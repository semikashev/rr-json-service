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
  "Label": ["Теория и практика"],
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

Типы блоков: `Paragraph`, `Heading2`–`Heading4`, `Image`, `Video`, `UnorderedList`, `OrderedList`, `Table`, `Quote`, `Factoid`, `Formula`, `CtaPrimaryBlock`, `CtaSecondaryBlock`.

Rich-text — массивы inline-элементов `Text` и `Link` с boolean-флагами форматирования:

- `IsHighlight` — маркерное выделение (зелёный `#DAFB9D` на сайте). Используется в Paragraph-блоках.
- `IsBold` — обычный жирный. Используется только в OrderedList/UnorderedList.
- `IsItalic`, `IsUnderline`, `IsStrike` и др. — определены в схеме.

Boolean-флаги присутствуют только со значением `true`.

## Служебные файлы

| Файл | Описание |
|---|---|
| [`SCHEMA.md`](SCHEMA.md) | Полная спецификация BlogArticle JSON Schema |
| `_mapping.json` | slug → {section, id, url} для всех 265 статей |
| `_leadtext_issues.json` | 4 статьи с пустым LeadText после очистки |

## Источник данных

Данные сконвертированы из WordPress-экспорта (v1 JSON) и обогащены через WP REST API (авторы, SEO-метаданные, категории). Скрипты конвертации (`convert.py`, `enrich.py`) находятся локально и не коммитятся — см. [`CLAUDE.md`](CLAUDE.md) для деталей.
