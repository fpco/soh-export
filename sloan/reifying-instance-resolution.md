# Reifying Instance Resolution

A week or two ago, [Chris Done](https://github.com/chrisdone) and I
were talking about how it'd be excellent if GHC could tell you *why*
an instance exists.  If this feature was in GHCI, here's roughly what
it might look like:

```haskell

Prelude> :explain Ord [Maybe Int]
instance Ord a => Ord [a] -- Defined in 'GHC.Classes'
  with a ~ Maybe Int

  instance Ord a => Ord (Maybe a) -- Defined in 'Data.Maybe'
    with a ~ Int

    instance Ord Int -- Defined in 'GHC.Classes'
```

This traces GHC's reasoning when resolving instances for a particular
type.  The first line tells us that the `Ord a => Ord [a]` instance
will be used for `Ord [Maybe Int]`.  Nested under this is a `with`
clause, which shows how the type variables are instantiated.  In this
case, it tells us that `a` in that instance is instantiated to `Maybe Int`.

After the `with` clause comes the instances that are needed to satisfy
the instance context.  In this case, an instance for `Ord (Maybe Int)`
is needed due to the `Ord a` constraint in the instance for `Ord [a]`.

Anyway, Chris posed a challenge:

> hmm, i wonder if there's some uber general way to debug the way
> haskell's types are resolved by generating clever code

I pondered this for a bit, and concluded that this is indeed
possible!  Last weekend I had a crack at implementing it, and it
worked out well beyond my expectations!

Ord demo
========

```active haskell
{-# START_FILE Demo.hs #-}
-- Usually not all of these pragmas are necessary. But they are
-- needed for some invocations of 'explainInstance'.
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE PolyKinds #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE OverlappingInstances #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE DeriveDataTypeable #-}

import ExplainInstance
-- show
explainInstance [t| Ord [Maybe Int] |]
-- /show
{-# START_FILE ExplainInstance.hs #-}
{-# LANGUAGE TupleSections #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE ViewPatterns #-}

-- show
-- /show
module ExplainInstance where

import qualified Control.Monad.State as State
import qualified Data.Set as S


import           Control.Applicative ((<$>), (<*>))
import           Control.Monad (filterM)
import           Data.Function (on)
import           Data.Generics
import           Data.List (groupBy, sortBy, sort, group, find)
import qualified Data.Map as M
import           Data.Maybe
import           Data.Ord (comparing)
import           Language.Haskell.TH
import           Language.Haskell.TH.Syntax (addTopDecls)

-- TODO:
--
-- * Rename to "explain-instance"
--
-- * Show info about type families by using Typeable
--
-- * Use a parser so that type vars can be provided.  Or, have a
-- wildcard datatype.
--
-- * Apply inverse renaming on output types (relevant to constraints
-- that end up in the types, as well as data types with contexts)

-- Issues due to TH limitations:
--
-- * No ConstraintKinds
--
-- * No PolyKinds
--
-- * No info about whether an instance is overlapping / incoherent
--   (TODO: is this truly an issue?)
--
-- * No GHCI support. This is because 'explainInstances' can't yield
-- an Exp, as:
--
--   Only function, value, and foreign import declarations may be
--   added with addTopDecl
--
-- Which seems to be a rather arbitrary limitation...
--
-- TODO: Followup on these limitations on trac / mailinglists

explainInstance :: Q Type -> Q [Dec]
explainInstance = explainInstance' False

explainInstanceError :: Q Type -> Q [Dec]
explainInstanceError = explainInstance' True

explainInstance' :: Bool -> Q Type -> Q [Dec]
explainInstance' addErrorInstance qty = do
    ty <- qty
    case unAppsT ty of
        (ConT clazz : tys) -> do
            (decs, methodMap, renameMap) <- instanceResolvers addErrorInstance [clazz]
            let tys' = applySubstMap renameMap tys
            decs' <- [d| main = putStrLn (displayInst $(return (invokeResolve methodMap clazz tys'))) |]
            return (decs' ++ decs)
        _ -> fail "explainInstance input should be a constraint"

-- | An explanation of why some constraint is satisfied.
data Inst = Inst
    { -- | Like an instance declaration, but without the declarations.
      instHead :: String
      -- | Describes how type variables are instantiated in the head.
    , instTypes :: [(String, TypeRep)]
      -- | Explanations of how the instance's constraints are satisfied.
    , instConstraints :: [Inst]
    }
    deriving (Show)

-- | Provides an indented string presentation of the instance resolution tree.
displayInst :: Inst -> String
displayInst = go 0
  where
    go i (Inst decl vars cxt) =
        addIndent i (decl ++ displayVars vars) ++
        concatMap (("\n" ++) . go (i + 2)) cxt
    displayVars [] = ""
    displayVars (var0 : vars) =
        "\n  with " ++ displayVar var0 ++
        "\n" ++ unlines (map (("       " ++) . displayVar) vars)
    displayVar (n, ty) = n ++ " ~ " ++ showsPrec 9 ty ""

instanceResolvers :: Bool -> [Name] -> Q ([Dec], M.Map Name Name, M.Map Name Name)
instanceResolvers addErrorInstance initial = do
    infos <- reifyMany recurse initial
    methodMap <- M.fromList <$> sequence
        [ (n, ) <$> chooseUnusedName True ("resolve" ++ nameBase n)
        | (n, ClassI {}) <- infos
        ]
    let names = map fst infos ++ concatMap (map conName . infoCons . snd) infos
    renameMap <- M.fromList <$>
        mapM (\n -> (n, ) <$> chooseUnusedName False (nameBase n)) names
    decs <- mapM (resolver methodMap) (concatMap (infoToDecs . snd) infos)
    return (map (applySubst (flip M.lookup renameMap)) decs, methodMap, renameMap)
  where
    -- Recursively enumerate all of the top level declarations which
    -- need to be copied / renamed.
    recurse :: (Name, Info) -> Q (Bool, [Name])
    recurse (name, info) = return $ do
        case info of
            ClassI (ClassD cxt _name tvs _fds decs) insts ->
                (True, allNames (cxt, filter isFamilyDec decs) ++
                       concatMap tvKindNames tvs ++
                       concatMap instNames insts)
            TyConI (TySynD _name tvs ty) ->
                (True, allNames ty ++ concatMap tvKindNames tvs)
            -- Only need to clone data declarations when they have
            -- datatype contexts.
            TyConI (DataD cxt _name _tvs _cons _deriving) ->
                (not (null cxt), allNames cxt)
            TyConI (NewtypeD cxt _name _tvs _con _deriving) ->
                (not (null cxt), allNames cxt)
            -- We might encounter this due to DataKinds.
            DataConI _name _ty typeName _fixity ->
                (False, [typeName])
            -- FamilyI dec insts -> return (True, [])
            _ -> (False, [])
    instNames :: Dec -> [Name]
    instNames (InstanceD cxt ty decs) =
        allNames cxt ++ allNames ty ++ allNames (filter isFamilyDec decs)
    instNames _ = []
    infoToDecs :: Info -> [Dec]
    -- TODO: check fundeps?
    infoToDecs (ClassI dec@(ClassD _ name tvs _ _) insts) =
        case addErrorInstance && not hasDefaultCase of
            False -> dec : insts
            True ->
                let errInst = InstanceD
                        []
                        (appsT $ ConT name : map (VarT . tvName) tvs)
                        errorInstanceDecs
                in dec : errInst : insts
      where
        -- If true then an overlapping instance like (Class v1 ..),
        -- where all arguments are type variables, already exists.
        -- In this case omit the error instance.
        hasDefaultCase = isJust $ find isDefaultCase insts
        isDefaultCase (InstanceD _ (unAppsT -> (_:tys)) _) =
            all isTyVar tys
        isDefaultCase _ = False
        isTyVar (VarT _) = True
        isTyVar _ = False
    infoToDecs (ClassI _ _) = error "impossible: ClassI which doesn't contain ClassD"
    infoToDecs (TyConI dec) = [dec]
    infoToDecs (FamilyI dec insts) = dec : insts
    infoToDecs (VarI _name _ty mdec _fixity) = maybeToList mdec
    infoToDecs ClassOpI {} = []
    infoToDecs PrimTyConI {} = []
    infoToDecs DataConI {} = []
    infoToDecs TyVarI {} = []
    errorInstanceDecs = [FunD (mkName "x") []]
    -- Modify a class or instance to instead just have a single
    -- "resolver*" function.
    resolver :: M.Map Name Name -> Dec -> Q Dec
    resolver methodMap (ClassD cxt' name tvs fds decs) = do
        let method = lookupMethod methodMap name
            ty = funT $
                map (AppT (ConT ''Proxy) . VarT . tvName) tvs ++
                [ConT ''Inst]
        cxt <- mapM trimConstraint cxt'
        return $ ClassD cxt name tvs fds $
            filter isFamilyDec decs ++
            [SigD method ty]
    resolver methodMap (InstanceD cxt' ty' decs) = do
        cxt <- mapM trimConstraint cxt'
        ty <- trimInstanceType ty'
        let substs = varTSubsts (cxt, ty)
            cleanTyVars = applySubstMap (M.fromList substs)
        cleanedHead <- cleanTyCons $ cleanTyVars $ InstanceD cxt ty []
        let (ConT clazzName : tvs) = unAppsT ty
            method = lookupMethod methodMap clazzName
            msg = case addErrorInstance of
                True | decs == errorInstanceDecs -> "ERROR " ++ pprint cleanedHead
                _ -> pprint cleanedHead
            expr = appsE'
                [ ConE 'Inst
                , LitE $ StringL msg
                , ListE $ flip map substs $ \(ty, cty) -> TupE
                    [ LitE (StringL (pprint cty))
                    , AppE (VarE 'typeRep) (proxyE (VarT ty))
                    ]
                , ListE $ flip mapMaybe cxt $ \case
                    EqualP {} -> Nothing
                    ClassP n tys -> Just (invokeResolve methodMap n tys)
                ]
            -- Need extra typeable constraints in order to use typeRep.
            extraCxt = map (ClassP ''Typeable . (: []) . VarT . fst) substs
        return $ InstanceD (cxt ++ extraCxt) ty $
            filter isFamilyDec decs ++
            [FunD method [Clause (map (\_ -> WildP) tvs) (NormalB expr) []]]
    resolver _ dec = return dec

invokeResolve :: M.Map Name Name -> Name -> [Type] -> Exp
invokeResolve methodMap name tys =
    appsE' $ VarE (lookupMethod methodMap name) : map proxyE tys

lookupMethod :: M.Map Name Name -> Name -> Name
lookupMethod methodMap name =
    fromMaybe (error ("Couldn't find resolve* method name for " ++ show name))
              (M.lookup name methodMap)

allNames :: Data a => a -> [Name]
allNames = listify (\_ -> True)

tvKindNames :: TyVarBndr -> [Name]
tvKindNames (KindedTV _name kind) = allNames kind
tvKindNames PlainTV {} = []

tvName :: TyVarBndr -> Name
tvName (KindedTV name _kind) = name
tvName (PlainTV name) = name

isFamilyDec FamilyD {} = True
isFamilyDec DataInstD {} = True
isFamilyDec NewtypeInstD {} = True
isFamilyDec TySynInstD {} = True
isFamilyDec ClosedTypeFamilyD {} = True
isFamilyDec _ = False

-- Work around a TH bug where PolyKinded constraints get too many
-- arguments.
trimConstraint :: Pred -> Q Pred
trimConstraint (ClassP n tys) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ ClassP n (drop (length tys - length tvs) tys)
trimConstraint x = return x

trimInstanceType :: Type -> Q Type
trimInstanceType (unAppsT -> (ConT n : tys)) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ appsT (ConT n : (drop (length tys - length tvs) tys))

chooseUnusedName :: Bool -> String -> Q Name
chooseUnusedName allowUnmodified name = do
    -- This will always match, since there can only be finite names.
    Just str <- findM (\str -> not <$> exists str) choices
    return (mkName str)
  where
    choices = map (name ++) $
        (if allowUnmodified then ("" : ) else id) $
        "_" : map ('_':) (map show [1..])
    -- 'recover' is used to handle errors due to ambiguous identifier names.
    exists str = recover (return True) $ do
        mtype <- lookupTypeName str
        mvalue <- lookupValueName str
        return $ isJust mtype || isJust mvalue

findM :: Monad m => (a -> m Bool) -> [a] -> m (Maybe a)
findM f (x:xs) = do
    b <- f x
    if b
        then return (Just x)
        else findM f xs

applySubst :: (Data a, Typeable b) => (b -> Maybe b) -> a -> a
applySubst f = everywhere (id `extT` (\x -> fromMaybe x (f x)))

applySubstMap :: (Data a, Ord b, Typeable b) => M.Map b b -> a -> a
applySubstMap m = applySubst (flip M.lookup m)

funT :: [Type] -> Type
funT [x] = x
funT (x:xs) = AppT (AppT ArrowT x) (funT xs)

appsT :: [Type] -> Type
appsT = go . reverse
  where
    go [x] = x
    go (x:xs) = AppT (go xs) x

unAppsT :: Type -> [Type]
unAppsT ty = go ty []
  where
    go (AppT l r) = go l . (r :)
    go ty = (ty :)

appsE' :: [Exp] -> Exp
appsE' = go . reverse
  where
    go [x] = x
    go (x:xs) = AppE (go xs) x

proxyE :: Type -> Exp
proxyE = SigE (ConE 'Proxy) . AppT (ConT ''Proxy)

freeVarsT :: Type -> [Name]
freeVarsT (ForallT tvs _ ty) = filter (`notElem` (map tvName tvs)) (freeVarsT ty)
freeVarsT (AppT l r) = freeVarsT l ++ freeVarsT r
freeVarsT (SigT ty k) = freeVarsT ty ++ freeVarsT k
freeVarsT (VarT n) = [n]
freeVarsT _ = []

-- Dequalify names which are unambiguous.
cleanTyCons :: Data a => a -> Q a
cleanTyCons = everywhereM (return `extM` subst1 `extM` subst2)
  where
    rename :: Name -> Q Name
    rename n = do
        inScope <- typeNameInScope n
        return $ if inScope then mkName (nameBase n) else n
    subst1 (ConT n) = ConT <$> rename n
    subst1 x = return x
    subst2 (ClassP n tys) = ClassP <$> rename n <*> return tys
    subst2 x = return x

typeNameInScope :: Name -> Q Bool
typeNameInScope n =
    recover (return False)
            ((Just n ==) <$> lookupTypeName (nameBase n))

-- Chooses prettier names for type variables.  Assumes that all type
-- variable names are unique.
varTSubsts :: Data a => a -> [(Name, Name)]
varTSubsts =
    concatMap munge . groupSortOn nameBase . sortNub . varTNames
  where
    munge = zipWith (\s x -> (x, mkName (nameBase x ++ s))) ("" : map show [1..])

varTNames :: Data a => a -> [Name]
varTNames x = [n | VarT n <- listify (\_ -> True) x]

infoCons :: Info -> [Con]
infoCons (TyConI (DataD _ _ _ cons _)) = cons
infoCons (TyConI (NewtypeD _ _ _ con _)) = [con]
infoCons _ = []

conName :: Con -> Name
conName (NormalC name _) = name
conName (RecC name _) = name
conName (InfixC _ name _) = name
conName (ForallC _ _ con) = conName con

groupSortOn :: Ord b => (a -> b) -> [a] -> [[a]]
groupSortOn f = groupBy ((==) `on` f) . sortBy (comparing f)

sortNub :: Ord a => [a] -> [a]
sortNub = map head . group . sort

addIndent :: Int -> String -> String
addIndent cnt = unlines . map (replicate cnt ' ' ++) . lines

reifyMany :: ((Name, Info) -> Q (Bool, [Name]))
          -> [Name]
          -> Q [(Name, Info)]
reifyMany recurse initial =
    State.evalStateT (fmap concat $ mapM go initial) S.empty
  where
    go :: Name -> State.StateT (S.Set Name) Q [(Name, Info)]
    go n = do
        seen <- State.get
        if S.member n seen
            then return []
            else do
                State.put (S.insert n seen)
                minfo <- State.lift $ recover (return Nothing) (fmap Just (reify n))
                case minfo of
                    Just info -> do
                        (shouldEmit, ns) <- State.lift $ recurse (n, info)
                        (if shouldEmit
                             then fmap ((n, info):)
                             else id) $ fmap concat $ mapM go ns
                    _ -> return []

```

Neat, it works!

`Ord` instances are pretty easy to follow since they tend to just be
structurally recursive.  Let's try this out on something a little more
interesting!

Printf demo
===========

Initially, it can be a bit tricky to grok how typeclasses allow
Haskell to have polyvariadic functions.  Lets see how
`explainInstances` can clarify!

```active haskell
{-# START_FILE Demo.hs #-}
-- Usually not all of these pragmas are necessary. But they are
-- needed for some invocations of 'explainInstance'.
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE PolyKinds #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE OverlappingInstances #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE DeriveDataTypeable #-}

import ExplainInstance
-- show
import Text.Printf

explainInstance [t| PrintfType (Int -> Int -> String) |]
-- /show
{-# START_FILE ExplainInstance.hs #-}
{-# LANGUAGE TupleSections #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE ViewPatterns #-}

-- show
-- /show
module ExplainInstance where

import qualified Control.Monad.State as State
import qualified Data.Set as S


import           Control.Applicative ((<$>), (<*>))
import           Control.Monad (filterM)
import           Data.Function (on)
import           Data.Generics
import           Data.List (groupBy, sortBy, sort, group, find)
import qualified Data.Map as M
import           Data.Maybe
import           Data.Ord (comparing)
import           Language.Haskell.TH
import           Language.Haskell.TH.Syntax (addTopDecls)

-- TODO:
--
-- * Rename to "explain-instance"
--
-- * Show info about type families by using Typeable
--
-- * Use a parser so that type vars can be provided.  Or, have a
-- wildcard datatype.
--
-- * Apply inverse renaming on output types (relevant to constraints
-- that end up in the types, as well as data types with contexts)

-- Issues due to TH limitations:
--
-- * No ConstraintKinds
--
-- * No PolyKinds
--
-- * No info about whether an instance is overlapping / incoherent
--   (TODO: is this truly an issue?)
--
-- * No GHCI support. This is because 'explainInstances' can't yield
-- an Exp, as:
--
--   Only function, value, and foreign import declarations may be
--   added with addTopDecl
--
-- Which seems to be a rather arbitrary limitation...
--
-- TODO: Followup on these limitations on trac / mailinglists

explainInstance :: Q Type -> Q [Dec]
explainInstance = explainInstance' False

explainInstanceError :: Q Type -> Q [Dec]
explainInstanceError = explainInstance' True

explainInstance' :: Bool -> Q Type -> Q [Dec]
explainInstance' addErrorInstance qty = do
    ty <- qty
    case unAppsT ty of
        (ConT clazz : tys) -> do
            (decs, methodMap, renameMap) <- instanceResolvers addErrorInstance [clazz]
            let tys' = applySubstMap renameMap tys
            decs' <- [d| main = putStrLn (displayInst $(return (invokeResolve methodMap clazz tys'))) |]
            return (decs' ++ decs)
        _ -> fail "explainInstance input should be a constraint"

-- | An explanation of why some constraint is satisfied.
data Inst = Inst
    { -- | Like an instance declaration, but without the declarations.
      instHead :: String
      -- | Describes how type variables are instantiated in the head.
    , instTypes :: [(String, TypeRep)]
      -- | Explanations of how the instance's constraints are satisfied.
    , instConstraints :: [Inst]
    }
    deriving (Show)

-- | Provides an indented string presentation of the instance resolution tree.
displayInst :: Inst -> String
displayInst = go 0
  where
    go i (Inst decl vars cxt) =
        addIndent i (decl ++ displayVars vars) ++
        concatMap (("\n" ++) . go (i + 2)) cxt
    displayVars [] = ""
    displayVars (var0 : vars) =
        "\n  with " ++ displayVar var0 ++
        "\n" ++ unlines (map (("       " ++) . displayVar) vars)
    displayVar (n, ty) = n ++ " ~ " ++ showsPrec 9 ty ""

instanceResolvers :: Bool -> [Name] -> Q ([Dec], M.Map Name Name, M.Map Name Name)
instanceResolvers addErrorInstance initial = do
    infos <- reifyMany recurse initial
    methodMap <- M.fromList <$> sequence
        [ (n, ) <$> chooseUnusedName True ("resolve" ++ nameBase n)
        | (n, ClassI {}) <- infos
        ]
    let names = map fst infos ++ concatMap (map conName . infoCons . snd) infos
    renameMap <- M.fromList <$>
        mapM (\n -> (n, ) <$> chooseUnusedName False (nameBase n)) names
    decs <- mapM (resolver methodMap) (concatMap (infoToDecs . snd) infos)
    return (map (applySubst (flip M.lookup renameMap)) decs, methodMap, renameMap)
  where
    -- Recursively enumerate all of the top level declarations which
    -- need to be copied / renamed.
    recurse :: (Name, Info) -> Q (Bool, [Name])
    recurse (name, info) = return $ do
        case info of
            ClassI (ClassD cxt _name tvs _fds decs) insts ->
                (True, allNames (cxt, filter isFamilyDec decs) ++
                       concatMap tvKindNames tvs ++
                       concatMap instNames insts)
            TyConI (TySynD _name tvs ty) ->
                (True, allNames ty ++ concatMap tvKindNames tvs)
            -- Only need to clone data declarations when they have
            -- datatype contexts.
            TyConI (DataD cxt _name _tvs _cons _deriving) ->
                (not (null cxt), allNames cxt)
            TyConI (NewtypeD cxt _name _tvs _con _deriving) ->
                (not (null cxt), allNames cxt)
            -- We might encounter this due to DataKinds.
            DataConI _name _ty typeName _fixity ->
                (False, [typeName])
            -- FamilyI dec insts -> return (True, [])
            _ -> (False, [])
    instNames :: Dec -> [Name]
    instNames (InstanceD cxt ty decs) =
        allNames cxt ++ allNames ty ++ allNames (filter isFamilyDec decs)
    instNames _ = []
    infoToDecs :: Info -> [Dec]
    -- TODO: check fundeps?
    infoToDecs (ClassI dec@(ClassD _ name tvs _ _) insts) =
        case addErrorInstance && not hasDefaultCase of
            False -> dec : insts
            True ->
                let errInst = InstanceD
                        []
                        (appsT $ ConT name : map (VarT . tvName) tvs)
                        errorInstanceDecs
                in dec : errInst : insts
      where
        -- If true then an overlapping instance like (Class v1 ..),
        -- where all arguments are type variables, already exists.
        -- In this case omit the error instance.
        hasDefaultCase = isJust $ find isDefaultCase insts
        isDefaultCase (InstanceD _ (unAppsT -> (_:tys)) _) =
            all isTyVar tys
        isDefaultCase _ = False
        isTyVar (VarT _) = True
        isTyVar _ = False
    infoToDecs (ClassI _ _) = error "impossible: ClassI which doesn't contain ClassD"
    infoToDecs (TyConI dec) = [dec]
    infoToDecs (FamilyI dec insts) = dec : insts
    infoToDecs (VarI _name _ty mdec _fixity) = maybeToList mdec
    infoToDecs ClassOpI {} = []
    infoToDecs PrimTyConI {} = []
    infoToDecs DataConI {} = []
    infoToDecs TyVarI {} = []
    errorInstanceDecs = [FunD (mkName "x") []]
    -- Modify a class or instance to instead just have a single
    -- "resolver*" function.
    resolver :: M.Map Name Name -> Dec -> Q Dec
    resolver methodMap (ClassD cxt' name tvs fds decs) = do
        let method = lookupMethod methodMap name
            ty = funT $
                map (AppT (ConT ''Proxy) . VarT . tvName) tvs ++
                [ConT ''Inst]
        cxt <- mapM trimConstraint cxt'
        return $ ClassD cxt name tvs fds $
            filter isFamilyDec decs ++
            [SigD method ty]
    resolver methodMap (InstanceD cxt' ty' decs) = do
        cxt <- mapM trimConstraint cxt'
        ty <- trimInstanceType ty'
        let substs = varTSubsts (cxt, ty)
            cleanTyVars = applySubstMap (M.fromList substs)
        cleanedHead <- cleanTyCons $ cleanTyVars $ InstanceD cxt ty []
        let (ConT clazzName : tvs) = unAppsT ty
            method = lookupMethod methodMap clazzName
            msg = case addErrorInstance of
                True | decs == errorInstanceDecs -> "ERROR " ++ pprint cleanedHead
                _ -> pprint cleanedHead
            expr = appsE'
                [ ConE 'Inst
                , LitE $ StringL msg
                , ListE $ flip map substs $ \(ty, cty) -> TupE
                    [ LitE (StringL (pprint cty))
                    , AppE (VarE 'typeRep) (proxyE (VarT ty))
                    ]
                , ListE $ flip mapMaybe cxt $ \case
                    EqualP {} -> Nothing
                    ClassP n tys -> Just (invokeResolve methodMap n tys)
                ]
            -- Need extra typeable constraints in order to use typeRep.
            extraCxt = map (ClassP ''Typeable . (: []) . VarT . fst) substs
        return $ InstanceD (cxt ++ extraCxt) ty $
            filter isFamilyDec decs ++
            [FunD method [Clause (map (\_ -> WildP) tvs) (NormalB expr) []]]
    resolver _ dec = return dec

invokeResolve :: M.Map Name Name -> Name -> [Type] -> Exp
invokeResolve methodMap name tys =
    appsE' $ VarE (lookupMethod methodMap name) : map proxyE tys

lookupMethod :: M.Map Name Name -> Name -> Name
lookupMethod methodMap name =
    fromMaybe (error ("Couldn't find resolve* method name for " ++ show name))
              (M.lookup name methodMap)

allNames :: Data a => a -> [Name]
allNames = listify (\_ -> True)

tvKindNames :: TyVarBndr -> [Name]
tvKindNames (KindedTV _name kind) = allNames kind
tvKindNames PlainTV {} = []

tvName :: TyVarBndr -> Name
tvName (KindedTV name _kind) = name
tvName (PlainTV name) = name

isFamilyDec FamilyD {} = True
isFamilyDec DataInstD {} = True
isFamilyDec NewtypeInstD {} = True
isFamilyDec TySynInstD {} = True
isFamilyDec ClosedTypeFamilyD {} = True
isFamilyDec _ = False

-- Work around a TH bug where PolyKinded constraints get too many
-- arguments.
trimConstraint :: Pred -> Q Pred
trimConstraint (ClassP n tys) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ ClassP n (drop (length tys - length tvs) tys)
trimConstraint x = return x

trimInstanceType :: Type -> Q Type
trimInstanceType (unAppsT -> (ConT n : tys)) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ appsT (ConT n : (drop (length tys - length tvs) tys))

chooseUnusedName :: Bool -> String -> Q Name
chooseUnusedName allowUnmodified name = do
    -- This will always match, since there can only be finite names.
    Just str <- findM (\str -> not <$> exists str) choices
    return (mkName str)
  where
    choices = map (name ++) $
        (if allowUnmodified then ("" : ) else id) $
        "_" : map ('_':) (map show [1..])
    -- 'recover' is used to handle errors due to ambiguous identifier names.
    exists str = recover (return True) $ do
        mtype <- lookupTypeName str
        mvalue <- lookupValueName str
        return $ isJust mtype || isJust mvalue

findM :: Monad m => (a -> m Bool) -> [a] -> m (Maybe a)
findM f (x:xs) = do
    b <- f x
    if b
        then return (Just x)
        else findM f xs

applySubst :: (Data a, Typeable b) => (b -> Maybe b) -> a -> a
applySubst f = everywhere (id `extT` (\x -> fromMaybe x (f x)))

applySubstMap :: (Data a, Ord b, Typeable b) => M.Map b b -> a -> a
applySubstMap m = applySubst (flip M.lookup m)

funT :: [Type] -> Type
funT [x] = x
funT (x:xs) = AppT (AppT ArrowT x) (funT xs)

appsT :: [Type] -> Type
appsT = go . reverse
  where
    go [x] = x
    go (x:xs) = AppT (go xs) x

unAppsT :: Type -> [Type]
unAppsT ty = go ty []
  where
    go (AppT l r) = go l . (r :)
    go ty = (ty :)

appsE' :: [Exp] -> Exp
appsE' = go . reverse
  where
    go [x] = x
    go (x:xs) = AppE (go xs) x

proxyE :: Type -> Exp
proxyE = SigE (ConE 'Proxy) . AppT (ConT ''Proxy)

freeVarsT :: Type -> [Name]
freeVarsT (ForallT tvs _ ty) = filter (`notElem` (map tvName tvs)) (freeVarsT ty)
freeVarsT (AppT l r) = freeVarsT l ++ freeVarsT r
freeVarsT (SigT ty k) = freeVarsT ty ++ freeVarsT k
freeVarsT (VarT n) = [n]
freeVarsT _ = []

-- Dequalify names which are unambiguous.
cleanTyCons :: Data a => a -> Q a
cleanTyCons = everywhereM (return `extM` subst1 `extM` subst2)
  where
    rename :: Name -> Q Name
    rename n = do
        inScope <- typeNameInScope n
        return $ if inScope then mkName (nameBase n) else n
    subst1 (ConT n) = ConT <$> rename n
    subst1 x = return x
    subst2 (ClassP n tys) = ClassP <$> rename n <*> return tys
    subst2 x = return x

typeNameInScope :: Name -> Q Bool
typeNameInScope n =
    recover (return False)
            ((Just n ==) <$> lookupTypeName (nameBase n))

-- Chooses prettier names for type variables.  Assumes that all type
-- variable names are unique.
varTSubsts :: Data a => a -> [(Name, Name)]
varTSubsts =
    concatMap munge . groupSortOn nameBase . sortNub . varTNames
  where
    munge = zipWith (\s x -> (x, mkName (nameBase x ++ s))) ("" : map show [1..])

varTNames :: Data a => a -> [Name]
varTNames x = [n | VarT n <- listify (\_ -> True) x]

infoCons :: Info -> [Con]
infoCons (TyConI (DataD _ _ _ cons _)) = cons
infoCons (TyConI (NewtypeD _ _ _ con _)) = [con]
infoCons _ = []

conName :: Con -> Name
conName (NormalC name _) = name
conName (RecC name _) = name
conName (InfixC _ name _) = name
conName (ForallC _ _ con) = conName con

groupSortOn :: Ord b => (a -> b) -> [a] -> [[a]]
groupSortOn f = groupBy ((==) `on` f) . sortBy (comparing f)

sortNub :: Ord a => [a] -> [a]
sortNub = map head . group . sort

addIndent :: Int -> String -> String
addIndent cnt = unlines . map (replicate cnt ' ' ++) . lines

reifyMany :: ((Name, Info) -> Q (Bool, [Name]))
          -> [Name]
          -> Q [(Name, Info)]
reifyMany recurse initial =
    State.evalStateT (fmap concat $ mapM go initial) S.empty
  where
    go :: Name -> State.StateT (S.Set Name) Q [(Name, Info)]
    go n = do
        seen <- State.get
        if S.member n seen
            then return []
            else do
                State.put (S.insert n seen)
                minfo <- State.lift $ recover (return Nothing) (fmap Just (reify n))
                case minfo of
                    Just info -> do
                        (shouldEmit, ns) <- State.lift $ recurse (n, info)
                        (if shouldEmit
                             then fmap ((n, info):)
                             else id) $ fmap concat $ mapM go ns
                    _ -> return []

```

How it works
============

This leverages Template Haskell in a rather extreme way - by
duplicating and renaming every top level declaration that could
possibly have anything to do with resolving the instance.  It turns
out that TH's `reify` function is actually quite sufficient for
recursively enumerating type declarations - something I'd previously
realized with my
[th-reify-many](http://hackage.haskell.org/package/th-reify-many)
package.

For example, lets look at what happens to a made up typeclass and its
instances:

```active haskell
{-# START_FILE Demo.hs #-}
-- Usually not all of these pragmas are necessary. But they are
-- needed for some invocations of 'explainInstance'.
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE PolyKinds #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE OverlappingInstances #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE DeriveDataTypeable #-}

import ExplainInstance
-- show
class Size a where
    size :: a -> Int

instance Size Int where
    size = id

instance Size a => Size (Maybe a) where
    size = maybe 0 size

instance (Size a, Size b) => Size (a, b) where
    size (a, b) = size a + size b

$(explainInstance [t| Size (Maybe Int, Int) |])
-- /show
{-# START_FILE ExplainInstance.hs #-}
{-# LANGUAGE TupleSections #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE ViewPatterns #-}

-- show
-- /show
module ExplainInstance where

import qualified Control.Monad.State as State
import qualified Data.Set as S


import           Control.Applicative ((<$>), (<*>))
import           Control.Monad (filterM)
import           Data.Function (on)
import           Data.Generics
import           Data.List (groupBy, sortBy, sort, group, find)
import qualified Data.Map as M
import           Data.Maybe
import           Data.Ord (comparing)
import           Language.Haskell.TH
import           Language.Haskell.TH.Syntax (addTopDecls)

-- TODO:
--
-- * Rename to "explain-instance"
--
-- * Show info about type families by using Typeable
--
-- * Use a parser so that type vars can be provided.  Or, have a
-- wildcard datatype.
--
-- * Apply inverse renaming on output types (relevant to constraints
-- that end up in the types, as well as data types with contexts)

-- Issues due to TH limitations:
--
-- * No ConstraintKinds
--
-- * No PolyKinds
--
-- * No info about whether an instance is overlapping / incoherent
--   (TODO: is this truly an issue?)
--
-- * No GHCI support. This is because 'explainInstances' can't yield
-- an Exp, as:
--
--   Only function, value, and foreign import declarations may be
--   added with addTopDecl
--
-- Which seems to be a rather arbitrary limitation...
--
-- TODO: Followup on these limitations on trac / mailinglists

explainInstance :: Q Type -> Q [Dec]
explainInstance = explainInstance' False

explainInstanceError :: Q Type -> Q [Dec]
explainInstanceError = explainInstance' True

explainInstance' :: Bool -> Q Type -> Q [Dec]
explainInstance' addErrorInstance qty = do
    ty <- qty
    case unAppsT ty of
        (ConT clazz : tys) -> do
            (decs, methodMap, renameMap) <- instanceResolvers addErrorInstance [clazz]
            let tys' = applySubstMap renameMap tys
            decs' <- [d| main = putStrLn (displayInst $(return (invokeResolve methodMap clazz tys'))) |]
            return (decs' ++ decs)
        _ -> fail "explainInstance input should be a constraint"

-- | An explanation of why some constraint is satisfied.
data Inst = Inst
    { -- | Like an instance declaration, but without the declarations.
      instHead :: String
      -- | Describes how type variables are instantiated in the head.
    , instTypes :: [(String, TypeRep)]
      -- | Explanations of how the instance's constraints are satisfied.
    , instConstraints :: [Inst]
    }
    deriving (Show)

-- | Provides an indented string presentation of the instance resolution tree.
displayInst :: Inst -> String
displayInst = go 0
  where
    go i (Inst decl vars cxt) =
        addIndent i (decl ++ displayVars vars) ++
        concatMap (("\n" ++) . go (i + 2)) cxt
    displayVars [] = ""
    displayVars (var0 : vars) =
        "\n  with " ++ displayVar var0 ++
        "\n" ++ unlines (map (("       " ++) . displayVar) vars)
    displayVar (n, ty) = n ++ " ~ " ++ showsPrec 9 ty ""

instanceResolvers :: Bool -> [Name] -> Q ([Dec], M.Map Name Name, M.Map Name Name)
instanceResolvers addErrorInstance initial = do
    infos <- reifyMany recurse initial
    methodMap <- M.fromList <$> sequence
        [ (n, ) <$> chooseUnusedName True ("resolve" ++ nameBase n)
        | (n, ClassI {}) <- infos
        ]
    let names = map fst infos ++ concatMap (map conName . infoCons . snd) infos
    renameMap <- M.fromList <$>
        mapM (\n -> (n, ) <$> chooseUnusedName False (nameBase n)) names
    decs <- mapM (resolver methodMap) (concatMap (infoToDecs . snd) infos)
    return (map (applySubst (flip M.lookup renameMap)) decs, methodMap, renameMap)
  where
    -- Recursively enumerate all of the top level declarations which
    -- need to be copied / renamed.
    recurse :: (Name, Info) -> Q (Bool, [Name])
    recurse (name, info) = return $ do
        case info of
            ClassI (ClassD cxt _name tvs _fds decs) insts ->
                (True, allNames (cxt, filter isFamilyDec decs) ++
                       concatMap tvKindNames tvs ++
                       concatMap instNames insts)
            TyConI (TySynD _name tvs ty) ->
                (True, allNames ty ++ concatMap tvKindNames tvs)
            -- Only need to clone data declarations when they have
            -- datatype contexts.
            TyConI (DataD cxt _name _tvs _cons _deriving) ->
                (not (null cxt), allNames cxt)
            TyConI (NewtypeD cxt _name _tvs _con _deriving) ->
                (not (null cxt), allNames cxt)
            -- We might encounter this due to DataKinds.
            DataConI _name _ty typeName _fixity ->
                (False, [typeName])
            -- FamilyI dec insts -> return (True, [])
            _ -> (False, [])
    instNames :: Dec -> [Name]
    instNames (InstanceD cxt ty decs) =
        allNames cxt ++ allNames ty ++ allNames (filter isFamilyDec decs)
    instNames _ = []
    infoToDecs :: Info -> [Dec]
    -- TODO: check fundeps?
    infoToDecs (ClassI dec@(ClassD _ name tvs _ _) insts) =
        case addErrorInstance && not hasDefaultCase of
            False -> dec : insts
            True ->
                let errInst = InstanceD
                        []
                        (appsT $ ConT name : map (VarT . tvName) tvs)
                        errorInstanceDecs
                in dec : errInst : insts
      where
        -- If true then an overlapping instance like (Class v1 ..),
        -- where all arguments are type variables, already exists.
        -- In this case omit the error instance.
        hasDefaultCase = isJust $ find isDefaultCase insts
        isDefaultCase (InstanceD _ (unAppsT -> (_:tys)) _) =
            all isTyVar tys
        isDefaultCase _ = False
        isTyVar (VarT _) = True
        isTyVar _ = False
    infoToDecs (ClassI _ _) = error "impossible: ClassI which doesn't contain ClassD"
    infoToDecs (TyConI dec) = [dec]
    infoToDecs (FamilyI dec insts) = dec : insts
    infoToDecs (VarI _name _ty mdec _fixity) = maybeToList mdec
    infoToDecs ClassOpI {} = []
    infoToDecs PrimTyConI {} = []
    infoToDecs DataConI {} = []
    infoToDecs TyVarI {} = []
    errorInstanceDecs = [FunD (mkName "x") []]
    -- Modify a class or instance to instead just have a single
    -- "resolver*" function.
    resolver :: M.Map Name Name -> Dec -> Q Dec
    resolver methodMap (ClassD cxt' name tvs fds decs) = do
        let method = lookupMethod methodMap name
            ty = funT $
                map (AppT (ConT ''Proxy) . VarT . tvName) tvs ++
                [ConT ''Inst]
        cxt <- mapM trimConstraint cxt'
        return $ ClassD cxt name tvs fds $
            filter isFamilyDec decs ++
            [SigD method ty]
    resolver methodMap (InstanceD cxt' ty' decs) = do
        cxt <- mapM trimConstraint cxt'
        ty <- trimInstanceType ty'
        let substs = varTSubsts (cxt, ty)
            cleanTyVars = applySubstMap (M.fromList substs)
        cleanedHead <- cleanTyCons $ cleanTyVars $ InstanceD cxt ty []
        let (ConT clazzName : tvs) = unAppsT ty
            method = lookupMethod methodMap clazzName
            msg = case addErrorInstance of
                True | decs == errorInstanceDecs -> "ERROR " ++ pprint cleanedHead
                _ -> pprint cleanedHead
            expr = appsE'
                [ ConE 'Inst
                , LitE $ StringL msg
                , ListE $ flip map substs $ \(ty, cty) -> TupE
                    [ LitE (StringL (pprint cty))
                    , AppE (VarE 'typeRep) (proxyE (VarT ty))
                    ]
                , ListE $ flip mapMaybe cxt $ \case
                    EqualP {} -> Nothing
                    ClassP n tys -> Just (invokeResolve methodMap n tys)
                ]
            -- Need extra typeable constraints in order to use typeRep.
            extraCxt = map (ClassP ''Typeable . (: []) . VarT . fst) substs
        return $ InstanceD (cxt ++ extraCxt) ty $
            filter isFamilyDec decs ++
            [FunD method [Clause (map (\_ -> WildP) tvs) (NormalB expr) []]]
    resolver _ dec = return dec

invokeResolve :: M.Map Name Name -> Name -> [Type] -> Exp
invokeResolve methodMap name tys =
    appsE' $ VarE (lookupMethod methodMap name) : map proxyE tys

lookupMethod :: M.Map Name Name -> Name -> Name
lookupMethod methodMap name =
    fromMaybe (error ("Couldn't find resolve* method name for " ++ show name))
              (M.lookup name methodMap)

allNames :: Data a => a -> [Name]
allNames = listify (\_ -> True)

tvKindNames :: TyVarBndr -> [Name]
tvKindNames (KindedTV _name kind) = allNames kind
tvKindNames PlainTV {} = []

tvName :: TyVarBndr -> Name
tvName (KindedTV name _kind) = name
tvName (PlainTV name) = name

isFamilyDec FamilyD {} = True
isFamilyDec DataInstD {} = True
isFamilyDec NewtypeInstD {} = True
isFamilyDec TySynInstD {} = True
isFamilyDec ClosedTypeFamilyD {} = True
isFamilyDec _ = False

-- Work around a TH bug where PolyKinded constraints get too many
-- arguments.
trimConstraint :: Pred -> Q Pred
trimConstraint (ClassP n tys) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ ClassP n (drop (length tys - length tvs) tys)
trimConstraint x = return x

trimInstanceType :: Type -> Q Type
trimInstanceType (unAppsT -> (ConT n : tys)) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ appsT (ConT n : (drop (length tys - length tvs) tys))

chooseUnusedName :: Bool -> String -> Q Name
chooseUnusedName allowUnmodified name = do
    -- This will always match, since there can only be finite names.
    Just str <- findM (\str -> not <$> exists str) choices
    return (mkName str)
  where
    choices = map (name ++) $
        (if allowUnmodified then ("" : ) else id) $
        "_" : map ('_':) (map show [1..])
    -- 'recover' is used to handle errors due to ambiguous identifier names.
    exists str = recover (return True) $ do
        mtype <- lookupTypeName str
        mvalue <- lookupValueName str
        return $ isJust mtype || isJust mvalue

findM :: Monad m => (a -> m Bool) -> [a] -> m (Maybe a)
findM f (x:xs) = do
    b <- f x
    if b
        then return (Just x)
        else findM f xs

applySubst :: (Data a, Typeable b) => (b -> Maybe b) -> a -> a
applySubst f = everywhere (id `extT` (\x -> fromMaybe x (f x)))

applySubstMap :: (Data a, Ord b, Typeable b) => M.Map b b -> a -> a
applySubstMap m = applySubst (flip M.lookup m)

funT :: [Type] -> Type
funT [x] = x
funT (x:xs) = AppT (AppT ArrowT x) (funT xs)

appsT :: [Type] -> Type
appsT = go . reverse
  where
    go [x] = x
    go (x:xs) = AppT (go xs) x

unAppsT :: Type -> [Type]
unAppsT ty = go ty []
  where
    go (AppT l r) = go l . (r :)
    go ty = (ty :)

appsE' :: [Exp] -> Exp
appsE' = go . reverse
  where
    go [x] = x
    go (x:xs) = AppE (go xs) x

proxyE :: Type -> Exp
proxyE = SigE (ConE 'Proxy) . AppT (ConT ''Proxy)

freeVarsT :: Type -> [Name]
freeVarsT (ForallT tvs _ ty) = filter (`notElem` (map tvName tvs)) (freeVarsT ty)
freeVarsT (AppT l r) = freeVarsT l ++ freeVarsT r
freeVarsT (SigT ty k) = freeVarsT ty ++ freeVarsT k
freeVarsT (VarT n) = [n]
freeVarsT _ = []

-- Dequalify names which are unambiguous.
cleanTyCons :: Data a => a -> Q a
cleanTyCons = everywhereM (return `extM` subst1 `extM` subst2)
  where
    rename :: Name -> Q Name
    rename n = do
        inScope <- typeNameInScope n
        return $ if inScope then mkName (nameBase n) else n
    subst1 (ConT n) = ConT <$> rename n
    subst1 x = return x
    subst2 (ClassP n tys) = ClassP <$> rename n <*> return tys
    subst2 x = return x

typeNameInScope :: Name -> Q Bool
typeNameInScope n =
    recover (return False)
            ((Just n ==) <$> lookupTypeName (nameBase n))

-- Chooses prettier names for type variables.  Assumes that all type
-- variable names are unique.
varTSubsts :: Data a => a -> [(Name, Name)]
varTSubsts =
    concatMap munge . groupSortOn nameBase . sortNub . varTNames
  where
    munge = zipWith (\s x -> (x, mkName (nameBase x ++ s))) ("" : map show [1..])

varTNames :: Data a => a -> [Name]
varTNames x = [n | VarT n <- listify (\_ -> True) x]

infoCons :: Info -> [Con]
infoCons (TyConI (DataD _ _ _ cons _)) = cons
infoCons (TyConI (NewtypeD _ _ _ con _)) = [con]
infoCons _ = []

conName :: Con -> Name
conName (NormalC name _) = name
conName (RecC name _) = name
conName (InfixC _ name _) = name
conName (ForallC _ _ con) = conName con

groupSortOn :: Ord b => (a -> b) -> [a] -> [[a]]
groupSortOn f = groupBy ((==) `on` f) . sortBy (comparing f)

sortNub :: Ord a => [a] -> [a]
sortNub = map head . group . sort

addIndent :: Int -> String -> String
addIndent cnt = unlines . map (replicate cnt ' ' ++) . lines

reifyMany :: ((Name, Info) -> Q (Bool, [Name]))
          -> [Name]
          -> Q [(Name, Info)]
reifyMany recurse initial =
    State.evalStateT (fmap concat $ mapM go initial) S.empty
  where
    go :: Name -> State.StateT (S.Set Name) Q [(Name, Info)]
    go n = do
        seen <- State.get
        if S.member n seen
            then return []
            else do
                State.put (S.insert n seen)
                minfo <- State.lift $ recover (return Nothing) (fmap Just (reify n))
                case minfo of
                    Just info -> do
                        (shouldEmit, ns) <- State.lift $ recurse (n, info)
                        (if shouldEmit
                             then fmap ((n, info):)
                             else id) $ fmap concat $ mapM go ns
                    _ -> return []

```

Here are the declarations generated by the `explainInstance` TH
splice, with comments and syntax sugar added:

```haskell
-- Calling 'resolveSize' yields an instance resolution tree which
-- explains concrete instances of the 'Size' typeclass.  Its first
-- argument is a Proxy type, allowing the caller to specify 'a', even
-- if its kind is not *.
class Size_ a where
    resolveSize :: Proxy a -> Inst

-- Prints out the instance resolution tree for (Size (Maybe Int, Int)).
main = putStrLn $ displayInst $ resolveSize (Proxy :: Proxy (Maybe Int, Int))
```

The following are exported by ExplainInstance:


```haskell
-- | An explanation of why some constraint is satisfied.
data Inst = Inst
    { -- | Like an instance declaration, but without the method declarations.
      instHead :: String
      -- | Describes how type variables are instantiated in the head.
    , instTypes :: [(String, TypeRep)]
      -- | Explanations of how the instance's constraints are satisfied.
    , instConstraints :: [Inst]
    }
    deriving (Show)

-- | Provides an indented string presentation of the instance resolution tree.
displayInst :: Inst -> String
```

Ok, so that's how the main function / class works.  Here's how the
actual instances look:

```haskell
instance Size_ Int where
  -- The explanation for (Size Int) is simple - there are no type
  -- variables or constraints, so it just needs to provide the
  -- instance head.
  resolveSize _ = Inst "instance Size Int" [] []

-- Here, things get a little more complicated as there's a type
-- variable and a constraint.
instance (Size_ a, Typeable a) => Size_ (Maybe a) where
  resolveSize _
    = Inst
        "instance Size a => Size (Maybe a)"
        -- We use the added 'Typeable' constraint to create a runtime
        -- representation of the type that 'a' got instantiated to.
        -- This provides the "with" clause in the output.
        [("a", typeRep (Proxy :: Proxy a))]
        -- We use the 'Size_' constraint to recursively explain why
        -- (Size a) has an instance.
        [resolveSize (Proxy :: Proxy a)]

instance (Size_ a, Size_ b, Typeable a, Typeable b) =>
         Size_ (a, b) where
  resolveSize _
    = Inst
        "instance (Size a, Size b) => Size ((a, b))"
        [("a", typeRep (Proxy :: Proxy a)),
         ("b", typeRep (Proxy :: Proxy b))]
        [resolveSize (Proxy :: Proxy a),
         resolveSize (Proxy :: Proxy b)]
```

So, in summary, the trick here is to copy and rename typeclass
hierarchies, giving all of them a single `resolve*` method.  Calling
this method provides a value representing the instance resolution
tree.  By directly piggy backing GHC's instance resolution, we can be
sure that this tool never gives misinformation!

Explaining unsatisfied constraints
==================================

While the above is quite handy for explaining instances used in code
that compiles, it isn't so handy when you're faced with a compilation
error.

Turns out that with `OverlappingInstances`, it's possible to catch a
subset of such errors in a much more descriptive way than GHC.  Take,
for example, the type error in this code:

```active haskell
import Text.Printf

data Thing = Thing

main = printf "" Thing
```

This yields the error

```
No instance for (PrintfArg Thing) arising from a use of printf
In the expression: printf "" Thing
In an equation for ‘main’: main = printf "" Thing
```

While the error itself is pretty good, if you're new to printf, one
might wonder "where the heck did this (PrintfArg Thing) constraint
come from anyway?".  With `explainInstanceError`, we can answer this
question!

```active haskell
{-# START_FILE Demo.hs #-}
-- Usually not all of these pragmas are necessary. But they are
-- needed for some invocations of 'explainInstance'.
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE PolyKinds #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE OverlappingInstances #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE DeriveDataTypeable #-}

import ExplainInstance
-- show
import Data.Typeable
import Text.Printf

data Thing = Thing deriving (Typeable)

$(explainInstanceError [t| PrintfType (Thing -> IO ()) |])
-- /show
{-# START_FILE ExplainInstance.hs #-}
{-# LANGUAGE TupleSections #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE ViewPatterns #-}

-- show
-- /show
module ExplainInstance where

import qualified Control.Monad.State as State
import qualified Data.Set as S


import           Control.Applicative ((<$>), (<*>))
import           Control.Monad (filterM)
import           Data.Function (on)
import           Data.Generics
import           Data.List (groupBy, sortBy, sort, group, find)
import qualified Data.Map as M
import           Data.Maybe
import           Data.Ord (comparing)
import           Language.Haskell.TH
import           Language.Haskell.TH.Syntax (addTopDecls)

-- TODO:
--
-- * Rename to "explain-instance"
--
-- * Show info about type families by using Typeable
--
-- * Use a parser so that type vars can be provided.  Or, have a
-- wildcard datatype.
--
-- * Apply inverse renaming on output types (relevant to constraints
-- that end up in the types, as well as data types with contexts)

-- Issues due to TH limitations:
--
-- * No ConstraintKinds
--
-- * No PolyKinds
--
-- * No info about whether an instance is overlapping / incoherent
--   (TODO: is this truly an issue?)
--
-- * No GHCI support. This is because 'explainInstances' can't yield
-- an Exp, as:
--
--   Only function, value, and foreign import declarations may be
--   added with addTopDecl
--
-- Which seems to be a rather arbitrary limitation...
--
-- TODO: Followup on these limitations on trac / mailinglists

explainInstance :: Q Type -> Q [Dec]
explainInstance = explainInstance' False

explainInstanceError :: Q Type -> Q [Dec]
explainInstanceError = explainInstance' True

explainInstance' :: Bool -> Q Type -> Q [Dec]
explainInstance' addErrorInstance qty = do
    ty <- qty
    case unAppsT ty of
        (ConT clazz : tys) -> do
            (decs, methodMap, renameMap) <- instanceResolvers addErrorInstance [clazz]
            let tys' = applySubstMap renameMap tys
            decs' <- [d| main = putStrLn (displayInst $(return (invokeResolve methodMap clazz tys'))) |]
            return (decs' ++ decs)
        _ -> fail "explainInstance input should be a constraint"

-- | An explanation of why some constraint is satisfied.
data Inst = Inst
    { -- | Like an instance declaration, but without the declarations.
      instHead :: String
      -- | Describes how type variables are instantiated in the head.
    , instTypes :: [(String, TypeRep)]
      -- | Explanations of how the instance's constraints are satisfied.
    , instConstraints :: [Inst]
    }
    deriving (Show)

-- | Provides an indented string presentation of the instance resolution tree.
displayInst :: Inst -> String
displayInst = go 0
  where
    go i (Inst decl vars cxt) =
        addIndent i (decl ++ displayVars vars) ++
        concatMap (("\n" ++) . go (i + 2)) cxt
    displayVars [] = ""
    displayVars (var0 : vars) =
        "\n  with " ++ displayVar var0 ++
        "\n" ++ unlines (map (("       " ++) . displayVar) vars)
    displayVar (n, ty) = n ++ " ~ " ++ showsPrec 9 ty ""

instanceResolvers :: Bool -> [Name] -> Q ([Dec], M.Map Name Name, M.Map Name Name)
instanceResolvers addErrorInstance initial = do
    infos <- reifyMany recurse initial
    methodMap <- M.fromList <$> sequence
        [ (n, ) <$> chooseUnusedName True ("resolve" ++ nameBase n)
        | (n, ClassI {}) <- infos
        ]
    let names = map fst infos ++ concatMap (map conName . infoCons . snd) infos
    renameMap <- M.fromList <$>
        mapM (\n -> (n, ) <$> chooseUnusedName False (nameBase n)) names
    decs <- mapM (resolver methodMap) (concatMap (infoToDecs . snd) infos)
    return (map (applySubst (flip M.lookup renameMap)) decs, methodMap, renameMap)
  where
    -- Recursively enumerate all of the top level declarations which
    -- need to be copied / renamed.
    recurse :: (Name, Info) -> Q (Bool, [Name])
    recurse (name, info) = return $ do
        case info of
            ClassI (ClassD cxt _name tvs _fds decs) insts ->
                (True, allNames (cxt, filter isFamilyDec decs) ++
                       concatMap tvKindNames tvs ++
                       concatMap instNames insts)
            TyConI (TySynD _name tvs ty) ->
                (True, allNames ty ++ concatMap tvKindNames tvs)
            -- Only need to clone data declarations when they have
            -- datatype contexts.
            TyConI (DataD cxt _name _tvs _cons _deriving) ->
                (not (null cxt), allNames cxt)
            TyConI (NewtypeD cxt _name _tvs _con _deriving) ->
                (not (null cxt), allNames cxt)
            -- We might encounter this due to DataKinds.
            DataConI _name _ty typeName _fixity ->
                (False, [typeName])
            -- FamilyI dec insts -> return (True, [])
            _ -> (False, [])
    instNames :: Dec -> [Name]
    instNames (InstanceD cxt ty decs) =
        allNames cxt ++ allNames ty ++ allNames (filter isFamilyDec decs)
    instNames _ = []
    infoToDecs :: Info -> [Dec]
    -- TODO: check fundeps?
    infoToDecs (ClassI dec@(ClassD _ name tvs _ _) insts) =
        case addErrorInstance && not hasDefaultCase of
            False -> dec : insts
            True ->
                let errInst = InstanceD
                        []
                        (appsT $ ConT name : map (VarT . tvName) tvs)
                        errorInstanceDecs
                in dec : errInst : insts
      where
        -- If true then an overlapping instance like (Class v1 ..),
        -- where all arguments are type variables, already exists.
        -- In this case omit the error instance.
        hasDefaultCase = isJust $ find isDefaultCase insts
        isDefaultCase (InstanceD _ (unAppsT -> (_:tys)) _) =
            all isTyVar tys
        isDefaultCase _ = False
        isTyVar (VarT _) = True
        isTyVar _ = False
    infoToDecs (ClassI _ _) = error "impossible: ClassI which doesn't contain ClassD"
    infoToDecs (TyConI dec) = [dec]
    infoToDecs (FamilyI dec insts) = dec : insts
    infoToDecs (VarI _name _ty mdec _fixity) = maybeToList mdec
    infoToDecs ClassOpI {} = []
    infoToDecs PrimTyConI {} = []
    infoToDecs DataConI {} = []
    infoToDecs TyVarI {} = []
    errorInstanceDecs = [FunD (mkName "x") []]
    -- Modify a class or instance to instead just have a single
    -- "resolver*" function.
    resolver :: M.Map Name Name -> Dec -> Q Dec
    resolver methodMap (ClassD cxt' name tvs fds decs) = do
        let method = lookupMethod methodMap name
            ty = funT $
                map (AppT (ConT ''Proxy) . VarT . tvName) tvs ++
                [ConT ''Inst]
        cxt <- mapM trimConstraint cxt'
        return $ ClassD cxt name tvs fds $
            filter isFamilyDec decs ++
            [SigD method ty]
    resolver methodMap (InstanceD cxt' ty' decs) = do
        cxt <- mapM trimConstraint cxt'
        ty <- trimInstanceType ty'
        let substs = varTSubsts (cxt, ty)
            cleanTyVars = applySubstMap (M.fromList substs)
        cleanedHead <- cleanTyCons $ cleanTyVars $ InstanceD cxt ty []
        let (ConT clazzName : tvs) = unAppsT ty
            method = lookupMethod methodMap clazzName
            msg = case addErrorInstance of
                True | decs == errorInstanceDecs -> "ERROR " ++ pprint cleanedHead
                _ -> pprint cleanedHead
            expr = appsE'
                [ ConE 'Inst
                , LitE $ StringL msg
                , ListE $ flip map substs $ \(ty, cty) -> TupE
                    [ LitE (StringL (pprint cty))
                    , AppE (VarE 'typeRep) (proxyE (VarT ty))
                    ]
                , ListE $ flip mapMaybe cxt $ \case
                    EqualP {} -> Nothing
                    ClassP n tys -> Just (invokeResolve methodMap n tys)
                ]
            -- Need extra typeable constraints in order to use typeRep.
            extraCxt = map (ClassP ''Typeable . (: []) . VarT . fst) substs
        return $ InstanceD (cxt ++ extraCxt) ty $
            filter isFamilyDec decs ++
            [FunD method [Clause (map (\_ -> WildP) tvs) (NormalB expr) []]]
    resolver _ dec = return dec

invokeResolve :: M.Map Name Name -> Name -> [Type] -> Exp
invokeResolve methodMap name tys =
    appsE' $ VarE (lookupMethod methodMap name) : map proxyE tys

lookupMethod :: M.Map Name Name -> Name -> Name
lookupMethod methodMap name =
    fromMaybe (error ("Couldn't find resolve* method name for " ++ show name))
              (M.lookup name methodMap)

allNames :: Data a => a -> [Name]
allNames = listify (\_ -> True)

tvKindNames :: TyVarBndr -> [Name]
tvKindNames (KindedTV _name kind) = allNames kind
tvKindNames PlainTV {} = []

tvName :: TyVarBndr -> Name
tvName (KindedTV name _kind) = name
tvName (PlainTV name) = name

isFamilyDec FamilyD {} = True
isFamilyDec DataInstD {} = True
isFamilyDec NewtypeInstD {} = True
isFamilyDec TySynInstD {} = True
isFamilyDec ClosedTypeFamilyD {} = True
isFamilyDec _ = False

-- Work around a TH bug where PolyKinded constraints get too many
-- arguments.
trimConstraint :: Pred -> Q Pred
trimConstraint (ClassP n tys) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ ClassP n (drop (length tys - length tvs) tys)
trimConstraint x = return x

trimInstanceType :: Type -> Q Type
trimInstanceType (unAppsT -> (ConT n : tys)) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ appsT (ConT n : (drop (length tys - length tvs) tys))

chooseUnusedName :: Bool -> String -> Q Name
chooseUnusedName allowUnmodified name = do
    -- This will always match, since there can only be finite names.
    Just str <- findM (\str -> not <$> exists str) choices
    return (mkName str)
  where
    choices = map (name ++) $
        (if allowUnmodified then ("" : ) else id) $
        "_" : map ('_':) (map show [1..])
    -- 'recover' is used to handle errors due to ambiguous identifier names.
    exists str = recover (return True) $ do
        mtype <- lookupTypeName str
        mvalue <- lookupValueName str
        return $ isJust mtype || isJust mvalue

findM :: Monad m => (a -> m Bool) -> [a] -> m (Maybe a)
findM f (x:xs) = do
    b <- f x
    if b
        then return (Just x)
        else findM f xs

applySubst :: (Data a, Typeable b) => (b -> Maybe b) -> a -> a
applySubst f = everywhere (id `extT` (\x -> fromMaybe x (f x)))

applySubstMap :: (Data a, Ord b, Typeable b) => M.Map b b -> a -> a
applySubstMap m = applySubst (flip M.lookup m)

funT :: [Type] -> Type
funT [x] = x
funT (x:xs) = AppT (AppT ArrowT x) (funT xs)

appsT :: [Type] -> Type
appsT = go . reverse
  where
    go [x] = x
    go (x:xs) = AppT (go xs) x

unAppsT :: Type -> [Type]
unAppsT ty = go ty []
  where
    go (AppT l r) = go l . (r :)
    go ty = (ty :)

appsE' :: [Exp] -> Exp
appsE' = go . reverse
  where
    go [x] = x
    go (x:xs) = AppE (go xs) x

proxyE :: Type -> Exp
proxyE = SigE (ConE 'Proxy) . AppT (ConT ''Proxy)

freeVarsT :: Type -> [Name]
freeVarsT (ForallT tvs _ ty) = filter (`notElem` (map tvName tvs)) (freeVarsT ty)
freeVarsT (AppT l r) = freeVarsT l ++ freeVarsT r
freeVarsT (SigT ty k) = freeVarsT ty ++ freeVarsT k
freeVarsT (VarT n) = [n]
freeVarsT _ = []

-- Dequalify names which are unambiguous.
cleanTyCons :: Data a => a -> Q a
cleanTyCons = everywhereM (return `extM` subst1 `extM` subst2)
  where
    rename :: Name -> Q Name
    rename n = do
        inScope <- typeNameInScope n
        return $ if inScope then mkName (nameBase n) else n
    subst1 (ConT n) = ConT <$> rename n
    subst1 x = return x
    subst2 (ClassP n tys) = ClassP <$> rename n <*> return tys
    subst2 x = return x

typeNameInScope :: Name -> Q Bool
typeNameInScope n =
    recover (return False)
            ((Just n ==) <$> lookupTypeName (nameBase n))

-- Chooses prettier names for type variables.  Assumes that all type
-- variable names are unique.
varTSubsts :: Data a => a -> [(Name, Name)]
varTSubsts =
    concatMap munge . groupSortOn nameBase . sortNub . varTNames
  where
    munge = zipWith (\s x -> (x, mkName (nameBase x ++ s))) ("" : map show [1..])

varTNames :: Data a => a -> [Name]
varTNames x = [n | VarT n <- listify (\_ -> True) x]

infoCons :: Info -> [Con]
infoCons (TyConI (DataD _ _ _ cons _)) = cons
infoCons (TyConI (NewtypeD _ _ _ con _)) = [con]
infoCons _ = []

conName :: Con -> Name
conName (NormalC name _) = name
conName (RecC name _) = name
conName (InfixC _ name _) = name
conName (ForallC _ _ con) = conName con

groupSortOn :: Ord b => (a -> b) -> [a] -> [[a]]
groupSortOn f = groupBy ((==) `on` f) . sortBy (comparing f)

sortNub :: Ord a => [a] -> [a]
sortNub = map head . group . sort

addIndent :: Int -> String -> String
addIndent cnt = unlines . map (replicate cnt ' ' ++) . lines

reifyMany :: ((Name, Info) -> Q (Bool, [Name]))
          -> [Name]
          -> Q [(Name, Info)]
reifyMany recurse initial =
    State.evalStateT (fmap concat $ mapM go initial) S.empty
  where
    go :: Name -> State.StateT (S.Set Name) Q [(Name, Info)]
    go n = do
        seen <- State.get
        if S.member n seen
            then return []
            else do
                State.put (S.insert n seen)
                minfo <- State.lift $ recover (return Nothing) (fmap Just (reify n))
                case minfo of
                    Just info -> do
                        (shouldEmit, ns) <- State.lift $ recurse (n, info)
                        (if shouldEmit
                             then fmap ((n, info):)
                             else id) $ fmap concat $ mapM go ns
                    _ -> return []

```

This gives us a clear picture of why the `PrintfArg` came to exist.
This works by adding additional typeclass instances like:

```haskell
instance Typeable t => PrintfType_ t where
  resolvePrintfType _ = Inst "ERROR instance PrintfType t"
                             [("t", typeRep (Proxy :: Proxy t))]
                             []
```

One cool thing about this is that it can tell you about multiple
failing constraints at once:


```active haskell
{-# START_FILE Demo.hs #-}
-- Usually not all of these pragmas are necessary. But they are
-- needed for some invocations of 'explainInstance'.
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE PolyKinds #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE OverlappingInstances #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE DeriveDataTypeable #-}

import ExplainInstance
-- show
import Data.Typeable
import Text.Printf

data Thing = Thing deriving (Typeable)

$(explainInstanceError [t| PrintfType (Thing -> Int -> Maybe (IO ())) |])
-- /show
{-# START_FILE ExplainInstance.hs #-}
{-# LANGUAGE TupleSections #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE LambdaCase #-}
{-# LANGUAGE ViewPatterns #-}

-- show
-- /show
module ExplainInstance where

import qualified Control.Monad.State as State
import qualified Data.Set as S


import           Control.Applicative ((<$>), (<*>))
import           Control.Monad (filterM)
import           Data.Function (on)
import           Data.Generics
import           Data.List (groupBy, sortBy, sort, group, find)
import qualified Data.Map as M
import           Data.Maybe
import           Data.Ord (comparing)
import           Language.Haskell.TH
import           Language.Haskell.TH.Syntax (addTopDecls)

-- TODO:
--
-- * Rename to "explain-instance"
--
-- * Show info about type families by using Typeable
--
-- * Use a parser so that type vars can be provided.  Or, have a
-- wildcard datatype.
--
-- * Apply inverse renaming on output types (relevant to constraints
-- that end up in the types, as well as data types with contexts)

-- Issues due to TH limitations:
--
-- * No ConstraintKinds
--
-- * No PolyKinds
--
-- * No info about whether an instance is overlapping / incoherent
--   (TODO: is this truly an issue?)
--
-- * No GHCI support. This is because 'explainInstances' can't yield
-- an Exp, as:
--
--   Only function, value, and foreign import declarations may be
--   added with addTopDecl
--
-- Which seems to be a rather arbitrary limitation...
--
-- TODO: Followup on these limitations on trac / mailinglists

explainInstance :: Q Type -> Q [Dec]
explainInstance = explainInstance' False

explainInstanceError :: Q Type -> Q [Dec]
explainInstanceError = explainInstance' True

explainInstance' :: Bool -> Q Type -> Q [Dec]
explainInstance' addErrorInstance qty = do
    ty <- qty
    case unAppsT ty of
        (ConT clazz : tys) -> do
            (decs, methodMap, renameMap) <- instanceResolvers addErrorInstance [clazz]
            let tys' = applySubstMap renameMap tys
            decs' <- [d| main = putStrLn (displayInst $(return (invokeResolve methodMap clazz tys'))) |]
            return (decs' ++ decs)
        _ -> fail "explainInstance input should be a constraint"

-- | An explanation of why some constraint is satisfied.
data Inst = Inst
    { -- | Like an instance declaration, but without the declarations.
      instHead :: String
      -- | Describes how type variables are instantiated in the head.
    , instTypes :: [(String, TypeRep)]
      -- | Explanations of how the instance's constraints are satisfied.
    , instConstraints :: [Inst]
    }
    deriving (Show)

-- | Provides an indented string presentation of the instance resolution tree.
displayInst :: Inst -> String
displayInst = go 0
  where
    go i (Inst decl vars cxt) =
        addIndent i (decl ++ displayVars vars) ++
        concatMap (("\n" ++) . go (i + 2)) cxt
    displayVars [] = ""
    displayVars (var0 : vars) =
        "\n  with " ++ displayVar var0 ++
        "\n" ++ unlines (map (("       " ++) . displayVar) vars)
    displayVar (n, ty) = n ++ " ~ " ++ showsPrec 9 ty ""

instanceResolvers :: Bool -> [Name] -> Q ([Dec], M.Map Name Name, M.Map Name Name)
instanceResolvers addErrorInstance initial = do
    infos <- reifyMany recurse initial
    methodMap <- M.fromList <$> sequence
        [ (n, ) <$> chooseUnusedName True ("resolve" ++ nameBase n)
        | (n, ClassI {}) <- infos
        ]
    let names = map fst infos ++ concatMap (map conName . infoCons . snd) infos
    renameMap <- M.fromList <$>
        mapM (\n -> (n, ) <$> chooseUnusedName False (nameBase n)) names
    decs <- mapM (resolver methodMap) (concatMap (infoToDecs . snd) infos)
    return (map (applySubst (flip M.lookup renameMap)) decs, methodMap, renameMap)
  where
    -- Recursively enumerate all of the top level declarations which
    -- need to be copied / renamed.
    recurse :: (Name, Info) -> Q (Bool, [Name])
    recurse (name, info) = return $ do
        case info of
            ClassI (ClassD cxt _name tvs _fds decs) insts ->
                (True, allNames (cxt, filter isFamilyDec decs) ++
                       concatMap tvKindNames tvs ++
                       concatMap instNames insts)
            TyConI (TySynD _name tvs ty) ->
                (True, allNames ty ++ concatMap tvKindNames tvs)
            -- Only need to clone data declarations when they have
            -- datatype contexts.
            TyConI (DataD cxt _name _tvs _cons _deriving) ->
                (not (null cxt), allNames cxt)
            TyConI (NewtypeD cxt _name _tvs _con _deriving) ->
                (not (null cxt), allNames cxt)
            -- We might encounter this due to DataKinds.
            DataConI _name _ty typeName _fixity ->
                (False, [typeName])
            -- FamilyI dec insts -> return (True, [])
            _ -> (False, [])
    instNames :: Dec -> [Name]
    instNames (InstanceD cxt ty decs) =
        allNames cxt ++ allNames ty ++ allNames (filter isFamilyDec decs)
    instNames _ = []
    infoToDecs :: Info -> [Dec]
    -- TODO: check fundeps?
    infoToDecs (ClassI dec@(ClassD _ name tvs _ _) insts) =
        case addErrorInstance && not hasDefaultCase of
            False -> dec : insts
            True ->
                let errInst = InstanceD
                        []
                        (appsT $ ConT name : map (VarT . tvName) tvs)
                        errorInstanceDecs
                in dec : errInst : insts
      where
        -- If true then an overlapping instance like (Class v1 ..),
        -- where all arguments are type variables, already exists.
        -- In this case omit the error instance.
        hasDefaultCase = isJust $ find isDefaultCase insts
        isDefaultCase (InstanceD _ (unAppsT -> (_:tys)) _) =
            all isTyVar tys
        isDefaultCase _ = False
        isTyVar (VarT _) = True
        isTyVar _ = False
    infoToDecs (ClassI _ _) = error "impossible: ClassI which doesn't contain ClassD"
    infoToDecs (TyConI dec) = [dec]
    infoToDecs (FamilyI dec insts) = dec : insts
    infoToDecs (VarI _name _ty mdec _fixity) = maybeToList mdec
    infoToDecs ClassOpI {} = []
    infoToDecs PrimTyConI {} = []
    infoToDecs DataConI {} = []
    infoToDecs TyVarI {} = []
    errorInstanceDecs = [FunD (mkName "x") []]
    -- Modify a class or instance to instead just have a single
    -- "resolver*" function.
    resolver :: M.Map Name Name -> Dec -> Q Dec
    resolver methodMap (ClassD cxt' name tvs fds decs) = do
        let method = lookupMethod methodMap name
            ty = funT $
                map (AppT (ConT ''Proxy) . VarT . tvName) tvs ++
                [ConT ''Inst]
        cxt <- mapM trimConstraint cxt'
        return $ ClassD cxt name tvs fds $
            filter isFamilyDec decs ++
            [SigD method ty]
    resolver methodMap (InstanceD cxt' ty' decs) = do
        cxt <- mapM trimConstraint cxt'
        ty <- trimInstanceType ty'
        let substs = varTSubsts (cxt, ty)
            cleanTyVars = applySubstMap (M.fromList substs)
        cleanedHead <- cleanTyCons $ cleanTyVars $ InstanceD cxt ty []
        let (ConT clazzName : tvs) = unAppsT ty
            method = lookupMethod methodMap clazzName
            msg = case addErrorInstance of
                True | decs == errorInstanceDecs -> "ERROR " ++ pprint cleanedHead
                _ -> pprint cleanedHead
            expr = appsE'
                [ ConE 'Inst
                , LitE $ StringL msg
                , ListE $ flip map substs $ \(ty, cty) -> TupE
                    [ LitE (StringL (pprint cty))
                    , AppE (VarE 'typeRep) (proxyE (VarT ty))
                    ]
                , ListE $ flip mapMaybe cxt $ \case
                    EqualP {} -> Nothing
                    ClassP n tys -> Just (invokeResolve methodMap n tys)
                ]
            -- Need extra typeable constraints in order to use typeRep.
            extraCxt = map (ClassP ''Typeable . (: []) . VarT . fst) substs
        return $ InstanceD (cxt ++ extraCxt) ty $
            filter isFamilyDec decs ++
            [FunD method [Clause (map (\_ -> WildP) tvs) (NormalB expr) []]]
    resolver _ dec = return dec

invokeResolve :: M.Map Name Name -> Name -> [Type] -> Exp
invokeResolve methodMap name tys =
    appsE' $ VarE (lookupMethod methodMap name) : map proxyE tys

lookupMethod :: M.Map Name Name -> Name -> Name
lookupMethod methodMap name =
    fromMaybe (error ("Couldn't find resolve* method name for " ++ show name))
              (M.lookup name methodMap)

allNames :: Data a => a -> [Name]
allNames = listify (\_ -> True)

tvKindNames :: TyVarBndr -> [Name]
tvKindNames (KindedTV _name kind) = allNames kind
tvKindNames PlainTV {} = []

tvName :: TyVarBndr -> Name
tvName (KindedTV name _kind) = name
tvName (PlainTV name) = name

isFamilyDec FamilyD {} = True
isFamilyDec DataInstD {} = True
isFamilyDec NewtypeInstD {} = True
isFamilyDec TySynInstD {} = True
isFamilyDec ClosedTypeFamilyD {} = True
isFamilyDec _ = False

-- Work around a TH bug where PolyKinded constraints get too many
-- arguments.
trimConstraint :: Pred -> Q Pred
trimConstraint (ClassP n tys) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ ClassP n (drop (length tys - length tvs) tys)
trimConstraint x = return x

trimInstanceType :: Type -> Q Type
trimInstanceType (unAppsT -> (ConT n : tys)) = do
    ClassI (ClassD _ _ tvs _ _) _ <- reify n
    return $ appsT (ConT n : (drop (length tys - length tvs) tys))

chooseUnusedName :: Bool -> String -> Q Name
chooseUnusedName allowUnmodified name = do
    -- This will always match, since there can only be finite names.
    Just str <- findM (\str -> not <$> exists str) choices
    return (mkName str)
  where
    choices = map (name ++) $
        (if allowUnmodified then ("" : ) else id) $
        "_" : map ('_':) (map show [1..])
    -- 'recover' is used to handle errors due to ambiguous identifier names.
    exists str = recover (return True) $ do
        mtype <- lookupTypeName str
        mvalue <- lookupValueName str
        return $ isJust mtype || isJust mvalue

findM :: Monad m => (a -> m Bool) -> [a] -> m (Maybe a)
findM f (x:xs) = do
    b <- f x
    if b
        then return (Just x)
        else findM f xs

applySubst :: (Data a, Typeable b) => (b -> Maybe b) -> a -> a
applySubst f = everywhere (id `extT` (\x -> fromMaybe x (f x)))

applySubstMap :: (Data a, Ord b, Typeable b) => M.Map b b -> a -> a
applySubstMap m = applySubst (flip M.lookup m)

funT :: [Type] -> Type
funT [x] = x
funT (x:xs) = AppT (AppT ArrowT x) (funT xs)

appsT :: [Type] -> Type
appsT = go . reverse
  where
    go [x] = x
    go (x:xs) = AppT (go xs) x

unAppsT :: Type -> [Type]
unAppsT ty = go ty []
  where
    go (AppT l r) = go l . (r :)
    go ty = (ty :)

appsE' :: [Exp] -> Exp
appsE' = go . reverse
  where
    go [x] = x
    go (x:xs) = AppE (go xs) x

proxyE :: Type -> Exp
proxyE = SigE (ConE 'Proxy) . AppT (ConT ''Proxy)

freeVarsT :: Type -> [Name]
freeVarsT (ForallT tvs _ ty) = filter (`notElem` (map tvName tvs)) (freeVarsT ty)
freeVarsT (AppT l r) = freeVarsT l ++ freeVarsT r
freeVarsT (SigT ty k) = freeVarsT ty ++ freeVarsT k
freeVarsT (VarT n) = [n]
freeVarsT _ = []

-- Dequalify names which are unambiguous.
cleanTyCons :: Data a => a -> Q a
cleanTyCons = everywhereM (return `extM` subst1 `extM` subst2)
  where
    rename :: Name -> Q Name
    rename n = do
        inScope <- typeNameInScope n
        return $ if inScope then mkName (nameBase n) else n
    subst1 (ConT n) = ConT <$> rename n
    subst1 x = return x
    subst2 (ClassP n tys) = ClassP <$> rename n <*> return tys
    subst2 x = return x

typeNameInScope :: Name -> Q Bool
typeNameInScope n =
    recover (return False)
            ((Just n ==) <$> lookupTypeName (nameBase n))

-- Chooses prettier names for type variables.  Assumes that all type
-- variable names are unique.
varTSubsts :: Data a => a -> [(Name, Name)]
varTSubsts =
    concatMap munge . groupSortOn nameBase . sortNub . varTNames
  where
    munge = zipWith (\s x -> (x, mkName (nameBase x ++ s))) ("" : map show [1..])

varTNames :: Data a => a -> [Name]
varTNames x = [n | VarT n <- listify (\_ -> True) x]

infoCons :: Info -> [Con]
infoCons (TyConI (DataD _ _ _ cons _)) = cons
infoCons (TyConI (NewtypeD _ _ _ con _)) = [con]
infoCons _ = []

conName :: Con -> Name
conName (NormalC name _) = name
conName (RecC name _) = name
conName (InfixC _ name _) = name
conName (ForallC _ _ con) = conName con

groupSortOn :: Ord b => (a -> b) -> [a] -> [[a]]
groupSortOn f = groupBy ((==) `on` f) . sortBy (comparing f)

sortNub :: Ord a => [a] -> [a]
sortNub = map head . group . sort

addIndent :: Int -> String -> String
addIndent cnt = unlines . map (replicate cnt ' ' ++) . lines

reifyMany :: ((Name, Info) -> Q (Bool, [Name]))
          -> [Name]
          -> Q [(Name, Info)]
reifyMany recurse initial =
    State.evalStateT (fmap concat $ mapM go initial) S.empty
  where
    go :: Name -> State.StateT (S.Set Name) Q [(Name, Info)]
    go n = do
        seen <- State.get
        if S.member n seen
            then return []
            else do
                State.put (S.insert n seen)
                minfo <- State.lift $ recover (return Nothing) (fmap Just (reify n))
                case minfo of
                    Just info -> do
                        (shouldEmit, ns) <- State.lift $ recurse (n, info)
                        (if shouldEmit
                             then fmap ((n, info):)
                             else id) $ fmap concat $ mapM go ns
                    _ -> return []

```

This project is quite new, and this is the newest part, so I imagine
there's still some problems with it to be worked out.  One problem I
know of is that it doesn't work for equality constraints.  For
example:

```active haskell
import Text.Printf

main = putStrLn "hi"
  where
    foo :: Int -> IO String
    foo = printf "%i"
```

and we get:

```
Couldn't match type [Char] with ()
In the expression: printf "%i"
```

Where did that come from?? Why is `()` correct and `String` not?  The
answer is that it comes from an equality constraint:

```
instance (a ~ ()) => PrintfType (IO a)
```

I haven't really given this much thought yet, but a general solution
to this hasn't yet occurred to me.

Limitations
===========

TODO
