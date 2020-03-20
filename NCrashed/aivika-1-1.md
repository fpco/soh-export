# Имитационное моделирование с Aivika. Часть 1

Постановка задачи
=================

Моделируется простая система массового обслуживания, состоящая из генератора, буфера и обслуживающего автомата. Если буфер переполняется, заявки откидываются. Время генерации и обработки заявок распределено по экспоненциальному закону.

<div style="text-align:center"><img src="https://raw.githubusercontent.com/NCrashed/bmstu-aivika-tutorial-01/master/tutorial/system.png"/></div>

Условные обозначения:

* Г - генератор заявок

* Б - буфер заявок

* ОА - обслуживающий автомат


Входные данные модели:

* ![](http://images.astronet.ru/pubd/tex/formula31061.gif) - среднее время между заявками

* m - емкость буфера

* ![](http://images.astronet.ru/pubd/tex/formula31062.gif)- среднее время обработки заявки

Объявим структуру, которая будет содержать входные данные:

``` haskell
data Input = Input {
    generationTime :: Double,
    processingTime :: Double,
    bufferCapacity :: Int
}
```

Выходные данные модели:

* Вероятность отказа системы

* Средний размер очереди в буфере

* Среднее количество заявок в системе

* Среднее время ожидания в буфере


Объявим структуру, которая будет содержать выходные данные:

``` haskell
data Output = Output {
    failChance :: Double,
    queueSize :: Double,
    requestsCount :: Double,
    awaitingTime :: Double
}
```

Необходимо написать функцию, которая вычислит `Output` на основе данного `Input`:
``` haskell
simulate :: Input -> Simulation Output
```

Возвращаемый тип обернут в монаду `Simulation`, которая отвечает за вычисления внутри симуляции. В рамках описанной задачи понимание концепции монады не требуется, читатель может рассматривать ее как контейнер, содержащий значение и правила его вычисления. Желающим ознакомиться предлагаем прочитать [статью](http://habrahabr.ru/post/183150/).

Введение в Aivika
=================
[Aivika](https://hackage.haskell.org/package/aivika-3.0) - библиотека для имитационного моделирования, поддерживающая несколько парадигм. Однако мы рассмотрим только одну из них - моделирование на основе процессов.

Библиотека использует сложную иерархию монад для представления различных типов вычислений. Перечислим основные монады, которые потребуются нам для решения задачи:

* `Simulation` - вычисления внутри симуляции. Является основной монадой в Aivika, так как `Simulation a` обозначает имитационную модель, которая вычисляет значение типа `a` и описывает все аспекты модели.

* `Event` - значение внутри этой монады является функцией от времени со строгой синхронизацией с очередью событий. Вычисления в монаде `Event` не могут быть приостановлены. Пример использования: отправка сообщения в буфер.

* `Process` - вычисление, которое может быть приостановлено в любой момент и возобновлено позже. В нашей задаче данная монада будет использована для представления процессов генератора и обслуживающего автомата.

* `Parameter` - значение внутри этой монады является внешним параметром для имитационной модели. Пример использования: получение максимального времени, до которого проводится симуляция; вычисление случайных величин.

Данные монады выстраивают иерархию, которая показана на рисунке ниже. Значение из менее общей монады можно преобразовать в значение более общей монады. Обратное преобразование запрещено, так как происходит потеря контекста. По схожей логике, целые числа можно преобразовать в дробные без потери информации, а дробные в целые - просходит потеря информации о дробной части числа.

Стрелка от монады `A` к монаде `B` означает, что вычисление `A t` может быть приобразовано в вычисление `B t` с помощью специальных функций, имена которых начинаются с `lift` (например, `liftSimulation` может перевести вычисление `Simulation t` в `Event t` и `Process t`):


<div style="text-align:center"><img src="https://raw.githubusercontent.com/NCrashed/bmstu-aivika-tutorial-01/master/tutorial/monads_hierarchy.png"/></div>

Построение обслуживающих автоматов
==================================

В данной модели буфер можно представить как FIFO (first in, first out), иначе называемую FCFS (first come, first served), очередь. В aivika есть подходящий тип очереди: `FCFSQueue`:

``` haskell
type Buffer a = FCFSQueue a
```

Здесь `a` отвечает за тип элементов, которые хранятся в очереди. Также для краткости определим синоним функции для создания нового буфера:

``` haskell
newBuffer :: Int -> Event (Buffer a)
newBuffer = newFCFSQueue 
```

Функция принимает максимальную емкость буфера, после которой элементы будет отбрасываться. Также нужно обратить внимание: так как новый буфер обернут в монаду `Event`, создание очереди происходит мгновенно в модельном времени.


Теперь определим обслуживающий автомат. Его будем моделировать с помощью зацикленного вычисления в монаде `Process`. Так как обслуживающий автомат использует входной буфер, то нам понадобится функция-обертка, которая предварительно создаст буфер и передаст его в обработчик обслуживающего автомата (далее ОА):

``` haskell
withBuffer :: Int -> (Buffer a -> Process b) -> Event (Process b, Buffer a)
withBuffer capacity f = do
   buffer <- newBuffer capacity
   return (f buffer, buffer)
```

Функция принимает размер буфера ОА, функцию-обработчик и возвращает кортеж из двух элементов: новый обработчик ОА, готовый к запуску симуляции; и созданный буфер, который будет использоваться в других ОА.

В качестве заявок, проходящих через систему будем использовать следующий тип:
``` haskell
data Request = Request
```
Он не несет никакой дополнительной информации в себе, значение этого типа просто обозначает наличие заявки.

Обрабатывающий автомат
----------------------

Теперь мы можем определить обработчик ОА с буфером:
``` haskell
processor :: Event (Process (), Buffer Request)
processor = withBuffer bufferCapacity $ \buffer-> forever $ do
   _ <- dequeue buffer
   holdExponential processingTime
```

Для создания ОА мы использовали только что определенную функцию `withBuffer`. ОА бесконечно (благодаря функции `forever`) забирает заявки из буфера с помощью `dequeue`, уничтожает их (так как заявка покидает систему после этого ОА и больше не нужна), и имитирует обработку путем остановки на случайное количество секунд, распределенных экспоненциально с парамером λ равным `processingTime` секунд. Для приостановки процесса используется вспомогательная функция `holdExponential`, которую мы определим как:

```haskell
holdExponential :: Double -> Process ()
holdExponential t = do
    htime <- liftParameter $ randomExponential t
    holdProcess htime
```

Нужно заметить, что функция из aivika `randomExponential` имеет тип:
``` haskell
randomExponential :: Double-> Parameter Double
```
Поэтому мы поднимаем вычисление из монады `Parameter` в монаду `Process` с помощью `liftProcess`.

Функция `holdProcess` из aivika приостанавливает выполнение процесса на переданное ей количество времени.

Генератор заявок
----------------

Определим генератор заявок, он будет бесконечно создавать новые `Request` и класть их во входной буфер ОА:
``` haskell
generator :: Buffer Request -> Process ()
generator buffer = forever $ do
       holdExponential generationTime
       liftEvent $ enqueueOrLost_ buffer Request
```

Функция `enqueueOrLost_` в aivika пытается поставить заявку в очередь, и если буфер переполнен, то отбрасывает ее. Функция выполняется в монаде `Event`, поэтому нам необходимо использовать `liftEvent`, чтобы поднять вычисление в монаду `Process`.


Собираем систему
================

Определим функцию, которая создаст описание модели, готовой для запуска симуляции:
``` haskell
simulate :: Input -> Simulation ()
simulate Input{..} = runEventInStartTime $ do
    (processorProc, buffer) <- processor
    generatorProc <- generator buffer
    liftSimulation $ do
        runProcessInStartTime processorProc
        runProcessInStartTime generatorProc
    where
    
    generator :: Buffer Request -> Process ()
    generator buffer = forever $ do
       holdExponential generationTime
       liftEvent $ enqueueOrLost_ buffer Request

    processor :: Event (Process (), Buffer Request)
    processor = withBuffer bufferCapacity $ \buffer-> forever $ do
       _ <- dequeue buffer
       holdExponential processingTime
```

Входной параметр с настройками модели записан с помощью расширения языка [RecordWildcards](https://ocharles.org.uk/blog/posts/2014-12-04-record-wildcards.html), поэтому поля структуры `Input` видны в функциях `generator` и `porcessor`.

`runEventInStartTime` позволяет запустить вычисление в монаде `Event` в момент старта симуляции, в этот момент мы создаем генератор и ОА и связываем их вместе, а функция `runProcessInStartTime` запускает обработчики генератора и ОА в момент старта симуляции.

Сбор статистики
===============

Модель готова, но ничего полезного из нее сейчас нельзя получить, поэтому добавим сбор статистики из генератора и ОА. 

Для отслеживания количества заявок в системе добавим переменную-счетчик `Var Int`, которая будет увеличиваться, когда заявка выходит из генератора и уменьшаться, когда ОА заканчивает обработку заявки. Особенность `Var` в том, что этот тип переменной сохраняет историю изменений, что позволит посчитать среднее количество заявок в системе:
``` haskell
generator :: Buffer Request -> Event (Process (), Var Int)
generator buffer = do
    stats <- liftSimulation $ newVar 0
    return $ (, stats) $ forever $ do
       holdExponential generationTime
       liftEvent $ do
           modifyVar stats (+1)
           enqueueOrLost_ buffer Request

processor :: Var Int -> Event (Process (), Buffer Request)
processor stats = withBuffer bufferCapacity $ \buffer-> forever $ do
   _ <- dequeue buffer
   holdExponential processingTime
   liftEvent $ modifyVar stats (subtract 1)
```

В функции создания генератора создается новая переменная с помощью `stats <- liftSimulation $ newVar 0`, и увеличивается на 1 каждый раз, когда новая заявка появляется в системе. 

Для ОА переменная является входным аргументом, каждый раз, когда заявка покидает систему, переменная уменьшается на единицу.

Однако при создании генератора и ОА в функции `simulate` мы столкнемся с рекурсивной зависимостью:
``` haskell
simulate :: Input -> Simulation ()
simulate Input{..} = runEventInStartTime $ do
    (processorProc, buffer) <- processor stats
    (generatorProc, stats) <- generator buffer -- рекурсивная зависимость
    liftSimulation $ do
        runProcessInStartTime processorProc
        runProcessInStartTime generatorProc
    where
    
    generator :: Buffer Request -> Process ()
    generator buffer = forever $ do
       holdExponential generationTime
       liftEvent $ enqueueOrLost_ buffer Request

    processor :: Event (Process (), Buffer Request)
    processor = withBuffer bufferCapacity $ \buffer-> forever $ do
       _ <- dequeue buffer
       holdExponential processingTime
```
Переменная `stats` используется в `processor` до того, как она создается в `generator`, но поменять порядок выполнения мы не можем, так как `processor` создает буфер `buffer`, который используется в `generator`.

Для разрешения этой проблемы воспользуемся расширением [`RecursiveDo`](https://www.haskell.org/haskellwiki/MonadFix),оно позволит "отложить" использование рекурсивной перменной `stats`, чтобы передать ее в `processor`:
``` haskell
simulate :: Input -> Simulation ()
simulate Input{..} = runEventInStartTime $ do
    -- ключевое слово rec обозначает блок с рекурсивными зависимостями
    rec (processorProc, buffer) <- processor stats
        (generatorProc, stats) <- generator buffer
    liftSimulation $ do
        runProcessInStartTime processorProc
        runProcessInStartTime generatorProc
    where
    
    generator :: Buffer Request -> Process ()
    generator buffer = forever $ do
       holdExponential generationTime
       liftEvent $ enqueueOrLost_ buffer Request

    processor :: Event (Process (), Buffer Request)
    processor = withBuffer bufferCapacity $ \buffer-> forever $ do
       _ <- dequeue buffer
       holdExponential processingTime
```

Теперь настало время заполнить структуру `Output` с помощью средств для сбора статистических данных в aivika. Функция `simulate` теперь будет возвращать тип `Output`:
``` haskell
simulate :: Input -> Simulation Output
```

А в конец функции мы добавим код с вычислением выходных данных модели:
``` haskell
runEventInStopTime $ do
   -- Вытаскиваем из буфера статистику по размеру
   sizeStats <- queueCountStats buffer
   -- Достаем из буфера количестов откинутых заявок и конвертируем в Double
   lostCount <- fromIntegral <$> enqueueLostCount buffer
   -- Достаем из буфера общее количество заявок за все вермя
   totalCount <- fromIntegral <$> enqueueCount buffer
   -- Статистика со временем ожидания заявки в буфере
   awaiting <- queueWaitTime buffer 
   -- Вытаскиваем статистику из нашей переменной
   requestsCountStats <- statsFromVar stats 
   return Output {
       -- Считаем вероятность отказа через определение вероятности
       failChance = lostCount / (lostCount + totalCount),
       -- Вычисляем среднее значение размера буфера
       queueSize = timingStatsMean sizeStats,
       -- Вычисляем среднее количество заявок в системе
       requestsCount = samplingStatsMean requestsCountStats,
       -- Вычисляем среднее время ожидания заявки в буфере
       awaitingTime = samplingStatsMean awaiting 
   }
```
Итого функция, описывающая модель поставленной задачи:
``` haskell
simulate :: Input -> Simulation Output
simulate Input{..} = runEventInStartTime $ do
    rec (processorProc, buffer) <- processor stats
        (generatorProc, stats) <- generator buffer
    liftSimulation $ do
        runProcessInStartTime processorProc
        runProcessInStartTime generatorProc
        runEventInStopTime $ do
           sizeStats <- queueCountStats buffer
           lostCount <- fromIntegral <$> enqueueLostCount buffer
           totalCount <- fromIntegral <$> enqueueCount buffer
           awaiting <- queueWaitTime buffer 
           requestsCountStats <- statsFromVar stats 
           return Output {
               failChance = lostCount / (lostCount + totalCount),
               queueSize = timingStatsMean sizeStats,
               requestsCount = samplingStatsMean requestsCountStats,
               awaitingTime = samplingStatsMean awaiting 
           }
    where
    generator :: Buffer Request -> Event (Process (), Var Int)
    generator buffer = do
        stats <- liftSimulation $ newVar 0
        return $ (, stats) $ forever $ do
           holdExponential generationTime
           liftEvent $ do
               modifyVar stats (+1)
               enqueueOrLost_ buffer Request

    processor :: Var Int -> Event (Process (), Buffer Request)
    processor stats = withBuffer bufferCapacity $ \buffer-> forever $ do
       _ <- dequeue buffer
       holdExponential processingTime
       liftEvent $ modifyVar stats (subtract 1)
```

Весь описанный выше код размещен в [отдельном проекте](https://www.fpcomplete.com/user/NCrashed/adopted/aivika-example), где его можно изменять и запускать:
``` 
Вероятность отказа: 0.80
Средний размер буфера: 1.83
Среднее число заявок в системе: 4.29
Среднее время ожидания в буфере: 99.99
```

В предоставленном проекте входные параметры изменены так, чтобы была возможность задавать закон распределения генерации заявок:

``` haskell
-- | Входные параметры модели
testInput :: Input
testInput = Input {
    -- | Закон распределения потока заявок
    -- Указывается один из: exponential, erlang, normal, uniform
    -- каждому закону нужно указать список параметров
    -- для exponential нужен 1 параметр (среднее время между заявками)
    -- для других (erlang и т.д.) нужно по 2 параметра
    generationDistribution = generationDistr "hyperexponential" [10, 0.5, 120, 0.5],
    -- | Закон распределения обработки заявок в обслуживающем автомате
    -- параметры аналогичны.
    processingDistribution = generationDistr "exponential" [50],
    -- | Емкость буффера
    bufferCapacity = 2,
    -- | Время иммитационного моделирования
    simulationTime = 500000.0,
    -- | Количество знаков после запятой в выводе результатов
    outputPrecision = 5
}
```

Поддерживаемые законы:

* `exponential` - экспоненциальное распределение, принимает один параметр (среднее время между событиями). Пример: `generationDistr "exponential" [50]`

* `hyperexponential` - гиперэкспоненциальное распределение, принимает пары параметров: вероятность i-ого экспоненциального распределения и его параметр (среднее время между событиями). Пример: `generationDistr "hyperexponential" [10, 0.5, 120, 0.5]`

* `erlang` - эрланговское распределение, принимает два параметра: среднее время (плавающая запятая) и порядок (положительное целое). Пример: `generationDistr "erlang" [50, 2]`

* `normal` - нормальное распределение, принимает два параметра: среднее значение (плавающая запятая) и среднеквадратичное отклонение (плавающая запятая). Пример: `generatonDist "normal" [25.5, 2.5]`

* `uniform` - равномерное распределение, принимает два параметра: минимальное время и максимальное (плавающая запятая). Пример: `generationDist "uniform" [10, 20]`