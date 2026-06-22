## Оглавление

1.  Текущее устройство раздела «База»
2.  Найденные проблемы
3.  Сравнение вариантов решения
4.  Рекомендуемый вариант: Read-model
5.  Как не допустить рассинхрона
6.  План внедрения по этапам
7.  План тестирования
8.  Риски и их митигация

---

## 1. Текущее устройство раздела «База» 

### 1.1. Архитектура 
В файле `app/routes/comments.py` функция `responded_comments()` строит виртуальную таблицу на лету, используя `UNION ALL` для объединения данных из нескольких таблиц.

*   **Источники данных:**
    *   Для комментариев: `comments` (финальные) + `pending_responses` (ожидающие модерации).
    *   Для жалоб: `reports` (финальные) + `pending_reports` (ожидающие модерации).
*   **Этапы формирования ответа:**
    *  Строится `UNION ALL`.
    *  Применяются фильтры, сортировка, пагинация (`OFFSET`), поиск (`LIKE '%...%'`) и `JOIN` с `post_context`.
    *  Выполняется отдельный запрос для точного `COUNT(*)`.

### 1.2. Ключевые файлы и их роль
| Файл | Роль |
| :--- | :--- |
| `app/routes/comments.py` | **Центральная логика раздела**. Здесь строятся `UNION ALL`. |
| `app/helpers/export_helpers.py` | Экспорт в Excel. Использует **те же** `UNION ALL`-запросы. |
| `app/scripts/sync_comments_analytics.py` | Отдельная синхронизация в `analytics_data`. **Третий источник правды**. |
| `app/routes/api/response_api.py` | Точки перехода из `pending` в финальные таблицы. **Ключевые места для синхронизации**. |
| `app/scripts/migrations.py` | Структура таблиц и индексов. |

---

## 2. Найденные проблемы 

### 2.1. Проблемы производительности
> **Факт:** На тестовой БД в таблице `comments` уже **50 456** записей, в `analytics_data` — **53 034**.
> **Вывод:** Проблема с производительностью **уже существует** на текущих данных.

*   **Тяжелые запросы (псевдо-SQL):**
    Для открытия страницы выполняется следующий тип запроса:
    ```sql
    SELECT * FROM (
        (SELECT ... FROM comments WHERE ...)
        UNION ALL
        (SELECT ... FROM pending_responses WHERE ...)
    ) t
    WHERE (platform = 'vk' AND date_processed > '2025-01-01')
    ORDER BY date_processed DESC
    LIMIT 20 OFFSET 0;
    ```

    **Почему это плохо:** `UNION ALL` + `ORDER BY` + `OFFSET` заставляют БД создавать и сортировать огромные временные таблицы.

    **Двойная нагрузка:** `COUNT(*)` выполняется как отдельный запрос, который полностью повторяет `UNION ALL`.

### 2.2. Проблемы с индексами
*   **Индексы есть, но они бесполезны:** В `comments` есть `FULLTEXT` индекс, но поиск в коде идёт через `LIKE '%...%'`. Это ломает возможность использовать индекс.
*   **Составные индексы:** В `comments` есть составной индекс `idx_comments_platform_status_date`, но из-за `UNION ALL` он не может быть эффективно использован для сортировки всего объединённого результата.

### 2.3. Риск рассинхрона
**Три источника данных:**
1.  Раздел «База» (виртуальная таблица).
2.  Экспорт (дублирует логику «Базы»).
3.  Аналитика (`analytics_data`), которая заполняется отдельным скриптом.

**Последствия:** Разная логика трансформации и асинхронное обновление `analytics_data` гарантируют рассинхрон.

---

## 3. Сравнение вариантов решения

