# Архитектура памяти для подагентов в системе MasterAgent

## Обзор

Документ описывает правильную реализацию системы памяти для подагентов в архитектуре с MasterAgent. Система обеспечивает:
- Память между запросами пользователя (через Redis)
- Память внутри одного запроса при множественных вызовах подагента
- Консистентность данных между компонентами

## Проблема

В RAG-системах с архитектурой MasterAgent → SubAgent возникает проблема с памятью подагента:

### Сценарий 1: Множественные вызовы в одном запросе
```
Пользователь: "Скажи подагенту что меня зовут Алия, а потом спроси у него моё имя"

ПЕРВЫЙ вызов подагента:
  История: []
  Подагент: "Запомнил, вас зовут Алия"

ВТОРОЙ вызов подагента:
  История: [] ← всё ещё пуста!
  Подагент: "Я не знаю вашего имени" ❌
```

### Сценарий 2: Разные запросы пользователя
```
ЗАПРОС 1: "Меня зовут Алия"
  → Подагент запоминает → сохраняет в Redis

ЗАПРОС 2: "Как меня зовут?"
  → Загружается из Redis: []
  → Подагент: "Я не знаю" ❌
```

## Архитектура решения

### Компоненты системы

```
┌─────────────────────────────────────────────────────────────┐
│                     Web Memory Service                       │
│  - Загрузка истории из Redis                                │
│  - Передача контекста в API                                 │
│  - Приём SSE событий и сохранение в Redis                   │
└─────────────────────────────────────────────────────────────┘
                           ↓ ↑
                    HTTP/SSE stream
                           ↓ ↑
┌─────────────────────────────────────────────────────────────┐
│                       API Server                             │
│  - Приём запросов с subagent_contexts                       │
│  - Передача в MasterAgent                                   │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                      MasterAgent                             │
│  - Хранит локальную копию _subagent_contexts               │
│  - Обновляет после каждого вызова подагента                │
│  - Передаёт историю в SubAgent                             │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                       SubAgent                               │
│  - Получает историю как параметр                           │
│  - Работает с контекстом                                   │
│  - Отправляет события через callback                       │
└─────────────────────────────────────────────────────────────┘
                           ↓
                    SSE события
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                    MemoryService                             │
│  - Кеш в памяти (memory_cache)                             │
│  - Персистентность в Redis                                  │
│  - Ключи: subagent:{name}:{chat_id}                        │
└─────────────────────────────────────────────────────────────┘
```

## Реализация

### 1. Сохранение событий подагента

**Файл**: `web_memory_service/app.py`

```python
# Обработка SSE событий от подагента
elif role == "subagent_delegation":
    # Задача от мастера к подагенту
    msg = MemoryMessage(
        role="user",
        content=data.get("task", ""),
        timestamp=time.time()
    )
    await memory_service.add_subagent_message(user_id, subagent_name, msg)

elif role == "subagent_completion":
    # Финальный ответ подагента
    msg = MemoryMessage(
        role="assistant",
        content=data.get("response", ""),
        timestamp=time.time()
    )
    await memory_service.add_subagent_message(user_id, subagent_name, msg)
```

**Что происходит**: События стримятся через SSE и немедленно сохраняются в `memory_cache` и Redis.

### 2. Загрузка истории в начале запроса

**Файл**: `web_memory_service/app.py`

```python
# Загружаем историю подагентов (если master)
subagent_contexts = {}
if agent_version == "master":
    subagent_contexts["analytic"] = await memory_service.get_subagent_context(
        user_id, "analytic", context_window=50
    )

# Отправляем в API
json={
    "message": message_text,
    "chat_id": user_id,
    "context": context,
    "subagent_contexts": subagent_contexts,  # ← Передаём историю
    "agent_version": agent_version
}
```

**Что происходит**:
- При новом запросе загружаем историю из `memory_cache` (или Redis если кеша нет)
- Передаём в API вместе с основным контекстом

### 3. Передача истории в MasterAgent

**Файл**: `skynet_simple/api_server.py`

```python
# Принимаем subagent_contexts
subagent_contexts = data.get('subagent_contexts', {})

# Передаём в MasterAgent
async for openai_message in agent.stream_with_context(
    messages,
    chat_id,
    session_logger,
    subagent_contexts=subagent_contexts  # ← Передаём
):
    # ...
```

