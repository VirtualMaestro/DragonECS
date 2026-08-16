# DragonECS — Bug-Hunting Review

**Дата:** 2026-08-16
**Ревизия:** `95efd83` (up version 1.1.9), ветка `main`
**Объём:** 68 файлов C#, ~26 300 строк — прочитаны полностью
**Метод:** направленный поиск логических ошибок, race conditions, edge cases и runtime-исключений. Каждая находка перепроверена трассировкой конкретного сценария; критические баги дополнительно верифицированы вручную по исходникам. Стиль, архитектура и производительность намеренно не рассматривались.

**Итог:** ~50 подтверждённых логических ошибок + ~16 краевых/неподтверждённых.

Обозначения типов: `Uncaught Exception` — необработанное исключение; `State Corruption` — порча внутреннего состояния; `Logic Flaw` — неверный результат без падения; `Memory Leak` — утечка; `Race Condition` — гонка; `Infinite Loop` — зависание/бесконечная рекурсия.

---

## 1. Критические: ядро `EcsGroup` (порча состояния при обычном Add/Remove)

Кластер из четырёх связанных багов в `Remove_Internal` / `ChangeIndexInSparse`. Воспроизводится тривиальными последовательностями и поражает все операции, использующие удаление: `ExceptWith`, `IntersectWith`, `ClearMarked/ClearUnmarked`, `RemoveUnusedEntityIDs`.

### 1.1. Глобальная порча общей `_nullPage`

- **Файл:** `src/Collections/EcsGroup.cs:791`
- **Сценарий:** `g.Add(5); g.Remove(5);` — удаляемая сущность единственная на своей 64-странице и последняя в dense. `--page.Count == 0` → вызывается `ChangeIndexInSparse(_dense[_count] = 5, ...)` для **самой удаляемой** сущности, чья страница уже имеет `Count == 0`. В `ChangeIndexInSparse` ветка `page.Count == 1` ложна → выполняется `page.Indexes[5] = 1`, где `Indexes` — статическая общая `_nullPage`.
- **Последствия:** запись в `_nullPage` глобальна для процесса. Во **всех группах всех миров** `Has(x)` для любой сущности с `x & 63 == 5` на пустой странице возвращает `true`; `Add(x)` ложно отказывает; `Remove(x)` на пустой группе уводит `page.Count` и `_count` в −1 — каскадная порча.
- **Тип:** State Corruption (глобальная)
- **Fix:** не вызывать `ChangeIndexInSparse`, когда перемещаемый элемент и есть удаляемый (`_dense[_count] == entityID`, т.е. удаляется последний dense-элемент); либо в `ChangeIndexInSparse` обрабатывать `page.Count == 0` как no-op.

```csharp
// Remove_Internal, ветка --page.Count == 0:
_dense[page.IndexesXOR] = _dense[_count];
ChangeIndexInSparse(_dense[_count], page.IndexesXOR); // <- для удаляемой сущности пишет в _nullPage
page.IndexesXOR = 0;
```

### 1.2. Затирание `IndexesXOR` при переходе страницы 2→1

- **Файл:** `src/Collections/EcsGroup.cs:797-798`
- **Сценарий:** `g.Add(1); g.Add(2); g.Remove(2);` (удаляется последний dense-элемент). Трассировка: после Add — `dense[1]=1, dense[2]=2, Indexes[1]=1, Indexes[2]=2, IndexesXOR=3`. `Remove(2)`: else-ветка; `ChangeIndexInSparse(2, 2)` попадает в ветку `page.Count == 1` и **перезаписывает** `IndexesXOR = 2`; затем `IndexesXOR ^= 2` → `0` (должно быть `1` — индекс оставшейся сущности).
- **Последствия:** `Has(1)` проверяет `_dense[0] == 1` → false. Сущность «потеряна» для `Has`/`IndexOf`/`Remove`, хотя `_count == 1`.
- **Тип:** State Corruption
- **Fix:** тот же, что в 1.1 — при удалении последнего dense-элемента не трогать sparse удаляемой сущности; тогда `IndexesXOR ^= Indexes[local]` корректно даст индекс оставшейся сущности.

### 1.3. XOR-агрегат не учитывает перемещение элемента внутри той же страницы

- **Файл:** `src/Collections/EcsGroup.cs:798`
- **Сценарий:** `g.Add(1); g.Add(2); g.Add(3); g.Remove(1);` — последний элемент (сущность 3) переезжает с dense-индекса 3 на 1, `ChangeIndexInSparse(3, 1)` обновляет `Indexes[3] = 1`, но агрегат корректируется только на индекс удалённого: `XOR ^= 1`. Инвариант «XOR = xor dense-индексов всех сущностей страницы» нарушен: стало 1, должно быть 3 (`1 ^ 2`).
- **Последствия:** латентная порча — при следующем переходе страницы к `Count == 1` (ещё один Remove) оставшаяся сущность получает неверный `IndexesXOR` → ложный `Has == false`, группа рассинхронизирована.
- **Тип:** State Corruption
- **Fix:** при перемещении последнего элемента внутри той же страницы дополнительно корректировать агрегат: `XOR ^= oldIndexOfMoved(_count) ^ newIndex`.

### 1.4. Грязная страница возвращается в пул мира

- **Файл:** `src/Collections/EcsGroup.cs:799-804` (+ `TakePage` :432-441)
- **Сценарий:** при переходе 2→1 обнуляется только `Indexes[localEntityID]` (слот удаляемой сущности); слот оставшейся сущности остаётся ненулевым, после чего страница уходит в `ReturnPage`. `TakePage()` выдаёт страницы из пула **без очистки** (зануляется только свежая аллокация через `AllocAndInit`).
- **Последствия:** другая группа того же мира, получив эту страницу, видит ложные `Has == true` / битые dense-индексы для локальных слотов, оставшихся от прежнего владельца. Пример: G1: `Add(1); Add(2); Remove(1)` → страница в пуле с `Indexes[2] == 2`; G2: `Add(65); Add(66)` берёт её из пула → `G2.Has(e)` ложно true для отсутствующих сущностей с `e & 63 == 2`.
- **Тип:** State Corruption
- **Fix:** обнулять оставшиеся ненулевые слоты перед `ReturnPage`, либо очищать страницу в `TakePage`/`ReturnPage`.

### 1.5. `EcsWorld.ReleaseGroup`: группа попадает в пулы двух миров (STABILITY_MODE)

- **Файл:** `src/Collections/EcsGroup.cs:498-509`
- **Сценарий:** сборка с `DRAGONECS_STABILITY_MODE` (без DEBUG). Ветка «группа чужого мира» вызывает `group.World.ReleaseGroup(group)`, но **без `return`** — выполнение продолжается и кладёт ту же группу ещё и в `_groupsPool` текущего мира (аналогично при неудачном `TryGetWorld`).
- **Последствия:** два мира выдают один экземпляр группы двум потребителям; `_sparsePages` размерён под ёмкость чужого мира → запись за границы unmanaged-массива страниц.
- **Тип:** State Corruption
- **Fix:** `return;` после делегирования чужому миру и при неудачном `TryGetWorld`.

