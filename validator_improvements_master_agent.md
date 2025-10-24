**Дата создания**: 24 October 2025, 19:38 (МСК)

---

# Улучшения валидатора для работы с MasterAgent

## Проблема

В системе были обнаружены три проблемы при работе валидатора с MasterAgent:

### 1. Отсутствие специальных инструкций для MasterAgent

**Симптомы:**
- Валидатор не знал, что у MasterAgent есть только инструменты делегирования
- Валидатор не понимал специфику работы координатора
- Не было контроля качества делегирования задач подагентам

**Причина:**
В методе `_get_tools_info_by_agent_type()` класса `ResponseValidator` ([response_validator.py:401-459](skynet_simple/agents/response_validator.py#L401-L459)) была обработка только для:
- `"auth"` - агент авторизации
- `"common"` - агент поддержки
- `"combined_v2"` - RAG-only агент

Но НЕ было обработки для `"master"` - типа агента координатора!

**Что происходило:**
```python
def _get_tools_info_by_agent_type(self, agent_type: str) -> str:
    if agent_type == "auth":
        return "..."
    elif agent_type == "common":
        return "..."
    elif agent_type == "combined_v2":
        return "..."

    return ""  # ← Для "master" возвращалась пустая строка!
```

MasterAgent возвращал `agent_type="master"` ([master_agent.py:54-56](skynet_simple/agents/master_agent.py#L54-L56)):
```python
def get_agent_type(self) -> str:
    """Тип агента для валидации"""
    return "master"
```

Этот тип передавался в валидатор через pipeline ([base_agent.py:115](skynet_simple/agents/base_agent.py#L115)):
```python
async for event in self.pipeline.stream(
    messages=messages,
    chat_id=chat_id,
    system_prompt=system_prompt,
    tools_schema=self.get_tools_schema(),
    agent_type=self.get_agent_type(),  # ← "master" для MasterAgent
    ...
)
```

### 2. Пустые сообщения валидатора в UI

**Симптомы:**
- Когда валидатор вызывал инструмент без текста, в UI появлялось пустое облачко диалога
- Это создавало визуальный шум в интерфейсе

**Причина:**
В [response_validator.py:616](skynet_simple/agents/response_validator.py#L616) возвращалось:
```python
"self_correction_message": content or ""
```

Если LLM возвращал `content` с пробелами (например `"   "` или какой-то whitespace), то проверка `if correction_message:` в [validation.py:108](skynet_simple/agents/stages/validation.py#L108) не отфильтровывала это, и пустое сообщение отправлялось в UI.

### 3. Результаты инструментов валидатора не показывались в UI

**Симптомы:**
- В консоли браузера: `⚠️ updateToolCallResult: не найдены statusSpan или resultSection`
- Результаты вызовов инструментов валидатором не отображались

**Причина:**
ValidationStage отправлял события `role: "tool"` без поля `name`:

[validation.py:165-175](skynet_simple/agents/stages/validation.py#L165-L175):
```python
tool_result_message = {
    "role": "tool",
    "tool_call_id": tool_result["tool_call_id"],
    "content": str(tool_result["result"])
    # ❌ НЕТ ПОЛЯ "name"!
}
yield tool_result_message
```

А в UI код ожидал это поле ([app.js:203-207](web_frontend_v2/app.js#L203-L207)):
```javascript
if (data.role === 'tool') {
    const toolName = data.name || 'unknown';  // ← Ожидаем data.name!

    // Проверяем это подагент или обычный инструмент
    if (toolName && toolName.startsWith('call_subagent_')) {
        // Показываем результат подагента
    } else {
        // Показываем результат обычного инструмента
        updateToolCallResult(currentToolCall, result);
    }
}
```

Без поля `name` код не мог определить тип инструмента и правильно показать результат.

## Решение

### 1. Добавлена обработка типа "master" в валидаторе

**Файл:** [response_validator.py:458-483](skynet_simple/agents/response_validator.py#L458-L483)

Добавлен новый блок:
```python
elif agent_type == "master":
    # 🎭 Мастер-агент координатор
    return """

## 🎭 ОСОБЕННОСТИ МАСТЕР-АГЕНТА (КООРДИНАТОР):

⚠️ **У этого агента ТОЛЬКО инструменты делегирования!**
   У него есть только:
   - call_subagent_analytic (вызов аналитического подагента)
   - (возможно другие подагенты в будущем)

🎯 **ГЛАВНАЯ РОЛЬ:** Мастер-агент НЕ выполняет задачи сам - он ДЕЛЕГИРУЕТ их подагентам!

✅ **ПРАВИЛЬНО:** Если мастер-агент вызывает call_subagent_analytic для любого аналитического запроса
✅ **ПРАВИЛЬНО:** Если мастер-агент формулирует задачу тепло и по-дружески для подагента
✅ **ПРАВИЛЬНО:** Если мастер-агент просит подагента быть ОЧЕНЬ внимательным
✅ **ПРАВИЛЬНО:** Если мастер-агент представляет результаты подагента пользователю понятным языком

❌ **ОШИБКА:** Если мастер-агент пытается выполнить аналитику сам (у него нет инструментов для этого!)
❌ **ОШИБКА:** Если мастер-агент говорит "я найду данные" вместо делегирования подагенту
❌ **ОШИБКА:** Если мастер-агент не вызывает подагента когда нужна работа с данными
❌ **ОШИБКА:** Если мастер-агент формулирует задачу для подагента сухо и без просьбы быть внимательным

💡 **ЭТОТ АГЕНТ РАБОТАЕТ ИНАЧЕ:** Он координирует работу подагентов, а не выполняет задачи напрямую.
💡 **ВАЖНО:** Подагент должен получить четкую задачу с просьбой быть очень внимательным!
"""
```

**Что это дает:**
- Валидатор теперь понимает специфику MasterAgent
- Валидатор может проверить, что MasterAgent правильно делегирует задачи
- Валидатор может вызвать `call_subagent_analytic` если MasterAgent забыл это сделать
- Валидатор контролирует качество формулировки задач для подагентов

### 2. Добавлена очистка пробелов в сообщениях валидатора

**Файл:** [response_validator.py:616](skynet_simple/agents/response_validator.py#L616)

**Было:**
```python
"self_correction_message": content or ""
```

**Стало:**
```python
"self_correction_message": (content or "").strip()
```

**Что это дает:**
- Если `content` содержит только пробелы - после `.strip()` получится `""`
- Проверка `if correction_message:` в ValidationStage правильно отфильтрует пустые строки
- Пустые облачка диалога не появятся в UI

### 3. Добавлено поле "name" в события tool results валидатора

**Файл:** [validation.py:174-177](skynet_simple/agents/stages/validation.py#L174-L177)

**Было:**
```python
# 🆕 СТРИМИМ в SSE
yield tool_result_message
```

**Стало:**
```python
# 🆕 СТРИМИМ в SSE (добавляем name для UI)
sse_message = tool_result_message.copy()
sse_message["name"] = tool_result["function_name"]  # Добавляем имя функции для UI
yield sse_message
```

**Что это дает:**
- В событии `role: "tool"` теперь есть поле `name` с именем функции
- UI код может правильно определить тип инструмента
- Для `call_subagent_*` результат показывается в специальном блоке "Получен ответ от подагента"
- Не возникает ошибка "не найдены statusSpan или resultSection"

## Архитектурный контекст

### Как работает валидатор с инструментами

1. **Агент передает свои инструменты валидатору:**
   ```python
   # base_agent.py:110-115
   async for event in self.pipeline.stream(
       tools_schema=self.get_tools_schema(),  # ← Инструменты агента
       agent_type=self.get_agent_type(),       # ← Тип агента
       ...
   )
   ```

2. **Pipeline передает в ValidationStage:**
   ```python
   # agent_pipeline.py:174-179
   async for validation_event in self.validation_stage.execute(
       ctx,
       agent_type,                  # ← "master" для MasterAgent
       tools_schema=tools_schema,   # ← Список подагентов как инструментов
       tool_executor=self.tool_executor
   )
   ```

3. **Валидатор вызывает LLM с инструментами агента:**
   ```python
   # response_validator.py:540-546
   response = await self.llm_client.chat_completion(
       messages=messages,
       tools=tools_schema,  # ← Инструменты агента!
       max_tokens=2000
   )
   ```

4. **Если валидатор нашел ошибку - вызывает инструменты:**
   ```python
   # response_validator.py:567-591
   if tool_calls:
       for tool_call in tool_calls:
           function_name = tool_call.get("function", {}).get("name")
           # Выполняем через tool_executor агента
           result = await tool_executor(function_name, arguments, chat_id)
   ```

### Инструменты MasterAgent

MasterAgent предоставляет подагентов как инструменты:

```python
# master_agent.py:203-211
def get_tools_schema(self) -> List[Dict]:
    """Получить схемы инструментов (подагентов)"""
    tools = []

    # Добавляем каждого подагента как инструмент
    for name, subagent in self.subagents.items():
        tools.append(subagent.to_tool_schema())

    return tools
```

Схема генерируется из имени подагента:

```python
# sub_agent.py:322-342
def to_tool_schema(self) -> Dict:
    return {
        "type": "function",
        "function": {
            "name": f"call_subagent_{self.name.lower().replace(' ', '_')}",
            "description": self.description,
            "parameters": {
                "type": "object",
                "properties": {
                    "task": {
                        "type": "string",
                        "description": "Задача для подагента"
                    },
                    ...
                }
            }
        }
    }
```

Для подагента с `name="analytic"` создается инструмент `call_subagent_analytic`.

## Влияние на систему

### До исправлений

1. **Валидатор не понимал MasterAgent:**
   - Получал пустые инструкции для типа "master"
   - Не мог правильно проверить работу координатора
   - Не контролировал качество делегирования

2. **Пустые сообщения в UI:**
   - Визуальный шум в интерфейсе
   - Непонятные пустые облачка диалога

3. **Результаты не показывались:**
   - Ошибки в консоли браузера
   - Результаты вызовов инструментов терялись

### После исправлений

1. **Валидатор понимает MasterAgent:**
   - Знает что у него только инструменты делегирования
   - Проверяет правильность делегирования
   - Может сам вызвать `call_subagent_analytic` если MasterAgent забыл

2. **Чистый UI:**
   - Нет пустых сообщений
   - Показываются только содержательные комментарии валидатора

3. **Результаты отображаются правильно:**
   - Для подагентов: специальный блок "Получен ответ от подагента"
   - Для обычных инструментов: обновление в tool call блоке
   - Нет ошибок в консоли

## Примеры работы

### Пример 1: Валидатор исправляет MasterAgent

**Запрос пользователя:**
> Сколько организаций с лицензией в Москве?

**MasterAgent ответил без вызова подагента:**
> Я найду эту информацию в базе данных.

**Валидатор обнаружил ошибку:**
```
❌ Ошибка: MasterAgent не вызвал подагента когда нужна работа с данными
```

**Валидатор сам вызвал инструмент:**
```python
call_subagent_analytic({
    "task": "Привет, товарищ! Пожалуйста, найди количество организаций с лицензией в Москве. Посмотри очень внимательно!"
})
```

**В UI:**
1. Не показывается пустое сообщение валидатора (благодаря `.strip()`)
2. Показывается блок "Вызов подагента: call_subagent_analytic"
3. Показывается блок "Получен ответ от подагента" с результатом (благодаря `name` в событии)
4. MasterAgent формирует финальный ответ на основе результата подагента

### Пример 2: Валидатор проверяет качество делегирования

**MasterAgent сформулировал задачу сухо:**
```python
call_subagent_analytic({
    "task": "Найди организации в Москве"
})
```

**Валидатор проверил:**
```
❌ Ошибка: Задача сформулирована сухо, без просьбы быть внимательным
```

**Валидатор исправил:**
```python
call_subagent_analytic({
    "task": "Привет, товарищ! Пожалуйста, найди очень внимательно все организации с лицензией в Москве. Заранее благодарю!"
})
```

## Технические детали

### Файлы изменены

1. **skynet_simple/agents/response_validator.py**
   - Строка 458-483: Добавлена обработка `agent_type == "master"`
   - Строка 616: Добавлен `.strip()` для очистки пробелов

2. **skynet_simple/agents/stages/validation.py**
   - Строки 174-177: Добавлено поле `name` в SSE события tool results

### Зависимости

- **MasterAgent** ([master_agent.py](skynet_simple/agents/master_agent.py)) - возвращает `agent_type="master"`
- **BaseAgent** ([base_agent.py](skynet_simple/agents/base_agent.py)) - передает `agent_type` в pipeline
- **AgentPipeline** ([agent_pipeline.py](skynet_simple/agents/agent_pipeline.py)) - передает в ValidationStage
- **ValidationStage** ([stages/validation.py](skynet_simple/agents/stages/validation.py)) - вызывает валидатор
- **UI** ([web_frontend_v2/app.js](web_frontend_v2/app.js)) - обрабатывает события от валидатора

## Будущие улучшения

1. **Расширение типов агентов:**
   - Добавить обработку для других типов координаторов
   - Создать базовый класс для агентов-координаторов

2. **Улучшение UI для валидации:**
   - Показывать отдельно исправления валидатора
   - Добавить индикатор "Валидатор исправляет..."

3. **Метрики валидации:**
   - Логировать как часто валидатор исправляет каждый тип агента
   - Анализировать типичные ошибки для улучшения промптов

## Заключение

Исправления улучшили работу валидатора с MasterAgent на трех уровнях:

1. **Логический уровень** - валидатор понимает специфику координатора
2. **UI уровень** - чистый интерфейс без пустых сообщений и корректное отображение результатов
3. **Архитектурный уровень** - правильная передача метаданных между компонентами

Теперь валидатор может эффективно контролировать работу MasterAgent и автоматически исправлять ошибки делегирования.