| Критерий | Вариант 1: Оптимизация | Вариант 2: Lifecycle-таблицы | Вариант 3: Read-model (рекомендуемый) |
| :--- | :--- | :--- | :--- |
| **Описание** | `keyset`-пагинация, `FULLTEXT`, кэширование `COUNT`. | Физическое объединение `comments + pending` в одну таблицу. | Создание projection-таблицы `base_items` как слоя чтения. |
| **Решает проблему 1M записей?** | Нет (`UNION ALL` останется). | Да | Да |
| **Решает рассинхрон?** | Нет (источников станет 4). | Да | Да |
| **Безопасность миграции** | Высокая | Очень высокая (сломается всё) | Высокая |
| **Объем работы** | Низкий | Огромный (месяцы) | Средний (2-3 недели) |
| **Риск сломать продакшен** | Низкий | Критический | Низкий |

---

## 4. Рекомендуемый вариант: Единая read-model (`base_items`)

### 4.1. Концепция
Мы создаём таблицу `base_items` как контролируемую проекцию данных из операционных таблиц. Эта таблица станет единственным каноническим источником для чтения.

```
comments + pending_responses + reports + pending_reports
                    ↓
                sync/rebuild
                    ↓
                base_items
                    ↓
      База / Аналитика / Экспорт
```

### 4.2. Структура таблицы `base_items` (предложение)

