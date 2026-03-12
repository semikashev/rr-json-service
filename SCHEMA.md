# JSON Schema — BlogArticle

Каждая статья хранится как JSON-файл в формате BlogArticle. Все поля используют PascalCase. Контент представлен как массив типизированных блоков с discriminator-полем `_t`.

## Верхнеуровневые поля

| Поле | Тип | Описание |
|---|---|---|
| `Slug` | `string` | URL-slug статьи |
| `Label` | `string[]` | Метки/теги (тематические теги из WP или название раздела как fallback) |
| `Title` | `string` | Заголовок статьи |
| `PublishDate` | `{ "$date": string }` | Дата публикации в формате MongoDB Extended JSON (ISO 8601) |
| `ImageUrl` | `string` | URL обложки (из Yoast og:image) |
| `AuthorIdList` | `string[]` | Список slug-ов авторов |
| `LeadText` | `Paragraph[]` | Краткое описание (превью) в формате rich-text |
| `MetaTitle` | `string` | SEO-заголовок (из Yoast, fallback к Title) |
| `ContentBlockList` | `Block[]` | Массив контент-блоков (см. ниже) |

Пример верхнеуровневой структуры:

```json
{
  "Slug": "kak-schitat-churn-rate",
  "Label": ["blog"],
  "Title": "Как считать Churn Rate",
  "PublishDate": { "$date": "2024-06-15T10:00:00" },
  "ImageUrl": "https://retailrocket.ru/wp-content/uploads/cover.jpg",
  "AuthorIdList": ["natalia-kazmina"],
  "LeadText": [
    {
      "Elements": [
        { "_t": "Text", "Content": "Churn Rate — ключевая метрика удержания..." }
      ]
    }
  ],
  "MetaTitle": "Как считать Churn Rate – Retail Rocket",
  "ContentBlockList": [ ... ]
}
```

---

## Inline-элементы (Text / Link)

Все текстовые поля внутри блоков (`Elements`, `Items`, `TitleElements`) содержат массивы inline-элементов двух типов, различаемых по полю `_t`.

### `Text` — текстовый фрагмент

```json
{
  "_t": "Text",
  "Content": "Текст фрагмента",
  "IsBold": true
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Text"` | Discriminator |
| `Content` | `string` | Текст |
| `IsBold` | `boolean?` | Жирный (присутствует только если `true`) |
| `IsItalic` | `boolean?` | Курсив |
| `IsUnderline` | `boolean?` | Подчёркнутый |
| `IsStrike` | `boolean?` | Зачёркнутый |
| `IsHighlight` | `boolean?` | Выделение (маркер) |
| `IsSuperscript` | `boolean?` | Надстрочный |
| `IsSubscript` | `boolean?` | Подстрочный |

Boolean-флаги опциональны — присутствуют только со значением `true`.

### `Link` — ссылка

```json
{
  "_t": "Link",
  "Href": "https://example.com",
  "Elements": [
    { "_t": "Text", "Content": "текст ссылки" }
  ]
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Link"` | Discriminator |
| `Href` | `string` | URL ссылки |
| `Elements` | `Text[]` | Массив Text-элементов (без вложенных Link) |

---

## Типы блоков

Каждый блок имеет поле `_t` (discriminator) для определения типа.

### `Paragraph` — Текстовый параграф

Основной блок контента. Самый частый тип.

```json
{
  "_t": "Paragraph",
  "Elements": [
    { "_t": "Text", "Content": "Текст с " },
    { "_t": "Text", "Content": "выделением", "IsBold": true },
    { "_t": "Text", "Content": " и " },
    {
      "_t": "Link",
      "Href": "https://example.com",
      "Elements": [{ "_t": "Text", "Content": "ссылкой" }]
    }
  ]
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Paragraph"` | |
| `Elements` | `(Text \| Link)[]` | Rich-text содержимое |

---

### `Heading2` .. `Heading4` — Заголовки

Заголовки уровней 2–4. `Heading1` пропускается при конвертации (дублирует Title).

```json
{
  "_t": "Heading2",
  "Title": "Заголовок второго уровня"
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Heading2"` / `"Heading3"` / `"Heading4"` | |
| `Title` | `string` | Текст заголовка (plain text) |

---

### `Image` — Изображение

```json
{
  "_t": "Image",
  "ImageUrl": "https://retailrocket.ru/wp-content/uploads/photo.jpg",
  "Alt": "Описание изображения",
  "Caption": "Подпись к изображению"
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Image"` | |
| `ImageUrl` | `string` | Абсолютный URL изображения |
| `Alt` | `string` | Alt-текст (пустая строка если отсутствует) |
| `Caption` | `string \| null` | Подпись (из `<figcaption>`) |

---

### `UnorderedList` — Маркированный список

```json
{
  "_t": "UnorderedList",
  "Items": [
    [
      { "_t": "Text", "Content": "Первый пункт с " },
      { "_t": "Text", "Content": "выделением", "IsBold": true }
    ],
    [
      { "_t": "Text", "Content": "Второй пункт" }
    ]
  ]
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"UnorderedList"` | |
| `Items` | `(Text \| Link)[][]` | Массив элементов, каждый элемент — массив inline-элементов |

---

### `OrderedList` — Нумерованный список

Структура идентична `UnorderedList`.

```json
{
  "_t": "OrderedList",
  "Items": [
    [{ "_t": "Text", "Content": "Шаг 1" }],
    [{ "_t": "Text", "Content": "Шаг 2" }]
  ]
}
```

---

### `Table` — Таблица