### 1.6. `EcsUnsafeSpan` в STABILITY_MODE: индексатор всегда возвращает NULL

- **Файл:** `src/Collections/EcsSpan.cs:526-528`
- **Сценарий:** сборка с `DRAGONECS_STABILITY_MODE`. Ветка `#elif DRAGONECS_STABILITY_MODE` содержит безусловный `return EcsConsts.NULL_ENTITY_ID;` (без проверки диапазона) — `return _values[index];` становится недостижимым.
- **Последствия:** любой поэлементный доступ к span в такой сборке возвращает нули вместо id сущностей.
- **Тип:** Logic Flaw
- **Fix:** `if ((uint)index >= (uint)_length) { return EcsConsts.NULL_ENTITY_ID; }`.

---

## 2. Критические: `EcsWorld`

### 2.1. Частичный `ReleaseDelEntityBuffer(count)` освобождает не те сущности

- **Файл:** `src/EcsWorld.cs:1385-1402`
- **Сценарий:** буфер удаления содержит 10 записей, вызывается `ReleaseDelEntityBuffer(4)`. Partition-цикл (разделение пустых/непустых сущностей) работает над индексами `[0, count)`, а спаны `fullBuffer`/`bufferSliced` строятся от смещения `_delEntBufferCount` — над **другим** диапазоном `[N-count, N)`. Циклы и спаны совпадают только при полном релизе (`_delEntBufferCount == 0`).
- **Последствия:** `_entityDispenser.Release` освобождает id сущностей, чьи компоненты остались в пулах и биты в `_entityComponentMasks` установлены. Следующий `NewEntity()` переиспользует эти id — новая сущность рождается с «призрачными» компонентами и `componentsCount > 0`.
- **Тип:** State Corruption
- **Fix:** вести partition в том же окне `[_delEntBufferCount, _delEntBufferCount + count)`, что и спаны (или наоборот — релизить первые `count` записей и сдвигать хвост).

### 2.2. `DeclareOrGetComponentTypeID` растит счётчик без ресайза, а ресайз однократный

- **Файл:** `src/EcsWorld.pools.cs:310-343` (+ :218-226)
- **Сценарий:** `DeclareOrGetComponentTypeID` (вызывается из публичного `GetComponentTypeID<T>()` и при построении масок) инкрементирует `_poolsCount` без ресайза `_pools`. Мир с `PoolsCapacity = 4`: объявить 10 типов → `_poolsCount = 10`, `_pools.Length = 4`. Инициализация пула для компонента с id 9: ресайз выполняется **однократным** удвоением до 8 → `_pools[9]` → IndexOutOfRangeException.
- **Тип:** Uncaught Exception
- **Fix:** `while (_poolsCount >= _pools.Length)` либо ресайз сразу до `CeilPow2(Math.Max(_poolsCount + 1, _pools.Length << 1))` (то же для `_poolSlots` и маски).

### 2.3. `InitEntitySlot`: off-by-one в `Upsize`

- **Файл:** `src/EcsWorld.cs:841-842`
- **Сценарий:** `InitEntitySlot(512, gen)` при capacity 512: `IdDispenser.Upsize(minSize)` ресайзит только при `minSize > _size` → `Upsize(512)` — no-op → `_entities[512]` → IndexOutOfRangeException. В соседнем `NewEntity(int)` корректно: `Upsize(entityID + 1)`.
- **Тип:** Uncaught Exception
- **Fix:** `_entityDispenser.Upsize(entityID + 1);`

### 2.4. `WorldComponentPool<T>.Abstract.GetRaw/SetRaw` используют worldID как внутренний индекс

- **Файл:** `src/EcsWorld.static.cs:340-343, 348-351`
- **Сценарий:** маппинг worldID → itemIndex хранится в `_mapping`, но `GetRaw(worldID)`/`SetRaw(worldID, ...)` вызывают `GetItem(worldID)` напрямую, минуя `_mapping`. При worldID больше числа элементов — IndexOutOfRangeException (например, из DebuggerProxy `AllWorldComponents`); при несовпадении порядка выдачи индексов — чтение/запись компонента **чужого мира**.
- **Тип:** State Corruption + Uncaught Exception
- **Fix:** `GetItem(GetItemIndex(worldID))` в обоих методах.

### 2.5. `EcsWorld.Destroy`: `IEcsWorldComponent.OnDestroy` получает `world == null`

- **Файл:** `src/EcsWorld.cs:366-367` + `src/EcsWorld.static.cs:308`
- **Сценарий:** `Destroy()` сначала выполняет `_worlds[ID] = null;`, затем `ReleaseData(ID);`. `Release` передаёт `_worlds[worldID]` (уже null) вторым аргументом в `OnDestroy`. Пользовательский world-компонент, обращающийся к миру в `OnDestroy` (отписка слушателей и т.п.), падает с NullReferenceException. Встроенные кэши выживают лишь потому, что игнорируют параметр.
- **Тип:** Uncaught Exception
- **Fix:** вызывать `ReleaseData(ID)` до обнуления `_worlds[ID]` (либо передавать сохранённую ссылку).

### 2.6. ABA переиспользования world ID ломает `entlong.IsAlive`

- **Файл:** `src/entlong.cs:58` + `src/EcsWorld.cs:863-867`
- **Сценарий:** мир A получает ID 1, вырастает до capacity 2048, выдаёт `entlong` с id 1500; `A.Destroy()` освобождает ID 1; новый мир B (capacity 512) получает ID 1. `e.IsAlive` → `TryGetWorld(1)` успешен (возвращает B) → `B._entities[1500]` → IndexOutOfRangeException. При меньшем id и случайно совпавшей генерации — молчаливый ложный `true` для чужой сущности.
- **Тип:** Uncaught Exception + Logic Flaw
- **Fix:** в `IsAlive(int, short)` проверять `entityID < _entitiesCapacity`; системно — не переиспользовать world ID немедленно или включать epoch мира в entlong.

### 2.7. `WorldComponentPool<T>`: `_count` никогда не декрементируется

- **Файл:** `src/EcsWorld.static.cs:255-263, 291-318`
- **Сценарий:** `_count` — одновременно вершина аллокатора (`itemIndex = ++_count`) и инкрементируется в recycled-ветке, но `Release` его не уменьшает. Цикл создания/уничтожения миров: после N циклов `_count = N+1` при одном живом элементе; когда понадобится действительно новый индекс — `Array.Resize(_items, NextPow2(_count))`, слоты между пустуют навсегда.
- **Последствия:** статический массив `_items` для каждого типа T растёт пропорционально общему числу когда-либо созданных миров, а не максимуму одновременных.
- **Тип:** Memory Leak
- **Fix:** не инкрементировать `_count` в recycled-ветке (или декрементировать в `Release`).

---

## 3. Критические: пулы компонентов

### 3.1. Все события пулов мертвы: `[Conditional]` + file-scoped `#define`

