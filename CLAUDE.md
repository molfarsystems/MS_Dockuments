# CLAUDE.md

Цей файл дає вказівки Claude Code (claude.ai/code) під час роботи з кодом у цьому репозиторії.

## Огляд проєкту

Molfar System (Мольфар Система) — це україномовний **мобільний застосунок на Kivy / KivyMD**
(Android, збирається через Buildozer/python-for-android), який дозволяє користувачу спілкуватись з
кількома AI-провайдерами (Google Gemini, NVIDIA NIM, OpenRouter) через налаштовувані ролі
"учасників", організовані за типами проектів (швидкий чат, зустрічі, конференції, "orduino"). Усі
дані застосунку (SQLite-бази, експортовані документи, `.md`-дзеркала ролей) зберігаються на
пристрої, а не в цьому репозиторії — бекенд/серверна частина відсутня.

Файлів `requirements.txt`/`pyproject.toml` немає; залежності оголошені лише в `buildozer.spec`
(`requirements = python3,kivy==2.3.0,kivymd==1.2.0,pillow,requests,urllib3,charset-normalizer,certifi,idna`).
Для розробки/запуску локально на десктопі встанови ці ж пакети через pip у середовищі Python 3.11.

## Команди

У цьому репозиторії немає лінтера, форматера чи налаштованого автоматизованого набору тестів (немає
використання pytest/unittest, немає `requirements*.txt`, немає CI окрім збірки APK). Будь-які зміни
потребують ручної/візуальної перевірки.

- **Запуск на десктопі** (найшвидший цикл розробки — Kivy нормально працює поза Android):
  `python main.py`
- **Локальна збірка debug APK для Android** (повторює CI): `buildozer android debug`
  — потребує повного тулчейну Android SDK/NDK; краще довірити це CI
  (`.github/workflows/main.yml`), якщо немає конкретної потреби тестувати саме пакування.
- **CI**: `.github/workflows/main.yml` ("Build Android APK") збирає debug APK при кожному push у
  `update/complete-overhaul` та кожному PR у `main`, і завантажує його як артефакт. Там зафіксовано
  `android.build_tools_version = 33.0.2`, `p4a.branch = v2024.01.21`, `cython==0.29.36` та
  `ubuntu-22.04` (не 24.04, бо там немає `libtinfo5`) — ці фіксації критичні для збірки; перед тим,
  як чіпати налаштування Android-збірки, читай докладні коментарі у workflow та в `buildozer.spec`.
- `main_test.py` — окрема діагностична точка входу (не частина застосунку, не набір тестів), яка
  перевіряє, які шляхи файлової системи доступні для запису на конкретному Android-пристрої; запускай
  окремо через `python main_test.py` під час діагностики проблем з дозволами на запис/логування.

## Архітектура

### Точка входу та послідовність запуску (`main.py`)

`main.py` огортає кожен імпорт і всю послідовність `MDApp.build()`/`_init_db()` захисними
try/except-блоками, які пишуть у файли логів на пристрої (`molfar_import.log`, `molfar_crash.log`), а
при збої показують екран аварії з повним трейсбеком замість мовчазного падіння — це зроблено тому,
що збої Android APK інакше вкрай важко діагностувати. Додаючи нові імпорти верхнього рівня в
`main.py`, дотримуйся наявного патерну `_try_import(label, fn)`, щоб збої лишались діагностованими.

Шляхи логів формуються за ланцюжком фолбеків (у порядку): app-specific зовнішня тека Android через
`getExternalFilesDir()` (не потребує runtime-дозволу) → `android.storage.app_storage_path()` → `cwd`
(десктоп). `_resolve_base_dir()` у `modules/creator.py` використовує той самий патерн для реальної
теки даних застосунку (`.../Android/data/<pkg>/files/Molfar_System/` на Android,
`user_data_dir/Molfar_System` на десктопі).

Порядок запуску в `MainApp._build_impl()`: `Lang` → `ConfigManager` → `ThemeManager` → побудова
кореня KV (`MDScreenManager` + drawer `MDNavigationLayout`) → реєстрація екранів →
`Clock.schedule_once` ініціалізації БД через 0.5с (`_init_db`, викликає
`modules.creator.initialize()`) → після ще однієї затримки перехід на `"home"`. Об'єкти
БД/менеджерів (`db_manager`, `chat_db_manager`, `ai_config_manager`, `participants_manager`,
`memory_manager`, `roles_manager`) дорівнюють `None`, доки не завершиться `_init_db`, тож будь-який
код екрана, що звертається до `MDApp.get_running_app().<manager>`, має витримувати їх тимчасову
відсутність одразу після запуску.

