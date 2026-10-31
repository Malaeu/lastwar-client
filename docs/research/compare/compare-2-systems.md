# Сверка досье lastwar.tech с нашими находками: игровые системы

Источники сверки: их страницы в `~/.hermes/cache/scratch/lwtech/` (city-building, military-battle, alliance-duel, world-map, shop-iap, data-tables, выборочно entity-id-reference) против нашего локального декомпайла `~/.local/share/lastwar-collector/decompiled/lua/` (файлы `view.lua`, `qdm.lua`, `qi.lua`, `mgr/manager.lua`, `Net__Msgs__*`) и `Global__EnumType.luac` / `Global__ConstDefine.luac`. Все ссылки на наш код — file:line в этом дереве. Их утверждения проверялись по сериализаторам сообщений, а не по UI-логике.

## 1. Подтверждаем совпадающее

Их разделение двух «очередей» (`BuildQueueManager` против `QueueDataManager`) верно и критично. Наш код это подтверждает механически: строительство живёт в `BuildingDate` (`updateTime`, `IsUpgradeFinish`, BuildingDate.luac:439-442), а наука/лечение/тренировка — в общем `QueueDataManager`, где тип различает только `queue.type` (`NewQueueType.Science = 6`, `Hospital = 3`, `RebirthHospital = 117`, `T11Break = 120` — EnumType). Общая схема `queue_new[]` совпадает: `uuid, qid, type, sT, uT, itemObj, funcUuid, para, isHelped`.

Кадр старта науки у них описан верно по существу: `science.research.new` несёт `itemId, useGold, bUuid` и опциональный `speedGoods` (`itemId;count|itemId;count`), ни ресурсов, ни времени (`Net__Msgs__ScienceResearchNewMessage`). Наш обработчик ответа один в один: `errorCode` либо `resource, gold, goodsRest, rescource_item` (опечатка в самом поле, на проводе), `queue.itemCostArr`, `science` (`mgr/DataCenter__ScienceData__ScienceManager`). То есть тарифицирует и возвращает остатки сервер — их вывод про «сервер решает, UI только гасит кнопку» подтверждён.

`queue.finish` — только `uuid` (Long), без полей стоимости (`Net__Msgs__QueueFinishMessage`), `queue.finish.batch` — `queueList` (SFSArray длинных) + `paraState` (Int) и только для фермы/пастбищ. `free.building.upgrade.finish` — только `uuid`, отправка под `IsUpgradeFinish()` = `updateTime > 0 and now >= updateTime` (BuildingDate.luac:439-442), плюс побочный расход стамины через `BuildManager:SetCurStamina()` в самом сериализаторе.

Кодировка id: `id = base_type + level`, база кратна `BuildLevelCap = 1000`, `10100000 + lv = FUN_BUILD_MAIN` (`Global__ConstDefine`).

Слоты науки по лабораториям, до трёх: `FUN_BUILD_SCIENE`, `LW_BUILE_SCIENCE_TWO`, `LW_BUILE_SCIENCE_THREE`; `HasExtraQueue` = «научных очередей ≥ 2» (mgr/manager.lua:183-187) — параллельность определяется числом свободных лабораторий, а не серверным локом.

Просьба помощи по своей очереди — `al.call.help` с `qType = 6` (view.lua:1002), их поле `qType` совпадает.

Сбор с производственных линий: `GetNextCollectTime` возвращает `-1` без `productStartTime` (ProductLineManager.luac:287-299), клиент в этом случае показывает локальный 602026 и ничего не отправляет — их правило `prodST` верно.

Боевая авторитетность сервера (арена/марши/мир) и ~20 PVE/mini-game команд с self-report (`lw.save.pve.record.isWin`, `user.finish.pve.level.isSuccess`) — согласуется с нашими выводами по маршам и `push.world.march.*`; в этом прогоне перепроверял только их перечень команд, не сам бой.

## 2. Новое у них, чего у нас нет