- **Файл:** `src/Pools/EcsPoolBase.cs:4-8` (определение) и `:386-440` (методы Invoke*); вызовы в `src/Pools/EcsPool.cs` и `src/Pools/EcsTagPool.cs`
- **Сценарий:** `#define DRAGONECS_ENABLE_POOLS_EVENTS` в C# действует **только в пределах файла** (`EcsPoolBase.cs`), а методы `InvokeOnAdd/InvokeOnAddAndGet/InvokeOnGet/InvokeOnDel` помечены `[Conditional("DRAGONECS_ENABLE_POOLS_EVENTS")]`. `[Conditional]` удаляет вызов, если символ не определён **в месте вызова** — а в `EcsPool.cs`/`EcsTagPool.cs` он не определён; на уровне проекта тоже (в `DragonECS.csproj` нет DefineConstants, в `DragonECS.asmdef` versionDefines пуст).
- **Последствия:** компилятор вырезает все вызовы событий. `AddListener` работает, но ни один колбек (`OnAdd`/`OnGet`/`OnDel`) никогда не вызывается — вся инфраструктура listeners мёртвая.
- **Тип:** Logic Flaw
- **Fix:** убрать `[Conditional]` и обернуть тела методов в `#if !DRAGONECS_DISABLE_POOLS_EVENTS`; либо добавлять `#define` (по тому же паттерну) в начало каждого файла с вызовами.

### 3.2–3.4. Copy-paste: `RemoveListener` вызывает `AddListener`

- **Файлы:** `src/Pools/EcsPool.cs:760`, `src/Pools/EcsTagPool.cs:524`, `src/Pools/EcsValuePool.cs:778`
- **Сценарий:** у readonly-обёрток всех трёх пулов:

```csharp
public void RemoveListener(IEcsPoolEventListener listener) { _pool.AddListener(listener); }
```

  Подписка не снимается, а добавляется второй раз: утечка ссылки на listener + двойные колбеки (после исправления 3.1). В `EcsValuePool` оба метода — no-op с предупреждением, но пользователь получает неверное предупреждение.
- **Тип:** Memory Leak / Logic Flaw
- **Fix:** `_pool.RemoveListener(listener);`

### 3.5. `EcsValuePool(capacity <= 1)`: буфер не растёт → исключение / порча unmanaged-кучи

- **Файл:** `src/Pools/EcsValuePool.cs:147` (проявляется в `Add`, :258-272)
- **Сценарий:** конструктор использует `ArrayUtility.NextPow2(capacity)` без минимума, в отличие от `EcsPool` (`CeilPow2Safe`, min 4). `NextPow2(1) == CeilPow2(1|1) == 1` → `_itemsLength = 1` (единственный слот — фейковый `_items[0]`). Первый `Add`: `itemIndex = 1 >= _itemsLength(1)` → «ресайз», но `capacity = NextPow2(1) = 1` — рост нулевой → `_dense.AsSpan().Slice(1, 1)` на span длиной 1 → ArgumentOutOfRangeException; без этого исключения запись `_items[1]` вышла бы за границы unmanaged-буфера (порча кучи).
- **Тип:** Uncaught Exception (+ риск порчи нативной памяти)
- **Fix:** `capacity = ArrayUtility.CeilPow2Safe(capacity);` как в `EcsPool`.

---

## 4. Критические: маски, аспекты, executors

### 4.1. `EcsStaticMask.IsSubarray` всегда возвращает true

- **Файл:** `src/EcsStaticMask.cs:253-271`
- **Сценарий:** `subI` инкрементируется каждую итерацию безусловно, `superI` — только на совпадении. При гарантии `super.Length >= sub.Length` (проверяется на входе) и `superI <= subI` цикл всегда завершается с `subI == sub.Length` → `true`. Пример: `[1,3].IsSubmaskOf([2])`… точнее, любые несовпадающие массивы проходят.
- **Последствия:** `IsSubmaskOf`/`IsSupermaskOf` (и обёртки в `EcsMask.cs:211-218`) вырождаются в сравнение длин — содержимое inc/any не проверяется, ложные `true`.
- **Тип:** Logic Flaw
- **Fix:** алгоритм как в соседнем корректном `IsSuperarray`: на несовпадении продвигать `superI`, на совпадении — `subI`:

```csharp
while (superI < super.Length && subI < sub.Length)
{
    if (super[superI] == sub[subI]) { subI++; }
    superI++;
}
return subI == sub.Length;
```

### 4.2. `WorldStateVersionsChecker.Check()`: инвертированное сравнение версий

- **Файл:** `src/Executors/MaskQueryExecutor.cs:199-207`
- **Сценарий:** при `_world.Version != снапшота` цикл делает `if (_versions[i] == slots[...].version) return false;` — возвращает «кэш невалиден», когда версия пула **не изменилась**, и «валиден», когда все отслеживаемые пулы изменились.
- **Последствия:** свойство `IsCached` у `EcsWhereExecutor`/`EcsWhereToGroupExecutor` даёт противоположный ответ.
- **Тип:** Logic Flaw
- **Fix:** `if (_versions[i] != slots[_componentIDs[i]].version) { return false; } ... return true;` (плюс паритет с `CheckAndNext` по `_isNotOnlyExc`).

### 4.3. `EcsStaticMask.Builder.Cancel` не инкрементирует версию билдера

- **Файл:** `src/EcsStaticMask.cs:473-477`
- **Сценарий:** `Cancel()` возвращает `BuilderInstance` в пул без `_version++` (в отличие от `Build()`). Следующий `EcsStaticMask.New()` получает тот же экземпляр с той же версией; старая копия билдера всё ещё проходит проверку `_version != _builder._version` → две «независимые» обёртки мутируют общее состояние. Двойной `Cancel` кладёт экземпляр в пул дважды.
- **Тип:** State Corruption
- **Fix:** инкрементировать `instance._version` при возврате в пул (в `Cancel`/`ReturnToPool`).

### 4.4. `BuilderInstance.Clear` не чистит `_combineds`/`_excepteds`

- **Файл:** `src/EcsStaticMask.cs:512-517`
- **Сценарий:** `EcsStaticMask.New().Combine(someMask).Cancel();` — `Clear()` чистит только `_incsSet/_excsSet/_anysSet`; `_combineds`/`_excepteds` очищаются только внутри `Build()`. Следующий билдер из пула молча вливает `someMask` в чужую маску.
- **Тип:** State Corruption
- **Fix:** добавить `_combineds.Clear(); _excepteds.Clear(); _sortedCombinedChecker = true;` в `Clear()`.

### 4.5. Executors в STABILITY_MODE: нет `return` + NRE до создания группы

- **Файл:** `src/Executors/EcsWhereExecutor.cs:121-124`, `src/Executors/EcsWhereToGroupExecutor.cs:134-139`
- **Сценарий:** сборка с `DRAGONECS_STABILITY_MODE`. При null-span / span чужого мира код обнуляет счётчик/группу, но не делает `return` и проваливается в `CacheTo` с невалидным span. В `EcsWhereToGroupExecutor` вдобавок `_filteredGroup.Clear()` вызывается **до** проверки `if (_filteredGroup == null)` → NullReferenceException при первом вызове.
- **Тип:** Uncaught Exception / Logic Flaw
- **Fix:** `return;` после обработки невалидного span; создавать `_filteredGroup` до `Clear()`.