**Файл**: `skynet_simple/agents/base_agent.py`

```python
async def stream_with_context(self,
                             messages: List[Dict[str, str]],
                             chat_id: int,
                             session_logger=None,
                             skip_validation: bool = False,
                             custom_system_prompt: str = None,
                             subagent_contexts: Dict[str, list] = None):
    # Устанавливаем в MasterAgent
    if subagent_contexts and hasattr(self, 'set_subagent_contexts'):
        self.set_subagent_contexts(subagent_contexts)
```

### 4. Локальное обновление истории (КЛЮЧЕВОЙ МОМЕНТ!)

**Файл**: `skynet_simple/agents/master_agent.py`

```python
async def call_subagent(self, subagent_name: str, task: str,
                       chat_id: int, context: list) -> str:
    try:
        # Получаем текущую историю подагента
        subagent_history = self._subagent_contexts.get(subagent_name, [])

        # Передаём подагенту
        result = await subagent.execute_task(
            task=task,
            chat_id=chat_id,
            context=context,
            tool_callback=self._subagent_tool_callback,
            session_logger=self._session_logger,
            history=subagent_history  # ← Передаём историю
        )

        # ВАЖНО: Обновляем локальную копию после выполнения
        if subagent_name not in self._subagent_contexts:
            self._subagent_contexts[subagent_name] = []

        # Добавляем задачу (user message)
        self._subagent_contexts[subagent_name].append({
            "role": "user",
            "content": task
        })

        # Добавляем ответ (assistant message)
        self._subagent_contexts[subagent_name].append({
            "role": "assistant",
            "content": result
        })

        return result
```

**Почему это важно**:
- При первом вызове: `subagent_history = []`
- После выполнения: обновляем локально `_subagent_contexts`
- При втором вызове: `subagent_history = [2 сообщения]` ✅
- Параллельно всё сохраняется в Redis через SSE события

### 5. Хранение в Redis

**Файл**: `alert_bot_skynet/domain/memory/service.py`

```python
def _get_subagent_key(self, chat_id: int, subagent_name: str) -> str:
    """Ключ для хранения истории подагента"""
    return f"subagent:{subagent_name}:{chat_id}"

async def add_subagent_message(self, chat_id: int,
                               subagent_name: str,
                               message: Message) -> None:
    """Добавить сообщение подагента в историю"""
    key = self._get_subagent_key(chat_id, subagent_name)

    # Загружаем из кеша или Redis
    if key not in self.memory_cache:
        if self.redis:
            messages = await self.redis.load_history(key)
            if messages:
                self.memory_cache[key] = messages
            else:
                self.memory_cache[key] = []

    # Добавляем сообщение
    self.memory_cache[key].append(message)

    # Сохраняем в Redis
    if self.redis:
        await self.redis.save_history(key, self.memory_cache[key])

async def get_subagent_context(self, chat_id: int,
                               subagent_name: str,
                               context_window: int = 50) -> List[Dict]:
    """Получить контекст подагента"""
    key = self._get_subagent_key(chat_id, subagent_name)

    # Проверяем кеш, если нет - загружаем из Redis
    if key not in self.memory_cache:
        if self.redis:
            messages = await self.redis.load_history(key)
            if messages:
                self.memory_cache[key] = messages
            else:
                self.memory_cache[key] = []

    history = self.memory_cache.get(key, [])

    # Берём последние N сообщений и форматируем для LLM
    context_messages = history[-context_window:] if history else []
    return [msg.to_dict() for msg in context_messages]
```

## Два уровня памяти

### Уровень 1: Локальная память (в рамках одного запроса)

**Где**: `MasterAgent._subagent_contexts`

**Как работает**:
1. Инициализируется при получении запроса (из Redis)
2. Обновляется после каждого вызова подагента
3. Используется для последующих вызовов в том же запросе

**Пример**:
```python
# Первый вызов
_subagent_contexts["analytic"] = []

# После первого вызова
_subagent_contexts["analytic"] = [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
]

# Второй вызов использует обновлённую историю
```

### Уровень 2: Персистентная память (между запросами)

**Где**: `MemoryService.memory_cache` + Redis

**Как работает**:
1. События от подагента сохраняются через SSE
2. `add_subagent_message()` обновляет кеш и Redis
3. `get_subagent_context()` читает из кеша (или Redis если кеш пуст)
4. При следующем запросе история загружается из кеша/Redis

