# Инструкции для агента

## Обзор проекта

Репозиторий содержит 265 статей с сайта retailrocket.ru, сконвертированных из WordPress-экспорта (v1) в формат **BlogArticle JSON Schema**. Схема описана в `SCHEMA.md`.

Статьи распределены по 8 секциям-папкам: `blog/`, `analytics/`, `glossary/`, `cases/`, `news/`, `updates/`, `pages/`, `events/`.

## Архитектура данных

### Формат файлов

- Каждый JSON-файл — одна статья в формате BlogArticle
- PascalCase для всех полей
- `_t` — discriminator-поле на каждом блоке и inline-элементе
- Rich-text: массивы `Text`/`Link` элементов с boolean-флагами форматирования (`IsBold`, `IsItalic` и т.д.)
- Boolean-флаги присутствуют ТОЛЬКО когда `true` (опущены если `false`)

### Labels (метки)

Источник — **категории WordPress** (НЕ теги). Маппинг:

| WP-категория (slug) | Отображаемое имя |
|---|---|
| `blog` | Теория и практика |
| `glossary` | Словарь |
| `cases` | Кейсы |
| `news` | Новости |
| `updates` | Продуктовые обновления |
| `analytics` | Аналитика |
| `cx-loyalty` | CX & Loyalty |
| `retail-media-360` | Retail Media 360 |
| `next-gen-comms` | Next-Gen comms |

- У статьи может быть **несколько меток** (например, `["CX & Loyalty", "Теория и практика"]`)
- Pages и events имеют пустой `Label: []`

### Вспомогательные файлы

| Файл | Описание | В git |
|---|---|---|
| `SCHEMA.md` | Полная спецификация BlogArticle JSON Schema | Да |
| `_mapping.json` | slug → {section, id, url} для всех 265 статей | Да |
| `_leadtext_issues.json` | 4 статьи с пустым LeadText после очистки | Да |
| `convert.py` | Конвертер v1 → BlogArticle | Нет (.gitignore) |
| `enrich.py` | Обогащение из WP REST API | Нет (.gitignore) |
| `_enrichment.json` | Кеш обогащения (авторы, SEO, категории) | Нет (.gitignore) |
| `output/` | Промежуточный вывод конвертера | Нет (.gitignore) |

## Скрипты конвертации

### convert.py

Конвертер из v1-формата в BlogArticle. Только stdlib Python 3.

Ключевые функции:
- `clean_excerpt()` — очистка LeadText от мусора (навигация, авторство, оглавления)
- `normalize_youtube_url()` — приведение YouTube URL к embed-формату
- `_clean_content_blocks()` — пост-обработка: удаление параграфов с «Содержание статьи/видео»

Пропускаемые блоки v1: `heading_1` (дублирует Title), `carousel`, `ImageCaption`, `contents` (оглавление).

### enrich.py

Обогащение через WP REST API (публичный, без авторизации). Только stdlib + `urllib`.

- `fetch_json()` / `fetch_json_with_headers()` — HTTP-запросы с retry (3 попытки, 2 сек задержка)
- `load_categories()` — загрузка категорий WP
- `per_page=20` (уменьшено с 100 из-за `IncompleteRead` ошибок)

## Правила при внесении изменений

### Что коммитить

- JSON-файлы статей (все 8 папок)
- `SCHEMA.md`
- `_mapping.json`
- `_leadtext_issues.json`
- `README.md`, `CLAUDE.md`

### Что НЕ коммитить

Всё перечисленное в `.gitignore`: скрипты (`convert.py`, `enrich.py`), кеши (`_enrichment.json`), промежуточные файлы (`output/`), архивы.

### Очистка LeadText

При правке LeadText удалять:
- `← Вернуться назад`
- `Читать далее →`
- `[…]`
- Строки авторства (имя + должность)
- Всё после `Содержание статьи` / `Содержание видео`

Если после очистки текст пуст — `LeadText: []`, добавить запись в `_leadtext_issues.json`.

### Очистка ContentBlockList

- Не включать блоки оглавления (`contents` в v1)
- Удалять/тримить параграфы содержащие «Содержание статьи» или «Содержание видео»

### Видео URL

Всегда в embed-формате: `https://www.youtube.com/embed/VIDEO_ID`

### Перезапуск конвертации

```bash
# 1. Обогатить данные из WP API
python3 enrich.py

# 2. Сконвертировать все статьи
python3 convert.py

# 3. Скопировать из output/ в корневые папки
# (convert.py пишет в output/{section}/{slug}.json)
```

## Известные особенности

- **WP API `IncompleteRead`** — при больших ответах. Решается retry + `per_page=20`
- **HTML entities в категориях** — `html.unescape()` обязателен (напр. `CX &amp; Loyalty`)
- **Event-файлы** — slug содержит URL-encoded кириллицу (`%d0%bd%d0%b0%d0%b7...`)
- **4 статьи с пустым LeadText** — `tolko-po-soglasiju-...` (blog), `chto-takoe-fmcg`, `chto-takoe-ots`, `kross-kanalnost-na-praktike` (glossary)
- **1 blog-статья без Label** — пустой массив
- **3 статьи без автора** — `AuthorIdList` пустой
- **EmphasisQuote** из v1 маппится на `Factoid` в новом формате