### 4.6. `EcsRunRunner`: в release-сборке `RunFinally` никогда не вызывается

- **Файл:** `src/Builtin/BaseProcesses.cs:215-226`
- **Сценарий:** система реализует `IEcsRun + IEcsRunFinally`. В DEBUG-ветке `pair.cleanup?.RunFinally()` вызывается в `finally`; в `#else`-ветке (release) — `finally { }` пустой.
- **Последствия:** cleanup-логика молча пропадает в продакшене; поведение debug и release расходится.
- **Тип:** Logic Flaw
- **Fix:** в release-ветке строить те же пары и вызывать `item.cleanup?.RunFinally()`.

### 4.7. `CacheTo(ref int[])`: ресайз нулевого массива в ноль

- **Файл:** `src/EcsMask.cs:1053-1066, 1224-1237`
- **Сценарий:** публичный вызов с `Array.Empty<int>()` при хотя бы одном совпадении: `array.Length <= count` (0 <= 0) → `Array.Resize(ref array, 0 << 1)` → длина остаётся 0 → `array[0]` → IndexOutOfRangeException.
- **Тип:** Uncaught Exception
- **Fix:** `Array.Resize(ref array, array.Length == 0 ? 4 : array.Length << 1);`

### 4.8. `EcsAspect.Builder.New`: нет try/finally вокруг пользовательского `Init`

- **Файл:** `src/EcsAspect.cs:270-333`
- **Сценарий:** исключение из `Init(Builder)` аспекта или инициализатора поля (реально в DEBUG: `Throw.ConstraintIsAlreadyContainedInMask` при дублирующем ограничении) — `_constructorBuildersStackIndex--` не выполняется. Thread-static индекс навсегда остаётся >= 0: статические `B`/`CurrentBuilder` вне инициализации проходят проверку и возвращают протухший билдер со старым `_world` и неотданным `_maskBuilder`.
- **Последствия:** молчаливая порча последующих аспектов вместо ожидаемого исключения `Aspect_CanOnlyBeUsedDuringInitialization`.
- **Тип:** State Corruption
- **Fix:** `try { ... } finally { _constructorBuildersStackIndex--; }` + отмена неотданного `_maskBuilder`.

### 4.9. Двойная инициализация executor в `WhereQueryCache.Init` (утечка)

- **Файл:** `src/EcsWorld.cache.cs:73-75`
- **Сценарий:** `GetExecutorForMask` уже создаёт и **инициализирует** executor; следом `instance.Initialize(world, mask)` вызывается второй раз. `OnInitialize` в `EcsWhereExecutor` создаёт второй `WorldStateVersionsChecker` — unmanaged-память первого утекает навсегда (struct без финализатора); в `EcsWhereToGroupExecutor` создаётся вторая `_filteredAllGroup`, первая брошена без Dispose.
- **Тип:** Memory Leak / State Corruption
- **Fix:** убрать повторный `instance.Initialize(world, mask)`.

### 4.10. Неинициализированный аспект → NRE без диагностики

- **Файл:** `src/EcsAspect.cs:513` + `src/Executors/MaskQueryExecutor.cs:27-33`
- **Сценарий:** `var a = new MyAspect();` (напрямую, без `world.GetAspect`) и использование как `IComponentMask`: `ToMask` возвращает `_mask == null` → `executorCore.Initialize(this, null)` → `new WorldStateVersionsChecker(null)` → NullReferenceException глубоко в стеке. Аналогично `a.IsMatches(e)` (`_source == null`).
- **Тип:** Uncaught Exception
- **Fix:** проверять `_isBuilt` в `ToMask`/`IsMatches` и бросать осмысленное исключение.

---

## 5. Пайплайн, раннеры, инъекции

### 5.1. `Builder.MergeWith` итерирует себя вместо `other`

- **Файл:** `src/EcsPipeline.Builder.cs:284`
- **Сценарий:** `builderA.Add(builderB)` (Builder реализует IEcsModule → `Import` → `into.MergeWith(this)`). `MergeWith(other)` итерирует `_systemNodes, _systemNodesCount, _startIndex` — связный список **себя**.
- **Последствия:** все системы `builderB` теряются, системы `builderA` дублируются.
- **Тип:** Logic Flaw
- **Fix:** `new LinkedListCountIterator<SystemNode>(other._systemNodes, other._systemNodesCount, other._startIndex)`.

### 5.2. `Remove<TSystem>()` пропускает головной узел и узел после удалённого

- **Файл:** `src/EcsPipeline.Builder.cs:327-337`
- **Сценарий:** (а) головной узел (`_startIndex`) никогда не проверяется: `Add(A).Add(B).Remove<A>()` — A остаётся. (б) после `RemoveAt` итерация перескакивает на следующий узел без проверки, а счётчик цикла (`_systemNodesCount`) уже сократился: `Add(A).Add(B1).Add(B2).Remove<B>()` — B2 остаётся.
- **Тип:** Logic Flaw
- **Fix:** проверять `_startIndex` отдельно (с переносом головы); после удаления не продвигать `enumIndex`; не завязывать цикл на убывающий счётчик.

### 5.3. `Remove<T>` при единственном узле не декрементирует `lazyInitSystemsCount`

- **Файл:** `src/EcsPipeline.Builder.cs:318-324` (проявляется в `Build()`, :363-410)
- **Сценарий:** `New().Add(A).Remove<A>().Build()` — ветка `_systemNodesCount <= 1` обнуляет счётчик узлов, но не `_layerLists[...].lazyInitSystemsCount` (в отличие от `RemoveAt`). `allSystemsLength` на 1 больше реального → в `_allSystems` хвостовой `null` → NullReferenceException при инициализации инъекций.
- **Тип:** Uncaught Exception / State Corruption
- **Fix:** в этой ветке уменьшать счётчик слоя аналогично `RemoveAt`.

### 5.4. `Add` объекта `IEcsProcess + IEcsModule` в пустой builder → IndexOutOfRange

- **Файл:** `src/EcsPipeline.Builder.cs:119-123, 178`
- **Сценарий:** первый `Add()` в пустой builder объекта, реализующего оба интерфейса, чей `Import` добавляет хотя бы одну систему: `importHeadIndex = _endIndex = -1`; после импорта `InsertAfterNode_Internal(-1, ...)` читает `_systemNodes[-1].next` → IndexOutOfRangeException.
- **Тип:** Uncaught Exception
- **Fix:** обрабатывать `insertAfterIndex == -1` как вставку в голову списка.

### 5.5. `GenerateSerializableTemplate` не заполняет `layers`

- **Файл:** `src/EcsPipeline.Builder.cs:643`
- **Сценарий:** `result.layers = new string[Layers.Count];` — массив никогда не заполняется (все элементы null). Порядок слоёв теряется; при импорте такого шаблона `Layers.MergeWith(layers)` → `GetVertexID(null)` → ArgumentNullException.
- **Тип:** Logic Flaw / Uncaught Exception
- **Fix:** заполнить массив перечислением `Layers`.