**Ключи в Redis**: `subagent:analytic:{chat_id}`

## Сравнение с памятью MasterAgent

| Аспект | MasterAgent | SubAgent |
|--------|-------------|----------|
| **Память внутри запроса** | В массиве `messages` LLM | В `_subagent_contexts` MasterAgent |
| **Память между запросами** | Redis (`chat_history:{chat_id}`) | Redis (`subagent:analytic:{chat_id}`) |
| **Обновление в запросе** | Автоматически (LLM context) | Вручную после каждого вызова |
| **Загрузка в начале** | `memory_service.get_context_for_agent()` | `memory_service.get_subagent_context()` |

## Поток данных (полная картина)

```
ЗАПРОС 1: "Скажи подагенту моё имя Алия, а потом спроси у него"

┌──────────────────────────────────────────────────────────────┐
│ 1. web_memory_service                                         │
│    - Загружает из Redis: []                                   │
│    - subagent_contexts = {"analytic": []}                     │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. MasterAgent                                                │
│    - _subagent_contexts = {"analytic": []}                    │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. ПЕРВЫЙ вызов SubAgent                                     │
│    - history = []                                             │
│    - Ответ: "Запомнил, вас зовут Алия"                       │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. Обновление MasterAgent._subagent_contexts                 │
│    - Добавляем user message (задача)                         │
│    - Добавляем assistant message (ответ)                     │
│    - _subagent_contexts = {"analytic": [2 сообщения]}        │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 5. SSE события → web_memory_service                          │
│    - subagent_delegation → сохраняется в Redis               │
│    - subagent_completion → сохраняется в Redis               │
│    - Redis: subagent:analytic:12345 = [2 сообщения]          │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 6. ВТОРОЙ вызов SubAgent                                     │
│    - history = [2 сообщения] ← из _subagent_contexts!        │
│    - Видит историю!                                           │
│    - Ответ: "Вас зовут Алия" ✅                               │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 7. Обновление _subagent_contexts                             │
│    - Добавляем ещё 2 сообщения                               │
│    - _subagent_contexts = {"analytic": [4 сообщения]}        │
│    - Сохраняется в Redis через SSE                           │
└──────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════

ЗАПРОС 2: "Как меня зовут?"

┌──────────────────────────────────────────────────────────────┐
│ 1. web_memory_service                                         │
│    - Загружает из Redis: [4 сообщения]                       │
│    - subagent_contexts = {"analytic": [4 сообщения]}         │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. MasterAgent                                                │
│    - _subagent_contexts = {"analytic": [4 сообщения]}        │
└──────────────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. Вызов SubAgent                                            │
│    - history = [4 сообщения]                                  │
│    - Видит всю историю!                                       │
│    - Ответ: "Вас зовут Алия" ✅                               │
└──────────────────────────────────────────────────────────────┘
```

## Важные детали реализации

### 1. Singleton MemoryService

```python
# alert_bot_skynet/domain/memory/service.py
memory_service = MemoryService(max_history=100, context_window=10)

# Один экземпляр на весь контейнер
# memory_cache живёт в памяти этого экземпляра
```

### 2. Кеш между запросами

`memory_cache` НЕ очищается между запросами, поэтому:
- При первой загрузке проверяется Redis
- При последующих запросах данные уже в кеше
- Обновления идут одновременно в кеш и Redis

### 3. Формат сообщений

Для LLM используется стандартный OpenAI формат:
```python
{
    "role": "user" | "assistant" | "tool",
    "content": "...",
    "tool_calls": [...],      # для assistant с вызовами
    "tool_call_id": "...",    # для tool результатов
    "name": "tool_name"       # для tool результатов
}
```

### 4. События SSE

Типы событий для подагента:
- `subagent_delegation` → `user` message в истории
- `subagent_assistant_with_tools` → `assistant` с `tool_calls`
- `subagent_tool_result` → `tool` результат
- `subagent_completion` → финальный `assistant`
- `subagent_tool` → только для UI (не сохраняется в LLM историю)

## Логирование по сессиям

Для отладки реализовано логирование в отдельные файлы:

**Файл**: `logs/memory_sessions/memory_{chat_id}.log`