- Ответ на `free.building.upgrade.finish` несёт `serverSystemTime` для ресинка часов клиента. Платные мгновенные финиши — отдельные `build.ccd.*` / `queue.ccd.*`.
- Точная форма ускорений (items-only): `build.ccd.m.new {bUUID, isFixRuins:false, itemIDs:"tmpl;n|..."}`, `queue.ccd.m.new {qUUID, itemIDs}`, `building.camp.accel {uuid, itemId}`. Алмазная форма = ключ `useGold` + пустой `itemIDs`; поля `isGold` на проводе нет.
- Диапазоны ускорителей (`goods`, `type2` = меню, `para3` = секунды): `200200-200207` универсальные, `200210-200217` стройка, `200220-200227` наука (8 градаций 60…432000 с), `200230-200235` тренировка, `200240-200245` лечение (6 градаций).
- Тип очков дуэли `51` привязан к очереди, которую ускоряют (`value`: 4 тренировка, 6 наука, 7 стройка, 9 лечение), а не к виду предмета; их фича `duel-speedups` отваливается, если в ответе нет `itemCostArr` или упал баланс алмазов.
- Чтение дуэли: `hero.event.info.get {activityId:"70000"}` (не `"55000"`, иначе голый `{success:true}`), активность `55000` типа 14, `type == 15` в ответе, клейм сундука `accept.personal.reward {actId, stage, eventType:15}`.
- Инфраструктура таблиц: CDN `hotupdate/table/table_{ver}_{md5}.data`, патчи BSDIFF40, `table_env=table_online` в check-version, split-таблицы через `link`-диапазоны, `_B`-варианты по `init.user.monopoly_ab`, 1 291 таблица, дифф 39432→39516 без изменения схем.
- `ub.store.upgrade {bUuid, resources{}, items{}}` — предзапас ресурсов под апгрейд; `worker.assign.all` / `assign.single.building`; `itemId` в `building.camp.training` — строка `"id;count|..."`, а не массив; семантика `status=1` и локали 602026 «Production in progress».
- Мир: `pointId = x + y*tileCount + 1`, упаковка `worldPos = serverId<<32 | pointId`, `world.get.block` до 160 индексов, `clearUuidSet` только в первом запросе, 3000×3000 на сезонных «девяти дворцах».
- Ловушка имён: `purchase.order.refresh/.finish/.delete` — не IAP, а очередь производства (заказы жителей/магазин).

## 3. Расхождения с нашими данными

1) В нашем скилле записано, что кадр старта науки несёт `robotUuid`. На проводе `robotUuid` в `ScienceResearchNewMessage` **нет вообще**: сериализатор пишет `itemId` (Int), `useGold` (Int), `bUuid` (Long) и опционально `speedGoods`. `robotUuid` есть только у `free.building.up.new`. Вызовы в UI передают `robotUuid = 0` — он теряется.
2) Там же: `bUuid` берётся не из здания, а из `queue.funcUuid` (view.lua:1039) — сервер ищет очередь по funcUuid, а не по uuid лаборатории из UI.
3) Наше «клиент отправляет `QueueFinish` только при `GetQueueState() == Finish`» — неполно. Автоотправку на истечении таймера делает `SetQueueState`, но **только для `NewQueueType.DragonHospital`** (qi.lua:100-125). Для науки финиш шлётся из `ScienceManager:CheckResearchFinish` / `CheckResearchFinishByBuildUuid` (mgr/manager.lua:120-139), а их зовёт только UI. Тихая до-истёкшая научная очередь сама не завершится.
4) `ScienceLevelCap` — **100**, а не как у зданий: `HasScienceById` разбирает `id % ScienceLevelCap` (mgr/manager.lua:156-160), `BuildLevelCap` = 1000. Складывать научные id по правилу зданий нельзя.
5) `free.building.up.new`: у них «uuid, upLevel, gold, workerUuid, robotUuid». У нас `uuid` — **UtfString** (не Long), и есть ещё `goldCost`, `clientParam`, `truckId`, `pathTime`, `workerUuid` (Long).
6) `useGold` (`ScienceResearchUseGold`): `NoUseGold = 0`, `UseGold = 1`, `Free = 2` (EnumType:3431-3435). Порядок не алфавитный и не очевидный — 2 это «бесплатно», а не «дороже».
7) Их «Tactical Center 10143000 — не производственная линия» верно, но для науки важнее другое: лаборатории — это `10123000 FUN_BUILD_SCIENE` («1st Tech Center»), `LW_BUILE_SCIENCE_TWO/THREE`. Tactical Center к владению научной очередью отношения не имеет; в их страницах этой связи нет вовсе.
8) Альянсовая наука (`al.science.research {scienceId}`, `al.science.donate {scienceId, option}`, `al.science.donate.gold`, `al.science.refreshNum`, `al.science.fresh`, `push.al.science*`) у них не описана ни в одной из сверенных страниц — это отдельное дерево со своими слотами и донатами.
9) `queue.finish.batch` у них «без поля стоимости» — верно, но опущено обязательное `paraState` (Int) и то, что команда не для науки.
10) Про `al.help.all` они пишут как про сбор «на уровне аккаунта»; у нас Help All закрывает чужие заявки, а своя очередь — только `al.call.help`. Смешивать нельзя: `isHelped` выставляется именно вторым.

## 4. Прямая польза для задачи «старт науки в обход условий + досрочный финиш»

