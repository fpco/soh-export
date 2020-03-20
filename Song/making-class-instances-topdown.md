# Making class instances topdown

# The problem

When I was reading SPJ's series of papers on SYB, I feel this is really an abstract and very good way to manipulate complex data. However, I quickly run into an annoying state when make instances for ``Data``, ``Typeable``, ``Show`` and many other class instances. There might be a quick solution with ``CPP`` language extension like the following:

```Haskell

#define DERIVINGS deriving (Typeable, Data ,Show, Eq,Generic)
data Company = C [Dept]  DERIVINGS
data Dept = D Name Manager [SubUnit] DERIVINGS
data SubUnit = PU Employee | DU Dept DERIVINGS
data Employee = E Person Salary DERIVINGS
data Person = P Name Address DERIVINGS
data Salary = S Float DERIVINGS
type Manager = Employee
type Name = String 
type Address = String

#undef DERIVINGS
```

You might think this is a joke and yes it is. And this kind of joke comes up over and over agagin. Back in days of GHC 7.8, the data types in ``Language.Haskell.TH`` are not instances of ``Generic``. If you want to make them instances of it then do it like the follwoing:

```Hakell
deriving instance Generic FixityDirection
deriving instance Generic Inline
deriving instance Generic RuleBndr
deriving instance Generic Match
deriving instance Generic Name
deriving instance Generic RuleMatch
deriving instance Generic Pred
deriving instance Generic Phases
deriving instance Generic Con
deriving instance Generic Module
deriving instance Generic AnnTarget
deriving instance Generic Type
deriving instance Generic TyVarBndr
deriving instance Generic TyLit
deriving instance Generic Exp
deriving instance Generic Lit
deriving instance Generic Pat
deriving instance Generic Dec
deriving instance Generic Clause
deriving instance Generic FunDep
deriving instance Generic Foreign
deriving instance Generic Fixity
deriving instance Generic Pragma
deriving instance Generic FamFlavour
...
```

You will not go crazy until you find you have ``Binary`` or other many empty instances to write. 

When I was working with ``jvm-codec`` to generate some bytecode with JVM, I want to pretty print the AST in it and the code is like the following:

```Haskell
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE StandaloneDeriving , CPP#-}
{-# LANGUAGE DeriveGeneric, DeriveAnyClass #-}
module Codec.JVM.Pretty 
    (module Text.PrettyPrint.GenericPretty)
    where

import Codec.JVM
import Codec.JVM.ASM.Code.Instr
import Codec.JVM.ConstPool
import Codec.JVM.Const
import Codec.JVM.Method
import Codec.JVM.Field
import Codec.JVM.Attr
import Codec.JVM.ASM.Code.CtrlFlow

import GHC.Generics
import Text.PrettyPrint.GenericPretty
import Text.PrettyPrint
import Codec.JVM.Pretty.GenericOut

#define DERIVE_INSTANCE(T) deriving instance Generic (T); deriving instance Out (T)
DERIVE_INSTANCE(ClassFile)
DERIVE_INSTANCE(Version)
DERIVE_INSTANCE(IClassName)
DERIVE_INSTANCE(Const)
DERIVE_INSTANCE(MethodInfo)
DERIVE_INSTANCE(FieldInfo)
DERIVE_INSTANCE(AccessFlag)
DERIVE_INSTANCE(Attr)
DERIVE_INSTANCE(NameAndDesc)
DERIVE_INSTANCE(MethodRef)
DERIVE_INSTANCE(FieldRef)
DERIVE_INSTANCE(ConstVal)
DERIVE_INSTANCE(UName)
DERIVE_INSTANCE(Desc)
DERIVE_INSTANCE(Code)
DERIVE_INSTANCE(Offset)
DERIVE_INSTANCE(FieldType)
DERIVE_INSTANCE(PrimType)
DERIVE_INSTANCE(StackMapFrame)
DERIVE_INSTANCE(InnerClassMap)
DERIVE_INSTANCE(InnerClass)
DERIVE_INSTANCE(VerifType)
DERIVE_INSTANCE(MethodDef)
DERIVE_INSTANCE(MethodDesc)
#undef DERIVE_INSTANCE

deriving instance Generic (Signature a)
deriving instance Out a => Out (Signature a)

deriving instance Generic (MethodSignature a)
deriving instance Out a => Out (MethodSignature a)

deriving instance Generic (FieldSignature a)
deriving instance Out a => Out (FieldSignature a)

deriving instance Generic (ClassSignature a)
deriving instance Out a => Out (ClassSignature a)

deriving instance Generic (TypeVariableDeclaration a)
deriving instance Out a => Out (TypeVariableDeclaration a)

deriving instance Generic (ReferenceParameter a)
deriving instance Out a => Out (ReferenceParameter a)

deriving instance Generic (Parameter a)
deriving instance Out a => Out (Parameter a)

deriving instance Generic (TypeParameter a)
deriving instance Out a => Out (TypeParameter a)

deriving instance Generic (Bound a)
deriving instance Out a => Out (Bound a)

instance Out Instr where
  docPrec n x = parens $ text (show x)
  doc = docPrec 0  
```

