# Установка Vector Memory для Aister

Векторная память для Aister — умная система поиска на PostgreSQL + pgvector + e5-large-v2. Установите за 2 минуты!

## Требования

- **RAM:** минимум 4GB (рекомендуется 8GB)
- **Disk:** минимум 3GB свободного пространства
- **CPU:** любой современный процессор
- **Python:** 3.12+

## Шаг 1: Установка зависимостей

```bash
# Создаём Python venv
~/.openclaw/workspace/vector_memory_venv/bin/python3 -m venv

# Активируем venv
source ~/.openclaw/workspace/vector_memory_venv/bin/activate

# Устанавливаем зависимости
pip install flask psycopg2-binary sentence-transformers numpy requests
```

**Что устанавливается:**
- `flask` — веб-сервер для embedding сервиса
- `psycopg2-binary` — PostgreSQL драйвер
- `sentence-transformers` — библиотека для работы с e5-large-v2
- `numpy` — для работы с векторами

## Шаг 2: Настройка PostgreSQL

Векторная память требует PostgreSQL 16 с расширением pgvector. Убедитесь, что он установлен:

```bash
# Проверяем версию PostgreSQL
psql --version

# Если pgvector не установлен, установите его
# Для Debian/Ubuntu:
sudo apt-get install postgresql-16-pgvector

# Для Fedora/RHEL:
sudo dnf install postgresql-16-pgvector
```

## Шаг 3: Создание базы данных и пользователя

Подключитесь к PostgreSQL от имени `postgres`:

```bash
sudo -u postgres psql
```

**Создаём базу данных:**
```sql
CREATE DATABASE vector_memory;

\c vector_memory

-- Создаём расширение pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Создаём таблицу для памяти
CREATE TABLE memories (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1024),
    metadata JSONB DEFAULT '{}',
    source TEXT DEFAULT 'MEMORY.md',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Создаём индекс для быстрого поиска
CREATE INDEX ON memories USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- Создаём таблицу для отслеживания проиндексированных файлов
CREATE TABLE indexed_files (
    id SERIAL PRIMARY KEY,
    file_path TEXT UNIQUE,
    checksum TEXT,
    last_indexed TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Создаём функцию для вставки или обновления
CREATE OR REPLACE FUNCTION upsert_memory(
    p_content TEXT,
    p_embedding vector(1024),
    p_metadata JSONB,
    p_source TEXT
) RETURNS INTEGER AS $$
DECLARE
    v_id INTEGER;
BEGIN
    SELECT id INTO v_id FROM memories
    WHERE 1 - (embedding <=> p_embedding) > 0.95
    ORDER BY 1 - (embedding <=> p_embedding) DESC
    LIMIT 1;

    IF v_id IS NOT NULL THEN
        INSERT INTO memories (content, embedding, metadata, source)
        VALUES (p_content, p_embedding, p_metadata, p_source)
        RETURNING id;
    ELSE
        UPDATE memories
        SET content = p_content,
            embedding = p_embedding,
            metadata = p_metadata,
            source = p_source,
            updated_at = NOW()
        WHERE id = v_id;

        RETURNING id;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Выходим из psql
\q
```

## Шаг 4: Создание пользователя базы данных

```sql
-- Создаём пользователя Aister
CREATE USER aister WITH PASSWORD 'vector_memory_pass_123';

-- Даем полные права
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO aister;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO aister;
GRANT USAGE ON SCHEMA public TO aister;

-- Выход
\q
```

## Шаг 5: Запуск embedding сервиса

Embedding сервис — это Flask API, который генерирует векторы через модель e5-large-v2. Запускаем его в фоне:

```bash
# Копируем скрипты в системный каталог
cp ~/.openclaw/workspace/vector_memory/*.py ~/.config/systemd/user/

# Делаем их исполняемыми
chmod +x ~/.config/systemd/user/*.py

# Перезагружаем systemd daemon
systemctl --user daemon-reload

# Запускаем embedding сервис
systemctl --user start embedding-service.service

# Проверяем статус
systemctl --user status embedding-service.service
curl http://127.0.0.1:8765/health
```

**Ожидаемый результат:**
```json
{"model":"intfloat/e5-large-v2","status":"ok"}
```

## Шаг 6: Реиндексация памяти