**Обход условий — реалистичен, потому что условий в кадре нет.** Условия уровня науки (`needBuild` — уровень лаборатории, `needScience` — предыдущая наука) читаются только локально в `GetUnreachedPreConditions` (view.lua:538-572, `condType 1` = здание, `condType 2` = наука) и в `preLockedId` (view.lua:429-436); кнопка гасится `SetBtnGoActive(false)`. В отправке (view.lua:998-1041) перепроверяются **только** `queue:GetQueueState() == Free` и наличие ресурсов/предметов в инвентаре — ни `needBuild`, ни `needScience` не сверяются повторно. Значит кадр собирается руками:

```
science.research.new {
  itemId:  <база+уровень, Int, напр. base+5 при ScienceLevelCap=100>,
  useGold: 0 | 1 | 2,          // NoUseGold | UseGold | Free
  bUuid:   <funcUuid свободной научной очереди>
}
```

Обязательное предусловие на нашей стороне: очередь берётся через `GetQueueByBuildUuidForScience(bUuid)` и должна быть `Free`; одна очередь = одна лаборатория. Если очередей нет — кадр бессмысленен.

**Ветки тех-центра — это те же `needBuild`-строки.** Когда `buildId == FUN_BUILD_SCIENE`, проверка идёт по уровню *той лаборатории, в которой исследуем* (view.lua:547-549 подменяет buildId на `buildingData.itemId`, view.lua:582-587 сравнивает `buildingData.level < v1.level`). То есть «нужен тех-центр N» — требование к уровню конкретной лаборатории, а не глобальное. Тест: собрать `itemId` на уровень, которого у лаборатории нет, и послать — если сервер не проверяет, исследование стартует на слабом здании и мы получаем ветку досрочно.

**Досрочный финиш — самая дешёвая проверка.** `queue.finish {uuid}` не несёт стоимости, клиентский гейт — единственное `now >= endTime`; при успехе `QueueFinishHandle` сбрасывает очередь в `Free` и применяет `science` (qdm.lua:326-353). При отказе приходит `errorCode`, состояние не меняется, денег не тратится. Значит: запускаем длинную науку, шлём `queue.finish {uuid}` сразу, смотрим на `errorCode` и на пришедший `science.itemId/level`, затем на `endTime` в `queue`.

**Сжатие таймера в том же кадре.** `speedGoods` опционален в старте — можно погасить время сразу (`speedGoods:"200220;3600"`), не открывая панель. Для уже идущей очереди: `queue.ccd.m.new {qUUID, itemIDs}` (предметы) либо `useGold` + пустой `itemIDs` (алмазы). Защита от тихой перетраты: проверять `itemCostArr` в ответе и падение баланса. Отдельный пул бесплатного времени — `free.speed.queue` (у них не описан).

**Стабильность последовательности.** Ни `science.research.new`, ни `queue.finish` не требуют ни `clientParam`, ни версии таблиц; ресурсы списывает сервер, локальные `lackResource/lackResourceItem/noBuyItem` — только индикаторы (жёсткий лок даёт `NO_ITEM`, но пакет всё равно уходит).

**Что не путать.** Альянсовая наука — другой кадр (`al.science.research {scienceId}`, один Int, без bUuid и без выбора очереди); её досрочный финиш идёт через те же `queue.finish`, если сервер заводит под неё очередь, но владелец — альянс, не наша лаборатория.

**Порядок тестов:** (1) `science.research.new` с заведомо невыполненным `needScience`; (2) то же с недостающим уровнем лаборатории (ветка тех-центра); (3) `queue.finish` на только что запущенной науке; (4) `speedGoods` в кадре старта с проверкой `itemCostArr`.

**Итог (5 строк)**
1. Условия уровня науки и тех-центра существуют только в клиенте: в кадре `science.research.new` их нет, летят `itemId, useGold, bUuid` (+опц. `speedGoods`) — обход реалистичен, проверка на сервере.
2. `needBuild` с `FUN_BUILD_SCIENE` — это требование к уровню той лаборатории, где идёт исследование, а не глобальный тех-центр; `ScienceLevelCap = 100`, id = база+уровень.
3. `queue.finish {uuid}` бесплатен, гейтится только `now >= endTime`; отказ = `errorCode` без потерь, поэтому ранний финиш — самый дешёвый тест.
4. `useGold`: 0 = без алмазов, 1 = за алмазы, 2 = бесплатно; `isGold` на проводе нет, `robotUuid` в старте науки тоже нет (это наша ошибка в скилле).
5. Альянсовая наука (`al.science.*`) и `free.speed.queue` у них не описаны — отдельные деревья, не смешивать с лабораторной наукой.