```sql
CREATE TABLE `base_items` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `source_table` ENUM('comments','pending_responses','reports','pending_reports') NOT NULL COMMENT 'Источник записи',
  `source_id` VARCHAR(255) NOT NULL COMMENT 'ID в исходной таблице',
  `source_updated_at` TIMESTAMP NULL COMMENT 'Время обновления в источнике',
  `projection_updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Время обновления в витрине',

  `content_type` ENUM('comment','review','complaint') NOT NULL COMMENT 'Тип контента',
  `platform` VARCHAR(50) NOT NULL,
  `normalized_platform` VARCHAR(50) GENERATED ALWAYS AS (LOWER(platform)) STORED COMMENT 'Для поиска',
  `status` VARCHAR(50) NOT NULL COMMENT 'Согласован, В работе, Удалён и т.д.',
  `date_order` TIMESTAMP NOT NULL COMMENT 'Основная дата для сортировки',

  `username` VARCHAR(255),
  `comment_text` TEXT,
  `reply_text` TEXT,
  `rating_value` INT,
  `location_name` VARCHAR(500),
  `location_address` VARCHAR(500),
  `post_id` VARCHAR(255),
  `post_text` TEXT,
  `tonality_score_openai` INT,
  `searchable_text` TEXT GENERATED ALWAYS AS (
    CONCAT_WS(' ', COALESCE(comment_text, ''), COALESCE(reply_text, ''), COALESCE(username, ''), COALESCE(post_text, ''))
  ) STORED COMMENT 'Объединённый текст для поиска',

  `approved` TINYINT DEFAULT 0,
  `created_at` TIMESTAMP,
  `deleted_at` TIMESTAMP NULL,

  UNIQUE KEY `uk_source` (`source_table`, `source_id`),
  KEY `idx_platform_status_date` (`platform`, `status`, `date_order` DESC),
  KEY `idx_username` (`username`),
  KEY `idx_platform_tonality` (`platform`, `tonality_score_openai`),
  FULLTEXT KEY `idx_searchable_text` (`searchable_text`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 4.3. Ключевые индексы 
*   **`idx_platform_status_date`**: Самый важный индекс. Позволит выполнять 99% запросов к «Базе» (фильтр по платформе и статусу + сортировка по дате) мгновенно.
*   **`idx_searchable_text` (FULLTEXT)**: Обеспечит быстрый и полноценный поиск по тексту вместо медленного `LIKE '%...%'`.
*   **`uk_source`**: Гарантирует уникальность и позволяет использовать `INSERT ... ON DUPLICATE KEY UPDATE` для идемпотентной синхронизации.

---

## 5. Как не допустить рассинхрона 

1.  **Single Source of Truth:** Операционные таблицы (`comments`, `pending_responses` и т.д.) остаются единственным источником правды.
2.  **Механизм синхронизации:** Внедрить асинхронный (или синхронный для критичных операций) вызов функции `sync_to_base_items()` в точках перехода:
    *   `app/routes/api/response_api.py` (approve/reject)
    *   `app/routes/api/reports_api.py` (send/reject)
    *   `app/routes/api/comments_api.py` (edit)
3.  **Идемпотентность:** Функция синхронизации использует `INSERT ... ON DUPLICATE KEY UPDATE`, что гарантирует, что повторные вызовы не создадут дубли.
4.  **Проверка рассинхрона:** Ежечасный скрипт `check_base_items_consistency.py`:
    *   Сравнивает `COUNT(*)` из `base_items` с суммой записей из источников.
    *   Выборочно сверяет `source_updated_at` и `projection_updated_at`.
    *   При обнаружении расхождения отправляет алерт в Telegram/почту.
5.  **Полный rebuild:** Скрипт `rebuild_base_items.py`, который можно запустить вручную. Он очищает `base_items` и заново наполняет её данными из всех источников. Это полезно при сбоях или после больших миграций данных.

---

## 6. План внедрения по этапам

*   **Этап 1: Подготовка (2 дня).** Создание таблицы `base_items`, написание скриптов синхронизации и первичной загрузки.
*   **Этап 2: Синхронизация (3 дня).** Внедрение вызова `sync_to_base_items()` в ключевые API (approve/reject). Настройка мониторинга.
*   **Этап 3: Переключение «Базы» (2 дня).** Изменение `responded_comments()`: вместо `UNION ALL` читать из `base_items`. Запуск на тестовом стенде.
*   **Этап 4: Переключение Экспорта и Аналитики (2 дня).** `export_helpers.py` и `sync_comments_analytics.py` переводятся на чтение из `base_items`.
*   **Этап 5: Нагрузочное тестирование (2 дня).** Замеры на 50k, 100k, 1M записей.

---

## 7. План тестирования на 50k / 100k / 1M записей

### 7.1. Подготовка данных
*   Наполнить `comments`, `pending_responses` до целевых объёмов (50k, 100k, 1M).
*   Выполнить `rebuild_base_items.py`.

### 7.2. Сценарии тестирования
Для каждого объёма данных выполнить следующие сценарии и замерить время ответа бэкенда (в миллисекундах).

| Сценарий | Что делаем | Целевое время (мс) |
| :--- | :--- | :--- |
| **Открытие первой страницы** | `GET /billing/responded_comments?partial=table&page=1` | **< 1000 мс** |
| **Пагинация** | Переход на 10-ю страницу (keyset) | **< 700 мс** |
| **Фильтр по платформе** | `?platform=vk` | **< 500 мс** |
| **Фильтр по дате** | `?date_from=...&date_to=...` | **< 500 мс** |
| **Фильтр по статусу** | `?status=approved` | **< 500 мс** |
| **Поиск по тексту** | `?keyword=слово` (FULLTEXT) | **< 700 мс** |
| **Открытие вкладки «Жалобы»** | `?contentType=complaints` | **< 1000 мс** |
| **`COUNT(*)`** | `?partial=count` | **< 200 мс** |
| **Экспорт** | `GET /billing/export_comments` | **< 10000 мс** |

### 7.3. Метрики успеха
*   Время открытия первой страницы при 1 000 000 записей < 1 секунды.
*   Отсутствие ошибок и таймаутов при всех сценариях.
*   Полное совпадение данных в «Базе», экспорте и аналитике (проверяется через скрипт сверки).

---

## 8. Риски и их митигация

| Риск | Вероятность | Влияние | Митигация |
| :--- | :--- | :--- | :--- |
| **Задержка синхронизации** | Средняя | Среднее | Синхронная синхронизация в критичных точках; автоматический повтор при сбое |
| **Ошибка в логике трансформации** | Средняя | Высокое | Тщательное тестирование на тестовом стенде; `dry-run` режим |
| **Сбой при `rebuild`** | Низкая | Высокое | Запуск в нерабочее время; процедура отката к `UNION ALL` |