**Відома поточна прогалина:** `main.py` імпортує
`from modules.project_chat_screen import ProjectChatScreen` і реєструє його як екран
`"project_chat"`, але `modules/project_chat_screen.py` було видалено з репозиторію (коміт
`ea86939`, "Delete modules/project_chat_screen.py") без заміни. Наразі застосунок не запуститься,
доки цей екран не буде відновлено або поки імпорт/реєстрацію не буде прибрано. Перевіряй це, перш
ніж вважати, що `main.py` запускається "як є".

### Структура UI

- Екрани лежать у `screens/` (`home.py`, `settings.py`, `answers.py`) і додаються до єдиного
  `MDScreenManager` (`id: sm`), побудованого з KV-рядка в `main.py`. Навігація йде через
  `MainApp.switch_screen(name)` / `go_back()` (веде список `screen_stack`), а не через пряму
  зміну `sm.current` з екранів.
- `mod/pop_ai.py` (`PopAIScreen`) — великий самодостатній односесійний чат-екран ("Швидкі
  запитання"), незалежний від системи проектів/учасників.
- `tools/left_menu.py` будує бічну навігаційну шухляду; `tools/mod_bottom_menu.py` будує
  перевикористовуване нижнє меню дій; `tools/menu_config.py` централізує визначення пунктів меню.
- Модальні вікна (діалоги) для функцій з активним CRUD лежать у `modules/*_modal.py`
  (`roles_modal.py`, `participants_modal.py`, `project_item_modal.py`, `create_proj_modal.py`,
  `md_editor_modal.py`, `models_modal.py`, `ai_settings_modal.py`, `haiduk_menu_modal.py`,
  `meeting_queue_modal.py`, `project_list_modal.py`).
- Кожен нетривіальний екран/віджет визначає свою розмітку як Kivy-рядок `KV = '''...'''`,
  побудований через `Builder.load_string(KV)` на початку свого модуля, а не як окремі `.kv`-файли.
  Дотримуйся цієї конвенції для нового UI.
- Kivy-віджети майже завжди беруть посилання `app = MDApp.get_running_app()`, щоб читати кольори
  теми (`app.theme_cls`), переклади (`app.lang.get(key)`) та менеджери БД.
- `lang/*.ini` (`ua`, `en`, `es`, `de`, `fr`) містять рядки UI за ключами в межах секцій (`[main]`
  тощо); `Lang.get(key, section="main")` у `main.py` шукає їх з кешем у пам'яті і повертає сирий
  ключ як фолбек, якщо запис відсутній. `lang/faq_*.json` містять контент FAQ для кожної мови.
  Українська (`ua`) — основна/дефолтна мова (дефолт у `config.ini` та дефолт `"ua"` в
  `ConfigManager`).

### Шар даних (SQLite, усе під `modules/`)

- **`modules/schemas.py`** — єдине джерело правди для всіх SQL-схем, констант версій кожної БД
  (`PROJECTS_DB_VERSION`, `CHAT_DB_VERSION`, `AI_CONFIG_DB_VERSION`, `MEMORY_DB_VERSION`,
  `ROLES_DB_VERSION`) і спільних лімітів (`MAX_ROLE_PROMPT`, `MAX_ROLE_SKILL`, `MAX_ROLE_DESC`).
  Будь-яка зміна схеми починається тут, плюс підняття відповідної `*_DB_VERSION` і гілка міграції
  (див. нижче) — ніколи не редагуй схеми "на льоту" всередині класів-менеджерів.
- **`modules/creator.py`** відповідає за bootstrap файлової системи/БД: визначає базову теку даних,
  створює фіксовану структуру `SUBDIRS` (`Documents/`, `DB/Projects/`, `Prompts/`, `Images/`,
  `Exports/`, `Roles/`, ...), створює кожен SQLite-файл, якщо його немає, через `_setup_database()`,
  і запускає `_migrate()` для наявних БД, чий `PRAGMA user_version` відстає від цільового. Міграції
  адитивні, пронумеровані, розбиті на гілки за `label` всередині `_migrate()` (наприклад,
  `if from_version < 5 <= to_version: ...`) — додавай нові міграції так само, не змінюючи наявні
  гілки. Він також засіває вбудовані системні промпти (`_seed_system_prompts`, ідемпотентно за
  `code`) і вбудовані ролі (`_seed_roles`, лише коли таблиця ролей порожня), і дзеркалить
  промпт/скіл кожної ролі у `Roles/<folder>/{prompt.md,skill.md}` — БД є джерелом правди, `.md`-файли
  — це одностороннє дзеркало для видимості користувачем.
- Під `DB/Projects/` є 5 окремих SQLite-баз: `projects.db`, `chat_history.db`, `ai_config.db`,
  `memory.db`, `roles.db` — кожна зі своїм класом-менеджером (`db_manager.py`/`DBManager`,
  `chat_db_manager.py`/`ChatDBManager`, `ai_config_manager.py`, неявний менеджер пам'яті,
  `roles_manager.py`/`RolesManager`) і своєю доріжкою версій/міграцій. Кожен тип проекту (`quick`,
  `conference`, `meeting`, `orduino`) отримує власну, структурно ідентичну таблицю (`TABLE_MAP` у
  `schemas.py`), звернення до якої йде через білий список на рівні Python (ніколи не підставляй
  сирий рядок типу в SQL напряму — завжди через `TABLE_MAP`/`ROLES_TABLE_MAP`).
- Рядки `project_participants` використовують `slot_index` 1–4 (робочі учасники), 5 (Оркестратор),
  6 (Секретар/"Архіваріус", доданий пізніше — не бере участі у звичайних раундах чату).
  Дотримуйся цієї нумерації слотів, торкаючись логіки учасників, замість введення паралельної
  концепції.
- Методи менеджерів БД зазвичай виконують блокуючу роботу з SQLite поза UI-потоком і повертають
  результати через `Clock.schedule_once` — дотримуйся цього патерну (див. `RolesManager`,
  `ParticipantsManager`) для будь-якого нового менеджера, замість прямих викликів SQLite в UI-потоці.

### Інтеграція з AI-провайдерами (`tools/ai_adapters/`, `tools/manage_api.py`, `tools/providers_registry.py`)

- `tools/providers_registry.py` (`PROVIDERS_REGISTRY`) — статичний каталог провайдерів (`google`,
  `nvidia`, `openrouter`) з `base_url`, списком моделей за замовчуванням, ознакою потреби API-ключа,
  порядком сортування. Новий провайдер додається сюди плюс відповідний адаптер.
- `tools/ai_adapters/base.py` визначає `BaseAdapter` (`call()`, `test_connection()`) та
  `AdapterError` (`message` + машинозчитуваний `code`, напр. `timeout`/`auth`/`quota`/`empty`/
  `network`/`http`). `tools/ai_adapters/openai_compat.py` надає спільні хелпери для
  OpenAI-сумісних API (`call_openai_chat`, `test_openai_models`), які повторно використовують
  адаптери NVIDIA та OpenRouter; адаптер Google звертається напряму до Gemini API.
  `tools/ai_adapters/__init__.py` реєструє кожен адаптер захищено (`_register`), щоб один зламаний
  модуль адаптера не обвалював увесь пакет/застосунок під час імпорту — зберігай цей патерн
  стійкості, додаючи нові адаптери.
- `APIManager.ask_with_config(cfg, messages, on_done, on_error)` у `tools/manage_api.py` — єдина
  точка входу, яку має викликати UI-код для звернення до будь-якого провайдера. Вона запускає виклик
  адаптера в daemon-потоці й одночасно ставить проти нього watchdog на базі `Clock` (`timeout + 10с`
  запасу), щоб зависла мережева операція ніколи не блокувала UI і не лишала `on_done`/`on_error`
  невикликаними — спрацьовує рівно один з них, під захистом прапорця `settled`. Використовуй цю
  точку входу повторно, а не викликай адаптери напряму з екранів.

### Ролі

`modules/roles_seed.py` містить вбудовані визначення ролей (`SEED_BY_KIND`), якими засівається
`roles.db` при першому запуску. Ролі бувають або `is_builtin=1` (редагуються вільно, але не
видаляються, стабільні `code`/`folder`), або створені користувачем
(`code = "custom_" + uuid8`). Значення `role_code`, обрані користувачем, зберігаються між сесіями
через `ConfigManager.get_haiduk()/set_haiduk_role()`.

### Конвенція коментарів

Багато коментарів мають префікс `KL` або `Claud` (наприклад, `# KL ...`, `# Claud ...`) — це
позначки, залишені попередніми контриб'юторами/AI-проходами, які часто пояснюють *чому* існує
обхідне рішення (особливості дозволів на запис в Android, необхідність перебудови таблиці замість
`ALTER TABLE` через обмеження SQLite `CHECK` тощо) — читай їх перед тим, як змінювати сусідній код,
зазвичай вони фіксують важко здобутий урок, а не є декоративними.

### Файли, про які варто знати

- `main_oridinal.py` — рання/альтернативна версія `main.py` (зверни увагу на помилку в назві) — її
  ніхто не імпортує; вважай історичним референсом, а не живим кодом.
- `assets/` містить іконки/сплеш/банер застосунку, на які посилаються `buildozer.spec` та
  KV-рядки.
