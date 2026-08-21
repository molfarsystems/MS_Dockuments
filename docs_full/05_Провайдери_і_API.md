# 05. Провайдери і API

> Актуально: 2026-07.

## Ланцюжок
UI → `tools/manage_api.APIManager.ask_with_config(cfg, messages, on_done, on_error)`
→ `tools/ai_adapters.get_adapter(kind)` → адаптер → HTTP (requests, імпорт
всередині методу) → `_finish` → колбек у Kivy-потоці.

## providers_registry (статичний)
Для кожного kind (google / nvidia / openrouter): base_url, дефолтні моделі,
прапорець web_search. Синхронізується в ai_config.db менеджером
(`ai_config_manager`, sync з PROVIDERS_REGISTRY при старті).

## Адаптери (tools/ai_adapters/)
- `base.py` — контракт: `call(model, messages, api_key, **opts) -> text|цитати`,
  `test_connection`, `AdapterError(code)` — code = ключ локалізації.
- `openai_compat.py` — спільний шар chat/completions (+/models тест, витяг
  тексту/помилок/url_citation). nvidia і openrouter — тонкі нащадки.
- `google_adapter.py` — Gemini REST generateContent; grounding-пошук;
  цитати з groundingMetadata.
- `openrouter_adapter.py` — web-плагін для пошуку.
- `__init__.py` — СТІЙКИЙ (див. [[03_Збірка_і_середовища]] §обмеження 2).

## APIManager: гарантії
- Фоновий daemon-потік на запит; UI не блокується.
- **Watchdog**: Clock-таймер `REQUEST_TIMEOUT+10с` — якщо потік завис
  (мережа без відповіді), запит завершується помилкою timeout.
- **Рівно один колбек**: `state.settled` — хто перший (відповідь, помилка,
  watchdog), той і доставляє; решта ігнорується. Колбеки — через Clock.
- Помилки локалізуються (`_localize`: ключ → текст мовою UI, fallback укр).
- `REQUEST_TIMEOUT = 220с` (повільні безкоштовні моделі OpenRouter).

## Формат messages
Стандартний список {role, content}; system збирається викликачем:
місце (schemas) + роль + блок документів (+ памʼять для Оркестратора).
Історія — живе вікно останніх HISTORY_WINDOW=25 повідомлень чату.

## Веб-пошук
Прапорець провайдера + тумблер-глобус у чаті. Цитати (url_citation /
grounding) повертаються поруч з текстом; Архіваріус вміє індексувати їх
як web-джерела ([[08_Двигун_знань]]).

## Точки розширення
- Новий провайдер: реєстр → адаптер (найчастіше нащадок openai_compat) →
  try-реєстрація в `__init__`. UI підхопить через ai_config sync.
- Мультимодальність (зображення, фаза 2): контракт messages розширюється
  до content=список блоків; конвертація — в openai_compat._build і
  google_adapter._build_payload. Див. [[16_Беклог]].