# The solution

So I post <https://ghc.haskell.org/trac/ghc/ticket/10607>. And in ICFP 2016 Nara, I demostrated another solution for this class instance deriving problem. Use a package called  [``derive-topdown``](http://hackage.haskell.org/package/derive-topdown) 

Here, I introduce only standalone deriving and template Haskell

## 1. Standalone deriving
There are functions named `deriving_`, `derivings`, `derivingss`. Please see the API for their types.

	{-# LANGUAGE StandaloneDeriving,
		ConstraintKinds,
		UndecidableInstances,
		GADTs,
		TemplateHaskell,
		DeriveGeneric #-}
	{-# OPTIONS_GHC -ddump-splices #-}
	
	import Data.Derive.TopDown
	import GHC.Generics
	import Data.Binary
	imoprt Data.Aeson
	import Data.Aeson.TH

	data Gender = Male | Female
	type Age = Int
	data Person a = P {name :: String , age :: Int, gender :: Gender}
	data Department a = D {dname :: String , 
						   head :: Person a, 
						   staff :: [Person a]}
	data Company a = C {cname :: String, 
	                    departments :: [Department a]}
	
	derivings [''Eq, ''Ord, ''Generic] ''Company

You will get:

		derivings [''Eq, ''Ord, ''Generic] ''Company
	  ======>
	    deriving instance Eq Gender
	    deriving instance Eq (Person a_acKV)
	    deriving instance Eq a_acKU => Eq (Department a_acKU)
	    deriving instance Eq a_acKT => Eq (Company a_acKT)
	    deriving instance Ord Gender
	    deriving instance Ord (Person a_acKV)
	    deriving instance Ord a_acKU => Ord (Department a_acKU)
	    deriving instance Ord a_acKT => Ord (Company a_acKT)
	    deriving instance Generic Gender
	    deriving instance Generic (Person a_acKV)
	    deriving instance Generic (Department a_acKU)
	    deriving instance Generic (Company a_acKT)

## 2. Usage with Template Haskell
For generating instances with a template Haskell function, `derivingTH`, `derivingTHs` and `derivingTHss` can be used:
	
	   derivingTHs
	      [(''ToJSON, deriveToJSON defaultOptions),
	       (''FromJSON, deriveFromJSON defaultOptions)]
	      ''Company
	  ======>
	    instance ToJSON Gender where
	      toJSON
	        = \ value_amQG
	            -> case value_amQG of {
	                 Male -> String (text-1.2.2.2:Data.Text.pack "Male")
	                 Female -> String (text-1.2.2.2:Data.Text.pack "Female") }
	      toEncoding
	        = \ value_amQH
	            -> case value_amQH of {
	                 Male
	                   -> Data.Aeson.Encoding.Internal.text
	                        (text-1.2.2.2:Data.Text.pack "Male")
	                 Female
	                   -> Data.Aeson.Encoding.Internal.text
	                        (text-1.2.2.2:Data.Text.pack "Female") }
	    instance ToJSON a_amqg => ToJSON (Person a_amqg) where
	      toJSON
	        = \ value_amQy
	        ...
	        ...



