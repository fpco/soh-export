# Монада для отзывчивого программирования. Часть I

_
Исходная статья «A monad for reactive programming. Part 1»<sup>[1][source]</sup>
написанная Альберто Гомез Корона (Alberto Gómez Corona).
Свободный (интерпретированный) перевод выполнен Артуром Файзрахмановым.
_

_Нетерпеливым: вы можете найти кое—что работающее в самом низу._

# Вступление

Отзывчивое _(реактивное)_ функциональное программирование не имеет понятия об
области видимости событий.  При получении сигнала повествовательное
_(декларативное)_ отзывчивое функциональное вычисление затрагивается целиком и
должно некоторым образом начаться заново, в силу своей повествовательной
природы.  Монада же может содержать в себе область видимости сигнала,– кое–что
уже вычислено заранее и эта часть не меняется, когда текущий компонент
встраивает сигнал в вычисление на каком то этапе.

Когда происходит вызов монады с более высокого уровня по сигналу, она может
разложить вычисление в цепочку виртуальных обработчиков событий _(установленных
монадным вычислением)_, даже в момент своего первого вызова.

Такой подход имеет большую область применения.  Представьте себе такое
вычисление:

```haskell
profits = do
    quantity ← waitEvent "количество"
    liftIO $ do
        putStr "количество="
        print quantity
        getChar
    price    ← waitEvent "стоимость"
    liftIO $ do
        putStr "стоимость="
        print price
        getChar
    liftIO $ do
        putStr "итог="
        print (quantity * price)
        getChar
    return (quantity * price)
```

Представим, что количество акций и их стоимость меняются в течении суток, таким
образом у нас есть два события: первое — когда меняется количество акций,
второе — когда изменяется стоимость акций.  Представим, что на самом деле
программа обновляет данные на информационном табло, вместо того чтобы печатать
вывод в консоль, и что в действительности вычисления гораздо более трудоёмкие и
затратные, производящие доступ к базам данных и отчётов в реальном времени,
сопровождаемые большим количеством других событий.  Мы хотим сделать так, чтобы
при изменении стоимости монада выполнила только те действия, которые нужны для
изменения стоимости и дохода, не затрагивая другие затратные ветки монады,
включая не только те вычисления, которые произошли выше, но и ветвления, которые
могут произойти в дальнейшем _(после события)_, если мы пожелаем.

# Воплощение задумки

Монада, подходящая для настройки отклика на события подобным образом, может
выглядить примерно так:

```haskell
data Transient m x = Transient (m (Maybe x))

instance (MonadState m, StateType m ~ EventF) ⇒ Monad (Transient m) where
    return x = Transient (return . Just $ x)
    x ↠ f = Transient $ do
        setEventCont f
        mk ← runTrans x
        case mk of
          Just k  → runTrans (f k)
          Nothing → return Nothing
```

Название `Transient`, что означает «мимолётный», взято из моей давней идеи
замыканий в монадическом выражении в виде мимолётных чистых состояний, которые
вычисляются заново только тогда, когда происходит изменение выше.

Мы можем остановить остаток вычисления ниже уровнем таким образом:

```haskell
stop = empty

empty :: Monad m => Transient m a
empty = Transient (return Nothing)
```

Это пустое значение `empty` будет использоваться в аппликативном образце для
«мимолётной» монады, но мы не будем описывать его прямо сейчас.

Мы используем монаду состояния, так как нам требуется передавать данные события
`EventF`, которые представляют из себя продолжения вычислений _(f)_ для каждого
вычисления _(x)_.

```haskell
data EventF = forall m b c. EventF (b → Transient m c)
```

`Transient` использует возможные значения `Maybe`, потому что мы хотим иметь
возможность остановливать продвижение вычислительного потока вглубь всегда,
когда в этом нет нужды.  Монада не устанавливает обработчики событий, вместо
этого она просто запоминает следующее продолжение вычисления с помощью
`setEventCont`:

```haskell
setEventCont f = do
    f' ← get
    put $ EventF f
    return f'
```

`waitEvent` и есть настроящее вычисление,– текущий _(активный)_ компонент,
который устанавливает обработчик события.  Этот обработчик – продолжение
вычисления текущего компонента (установленный монадой) для ожидающего события.