### 5.6. `RunHelperWithFinally.Run`: отсутствует null-check в release

- **Файл:** `src/EcsRunner.cs:440-443`
- **Сценарий:** кастомный раннер, среди систем есть не реализующая `TProcessFinally` (`runFinally == null`). В release-пути `finally` вызывает колбек с null → NullReferenceException каждый кадр. В DEBUG-пути и в перегрузке `Run<TData>` проверка есть.
- **Тип:** Uncaught Exception
- **Fix:** `if (item.runFinally != null)` как в остальных трёх путях.

### 5.7. `Injector.Inject<Base>` после `Inject<Derived>` того же runtime-типа не создаёт узел Base

- **Файл:** `src/Injections/Injector.cs:101-116`
- **Сценарий:** проверка `_nodes.ContainsKey(tType)` находится внутри ветки «ветка по runtime-типу не найдена». Второй вызов с тем же runtime-типом объекта (`Inject<IService>(impl); Inject(impl)` или наоборот) пропускает создание узла для `typeof(T)`.
- **Последствия:** системы `IEcsInject<Base>` молча не получают зависимость; `Extract<Base>()` бросает NodeNotFound; в DEBUG init падает с `Injection_RequiredNodeNotFound`.
- **Тип:** Logic Flaw
- **Fix:** вынести создание узла из ветки `if (branch == false)`.

### 5.8. Раннеры получают каждую инъекцию дважды

- **Файл:** `src/Injections/Graph/InjectionNode.cs:51-54` + `src/EcsPipeline.cs:283-284`
- **Сценарий:** каждый раннер регистрируется в `_runners` под двумя ключами (тип раннера и тип интерфейса). `InjectionNode<T>.Inject` итерирует словарь и вызывает `ExtractTo_Internal(runner.Value)` — каждый раннер с `IEcsInject<T>` получает инъекцию дважды.
- **Последствия:** для неидемпотентных Inject (накопление в список) — дублирование данных.
- **Тип:** Logic Flaw
- **Fix:** итерировать уникальные значения (пропускать записи, где `Key != Value.Interface`).

### 5.9. Финализатор `EcsPipeline` исполняет `Init()`/`Destroy()` на потоке GC

- **Файл:** `src/EcsPipeline.cs:220-225`
- **Сценарий:** построенный, но не инициализированный пайплайн становится недостижим → финализатор вызывает `Init()` + `Destroy()`: пользовательские `PreInit/Init/Destroy` систем исполняются на потоке финализатора — обращения к уже финализированным объектам (миры, Unity-объекты) → NRE/крэш; гонка с основным потоком.
- **Тип:** Race Condition / Uncaught Exception
- **Fix:** не исполнять жизненный цикл в финализаторе (максимум — предупреждение в лог).

### 5.10. Два раннера с одним интерфейсом → ArgumentException и рассогласованный словарь

- **Файл:** `src/EcsPipeline.cs:283-284`
- **Сценарий:** `AddRunner<MyCustomRunRunner>()` (Interface = IEcsRun), затем `Init()` → внутренний `GetRunnerInstance<EcsRunRunner>()`: по ключу типа не найден → создаётся новый → второй `_runners.Add(IEcsRun, ...)` бросает ArgumentException, причём первый `Add(runnerType)` уже выполнен — словарь рассогласован.
- **Тип:** Uncaught Exception / State Corruption
- **Fix:** проверять занятость ключа интерфейса до мутации словаря; переиспользовать существующий раннер или бросать осмысленную ошибку.

### 5.11. `DependencyGraph.MergeWith`: слот 0 (null) + отсутствие `return`

- **Файл:** `src/Utils/DependencyGraph.cs:235-257`
- **Сценарий:** типизированная ветка (`other is DependencyGraph<T>`) итерирует все vertexInfo с i=0, включая служебный слот 0 (`value == null` для ссылочного T) → `Dictionary.TryGetValue(null)` → ArgumentNullException. После ветки нет `return` — зависимости добавились бы второй раз общим путём.
- **Тип:** Uncaught Exception / Logic Flaw
- **Fix:** начинать с i=1, пропускать `isContained == false`, добавить `return` после типизированной ветки.

### 5.12. Off-by-one в энумераторе `DependencyGraph`

- **Файл:** `src/Utils/DependencyGraph.cs:545-556`
- **Сценарий:** `if (_index++ >= Count) return false;` проверяет старое значение, затем `GetVertexInfo(_index)` — на последней итерации читается `_items[Count]`. Пока capacity > Count — читается мусор; когда число vertexInfo точно равно ёмкости StructList (32, 64, ...) — IndexOutOfRangeException при любом `foreach` по `Layers` (в т.ч. в `Builder.MergeWith`).
- **Тип:** Uncaught Exception
- **Fix:** `if (++_index >= count) { return false; }` до доступа к элементу.

### 5.13. `Before/After` на несуществующий слой молча ломает автопривязку

- **Файл:** `src/Utils/DependencyGraph.cs:223-231` (+ `src/Utils/LayersMap.cs:153-167`)
- **Сценарий:** `Layers.Add("X").Before("Опечатка")` — `AddDependency` безусловно ставит `hasAnyDependency = true` для обеих вершин, но само ребро отбрасывается в топосортировке (вторая вершина не contained). X теряет автопривязку к basic-слою (она делается только при `hasAnyDependency == false`) и молча сортируется в крайнюю позицию.
- **Тип:** Logic Flaw
- **Fix:** учитывать `hasAnyDependency` только по рёбрам между contained-вершинами либо валидировать существование целевого слоя.

---

## 6. Внутренние структуры данных

### 6.1. `StructList.RemoveAtWithOrder`: чтение за границей и `_count` не уменьшается

- **Файл:** `src/Internal/StructList.cs:124-133`
- **Сценарий:**

```csharp
for (int i = index; i < _count;)
{
    _items[i++] = _items[i];
}
```

  Последняя итерация выполняет `_items[_count-1] = _items[_count]` — чтение за логическим концом (мусор в последний слот; при `_count == _items.Length` — IndexOutOfRangeException). Кроме того `_count` **не декрементируется** — элемент логически не удалён.
- **Достижимость:** `pool.RemoveListener` → `_listeners.RemoveWithOrder(listener)` (`EcsPool.cs:618`, `EcsTagPool.cs:389`) — список слушателей сохраняет старый размер со stale/default-слотом → NRE или вызов протухшего слушателя.
- **Тип:** State Corruption + Uncaught Exception
- **Fix:** сдвигать до `_count - 1`, затем `_count--` (+ занулить хвост для managed T).

### 6.2. `HMem<T>.As<U>`: инвертированное условие

- **Файл:** `src/Internal/Allocators/MemoryAllocator.cs:336-341`
- **Сценарий:** `if (IsCreated) { return default; }` — для валидного буфера возвращается пустой хэндл (null-указатель, длина 0); для дефолтного — продолжаются вычисления на null Handler.
- **Тип:** State Corruption
- **Fix:** `if (IsCreated == false) { return default; }`