```python
def get_session_logger(chat_id: int):
    """Создать логгер для конкретной сессии"""
    log_dir = Path("/app/logs/sessions")
    log_file = log_dir / f"memory_{chat_id}.log"

    session_logger = logging.getLogger(f"session_{chat_id}")
    file_handler = logging.FileHandler(log_file, encoding='utf-8')
    # ...
    return session_logger

# Использование
slog = get_session_logger(user_id)
slog.info(f"📚 История подагента загружена: {len(messages)} сообщений")
```

**Пример лога**:
```
15:26:14 - ================================================================================
15:26:14 - 📥 НОВЫЙ ЗАПРОС от пользователя: Привет!) мы тестируем память...
15:26:14 - ✅ Сообщение пользователя сохранено в Redis
15:26:14 - 📚 История подагента 'analytic' загружена: 0 сообщений
15:26:14 -    📭 История подагента пуста
15:26:16 - 💾 Задача для подагента сохранена: пользователя зовут Алия...
15:26:17 - 💾 Финальный ответ подагента сохранён: Я запомнил...
15:26:18 - 💾 Задача для подагента сохранена: как зовут пользователя?...
15:26:18 - 💾 Финальный ответ подагента сохранён: Вас зовут Алия...
```

## Проблемы которые были решены

### Проблема 1: Пустой кеш при первой загрузке
```python
# БЫЛО (неправильно):
if key not in self.memory_cache:
    messages = await self.redis.load_history(key)
    self.memory_cache[key] = []  # ← Всегда создавали пустой!

# СТАЛО (правильно):
if key not in self.memory_cache:
    messages = await self.redis.load_history(key)
    if messages:
        self.memory_cache[key] = messages
    else:
        self.memory_cache[key] = []
```

### Проблема 2: Тип ключа в Redis
```python
# БЫЛО: ожидали только int
async def save_history(self, chat_id: int, messages):
    key = f"chat_history:{chat_id}"

# СТАЛО: поддержка строковых ключей
async def save_history(self, chat_id: int | str, messages):
    if isinstance(chat_id, str):
        key = chat_id  # Уже готовый ключ
    else:
        key = f"chat_history:{chat_id}"
```

### Проблема 3: Локальная память не обновлялась
```python
# БЫЛО: использовали ту же историю для всех вызовов
subagent_history = self._subagent_contexts.get(subagent_name, [])
result = await subagent.execute_task(history=subagent_history)
# Следующий вызов: опять та же история!

# СТАЛО: обновляем после каждого вызова
result = await subagent.execute_task(history=subagent_history)

# Обновляем локально
self._subagent_contexts[subagent_name].append({"role": "user", ...})
self._subagent_contexts[subagent_name].append({"role": "assistant", ...})
# Следующий вызов: свежая история!
```

## Тестирование

### Тест 1: Память внутри запроса
```
Запрос: "Скажи подагенту что меня зовут Алия, а потом спроси как меня зовут"

Ожидаемый результат:
- Первый вызов: подагент запоминает
- Второй вызов: подагент помнит имя ✅
```

### Тест 2: Память между запросами
```
Запрос 1: "Меня зовут Алия"
Запрос 2: "Как меня зовут?"

Ожидаемый результат:
- Запрос 2 загружает историю из Redis
- Подагент помнит имя ✅
```

### Тест 3: Проверка Redis
```bash
# Проверить что данные сохранены
docker compose exec redis redis-cli GET "subagent:analytic:12345"

# Должно вернуть JSON массив с сообщениями
```

## Заключение

Система памяти для подагентов реализована через два взаимодополняющих механизма:

1. **Локальная память** (`_subagent_contexts`) - решает проблему множественных вызовов в одном запросе
2. **Персистентная память** (Redis через `MemoryService`) - решает проблему памяти между запросами

Ключевое отличие от памяти мастер-агента:
- Мастер-агент автоматически накапливает контекст в `messages` массиве LLM
- Подагент требует явного обновления `_subagent_contexts` после каждого вызова

Это решение обеспечивает полную функциональность памяти для подагентов, аналогичную памяти мастер-агента, и позволяет подагентам корректно работать с контекстом как внутри одного запроса, так и между разными запросами пользователя.

---

**Дата**: 24 октября 2025
**Авторы**: Система разработана совместно с Claude (Anthropic)