_(Заметим, что проще было бы обойтись без состояния, но, как покажет дальнейшее
повествование, это на самом деле необходимо.)_

В конце концов, примитив `waitEvent` приостанавливает вычисление, возвращая
`Nothing`:

``` haskell
waitEvent name = Transient $ do
    f   ← get
    evs ← liftIO $ takeMVar eventHandlers
    liftIO . putMVar eventHandlers . M.insert name f $ evs -- !> ("set "++name)
    return Nothing
```

Он использует карту обработчиков событий:

``` haskell
eventHandlers ∷ MVar (M.Map String EventF)
eventHandlers = unsafePerformIO $ newMVar M.empty
```

После завершения выполнения монадического выражения эта структура по меньшей
мере имеет первый обработчик событий.  `eventLoop` ожидает события и выполняет
соответствующий обработчик:

``` haskell
type EvType = String
data Event = forall a. Event EvType a

eventLoop [] = return()
eventLoop (Event name r : evs) = do
    liftIO . print $ "новое событие: " ++ name
    ths ← liftIO . readMVar $ eventHandlers
    case M.lookup name ths of
      Just (EventF f) → runTrans'' $ (unsafeCoerce f) r
      Nothing         → return ()
    eventLoop evs
```

Событие `Event` содержит в себе тип _(который используется для поиска
обработчиков)_ и значение.

Вместо использования списка, `eventLoop` может быть описан как процесс,
считывающий некую очередь _[событий]_.


Обратите внимание на использование небезопасного преобразования `unsafeCoerce`.
Конструкция диктует то, что это сработает как нужно, так как значение `x` из
монады будет передано продолжению `f`, записанному в состоянии, но компилятор
об этом не знает.  Сделаем это допущение в целях экспиремента.  Я – физик, не
математик и не инженер.  Будь я математиком, я бы провёл три года в борьбе с
денотационной семантикой обработки событий, производя заумные работы прежде чем
представить что-то, что можно было бы использовать.  Если же я был инженером, я
бы написал простую показательную программу конкретной задачи.  Я люблю
математику и инженерное дело, но я с радостью приношу в жертву красоту концепции
и сроки поставки во имя решения общей проблемы.

Вычисление `runTrans''` запускает продолжение, принимая трансформер `StateT`,
который переносит состоние `EventF`.

``` haskell
runTrans'' ∷ Transient (StateT EventF IO) a → IO ()
runTrans'' tmx = runTrans' tmx >> return ()

runTrans' ∷ Monad m ⇒ Transient (StateT EventF m) x → m (Maybe x)
runTrans' tmx = evalStateT (runTrans tmx) undefined
```

Продолжение вызываемое `runTrans''` может содержать в себе последующие выражения
`waitEvent`, добавляющие последующие обработчики событий в петление таким же
образом.

# Запуск

Давайте скормим нашему чудовищу такой набор событий:

``` haskell
eventList =
    [ Event "количество" 10
    , Event "стоимость"   2
    , Event "стоимость"   3
    , Event "количество" 30
    , Event "стоимость"   4
    ]
```

Теперь запускаем (**нажимайте <kdb>Ввод</kbd>, чтобы продолжать вычисления**,
потому что функция доходов `profits` использует `getChar`).


