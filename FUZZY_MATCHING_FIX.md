# 🔧 Исправление: Fuzzy-подсказки для get_table_schema

## Проблема

При вызове `get_table_schema` с **неправильным названием таблицы**, бэкенд возвращал fuzzy-подсказки в поле `message`, но UI **не отображал их**, показывая только короткое сообщение об ошибке из поля `error`.

### Пример проблемы

**Запрос:**
```
get_table_schema("свод_впо1_всего_р2_1_2_1")  # Неправильное название
```

**Ответ бэкенда (Python):**
```json
{
  "success": false,
  "error": "Таблица 'свод_впо1_всего_р2_1_2_1' не найдена",
  "suggestions": [
    "свод_впо1_всего_р2_1_21",
    "свод_впо1_всего_р2_1_2_2",
    "свод_впо1_всего_р2_1_2_3",
    "свод_впо1_всего_р2_1_2_4",
    "свод_впо1_всего_р2_1_5_1"
  ],
  "message": "❌ Таблица 'свод_впо1_всего_р2_1_2_1' не найдена\n\n💡 Возможно, вы имели в виду одну из этих таблиц:\n  • свод_впо1_всего_р2_1_21\n  • свод_впо1_всего_р2_1_2_2\n  • ...\n\n🔍 Попробуйте вызвать get_table_schema с одним из предложенных названий."
}
```

**Что показывал UI до исправления:**
```
❌ Ошибка
Таблица 'свод_впо1_всего_р2_1_2_1' не найдена
```

**Что было утеряно:**
- Список из 5 похожих таблиц (`suggestions`)
- Подсказка с правильными названиями

---

## Решение

### 1. Изменения в UI ([web_frontend_v2/ui.js:597-628](web_frontend_v2/ui.js#L597-L628))

Добавили логику для отображения fuzzy-подсказок:

```javascript
} else if (result.error) {
    // 🔧 FIX: Показываем fuzzy-подсказки если они есть
    let errorContent = '';

    if (result.suggestions && result.suggestions.length > 0) {
        // Есть fuzzy подсказки - показываем красиво
        const suggestionsHTML = result.suggestions.map(name =>
            `<div class="suggestion-item">${getIcon('arrow-right', 12)} <code>${name}</code></div>`
        ).join('');

        errorContent = `
            <div class="tool-argument" style="color: #f44336;">${result.error}</div>
            <div class="tool-argument" style="margin-top: 12px;">
                <strong>${getIcon('lightbulb', 14)} Возможно, вы имели в виду:</strong>
                <div class="suggestions-list" style="margin-top: 8px;">
                    ${suggestionsHTML}
                </div>
            </div>
        `;
    } else if (result.message) {
        // Используем message вместо error, если оно есть
        errorContent = `<div class="tool-argument" style="color: #f44336; white-space: pre-wrap;">${result.message}</div>`;
    } else {
        // Простая ошибка
        errorContent = `<div class="tool-argument" style="color: #f44336;">${result.error}</div>`;
    }

    resultSection.innerHTML = `
        <div class="tool-section-title">${getIcon('x-circle', 12)} Ошибка</div>
        ${errorContent}
    `;
    if (window.lucide) lucide.createIcons();
}
```

**Логика:**
1. Проверяем наличие поля `suggestions` (массив похожих таблиц)
2. Если есть — отображаем их красиво с иконками
3. Если нет — показываем `message` или просто `error`

### 2. Стили для подсказок ([web_frontend_v2/style.css:620-654](web_frontend_v2/style.css#L620-L654))

Добавили красивые стили для подсказок:

```css
/* 💡 Fuzzy-подсказки для таблиц */
.suggestions-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.suggestion-item {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    background: #FFF8E1;
    border-left: 3px solid #FFA726;
    border-radius: 4px;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
    font-size: 13px;
    color: #333;
    transition: all 0.2s ease;
}

.suggestion-item:hover {
    background: #FFE0B2;
    transform: translateX(4px);
    cursor: pointer;
}

.suggestion-item code {
    background: rgba(0, 0, 0, 0.05);
    padding: 2px 6px;
    border-radius: 3px;
    font-family: 'Courier New', monospace;
    font-size: 12px;
    color: #D84315;
}
```

**Дизайн:**
- Теплый желтый фон (#FFF8E1) с оранжевой левой границей
- Hover эффект (светлее + сдвиг вправо)
- Моноширинный шрифт для названий таблиц

---

## Результат

**Теперь UI отображает:**

```
❌ Ошибка
Таблица 'свод_впо1_всего_р2_1_2_1' не найдена

💡 Возможно, вы имели в виду:
  → свод_впо1_всего_р2_1_21
  → свод_впо1_всего_р2_1_2_2
  → свод_впо1_всего_р2_1_2_3
  → свод_впо1_всего_р2_1_2_4
  → свод_впо1_всего_р2_1_5_1
```

**С визуальными улучшениями:**
- Каждая подсказка в отдельной рамке
- Иконка стрелки → перед каждым названием
- Hover эффект при наведении
- Названия таблиц в `<code>` блоках

---

## Как работает Fuzzy Matching

**Алгоритм:** Расстояние Левенштейна (Levenshtein Distance)

**Параметры:**
- `max_suggestions = 5` — максимум 5 подсказок
- `max_distance = max(len(table_name) * 0.3, 3)` — максимум 30% от длины названия

**Пример:**
```
Искомое: "свод_впо1_всего_р2_1_2_1"  (24 символа)
Найдено:  "свод_впо1_всего_р2_1_21"   (23 символа, distance=1)
          "свод_впо1_всего_р2_1_2_2"  (24 символа, distance=1)
          ...

max_distance = max(24 * 0.3, 3) = 7.2 → все с distance ≤ 7 попадут в список
```

**Код (Python):**
- [sql_tool.py:450-507](../skynet_simple/tools/rag/sql_tool.py#L450-L507) — функции `_levenshtein_distance` и `_find_similar_table_names`
- [sql_tool.py:550-606](../skynet_simple/tools/rag/sql_tool.py#L550-L606) — использование fuzzy matching в `get_table_schema_tool`

---

## Тестирование

Создан тест: [test_fuzzy_ui.py](test_fuzzy_ui.py)

**Запуск:**
```bash
python3 test_fuzzy_ui.py
```

**Результат:**
```
======================================================================
ТЕСТ FUZZY MATCHING ДЛЯ GET_TABLE_SCHEMA
======================================================================

🔍 Проверяем таблицу: 'свод_впо1_всего_р2_1_2_1'

❌ Таблица не найдена (0 колонок)

💡 Найдено 5 похожих таблиц:

  • свод_впо1_всего_р2_1_21 (distance=1)
  • свод_впо1_всего_р2_1_2_2 (distance=1)
  • свод_впо1_всего_р2_1_2_3 (distance=1)
  • свод_впо1_всего_р2_1_2_4 (distance=1)
  • свод_впо1_всего_р2_1_5_1 (distance=1)
```

---

## Итог

✅ **Исправлено:** UI теперь отображает fuzzy-подсказки при ошибке "таблица не найдена"
✅ **Бэкенд:** Fuzzy matching уже работал, просто UI не показывал результаты
✅ **UX:** Пользователь видит 5 похожих таблиц и может выбрать правильную
✅ **Дизайн:** Красивое оформление с иконками и hover эффектами

**Дата исправления:** 2025-10-24
**Автор:** Claude Code (Sonnet 4.5)