### 6.3. `Marshal.SizeOf<T>()` вместо `sizeof(T)` → недоаллокация для `char`

- **Файл:** `src/Internal/Allocators/TempBuffer.cs:50`, `src/Internal/Allocators/MemoryAllocator.cs:53, 70`
- **Сценарий:** `Marshal.SizeOf<char>() == 1` (ANSI marshal size), а `sizeof(char) == 2`. `MetaIDAttribute.ParseIDFromTypeName` (`MetaIDAttribute.cs:116`) вызывает `TempBuffer<MetaIDAttribute, char>.Get(name.Length)` → аллоцируется `name.Length * 1` байт, записывается `2 * name.Length` байт → запись ~name.Length байт **за пределами** аллокации, затем `new string(buffer, ...)` читает там же. Для `bool` — переаллокация 4x (ломается семантика Length/AsSpan).
- **Тип:** Buffer Overflow
- **Fix:** везде использовать `sizeof(T)`.

### 6.4. `MemoryAllocator.Free(null)` → порча нативной кучи

- **Файл:** `src/Internal/Allocators/MemoryAllocator.cs:233-244`; потребитель: `src/Internal/UnsafeArray.cs:98-103`
- **Сценарий:** `Free(null)` вычисляет `((Meta*)null) - 1` — адрес `-sizeof(Meta)`. DEBUG-проверка `handledPtr == null` ложна → разыменование `handledPtr->ID` → AV. В release проверки нет вовсе: `Marshal.FreeHGlobal` по мусорному адресу → порча кучи. Тривиально достижимо: `default(UnsafeArray<T>).Dispose()`, `UnsafeArray<T>.Empty.Dispose()`, двойной `Dispose()`.
- **Тип:** State Corruption / Uncaught Exception
- **Fix:** ранний `return` при `dataPtr == null` в `Free(void*)`.

### 6.5. Пустая структура `Meta` в release → все аллокации смещены на 1 байт

- **Файл:** `src/Internal/Allocators/MemoryAllocator.cs:274-280` (использование: :90, :172-174, :400)
- **Сценарий:** в non-DEBUG `Meta` без полей, но `sizeof(пустая структура) == 1`. Дата-указатель — `base + 1`: каждый `T*`, выдаваемый аллокатором в release, невыровнен на 1 байт (все `HMem<T>`, `UnsafeArray<T>`).
- **Последствия:** на x86/x64 — замедление; на alignment-строгих таргетах (IL2CPP ARM32, Mono ARM, atomics/Interlocked поверх этой памяти) — фолт или tearing.
- **Тип:** State Corruption (платформозависимая)
- **Fix:** фиксированный layout: `[StructLayout(LayoutKind.Sequential, Size = 8)]` либо держать поля во всех конфигурациях.

### 6.6. `SparseArray.Resize` хеширует через `%` вместо `&`