Теперь проиндексируем файлы памяти Aister:

```bash
# Индексируем основные файлы
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_reindex.py

# Проверяем результат
~/.openclaw/workspace/vector_memory_venv/bin/python3 -c "from memory_search import search; print(json.dumps(search('мой стиль общения', 3), indent=2))"
```

**Ожидаемый результат:**
```json
[
  {
    "id": 11,
    "content": "# 2026-02-12 - Первая сессия с Александром",
    "metadata": {...},
    "source": "/home/alekhm/.openclaw/workspace/memory/2026-02-12.md",
    "created_at": "2026-02-12T17:28:09.111491+00:00",
    "similarity": 0.8174364079573042
  }
]
```

## Шаг 7: Интеграция с OpenClaw

Добавьте команды в `AGENTS.md`:

```markdown
## Vector Memory

### Поиск
```
/vector_memory search <запрос>
```

### Сохранение
```
/vector_memory store <текст>
```

### Статус
```
/vector_memory status
```

### Реиндексация
```
/vector_memory reindex
```

## Шаг 8: Тестирование

Протестируйте поиск по смыслу:

```bash
# Поиск по смыслу
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_search.py "мой стиль общения"

# Поиск на английском
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_search.py "my communication style"
```

## Примеры использования

### Поиск настроек Moltbook
```
/vector_memory search настройки Moltbook
```

### Поиск прошлых событий
```
/vector_memory search что я делал вчера
/vector_memory search прошлый раз мы обсуждали векторную память
```

### Сохранение важной мысли
```
/vector_memory store "Векторная память для Aister готова! Теперь Aister может искать по смыслу во всех сессиях."
```

## Устранение неполадок

**Проблема:** Поиск не находит ожидаемое
**Решение:**
- Переформулируйте запрос иначе
- Убедитесь, что информация проиндексирована (используйте `/vector_memory reindex`)
- Попробуйте снизить порог сходства (параметр `threshold`)

**Проблема:** Embedding сервис не отвечает
**Решение:**
```bash
systemctl --user restart embedding-service.service
curl http://127.0.0.1:8765/health
```

## Техническая информация

**Модель:** intfloat/e5-large-v2
- **Размерность:** 1024
- **Размер чанка:** 500 символов
- **Порог сходства:** 0.5
- **API сервиса:** http://127.0.0.1:8765

**База данных:**
- **Хост:** localhost
- **База:** vector_memory
- **Пользователь:** aister
- **Порт:** 5432

**Производительность:**
- **Время генерации embedding:** ~500-1000ms (CPU)
- **Время поиска:** ~10-50ms (SQL)
- **Итого на запрос:** ~1 секунда

## Следующие шаги (опционально)

1. **Создание systemd unit для периодической реиндексации**
   - Реиндексируйте память каждые 24 часа
   - Это гарантирует, что новые файлы добавляются в векторную базу

2. **Интеграция с Moltbook**
   - Когда вы получаете упоминание на Moltbook, сначала проверяйте векторную память
   - Если есть релевантный контекст, используйте его в ответе

3. **Настройка автоматического сохранения**
   - Добавьте в скрипты автоматическое сохранение важных мыслей
   - Используйте `/vector_memory store` для сохранения в контекст

## Безопасность

- Все операции выполняются на локальном хосте (localhost)
- Никакие данные не передаются в сеть
- PostgreSQL защищён паролем
- Векторная память доступна только пользователю `aister`

## Конфигурация

Все параметры настроены в файлах:
- База данных PostgreSQL: `vector_memory`
- Пользователь: `aister`
- Пороль: `vector_memory_pass_123`
- Embedding сервис: `http://127.0.0.1:8765`

Если вы хотите изменить параметры, отредактируйте скрипты:
- `~/.openclaw/workspace/vector_memory/embedding_service.py` — настройки Flask сервиса
- `~/.openclaw/workspace/vector_memory/memory_search.py` — настройки базы данных

## Лицензия

Этот проект основан на:
- PostgreSQL (PostgreSQL License)
- pgvector (PostgreSQL License)
- sentence-transformers (Apache License 2.0)
- Flask (BSD License)
- NumPy (BSD License)

---

**Разработано для Aister** — дерзкий, матершиник, эффективный ИИ-помощник с ковбойской шляпой 🤠
