<div align="center">

# rlm.pi

**Рекурсивные языковые модели для Pi.**

Дешёвая модель читает очень большой контекст с высокой точностью.

<p>
<a href="https://www.npmjs.com/package/@hicaru/pi-rlm"><img src="https://img.shields.io/npm/v/@hicaru/pi-rlm?color=cb3837&label=npm" alt="npm" /></a>
<a href="https://arxiv.org/abs/2512.24601"><img src="https://img.shields.io/badge/arXiv-2512.24601-b31b1b" alt="RLM paper" /></a>
<img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT" />
</p>

<sub><a href="README.md">English</a> · <a href="README.zh-CN.md">中文</a> · <b>Русский</b></sub>

</div>

## Установка

| | Установить | Обновить | Удалить |
|---|---|---|---|
| **Pi** | `pi install npm:@hicaru/pi-rlm` | добавьте `--force` | `pi uninstall npm:@hicaru/pi-rlm` |
| **oh-my-pi** | `omp plugin install @hicaru/pi-rlm` | добавьте `--force` | `omp plugin uninstall @hicaru/pi-rlm` |

1. Установите плагин.
2. Введите `/reload` или перезапустите Pi.
3. Проверьте, что `/rlm`, `/rlm-config` и `/rlm-stop` есть в **[Extensions]**.
4. Нажмите `Ctrl+Shift+R` или введите `/rlm`. Теперь ваши запросы идут в движок RLM.

## Как это работает

```
 вы ──► запрос ──► движок RLM ──► основная модель (Python в repl(), по одному ходу)
                       │              ├─► search / grep / outline   (без токенов)
                       │              ├─► llm_query ──► дешёвые воркеры
                       │              └─► rlm_query ──► дочерние RLM (лимит глубины)
                       └──► состояние Σ + проверка + лимиты (токены, время, ошибки)
```

- Контекст хранится в песочнице Python. Модель видит только индекс.
- Основная модель пишет код на Python. Код находит нужные модели данные.
- Дешёвые модели-воркеры читают и обобщают. Основная модель не читает весь текст.
- Дочерний RLM решает сложную подзадачу. Он получает всю песочницу без затрат токенов.
- Движок хранит состояние запуска `Σ` и проверяет каждое его изменение.
- `/compact` создаёт дайджест сессии. Он не использует LLM.

## Бенчмарки

Бенчмарк — **OOLONG** (oolong-synth): 24 задачи, контекст до 65 536 токенов, температура 0.

| Модель | OOLONG | Токенов/задача | Цена/задача |
|---|---|---|---|
| `zai/glm-5.3` | **95.8%** (23/24) | **~28k** | ~$0.00 \* |
| `b-ai/qwen3.8-27b` | **91.7%** (22/24) | ~37k | ~$0.00 \* |
| `openrouter/google/gemma-4-31b-it:free` | ожидается | — | — |

Список моделей — в [`bench/panel.json`](bench/panel.json). У каждой модели своя папка
результатов: `bench/runs/<model>/`. Запуск всех моделей:

```bash
export ZAI_API_KEY=... BAI_API_KEY=... OPENROUTER_API_KEY=...
bun run bench --oolong-max-cl 65536 --oolong-limit 24
```

Подробнее — в [`bench/README.md`](bench/README.md).

## Команды

| Команда | Клавиши | Функция |
|---|---|---|
| `/rlm` | `Ctrl+Shift+R` | Включить или выключить режим RLM |
| `/rlm-stop` | | Остановить текущий запуск |
| `/rlm-config` | | Настроить сэмплинг, бюджеты и лимиты |

## API песочницы

Модель может использовать только эти функции поиска и делегирования:

```python
search("needle", k=10)        # совпадения BM25 в контексте
grep_context(pattern, k=50)   # совпадения regex с номерами строк
outline(path)                 # определения в файле
add_context("report.pdf")     # добавить файл PDF, DOCX, XLSX, EPUB, CSV или HTML
llm_query(prompt)             # отправить текст дешёвому воркеру
map_files(paths, prompt)      # задать один вопрос по многим файлам
rlm_query(task, paths=...)    # запустить дочерний RLM
answer["content"] = "..."     # записать финальный ответ
answer["ready"] = True        # завершить запуск
```

## Безопасность

- Движок не пишет на диск. Ответ — единственный результат запуска.
- API-ключи не попадают в песочницу. Все вызовы sub-LLM делает хост.
- Python работает в защищённом подпроцессе со списком зарезервированных имён.

## Лицензия

[MIT](pi-plugin/rlm/LICENSE) © hicaru contributors
