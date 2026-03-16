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
| `_image_mapping.json` | original URL → relative path для всех 1538 изображений | Да |
| `_leadtext_issues.json` | 4 статьи с пустым LeadText после очистки | Да |
| `convert.py` | Конвертер v1 → BlogArticle | Нет (.gitignore) |
| `enrich.py` | Обогащение из WP REST API | Нет (.gitignore) |
| `download_images.py` | Скачивание изображений из JSON-статей | Нет (.gitignore) |
| `_enrichment.json` | Кеш обогащения (авторы, SEO, категории) | Нет (.gitignore) |
| `_image_errors.json` | Ошибки скачивания изображений (если есть) | Нет (.gitignore) |
| `images/` | Скачанные изображения (1538 файлов, ~454 МБ) | Нет (.gitignore) |
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
- `_image_mapping.json`
- `_leadtext_issues.json`
- `README.md`, `CLAUDE.md`

### Что НЕ коммитить

Всё перечисленное в `.gitignore`: скрипты (`convert.py`, `enrich.py`, `download_images.py`), кеши (`_enrichment.json`), промежуточные файлы (`output/`), изображения (`images/`), архивы.

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

### download_images.py

Скачивание всех изображений из JSON-статей. Python 3, stdlib + `urllib`.

- Извлекает URL из трёх полей: `ImageUrl` (top-level обложка + Image-блоки), `AuthorImageUrl` (Quote-блоки)
- 3 источника: `retailrocket.ru` (~1445 URL), `gallery.retailrocket.net` (88), `secure.gravatar.com` (5)
- Транслитерация кириллицы в именах файлов (встроенная таблица, без зависимостей)
- Gravatar URL: декодирование HTML entities (`&#038;` → `&`), имя файла `gravatar-{hash[:12]}-s{size}.jpg`
- Дедупликация: один URL → одна копия в папке первой встретившейся статьи
- Скачивание: чанками по 64KB, проверка `Content-Length`, retry 3x с задержкой 2 сек
- Параллельность: `ThreadPoolExecutor` (по умолчанию 10 потоков)
- Результат: `images/{section}/{slug}/{filename}`, маппинг в `_image_mapping.json`

```bash
python3 download_images.py              # скачать все 1538 изображений
python3 download_images.py --dry-run    # только собрать URL и сохранить маппинг
python3 download_images.py --retry-errors  # повторить только неудачные из _image_errors.json
python3 download_images.py --workers 5  # ограничить число потоков
```

### Перезапуск конвертации

```bash
# 1. Обогатить данные из WP API
python3 enrich.py

# 2. Сконвертировать все статьи
python3 convert.py

# 3. Скопировать из output/ в корневые папки
# (convert.py пишет в output/{section}/{slug}.json)

# 4. Скачать изображения
python3 download_images.py
```

## Известные особенности

- **WP API `IncompleteRead`** — при больших ответах. Решается retry + `per_page=20`
- **HTML entities в категориях** — `html.unescape()` обязателен (напр. `CX &amp; Loyalty`)
- **Event-файлы** — slug содержит URL-encoded кириллицу (`%d0%bd%d0%b0%d0%b7...`)
- **4 статьи с пустым LeadText** — `tolko-po-soglasiju-...` (blog), `chto-takoe-fmcg`, `chto-takoe-ots`, `kross-kanalnost-na-praktike` (glossary)
- **1 blog-статья без Label** — пустой массив
- **3 статьи без автора** — `AuthorIdList` пустой
- **EmphasisQuote** из v1 маппится на `Factoid` в новом формате
- **proxy-metrics part-2 в blog/ вместо analytics/** — серия из 3 статей `proxy-metrics-v-e-commerce-part-{1,2,3}`: part-1 и part-3 имеют WP-категорию `analytics` (Label: `"Аналитика"`), но part-2 имеет категорию `blog` (Label: `"Теория и практика"`). Это расхождение в категоризации на стороне WordPress, данные точно отражают WP-категории.
