# Инструкции для агента

## Обзор

Репозиторий — **265 статей** с сайта retailrocket.ru в формате **BlogArticle JSON Schema**.
8 секций: `blog/`(125), `glossary/`(42), `updates/`(30), `cases/`(24), `news/`(21), `pages/`(18), `events/`(3), `analytics/`(2).

## Целевая схема

Каждый JSON-файл — одна статья. PascalCase, `_t` — discriminator.

Целевые MongoDB-схемы: [`gl.retailrocket.ru/sandbox/landing`](https://gl.retailrocket.ru/sandbox/landing/-/tree/master/Migrations) → `Migrations/*.schema.json`

### Общие поля (blog, analytics, news, updates)

```
Slug            string          URL-slug статьи
Label           string[]        WP-категории (может быть несколько, или пустой для pages/events)
Title           string          Заголовок (H1)
MetaTitle       string          SEO-заголовок
LeadText        Paragraph[]     Вводный абзац из WP excerpt (может быть пустой)
PublishDate     {$date: string} MongoDB Extended JSON
ImageUrl        string|null     Обложка (CDN URL или null)
AuthorIdList    string[]        Slug-и авторов (может быть пустой)
ContentBlockList Block[]        Тело статьи
```

### Дополнительные поля по секциям

- **glossary/** → `Letter` (string, required): буква алфавита. Источник: WP-таксономия `alphabet_letter`. **Нет поля ImageUrl** (в отличие от других секций).
- **cases/** → `IndustryList` (string[]|null): индустрия клиента. Источник: WP-таксономия `case_industry`. 10 значений: DIY, FMCG, HoReCa, Ритейл, Бьюти, Детские товары, Искусство, Образование, Строительство, 18+. `AuthorIdList` может быть `null` (для кейсов-подрядчиков).

### Типы блоков ContentBlockList

`Paragraph`, `Heading2`–`Heading6`, `Image`, `Video`, `UnorderedList`, `OrderedList`, `Table`, `Quote`, `Factoid`, `Formula`, `CtaPrimaryBlock`, `CtaSecondaryBlock`, `BigContentBg`, `SmallContentBg`, `Carousel`.

### Rich-text элементы

Массивы `Text` и `Link` с boolean-флагами (присутствуют ТОЛЬКО когда `true`):

| Флаг | Значение | Где встречается |
|---|---|---|
| `IsHighlight` | Маркерное выделение (зелёный `#DAFB9D`) | Paragraph, списки из `icon-list.default` |
| `IsBold` | Обычный жирный | Списки из `text-editor.default` |
| `IsItalic`, `IsUnderline`, `IsStrike` | Курсив, подчёркивание, зачёркивание | Везде |

**Важно:** `IsHighlight` и `IsBold` — разные вещи. На сайте CSS-правило `.elementor-heading-title b, .elementor-icon-list-text b, .rr-table-cell b { color: #DAFB9D }` делает bold зелёным в трёх контекстах: Paragraph (heading.default), icon-list, кастомные таблицы. В обычных списках (text-editor.default) bold остаётся жирным.

### Labels (метки)

Источник — **категории WordPress** (НЕ теги):

| WP-категория | Отображаемое имя |
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

У статьи может быть несколько меток. Pages и events — `Label: []`.

## Актуальные скрипты-обработчики

Все скрипты — Python 3 stdlib, без внешних зависимостей (кроме экспортера: beautifulsoup4 + lxml).

### 1. Экспортер: `../export_retailrocket.py`

Скачивает HTML из WP REST API → парсит BeautifulSoup → выдаёт v1 JSON (плоские блоки с Markdown-разметкой).

**Фильтрация шаблонного мусора** — по `data-widget_type`:

```python
_SKIP_WIDGET_TYPES = {
    "theme-post-excerpt.default",          # дубль LeadText
    "rrposts-authors-list-item.default",   # блок автора (уже в AuthorIdList)
    "rrposts_back_to_category.default",    # навигация «← Вернуться назад»
    "post-info.default",                   # лейблы категорий
    "clients_widget.default",             # декоративный marquee
    "counter.default",                    # JS-анимация
    "team-slider.default",               # фото-marquee
    "steps_slider.default",              # слайдер шагов
    "pricing-form-v2.default",           # форма
    "rr-form-builder.default",           # лид-форма
}
```

### 2. Конвертер: `convert.py` (в retailrocket-json/)

v1 JSON → BlogArticle (v2). Читает из корневых папок, пишет в `output/`.

Ключевые функции:
- `clean_excerpt()` — очистка LeadText (навигация, авторство, оглавления)
- `_strip_leading_garbage()` — удаление мусора с начала ContentBlockList
- `_strip_trailing_garbage()` — удаление мусора с конца
- `_clean_content_blocks()` — обрезка параграфов с мусорными маркерами
- `_bold_to_highlight()` — `IsBold → IsHighlight` в Paragraph
- LeadText дубликат: **prefix-match** (не строгое `==`, т.к. WP обрезает excerpt)
- Авторский параграф: regex на 6 авторов (Казьмина, Морозов, Криеву, Тимохин, Козлова, Давыдова)

### 3. Клинер: `clean_articles.py` (в retailrocket-json/)

Чистит BlogArticle JSON файлы **in-place** (без v1 исходников). Идемпотентный.

```bash
python3 clean_articles.py              # dry-run
python3 clean_articles.py --apply      # применить
python3 clean_articles.py cases/slug.json --apply  # один файл
```

Что чистит:
- Дубликат LeadText (prefix-match)
- Авторский параграф (6 авторов)
- Gravatar-аватарки
- Навигационные ссылки «Читать далее →»
- Мусорные маркеры («Содержание статьи», «Авторы статьи», «Похожие статьи»)
- Trailing-блоки «Похожие кейсы/статьи» (≥2 карточек-ссылок на retailrocket.ru)
- `IsBold → IsHighlight` в Paragraph
- Декодирование HTML entities

### 4. Обогащение: `enrich.py` (в retailrocket-json/)

WP REST API → `_enrichment.json` (авторы, SEO, категории, quote-авторы).

### 5. Изображения: `download_images.py` (в retailrocket-json/)

Скачивает изображения из JSON-статей → `images/`, маппинг в `_image_mapping.json`.

## Стратегии решения типовых задач

### Добавление нового поля из WordPress

1. Определить источник: WP-таксономия (`wp/v2/<taxonomy>?post=ID`), поле поста, или данные из HTML
2. Загрузить все значения через WP REST API
3. Добавить поле в JSON-файлы (после `Label` для section-specific полей)
4. Обновить `convert.py` для будущих конвертаций
5. Обновить `README.md` и `CLAUDE.md`

### Очистка мусорных блоков

**Два уровня фильтрации:**

1. **Экспортер** (структурный) — если у мусора есть уникальный `data-widget_type`, добавить его в `_SKIP_WIDGET_TYPES`. Это самый надёжный способ.
2. **Клинер** (текстовый) — если мусор приходит из `text-editor.default` (тот же виджет, что и полезный контент), используем текстовые эвристики в `clean_articles.py`.

Порядок: сначала проверить структуру WP HTML, затем — текстовые паттерны.

### Исправление IsHighlight / IsBold

CSS-правило определяет контекст:

```css
.elementor-heading-title b { color: #DAFB9D }   /* Paragraph → IsHighlight */
.elementor-icon-list-text b { color: #DAFB9D }   /* icon-list → IsHighlight */
.rr-table-cell b { color: #DAFB9D }              /* rr-table → IsHighlight */
/* text-editor.default <b> → IsBold (обычный жирный) */
```

Для патча существующих файлов: загрузить HTML из WP API, найти bold-тексты внутри `icon-list.default` виджетов, сопоставить с JSON и заменить `IsBold → IsHighlight`.

### Полная переконвертация

```bash
# 1. Экспорт v1 (перезаписывает JSON-файлы в корневых папках)
cd .. && python3 export_retailrocket.py

# 2. Конвертация v1 → v2 (пишет в output/)
cd retailrocket-json && python3 convert.py

# 3. Копирование v2 обратно
for s in blog analytics glossary cases news updates pages events; do
  cp output/$s/*.json $s/ 2>/dev/null
done

# 4. Очистка
python3 clean_articles.py --apply

# 5. Обогащение (IndustryList, Letter и т.д.) — отдельными скриптами
```

**Внимание:** после шага 1 файлы в корневых папках будут в v1 формате. Шаги 2–3 возвращают их в v2. Не коммитить между шагами.

## Что коммитить

- JSON-файлы статей (все 8 папок)
- `SCHEMA.md`, `README.md`, `CLAUDE.md`
- `_mapping.json`, `_image_mapping.json`, `_leadtext_issues.json`

## Что НЕ коммитить

Всё в `.gitignore`: скрипты (`convert.py`, `enrich.py`, `clean_articles.py`, `download_images.py`), кеши (`_enrichment.json`, `_alt_vision_cache.json`), промежуточные файлы (`output/`), изображения (`images/`), `.env`.

## Известные особенности

- **4 статьи с пустым LeadText** — `tolko-po-soglasiju-...` (blog), `chto-takoe-fmcg`, `chto-takoe-ots`, `kross-kanalnost-na-praktike` (glossary)
- **3 кейса без AuthorIdList** — новый формат кейса-подрядчика (2026), на сайте блок авторов не рендерится
- **2 кейса без IndustryList** — `podrygka-online`, `rfm-segmentatsiya-keys-tehnosila` (не заполнено в WP)
- **proxy-metrics part-2 в blog/ вместо analytics/** — расхождение в WP-категоризации, данные точно отражают WP
- **WP API `IncompleteRead`** — retry + `per_page=20`
- **Видео URL** — всегда embed-формат: `https://www.youtube.com/embed/VIDEO_ID`