``` active haskell
{-# LANGUAGE ExistentialQuantification #-}
{-# LANGUAGE FlexibleContexts          #-}
{-# LANGUAGE UndecidableInstances      #-}
{-# LANGUAGE MultiParamTypeClasses     #-}

import Control.Concurrent.MVar
import Control.Monad.State
import Data.Map                as M
import System.IO.Unsafe
import Unsafe.Coerce


data EventF = forall m b c. EventF (b -> Transient m c)
data Transient m x = Transient (m (Maybe x))

runTrans :: Transient m x -> m (Maybe x)
runTrans (Transient mx) = mx

setEventCont f = do
    f' <- get
    put $ EventF f
    return f'

empty :: Monad m => Transient m a
empty = Transient $ return Nothing

instance (MonadState EventF m) => Monad (Transient m) where
    return x = Transient . return . Just $ x
    x >>= f = Transient $ do
        setEventCont f
        mk <- runTrans x
        case mk of
          Just k  -> runTrans (f k)
          Nothing -> return Nothing

instance MonadTrans Transient where
    lift mx = Transient $ mx >>= return . Just

instance (MonadState EventF m, MonadIO m) => MonadIO (Transient m) where
    liftIO io = let x = liftIO io in x `seq` lift x

instance (MonadState EventF m) => MonadState EventF (Transient  m) where
--  type StateType (Transient m) = EventF
    get = Transient $ get >>= return . Just
    put x = Transient $ put x >> return (Just ())

eventHandlers :: MVar (M.Map String EventF)
eventHandlers = unsafePerformIO $ newMVar M.empty

type EvType = String
data Event = forall a. Event EvType a

waitEvent name = Transient $ do
    f <- get
    evs <- liftIO $ takeMVar eventHandlers
    liftIO . putMVar eventHandlers . M.insert name f $ evs
    return Nothing

eventLoop [] = return ()
eventLoop (Event name r : evs) = do
    liftIO . putStrLn $ "новое событие: " ++ name
    ths <- liftIO . readMVar $ eventHandlers
    case M.lookup name ths of
      Just (EventF f) -> runTrans'' $ (unsafeCoerce f)  r
      Nothing         -> return ()
    eventLoop evs

runTrans' :: Monad m => Transient (StateT EventF m) x -> m (Maybe x)
runTrans' tmx = evalStateT (runTrans tmx) undefined

runTrans'' :: Transient (StateT EventF IO) a -> IO ()
runTrans'' tmx = runTrans' tmx >> return ()

-- show
main :: IO ()
main = do
    runTrans'' profits
    eventLoop eventList
    putStrLn "КОНЕЦ"

eventList :: [Event]
eventList =
    [ Event "количество" 10
    , Event "стоимость"   2
    , Event "стоимость"   3
    , Event "количество" 30
    , Event "стоимость"   4
    ]

profits :: Transient (StateT EventF IO) Integer
profits = do
    quantity <- waitEvent "количество"
    liftIO $ do
        putStr "количество="
        print quantity
        getChar
    price <- waitEvent "стоимость"
    liftIO $ do
        putStr "стоимость="
        print price
        getChar
    liftIO $ do
        putStr $ "итог="
        print $ quantity * price
        getChar
    return $ quantity * price

-- /show
```

Как видите, выполняется только тот код, который следует за событием, и
используется только тот вышестоящий контекст, который необходим продолжению для
каждого вызова по событию.

Вот и всё.

Хотя, нет.  Взгляните на этот пример:

``` haskell
main = do
    runTrans'' $ do
        let threshold = 100
        pr <- profits
        liftIO $ do
            when (pr > threshold) $
                putStr "Порог пройден! Отправлено письмо начальнику."
            print pr
    eventLoop eventList
    putStrLn "END"
```

Здесь выполняется то же самое вычисление прибыли `profits` с теми же событиями,
но в самом конце если доход превышает заданный порог производится дополнительно
сообщение.

Возможно, вы думаете, что последним сообщением программы будет

```
"Порог пройден…"
```

, если заменить главное выражение `main` в программе на приведённое в
примере, но это не так.  Это значит, что в таком виде монада выполняет неполный
набор действий по событию.  Необходимо сохранять состояние полного набора всех
вложенных продолжений во всех вложенных вычислениях, затрагиваемых событием, а
не отдельно взятые продолжения.

Кроме того, небезопасное приведение типов `unsafeCoerce` может привести к ошибке
сегментации, к примеру, если в монадическом вычислении мы заменим

``` haskell
waitEvent "quantity"
```

на

``` haskell
(*) <$> return units <*> waitEvent "quantity"
```

.

Но об этом во второй части.

Кстати говоря, там я представлю улучшенную версию этой монады, используемую в
[клиентском фреймворке `hplayground`][hplay].

[source]: https://www.fpcomplete.com/user/agocorona/a-monad-for-reactive-programming-part-1
[hplay]: https://github.com/agocorona/hplayground