```json
{
  "_t": "Table",
  "Header": ["Метрика", "Описание", "Формула"],
  "Rows": [
    ["CPA", "Стоимость привлечения", "Расходы / Конверсии"],
    ["ROAS", "Возврат на рекламу", "Доход / Расходы"]
  ]
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Table"` | |
| `Header` | `string[]` | Заголовки столбцов |
| `Rows` | `string[][]` | Строки данных (plain text) |

---

### `Quote` — Цитата с атрибуцией

```json
{
  "_t": "Quote",
  "Elements": [
    { "_t": "Text", "Content": "Текст цитаты." }
  ],
  "Author": "Алексей Клотц",
  "AuthorPosition": "Head of digital products в Cofix",
  "AuthorImageUrl": "https://retailrocket.ru/wp-content/uploads/photo.webp"
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Quote"` | |
| `Elements` | `(Text \| Link)[]` | Текст цитаты в rich-text |
| `Author` | `string` | Имя автора цитаты |
| `AuthorPosition` | `string \| null` | Должность автора (из Elementor author-box) |
| `AuthorImageUrl` | `string \| null` | URL фото автора (из Elementor author-box) |

---

### `Factoid` — Факт / Выделенная цитата

Объединяет бывшие `EmphasisQuote` и `Factoid` из v1.

```json
{
  "_t": "Factoid",
  "Elements": [
    { "_t": "Text", "Content": "67% покупателей бросают корзину до оплаты." }
  ]
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Factoid"` | |
| `Elements` | `(Text \| Link)[]` | Текст факта в rich-text |

---

### `Formula` — Математическая формула

```json
{
  "_t": "Formula",
  "Latex": "Churn Rate = \\frac{Lost}{Start} \\times 100\\%"
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Formula"` | |
| `Latex` | `string` | LaTeX-выражение |

---

### `Video` — Видео

```json
{
  "_t": "Video",
  "VideoUrl": "https://www.youtube.com/watch?v=..."
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"Video"` | |
| `VideoUrl` | `string` | URL видео (YouTube, Vimeo и др.) |

---

### `CtaPrimaryBlock` — Основной Call-to-Action

Промо-блок с основным призывом к действию (заявка, демо).

```json
{
  "_t": "CtaPrimaryBlock",
  "TitleElements": [
    { "_t": "Text", "Content": "Хотите так же?" }
  ],
  "Elements": [],
  "BtnText": "Запросить демо",
  "BtnUrl": "https://retailrocket.ru/demo/"
}
```

| Поле | Тип | Описание |
|---|---|---|
| `_t` | `"CtaPrimaryBlock"` | |
| `TitleElements` | `(Text \| Link)[]` | Заголовок промо-блока |
| `Elements` | `(Text \| Link)[]` | Описание (обычно пустой массив) |
| `BtnText` | `string` | Текст кнопки |
| `BtnUrl` | `string` | URL кнопки |

---

### `CtaSecondaryBlock` — Вторичный Call-to-Action

Промо-блок с вторичным призывом (подробнее, кейсы).

```json
{
  "_t": "CtaSecondaryBlock",
  "TitleElements": [
    { "_t": "Text", "Content": "Узнать подробнее" }
  ],
  "Elements": [],
  "BtnText": "Посмотреть кейсы",
  "BtnUrl": "https://retailrocket.ru/cases/"
}
```

Структура полей аналогична `CtaPrimaryBlock`.

---

## Разделы экспорта

| Раздел | Папка | Кол-во файлов |
|---|---|---|
| Блог | `blog/` | 125 |
| Глоссарий | `glossary/` | 42 |
| Обновления | `updates/` | 30 |
| Кейсы | `cases/` | 24 |
| Новости | `news/` | 21 |
| Страницы | `pages/` | 18 |
| Мероприятия | `events/` | 3 |
| Аналитика | `analytics/` | 2 |
| **Итого** | | **265** |

## Файл маппинга

`_mapping.json` содержит маппинг `slug → { section, id, url }` для всех статей:

```json
{
  "kak-schitat-churn-rate": {
    "section": "blog",
    "id": 12345,
    "url": "https://retailrocket.ru/blog/kak-schitat-churn-rate/"
  }
}
```

| Поле | Тип | Описание |
|---|---|---|
| `section` | `string` | Раздел на сайте |
| `id` | `number` | WordPress post ID |
| `url` | `string` | Полный URL на retailrocket.ru |

## Статистика блоков (265 файлов, 14 514 блоков)

| Тип блока | Количество |
|---|---|
| `Paragraph` | 7 933 |
| `Image` | 1 724 |
| `Heading3` | 1 387 |
| `Heading2` | 1 366 |
| `UnorderedList` | 1 190 |
| `CtaSecondaryBlock` | 202 |
| `Table` | 201 |
| `OrderedList` | 160 |
| `Quote` | 148 |
| `CtaPrimaryBlock` | 92 |
| `Formula` | 85 |
| `Factoid` | 17 |
| `Video` | 8 |
| `Heading4` | 1 |

## Статистика inline-элементов (19 140 элементов)

| Тип | Количество |
|---|---|
| `Text` | 17 746 |
| `Link` | 1 394 |

### Boolean-флаги на Text-элементах

| Флаг | Элементов |
|---|---|
| `IsBold` | 2 401 |
| `IsItalic` | 34 |
| `IsUnderline` | 0 |
| `IsStrike` | 0 |
| `IsHighlight` | 0 |
| `IsSuperscript` | 0 |
| `IsSubscript` | 0 |

## Авторы

| Author ID | Статей |
|---|---|
| `natalia-kazmina` | 140 |
| `kirill-morozov` | 102 |
| `rradmin` | 19 |
| `elizaveta-krievu` | 1 |

3 статьи не имеют привязанного автора (`AuthorIdList` пуст).
