# retailrocket-json

265 статей с сайта [retailrocket.ru](https://retailrocket.ru) в формате **BlogArticle JSON Schema**.

## Структура

```
blog/           125 статей — теория и практика e-commerce
glossary/        42 статьи — словарь терминов
updates/         30 статей — продуктовые обновления
cases/           24 статьи — кейсы клиентов
news/            21 статья  — новости компании
pages/           18 страниц — служебные страницы
events/           3 события — мероприятия
analytics/        2 статьи — аналитические отчёты
```

## Формат

Каждый JSON-файл — одна статья. Полная спецификация: [`SCHEMA.md`](SCHEMA.md).

```json
{
  "Slug": "kak-schitat-churn-rate",
  "Label": ["CX & Loyalty", "Теория и практика"],
  "Title": "Как считать Churn Rate",
  "PublishDate": { "$date": "2024-06-15T10:00:00" },
  "ImageUrl": "https://retailrocket.ru/wp-content/uploads/cover.jpg",
  "AuthorIdList": ["natalia-kazmina"],
  "LeadText": [{ "Elements": [{ "_t": "Text", "Content": "..." }] }],
  "MetaTitle": "Как считать Churn Rate – Retail Rocket",
  "ContentBlockList": [
    { "_t": "Paragraph", "Elements": [...] },
    { "_t": "Heading2", "Title": "Формула расчёта" },
    { "_t": "Image", "ImageUrl": "...", "Alt": "..." },
    { "_t": "UnorderedList", "Items": [...] }
  ]
}
```

Типы блоков: `Paragraph`, `Heading2`–`Heading6`, `Image`, `Video`, `UnorderedList`, `OrderedList`, `Table`, `Quote`, `Factoid`, `Formula`, `CtaPrimaryBlock`, `CtaSecondaryBlock`, `BigContentBg`, `SmallContentBg`, `Carousel`.

### Промо-блоки с фоном

- **BigContentBg** — промо-контейнер с фоновым изображением, заголовком и CTA-кнопкой. Поля: `ImageUrl`, `Alt`, `Elements`, `BtnText`, `BtnUrl`. (1 блок)
- **SmallContentBg** — промо-контейнер без изображения, с заголовком и CTA-кнопкой. Поля: `TitleElements`, `Elements`, `BtnText`, `BtnUrl`. (78 блоков)

Детектируются в WordPress HTML по контейнерам с `background_background: classic` в `data-settings`.

Rich-text — массивы inline-элементов `Text` и `Link` с boolean-флагами форматирования:

- `IsHighlight` — маркерное выделение (зелёный `#DAFB9D` на сайте). Используется в Paragraph-блоках и в списках из `icon-list.default` виджетов Elementor.
- `IsBold` — обычный жирный. Используется в списках из `text-editor.default` виджетов.
- `IsItalic`, `IsUnderline`, `IsStrike` и др. — определены в схеме.

Boolean-флаги присутствуют только со значением `true`.

### Дополнительные поля по разделам

- **cases/** — `IndustryList` (массив строк | null): индустрия клиента из WP-таксономии `case_industry`. 10 значений: DIY, FMCG, HoReCa, Ритейл, Бьюти, Детские товары, Искусство, Образование, Строительство, 18+.
- **glossary/** — `Letter` (строка): буква алфавита из WP-таксономии `alphabet_letter`. 14 букв: А, Б, В, Д, К, М, Н, О, П, Р, С, Т, Ф, Ц.

## Служебные файлы

| Файл | Описание |
|---|---|
| [`SCHEMA.md`](SCHEMA.md) | Полная спецификация BlogArticle JSON Schema |
| `_mapping.json` | slug → {section, id, url} для всех 265 статей |
| `_image_mapping_cdn.json` | WP URL → CDN URL для 1538 изображений |
| `_leadtext_issues.json` | 4 статьи с пустым LeadText после очистки |
| `_unpatched_content_bg.json` | 19 SmallContentBg, не запатченных автоматически |

## Очистка данных

Двухуровневая фильтрация шаблонного мусора WordPress:

1. **Экспортер** — фильтрация по `data-widget_type` (4 типа: excerpt, автор, навигация, лейблы категорий)
2. **Клинер** (`clean_articles.py`) — текстовые эвристики для in-content дублей:
   - Удаление параграфа-дубликата LeadText (prefix-match, т.к. WordPress обрезает excerpt)
   - Удаление авторского параграфа (6 известных авторов, уже есть в `AuthorIdList`)
   - Удаление Gravatar-аватарок, навигационных ссылок, мусорных маркеров
   - Удаление trailing-блоков «Похожие кейсы/статьи» (карточки-ссылки на другие материалы)

## Целевые MongoDB-схемы

JSON-файлы должны соответствовать MongoDB JSON Schema из репозитория:

[`gl.retailrocket.ru/sandbox/landing`](https://gl.retailrocket.ru/sandbox/landing/-/tree/master/Migrations) → папка `Migrations/`:

| Файл схемы | Секция |
|---|---|
| `blogArticles.schema.json` | blog/ |
| `analyticsArticles.schema.json` | analytics/ |
| `casesArticles.schema.json` | cases/ |
| `glossaryArticles.schema.json` | glossary/ |
| `newsArticles.schema.json` | news/ |
| `updatesArticles.schema.json` | updates/ |

Схема `casesArticles` обновлена: поле `IndustryList` (массив строк | null).

## Источник данных

Данные сконвертированы из WordPress-экспорта (v1 JSON) и обогащены через WP REST API (авторы, SEO-метаданные, категории). Скрипты конвертации (`convert.py`, `enrich.py`, `clean_articles.py`) находятся локально и не коммитятся — см. [`CLAUDE.md`](CLAUDE.md) для деталей.