- **Файл:** `src/Internal/SparseArray.cs:208`
- **Сценарий:** lookup/insert/remove используют `key & _modBitMask` (всегда неотрицательный), а `Resize()` — `hashKey % newSize` (знак сохраняется в C#). Отрицательный ключ (класс публичный; `(keyX << 16) | keyY` с keyX >= 0x8000 даёт отрицательный ключ) → на ресайзе `newBuckets[-5]` → IndexOutOfRangeException; без падения — бакеты `%` расходятся с lookup `&`.
- **Тип:** Uncaught Exception / State Corruption
- **Fix:** `newEntries[i].hashKey & _modBitMask`.

### 6.7. `SparseArray.Clear` не сбрасывает freelist

- **Файл:** `src/Internal/SparseArray.cs:177-188`
- **Сценарий:** `Add(k1); Add(k2); Remove(k1); Clear(); Add(k3); Add(k4);` — `Clear` не сбрасывает `_freeList/_freeCount`. `Add(k3)` берёт слот 0 из freelist (не увеличивая `_count`), `Add(k4)` берёт `index = _count++ = 0` — **перезаписывает** запись k3; `entry.next` замыкается сам на себя → `FindEntry` любого отсутствующего ключа этого бакета зацикливается навсегда.
- **Тип:** Infinite Loop / State Corruption
- **Fix:** `_freeList = 0; _freeCount = 0;` в `Clear()` (и чистить entries при `_count == 0 && _freeCount > 0`).

### 6.8. `SparseArray.Count` — high-water mark, а не число элементов

- **Файл:** `src/Internal/SparseArray.cs:46`
- **Сценарий:** `Add(a); Add(b); Remove(a);` → `Count == 2`. `EcsMask` использует `_staticMasks.Count` как следующий ID маски (`EcsMask.cs:363/413`) — при Remove-до-Add возможны дубликаты ID.
- **Тип:** State Corruption
- **Fix:** `return _count - _freeCount;`

### 6.9. Энумератор `AppendOnlyTable` пропускает слот 0

- **Файл:** `src/Internal/AppendOnlyTable.cs:80, 105-119`
- **Сценарий:** `Provider.GetEnumerator()` возвращает `new Enumerator()` — `_index` зануляется (0, а не −1); `MoveNext` делает `while (++_index < _capacity)` — первый проверяемый слот — 1, слот 0 никогда не посещается.
- **Последствия:** `EcsTypeCodeManager.FindTypeOfCode`/`GetDeclaredTypes` и обход кэша TypeMeta молча теряют одну запись (тип, чей probe попал в слот 0).
- **Тип:** State Corruption (тихая потеря элемента)
- **Fix:** использовать конструктор, ставящий `_index = -1`.

### 6.10. `EcsTypeCodeManager`: читатели без блокировки при Resize под локом

- **Файл:** `src/Internal/EcsTypeCodeManager.cs:43-55` + `src/Internal/AppendOnlyTable.cs:349-373`
- **Сценарий:** `Get(Type)` мутирует общую статическую таблицу под `_lock`, но `Has()`, `FindTypeOfCode()`, `GetDeclaredTypes()`, `Count` читают те же `_keys/_occupied/_capacity/_mask` без лока. `Resize()` публикует `_capacity/_mask` **до** аллокации новых массивов — конкурентный читатель видит новую маску со старыми (меньшими) массивами → IndexOutOfRangeException или ложный «not found».
- **Тип:** Race Condition
- **Fix:** брать `_lock` в читателях либо публиковать состояние атомарно (одна volatile-ссылка на иммутабельный снапшот).

---

## 7. DebugUtils (диагностика; проявляется при использовании отладочных API)

### 7.1. `FindRootTypeMeta`: бесконечный цикл

- **Файл:** `src/DebugUtils/TypeMeta.cs:36`
- **Сценарий:** `while (result.BaseMeta != null) { result = meta.BaseMeta; }` — присваивается `meta.BaseMeta` (первый уровень) вместо `result.BaseMeta`. Цепочка BaseMeta глубиной >= 2 → зависание.
- **Тип:** Infinite Loop
- **Fix:** `result = result.BaseMeta;`

### 7.2. `TypeMeta`: ленивая инициализация без синхронизации

- **Файл:** `src/DebugUtils/TypeMeta.cs:186-331` (все ленивые геттеры)
- **Сценарий:** `TypeMeta.Get()` защищён локом, но возвращённый экземпляр шарится между потоками, а ленивые `InitName()/InitColor()/...` — нет: неатомарный `_initFlags |= ...` теряет флаги при конкуренции; на слабых моделях памяти (ARM/IL2CPP) флаг может стать видимым раньше данных → чужой поток читает `_name == null`.
- **Тип:** Race Condition
- **Fix:** инициализация под lock либо volatile-публикация (данные до флага) с атомарным обновлением флагов.

### 7.3. `IsHasCustomMeta`: нет guard'ов для generic-прокси

- **Файл:** `src/DebugUtils/TypeMeta.cs:475-478`
- **Сценарий:** тип с `[MetaProxy(typeof(Proxy<>))]` (open-generic): конструктор TypeMeta имеет guard (`ContainsGenericParameters` + `MakeGenericType` + `as`), а здесь прямой `Activator.CreateInstance` → ArgumentException; прямой каст `(MetaProxyBase)` → InvalidCastException.
- **Тип:** Uncaught Exception
- **Fix:** повторить guard из конструктора.

### 7.4. `InitializeAll` не инициализирует `ReflectionInfo`

- **Файл:** `src/DebugUtils/TypeMeta.cs:398-411`
- **Сценарий:** метод дергает Name/Group/Color/Description/Tags/MetaID/TypeCode, но не IsComponent/IsProcess/IsPool, при этом `InitFlag.All` включает `ReflectionInfo` → `_initFlags != InitFlag.All` навсегда, повторные вызовы всегда перевыполняют работу.
- **Тип:** Logic Flaw
- **Fix:** добавить `_ = IsComponent;`

### 7.5. `EcsDebug.RegisterMark`: чтение словаря вне лока

- **Файл:** `src/DebugUtils/EcsDebug.cs:333-351`
- **Сценарий:** первый `_nameIdTable.TryGetValue` выполняется без `_lock`, пока другой поток внутри лока делает `Add` (с resize словаря) — конкурентное чтение Dictionary во время записи: исключение/зависание/мусорный id.
- **Тип:** Race Condition
- **Fix:** убрать незащищённую проверку либо ConcurrentDictionary.

### 7.6. `EcsDebug.DeleteMark` вызывает на клонах `OnNewProfilerMark`

- **Файл:** `src/DebugUtils/EcsDebug.cs:352-365`
- **Сценарий:** в цикле по thread-клонам сервиса вызывается `OnNewProfilerMark(id, name)` вместо `OnDelProfilerMark(id)` — клоны заново регистрируют удаляемый маркер; при переиспользовании id через `_idDispenser` в клонах остаются протухшие данные.
- **Тип:** Logic Flaw
- **Fix:** вызывать `service.OnDelProfilerMark(id)`.

### 7.7. `TryGetTags`: инвертированный результат

- **Файл:** `src/DebugUtils/EcsDebugUtility.cs:253, 259, 265`
- **Сценарий:** `return tags.Count <= 0;` — для типа с тегами возвращается false, без тегов — true.
- **Тип:** Logic Flaw
- **Fix:** `return tags.Count > 0;`

### 7.8. `GetGenericTypeFullName`: NRE и неверная обрезка вложенных generic

- **Файл:** `src/DebugUtils/EcsDebugUtility.cs:39-47`
- **Сценарий:** (а) для сконструированного generic, содержащего generic-параметры, `type.FullName == null` → NRE на `typeName.LastIndexOf('`')`. (б) для вложенных закрытых generic (`List<List<int>>`) `LastIndexOf` находит бэктик внутреннего аргумента → искалеченное имя.
- **Тип:** Uncaught Exception / Logic Flaw
- **Fix:** null-fallback на `type.Name`; `IndexOf` вместо `LastIndexOf`.

### 7.9. `JsonDebugger`: пакет дефектов (невалидный JSON / падения)

- **Файл:** `src/DebugUtils/JsonDebugger.cs`
- Между членами объекта не ставятся запятые (:216-276) → `{"Type": "X" "field1": 1}` — невалидный JSON. **Fix:** запятая перед каждым членом после первого.
- `del.Target.GetType()` → NRE для делегата на статический метод (:157-164). **Fix:** `del.Target?.GetType().FullName ?? ...`.
- `e.Message` пишется без `EscapeString` (:125-132) — кавычки/переводы строк ломают документ. **Fix:** экранировать.
- enum и `char` сериализуются без кавычек/экранирования (:91-105, :120-123) — `Flags`-enum с запятой рвёт структуру. **Fix:** кавычки + EscapeString.
- `_indentsChache` — static List без синхронизации (:11, :22-30): конкурентные `PrintJson` портят список. **Fix:** `[ThreadStatic]` или lock.
- Рекурсия по struct-свойствам не ограничена (visited учитывает только reference types; :170-182, :242-271): свойство, возвращающее тот же struct-тип (типовой пример — `Vector3.normalized`) → StackOverflowException. **Fix:** maxDepth / учёт value types в стеке обхода.
- Не-строковая `IEnumerable<char>` (например `char[]`) сериализуется как имя типа через `ToString()` (:58-68). **Fix:** `new string(chars.ToArray())`.
- **Тип:** Logic Flaw / Uncaught Exception / Race Condition / Infinite Loop

### 7.10. `MetaColor.ParseHex`: перепутанная валидация

- **Файл:** `src/DebugUtils/MetaAttributes/MetaColorAttribute.cs:264-280`
- **Сценарий:** `if (hex[0] != '#' && hex.Length != 7 && hex.Length != 9)` — бросает только когда неверно всё сразу (`&&` вместо `||`). `Parse("#fff")` → ArgumentOutOfRangeException на `Slice(2, 2)`; `Parse("")` → IndexOutOfRangeException на `hex[0]`; 7-символьный мусор без `#` парсится молча.
- **Тип:** Uncaught Exception / Logic Flaw
- **Fix:** `if (hex.Length == 0 || hex[0] != '#' || (hex.Length != 7 && hex.Length != 9)) { throw; }`

### 7.11. `MetaDescription.ToString`: потерянная интерполяция

- **Файл:** `src/DebugUtils/MetaAttributes/MetaDescriptionAttribute.cs:47`
- **Сценарий:** `return $"[{Author}] Text";` — литеральная строка "Text" вместо `{Text}`.
- **Тип:** Logic Flaw
- **Fix:** `return $"[{Author}] {Text}";`

### 7.12. `MetaID.IsGenericID("")` → IndexOutOfRangeException

- **Файл:** `src/DebugUtils/MetaAttributes/MetaIDAttribute.cs:52-55`
- **Сценарий:** `id[id.Length - 1]` без guard на пустую строку. Дополнительно: за счёт `||` с regex `^[^,<>\s]*$` метод возвращает true и для любых не-generic id — семантика подозрительна.
- **Тип:** Uncaught Exception
- **Fix:** guard на null/empty; пересмотреть условие.

### 7.13. `ConvertIDToTypeName`/`ParseIDFromTypeName`: коллизии экранирования

- **Файл:** `src/DebugUtils/MetaAttributes/MetaIDAttribute.cs:103-136`
- **Сценарий:** `"_1"`, `"_2"`, `"_3"` — валидные подстроки ID — схлопываются в `"__"`, обратное преобразование даёт `"_"`: id `"A_1B"` → `"A__B"` → обратно `"A_B"`. Round-trip необратим; разные ID (`"A_1B"`, `"A_2B"`, `"A__B"`) коллидируют в одно имя типа.
- **Тип:** State Corruption
- **Fix:** однозначная схема экранирования с корректным декодером.

### 7.14. `AllowedInWorldsAttribute`: AllowMultiple + `GetCustomAttribute`

- **Файл:** `src/Utils/AllowedInWorldsAttribute.cs:9, 21`
- **Сценарий:** атрибут объявлен `AllowMultiple = true`, но читается через `GetCustomAttribute<T>()` — при двух атрибутах на типе → AmbiguousMatchException при первой же проверке; без исключения проверялся бы только один.
- **Тип:** Uncaught Exception
- **Fix:** `GetCustomAttributes<T>()` с объединением списков.

---

## 8. Краевые / неподтверждённые (логика дефектна, но сценарий требует нестандартных условий)

| # | Файл | Суть | Условие проявления |
|---|------|------|--------------------|
| 8.1 | `src/Collections/EcsGroup.cs:718-721` | `NextPow2(1) == 1` → `_dense[1]` → IndexOutOfRange | пользовательский `GroupCapacity` 0/1 (дефолт 512; валидации конфига нет) |
| 8.2 | `src/Collections/EcsGroup.cs:1381-1740` | set-операции: проверка размера через **негенерик** `ICollection` (который `HashSet<int>` не реализует) + двойной счёт дубликатов входа → ложные `SetEquals`/`IsSubsetOf`/`IsProperSupersetOf`; `SymmetricExceptWith` с дубликатами аннулирует элементы (:1276-1290) | вход `IEnumerable<int>` с дубликатами или `HashSet<int>` |
| 8.3 | `src/EcsWorld.static.cs:65-77` + `src/entlong.cs:257` | нет проверки отрицательного/завышенного `_world` → IndexOutOfRange | raw-каст `(entlong)long`, десериализация повреждённых данных |
| 8.4 | `src/EcsWorld.cs:1732-1804` | `return itemsCount;` вместо `return arrayIndex;` | только при уже нарушенном инварианте componentsCount |
| 8.5 | `src/EcsWorld.static.cs:247-289` | `_mapping[worldID]` публикуется до `Array.Resize` и `Init` → IOR/NRE у fast-path читателя | многопоточная работа с разными мирами одного типа компонента |
| 8.6 | `src/EcsMask.cs:269` | `IComponentMask.ToMask(world)` игнорирует параметр — маска мира A в запросе мира B читает чужие данные | кросс-мировое использование маски |
| 8.7 | `src/EcsStaticMask.cs:111-114` | `(_incs.Length & _excs.Length) == 1` — побитовое AND длин вместо логического условия; детект Broken несистемный | редкие сочетания длин |
| 8.8 | `src/EcsAspect.cs:411-435` | параметр `order` в `Combine/Except` полностью игнорируется (в `EcsStaticMask.Builder` не прокидывается; `_excepteds` не сортируется) | конфликты inc/exc при комбинировании аспектов |
| 8.9 | `src/Executors/MaskQueryExecutor.cs:46-64` | один executor под двумя ключами → дубликаты в `GetMaskQueryExecutors` | запрос той же маски по EcsMask и EcsStaticMask |
| 8.10 | `src/EcsPipeline.cs:323-326` | `OnRunnerDestroy_Internal` удаляет один ключ из двух → stale-раннер | в репозитории метод не вызывается (задел API) |
| 8.11 | `src/Internal/Allocators/TempBuffer.cs:23-31` | use-after-free при двух разных `T` на одном `TContext` (кэш `_ptr/_size` per-(TContext,T), буфер общий) | внешний код с двумя T на контекст (текущие вызовы — по одному) |
| 8.12 | `src/Internal/Allocators/AllocatorUtility.cs:73-91` | `Realloc(Length << 1)` при `Length == 0` не растёт → перезапись кучи | передача default-буфера (текущие вызовы стартуют с 32) |
| 8.13 | `src/Internal/Allocators/MemoryAllocator.cs:93-103, 176-185` | записи `_debugInfos` и `++_inrement` вне лока | только DEBUG-диагностика, многопоточность |
| 8.14 | `src/DataInterfaces.cs:14-31` | `EcsWorldComponent<T>` без `where T : struct`: для reference-type `default(T) == null` → кастомные Init/OnDestroy молча не вызываются | reference-type world-компонент |
| 8.15 | `src/DebugUtils/MetaAttributes/MetaGroupAttribute.cs:29, 76` | regex `"Module(?=/)"` вырезает подстроку в любом месте имени → `//`, пустые сегменты | имя группы содержит "Module" |
| 8.16 | `src/DebugUtils/MetaAttributes/MetaColorAttribute.cs:327-337` | `UpContrast` для ахроматического цвета возвращает `default` (a=0, прозрачный) вместо a=255 | автоцвет с r==g==b |
| 8.17 | `src/EcsMask.cs:876-929` | единственный кэшированный `EcsMaskIterator` на маску мутирует общие `_sort*Buffer` при каждом `GetEnumerator` | параллельные запросы по одной маске (если декларируется один поток на мир — не баг) |

---

## 9. Приоритеты для исправления

1. **Кластер `EcsGroup.Remove_Internal` (1.1–1.4)** — тривиальные Add/Remove портят и локальное состояние группы, и глобальную `_nullPage`, заражая все группы всех миров. Ядро всех запросов и set-операций.
2. **`ReleaseDelEntityBuffer` с частичным count (2.1)** — освобождение не тех сущностей, «призрачные» компоненты у новых entity.
3. **`IsSubarray` (4.1) и `WorldStateVersionsChecker.Check` (4.2)** — сравнение масок и валидность кэша executors возвращают инвертированные/константные результаты.
4. **Нативная память: `Marshal.SizeOf` (6.3), `Free(null)` (6.4), невыровненный `Meta` (6.5), `HMem.As` (6.2)** — переполнение буфера и порча кучи.
5. **События пулов (3.1) + `RemoveListener` (3.2–3.4)** — функциональность полностью мертва; после «оживления» вскроются баги listeners.
6. **`StructList.RemoveAtWithOrder` (6.1)** — задет тот же механизм listeners.
7. **Пулы билдеров масок (4.3, 4.4) и стек билдеров аспектов (4.8)** — тихая порча масок, самая труднодиагностируемая категория.

---

*Отчёт составлен автоматизированным ревью (Claude Code). Каждая находка верифицирована трассировкой кода конкретного сценария; номера строк соответствуют ревизии `95efd83`.*
