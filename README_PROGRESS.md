# Free Visual Novel — Прогресс

## Суть проекта
Модуль для Foundry VTT — полноэкранная оверлейная VN-сцена. GM управляет сценой (фон, портреты, диалог, броадкаст), игроки видят её в реальном времени, могут управлять своими портретами (эмоции, кнопка Attention/Claim).

## Структура файлов
- `scripts/visualnovel.js` — вся логика (класс приложения, панели, сокеты, настройки)
- `templates/visualnovel.hbs` — Handlebars-шаблон всей VN
- `style/visualnovel.css` — вся стилизация
- `module.json` — манифест модуля (с `"socket": true`)

## Ключевые возможности (сделано)

### Архитектура
- Двухслойная: `.vn-fullscreen` (z-index 25, под UI Foundry) + `.vn-interactive-layer` (z-index 75, выше UI) — отдельный `<div>` на `body`
- Pointer-events цепочка: контейнеры `pointer-events: none`, интерактивные дети `pointer-events: auto`
- Замена `innerHTML` без jQuery (стандартный `_replaceHTML`)
- Proxy `_el()` — направляет `querySelector*`/`addEventListener` на `document`

### Сцена
- Полноэкранный оверлей с fade-анимацией открытия/закрытия
- Фон — изображение или градиент, регулировка яркости (Scene Settings)
- Смена эмоций портрета — cross-fade через overlay `<img>` (0.4s)
- Перетаскивание портретов мышью (только GM)
- Выделение портрета кликом (только GM)
- Ховер-контролы: scale, bring forward/send backward, flip, lock, remove

### Панели (открываются из тулбара)
- **Locations** — список + фильтр (search, tag, group) + пагинация (30/Show More) + CRUD
- **Portraits (GM)** — то же + импорт из актёров + добавление на сцену + удаление
- **Portraits (Player)** — открывается по "My Portrait"; видит ТОЛЬКО свои портреты; может редактировать (имя, заголовок, теги, изображения, эмоции)
- **Scene Settings** — яркость, theme colors (bg + accent), диалог (ширина/высота/прозрачность/выравнивание/текст/шрифт/режим/Y-offset), спикер бокс (шрифт)
- **Presets** — сохранение/загрузка снепшотов сцены (временно отключено)

### Диалог
- Два режима: **Single + Speaker** (одно окно + имя говорящего) и **Dual (2 boxes)** (левая/правая колонки)
- Y-offset — вертикальное позиционирование
- Inline editing: клик на тексте диалога → `contenteditable`, Enter/blur → сохранение, Esc → отмена
- Размеры/прозрачность/выравнивание/шрифт — настраиваются
- Спикер индикатор — всегда виден (авто-размер, только шрифт настраивается)
- Спикер бар — выбор говорящего (GM), клик по claimed портрету → снятие claim + назначение спикером

### Attention / Claim
- Игрок нажимает ✋ на своём портрете → сокет `{type: "claim"}` → GM видит пульсацию + кнопку ✓
- GM нажимает ✓ → портрет становится спикером, claim снимается
- Визуал: dim + drop-shadow + пульсация (2s, цвет акцента)

### Player Portrait Control
- Опционально (вкл/выкл в Configure Module — `playablePortraits`)
- Привязка портрета к игроку через `userId` + авто-тег `player:Name`
- Emotion strip на своих портретах (макс 5)
- Emotion change → сокет `{type: "emotion"}` → GM обновляет и броадкастит всем

### Броадкаст + Ре-вход
- GM нажимает Play → сокет `{type: "state"}` с портретами, спикером, claimed, inviteMode
- **inviteMode**: "All Online" или "Stage Only" (фильтр по userId на сцене)
- Stop → сокет `{type: "stop"}` → все закрывают VN; `_claimed` очищается
- **Session persistence**: `_saveSession()` сохраняет полное состояние в user flag GM-а; `_restoreSession()` восстанавливает при открытии
- **Rejoin**: `/vnrejoin` → `_rejoinVN()` → применяет `_lastBroadcastState`
- **Late-joiners**: `broadcastStore` (world setting) сохраняется на каждый броадкаст; `ready` hook читает его и открывает VN
- **Targeted invite**: сокет `{type: "invite", userId}` → только целевой игрок обрабатывает

### Приглашение (Invite)
- Кнопка + выпадающий список онлайн-игроков (только для `permManage`)
- Создаётся **программно** в `_buildInviteUI()`, не в шаблоне
- `_rebuildMenu()` — перезаполняет список игроков при каждом клике (чтобы видеть новых)
- `toolbar.appendChild(menu)` — меню внутри тулбара для правильного `position: absolute; top: 100%`
- CSS-классы (без inline `style.cssText` — CSP-safe для Foundry v13)
- `overflow: visible`, `min-width: 240px`

### Права доступа (гранулярные)
Четыре настройки в Configure Module (каждая — выбор роли: Player/Trusted/Assistant/GM):
- `permManage` — управление VN (по умолчанию GM)
- `permBroadcast` — броадкаст (по умолчанию GM)
- `permApproveClaims` — утверждение Attention (по умолчанию GM)
- `permAddRequests` — отправка запросов (по умолчанию Player)
- `_userCan(permKey)` / `_roleCan(role, permKey)` — универсальная проверка

### Разное
- Theme colors (bg + accent) — и в `vndata`, и в module settings
- DOM-фильтрация для search/tag; context-based для group/pagination
- Ограничение: макс 10 портретов на сцене
- `game.freevisualnovel` API: `open()`, `close()`, `setBackground()`, `addPortrait()`, `addRequest()`, `clearStage()`, `importActorPortraits()`
- Чат команды: `/vnreq <text>`, `/vnportrait` / `/vnedit`, `/vnrejoin`
- Два слоя затухания при close: fade 250ms → очистка → `super.close()`
- Прокси `_el()` для корректной работы binding-ов независимо от расположения элементов
- `_onRender` всегда вызывает `_bindMainUI()` — тулбар работает при любом открытом panel
- `close()` ждёт `await _saveSession()` перед очисткой, если broadcast активен
- Inline editing: флаг `_inlineEditBound` — слушатели вешаются один раз, а не на каждый render

## Статус тестирования
- **Броадкаст GM ↔ игрок** — ✅ Работает
- **Emotion change от игрока** — ✅ Работает
- **Игрок не может двигать/выделять чужие портреты** — ✅ Работает
- **Attention/Approve (player ✋ → GM ✓)** — ✅ Работает
- **Rejoin (/vnrejoin)** — ✅ Работает
- **Late-join через broadcastStore** — ✅ Работает
- **Invite menu (кнопка + список + перестроение)** — ✅ Работает
- **Гранулярные права** — ✅ Работает
- **2-dialogue mode** — ✅ Работает (переключение, Y-offset, left/right)
- **Inline editing** — ✅ Работает (click, blur/Enter save, Esc cancel)
- **Session persistence (save/restore)** — ✅ Работает (merge через Object.assign)

## Планы на будущее
1. **Список игроков + per-player show/hide** — GM видит кто онлайн, может скрыть VN для конкретного игрока
2. **Селектор вкладок Foundry** — кнопка с иконками чата/актёров/сцен, клик открывает поп-аут (не закрывая VN)
3. **Погода/время/температура** — интеграция с Simple Calendar
4. **CSS-анимации** — fade/slide для показа/скрытия элементов вместо бинарного show/hide
