# Установка Vector Memory для Aister

Векторная память для Aister — умная система поиска на PostgreSQL + pgvector + e5-large-v2.

## Предупреждения

> **Важно:** Перед установкой ознакомьтесь с требованиями:
> - **Сеть:** Первый запуск скачает модель e5-large-v2 (~1.3GB) с HuggingFace
> - **Привилегии:** Требуются root для системных пакетов и PostgreSQL superuser
> - **Пароли:** Никогда не используйте hardcoded пароли из примеров в продакшене

## Требования

- **RAM:** минимум 4GB (рекомендуется 8GB)
- **Disk:** минимум 3GB свободного пространства
- **CPU:** любой современный процессор
- **Python:** 3.12+

## Шаг 1: Установка зависимостей

```bash
# Создаём Python venv
python3 -m venv ~/.openclaw/workspace/vector_memory_venv

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

Векторная память требует PostgreSQL 16 с расширением pgvector:

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

> **Безопасность:** Замените `YOUR_SECURE_PASSWORD` на надёжный уникальный пароль!

```sql
-- Создаём пользователя (замените пароль!)
CREATE USER aister WITH PASSWORD 'YOUR_SECURE_PASSWORD';

-- Даем права (минимально необходимые)
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO aister;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO aister;
GRANT USAGE ON SCHEMA public TO aister;

-- Выход
\q
```

## Шаг 5: Настройка переменных окружения

Создайте файл с переменными окружения:

```bash
# Создаём конфигурационный файл
mkdir -p ~/.config/vector-memory
cat > ~/.config/vector-memory/env << 'EOF'
# Database configuration
export VECTOR_MEMORY_DB_HOST="localhost"
export VECTOR_MEMORY_DB_PORT="5432"
export VECTOR_MEMORY_DB_NAME="vector_memory"
export VECTOR_MEMORY_DB_USER="aister"
export VECTOR_MEMORY_DB_PASSWORD="YOUR_SECURE_PASSWORD"

# Embedding service
export EMBEDDING_SERVICE_URL="http://127.0.0.1:8765"
export EMBEDDING_PORT="8765"

# Memory settings
export VECTOR_MEMORY_DIR="$HOME/.openclaw/workspace/memory"
export VECTOR_MEMORY_CHUNK_SIZE="500"
export VECTOR_MEMORY_THRESHOLD="0.5"
export VECTOR_MEMORY_LIMIT="5"
EOF

# Ограничиваем доступ к файлу с паролем
chmod 600 ~/.config/vector-memory/env
```

## Шаг 6: Копирование скриптов

```bash
# Создаём директорию для скриптов
mkdir -p ~/.openclaw/workspace/vector_memory

# Копируем скрипты из skill
cp embedding_service.py ~/.openclaw/workspace/vector_memory/
cp memory_search.py ~/.openclaw/workspace/vector_memory/
cp memory_reindex.py ~/.openclaw/workspace/vector_memory/

# Делаем исполняемыми
chmod +x ~/.openclaw/workspace/vector_memory/*.py
```

## Шаг 7: Запуск embedding сервиса

> **Важно:** Первый запуск скачает модель ~1.3GB с HuggingFace!

```bash
# Загружаем переменные окружения
source ~/.config/vector-memory/env

# Запускаем сервис вручную (для тестирования)
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/embedding_service.py

# В другом терминале проверяем статус
curl http://127.0.0.1:8765/health
```

**Ожидаемый результат:**
```json
{"model":"intfloat/e5-large-v2","status":"ok","embedding_dim":1024}
```

### Автозапуск через systemd (опционально)

Создайте systemd unit для автоматического запуска:

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/embedding-service.service << 'EOF'
[Unit]
Description=Vector Memory Embedding Service
After=network.target

[Service]
Type=simple
EnvironmentFile=%h/.config/vector-memory/env
ExecStart=%h/.openclaw/workspace/vector_memory_venv/bin/python3 %h/.openclaw/workspace/vector_memory/embedding_service.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
EOF

# Перезагружаем systemd
systemctl --user daemon-reload

# Включаем автозапуск
systemctl --user enable embedding-service.service

# Запускаем
systemctl --user start embedding-service.service

# Проверяем статус
systemctl --user status embedding-service.service
```

## Шаг 8: Реиндексация памяти

```bash
# Загружаем переменные окружения
source ~/.config/vector-memory/env

# Индексируем файлы памяти
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_reindex.py
```

**Ожидаемый результат:**
```
Indexing MEMORY.md...
  Generating embeddings for 42 chunks...
  Indexed 42 chunks from MEMORY.md

Reindex complete:
  Files processed: 1
  Chunks indexed: 42
  Total memories in DB: 42
```

## Шаг 9: Тестирование поиска

```bash
# Загружаем переменные окружения
source ~/.config/vector-memory/env

# Тестовый поиск
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_search.py "мой стиль общения" -j
```

## Примеры использования

### Поиск по смыслу
```bash
source ~/.config/vector-memory/env
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_search.py "настройки Moltbook"
```

### Поиск на английском
```bash
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_search.py "my communication style"
```

### JSON вывод
```bash
~/.openclaw/workspace/vector_memory_venv/bin/python3 ~/.openclaw/workspace/vector_memory/memory_search.py "что я делал вчера" -j
```

## Переменные окружения

| Переменная | Описание | По умолчанию |
|------------|----------|--------------|
| `VECTOR_MEMORY_DB_HOST` | Хост PostgreSQL | `localhost` |
| `VECTOR_MEMORY_DB_PORT` | Порт PostgreSQL | `5432` |
| `VECTOR_MEMORY_DB_NAME` | Имя базы данных | `vector_memory` |
| `VECTOR_MEMORY_DB_USER` | Пользователь БД | `aister` |
| `VECTOR_MEMORY_DB_PASSWORD` | Пароль БД | *(обязательно)* |
| `EMBEDDING_SERVICE_URL` | URL embedding сервиса | `http://127.0.0.1:8765` |
| `EMBEDDING_MODEL` | Модель для embeddings | `intfloat/e5-large-v2` |
| `EMBEDDING_PORT` | Порт embedding сервиса | `8765` |
| `VECTOR_MEMORY_DIR` | Директория с файлами памяти | `~/.openclaw/workspace/memory` |
| `VECTOR_MEMORY_CHUNK_SIZE` | Размер чанка | `500` |
| `VECTOR_MEMORY_THRESHOLD` | Порог сходства | `0.5` |
| `VECTOR_MEMORY_LIMIT` | Макс. результатов | `5` |

## Устранение неполадок

**Проблема:** Модель не скачивается
**Решение:** Проверьте интернет-соединение. Модель скачивается с HuggingFace.

**Проблема:** Ошибка подключения к БД
**Решение:** Проверьте переменные окружения и убедитесь, что PostgreSQL запущен.

**Проблема:** Embedding сервис не отвечает
**Решение:**
```bash
systemctl --user restart embedding-service.service
curl http://127.0.0.1:8765/health
```

**Проблема:** Поиск не находит ожидаемое
**Решение:**
- Переформулируйте запрос
- Запустите `/reindex_memory`
- Снизьте порог: `VECTOR_MEMORY_THRESHOLD=0.4`

## Безопасность

- Все операции выполняются локально (после скачивания модели)
- Пароли хранятся в файле с правами 600
- База данных защищена паролем
- Не коммитьте файл `~/.config/vector-memory/env` в git!

---

**Разработано для Aister** — дерзкий, матершинник, эффективный ИИ-помощник с ковбойской шляпой 🤠
