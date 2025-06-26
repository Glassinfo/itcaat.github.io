---
title: "Как я научил TeamCity говорить с ИИ через MCP, или История одного протокола"
date: 2025-06-26T13:39:35+03:00
description: "Если вы здесь для быстрого понимания: **teamcity-mcp** — это Go-сервер, который превращает ваш TeamCity в AI-френдли ресурс через протокол Model Context Protocol. Теперь ваш ChatGPT или Claude может тригерить билды, читать логи и даже троллить вас за красные тесты. Всё это работает через JSON-RPC 2.0, потому что в 2024 году мы до сих пор используем технологии из нулевых, но теперь с ИИ."
tags: [ai]
---

_Истина где то рядом, но если хочешь разобраться как оно работает - напиши программу._

---

Если вы здесь на пару минут, то тут будем говорить техническую реализацию **teamcity-mcp**. Он превращает ваш TeamCity в AI-френдли ресурс через протокол Model Context Protocol. Теперь ваш ChatGPT или Claude может тригерить билды, читать логи и даже троллить вас за красные тесты. Всё это работает через JSON-RPC 2.0, потому что в 2024 году мы до сих пор используем технологии из нулевых, но теперь с ИИ.

Подключается в конфигах MCP очень просто:

```json
{
    "mcpServers": {
        "teamcity": {
            "command": "docker",
            "args": [
                "run",
                "--rm",
                "-i",
                "-e",
                "TC_URL",
                "-e",
                "TC_TOKEN",
                "teamcity-mcp:latest",
                "--transport",
                "stdio"
            ],
            "env": {
                "TC_URL": "https://teamcity.com",
                "TC_TOKEN": "you-token-here"
            }
        }
    }
}
```


## Что такое MCP и почему он вообще нужен?

### Теория: Проблема интеграции LLM с внешними системами

Model Context Protocol — это попытка Anthropic (создателей Claude) стандартизировать то, как LLM взаимодействуют с внешними системами. Но чтобы понять, зачем это нужно, давайте разберёмся с фундаментальными проблемами.

**Проблема #1: Context Window Limitation**

LLM имеют ограниченное окно контекста. GPT-4 может обработать ~128K токенов, Claude 3.5 — до 200K. Звучит много? А теперь попробуйте засунуть туда:
- Документацию по TeamCity API (>500K токенов)
- Историю последних 1000 билдов
- Конфигурации всех проектов
- Логи упавших тестов

Спойлер: не поместится. И даже если поместится, это будет стоить как небольшая квартира в Москве.

**Проблема #2: Hallucination в API вызовах**

LLM прекрасно генерируют код, но когда дело доходит до API вызовов, они начинают фантазировать:

```bash
# Что попросил пользователь:
"Запусти билд для проекта MyApp"

# Что сгенерировал GPT:
curl -X POST https://teamcity.com/api/builds/trigger \
  -d '{"project": "MyApp", "action": "build"}' # 🚨 Неправильный endpoint и формат
```

**Проблема #3: Отсутствие типизации и валидации**

Промпт-инжиниринг для API — это как программирование на языке без типов, компилятора и здравого смысла:

```
Промпт: "Используй TeamCity REST API для получения списка билдов"
LLM: "Конечно! Вот запрос: GET /builds" 
Реальность: Endpoint /app/rest/builds, нужна аутентификация, локаторы, пагинация...
```

### MCP как решение: Структурированный подход

MCP решает эти проблемы через:

1. **Ресурсы** — структурированные данные с чёткими схемами
2. **Инструменты** — типизированные функции с валидацией входных данных  
3. **Протокол** — стандартизированный JSON-RPC 2.0 интерфейс

По сути, MCP — это такой переводчик между миром ИИ и миром ваших любимых инструментов. Только вместо того чтобы объяснять ChatGPT, как работает ваш TeamCity через промпты (что, скажем честно, работает примерно как объяснение алгоритмов кошке), MCP предоставляет структурированный интерфейс.

### Сравнение подходов

| Подход | Промпт-инжиниринг | MCP |
|--------|------------------|-----|
| **Типизация** | Отсутствует | Строгие JSON схемы |
| **Валидация** | На совести LLM | На уровне протокола |
| **Кэширование** | Невозможно | Встроенное |
| **Отладка** | "Попробуй ещё раз" | Структурированные ошибки |
| **Масштабируемость** | Линейная деградация | Горизонтальное масштабирование |

### Почему именно TeamCity?

Да на самом деле потому что я не нашел готового решения. Ну и еще TeamCity — это как старый добрый швейцарский нож DevOps-а. Он делает всё: и билды собирает, и тесты гоняет, и артефакты хранит, и администраторов расстраивает конфигурационными файлами размером с "Войну и мир". 

И раз уж мы живём в эпоху, когда ИИ помогает нам писать код (а иногда и думать), то почему бы не дать ему доступ к нашей CI/CD-системе? Что может пойти не так? 😏

## Архитектура: Как это работает под капотом

### Основные компоненты

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   AI Agent      │    │   teamcity-mcp   │    │    TeamCity     │
│   (Claude)      │◄──►│     Server       │◄──►│    REST API     │
│                 │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
     JSON-RPC 2.0            Go 1.23+               HTTP/JSON
```

**teamcity-mcp** — это мидлвэр, который:
1. Говорит с ИИ на языке MCP (JSON-RPC 2.0 поверх HTTP/WebSocket)
2. Переводит MCP-запросы в TeamCity REST API вызовы
3. Кэширует результаты, чтобы не DDOS-ить ваш TeamCity
4. Логирует всё в JSON, как и положено в 2024 году
5. Экспортирует метрики в Prometheus, потому что мониторинг — это наша религия

### Протокол MCP: Ресурсы vs Инструменты

MCP основан на двух фундаментальных концепциях, которые отражают паттерны взаимодействия с любыми системами:

#### Теория разделения чтения и записи (CQRS для ИИ)

MCP различает две сущности по принципу Command Query Responsibility Segregation:

**Ресурсы** (Resources) — это read-only данные:
- `teamcity://projects` — все проекты
- `teamcity://buildTypes` — конфигурации сборок  
- `teamcity://builds` — история сборок
- `teamcity://agents` — билд-агенты
- `teamcity://artifacts` — артефакты сборок

**Инструменты** (Tools) — это действия:
- `trigger_build` — запустить сборку
- `cancel_build` — отменить сборку
- `pin_build` — закрепить сборку
- `set_build_tag` — установить теги
- `download_artifact` — скачать артефакты

#### Почему такое разделение важно?

1. **Безопасность**: ИИ может читать всё, но писать только через контролируемые инструменты
2. **Кэширование**: Ресурсы можно кэшировать агрессивно, инструменты — нет
3. **Аудит**: Все изменения проходят через инструменты и логируются
4. **Типизация**: Инструменты имеют строгие схемы входных данных

```json
// Ресурс — просто данные
{
  "uri": "teamcity://builds/12345",
  "name": "Build #12345",
  "mimeType": "application/json",
  "text": "{\"id\": 12345, \"status\": \"SUCCESS\"}"
}

// Инструмент — функция с валидацией
{
  "name": "trigger_build",
  "inputSchema": {
    "type": "object",
    "properties": {
      "buildTypeId": {"type": "string", "pattern": "^[A-Za-z0-9_]+$"}
    },
    "required": ["buildTypeId"]
  }
}
```

Это как readonly vs readwrite в базах данных, только для ИИ.

### Аутентификация: Потому что безопасность важна

У нас есть двухуровневая аутентификация:

1. **HMAC-токены** для аутентификации MCP-клиентов
2. **TeamCity API токены** для доступа к TeamCity

```bash
# Клиент должен прислать HMAC-токен в заголовке
Authorization: Bearer your-hmac-secret-key

# А сервер использует TeamCity API токен для вызовов
TC_TOKEN=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

**Почему только API токены?**

TeamCity API токены — это современный и безопасный способ аутентификации:
- **Ограниченные права** — можно настроить токен только для нужных операций
- **Время жизни** — токены могут автоматически истекать
- **Отзыв** — токен можно мгновенно деактивировать
- **Аудит** — все действия привязаны к токену, а не к пользователю

Да, это может показаться избыточным, но лучше иметь два уровня защиты, чем объяснять начальству, почему ИИ потер все ваши билды.

## Техническая реализация: Где магия встречается с реальностью

### Go 1.23 и выбор технологий

#### Теория выбора языка для сетевых сервисов

Выбор языка программирования для сетевых сервисов — это всегда компромисс между несколькими факторами:

**Performance vs Development Speed**
```
C/C++     ████████████████████████████████████████ (performance)
Rust      ████████████████████████████████████████ (performance)
Go        ███████████████████████████████████████  (performance)
Java      ████████████████████████████████████     (performance)
Node.js   ████████████████████████████             (performance)
Python    ████████████████████                     (performance)

Python    ████████████████████████████████████████ (dev speed)
Go        ███████████████████████████████████████  (dev speed)
Node.js   ████████████████████████████████████     (dev speed)
Java      ████████████████████████████             (dev speed)
Rust      ████████████████████                     (dev speed)
C/C++     ████████████                             (dev speed)
```

**Concurrency Models**

Разные языки используют разные модели конкурентности:

- **Thread-based** (Java, C#): Тяжёлые потоки ОС, context switching overhead
- **Event Loop** (Node.js, Python asyncio): Один поток, неблокирующий I/O
- **Actor Model** (Erlang, Akka): Изолированные процессы с message passing
- **CSP** (Go): Горутины + каналы, легковесные потоки

#### Почему именно Go?

```go
// Конкурентность из коробки
go func() {
    // Горутина весит ~2KB памяти
    // vs ~2MB для thread в Java
    handleMCPRequest(request)
}()

// Каналы для синхронизации
results := make(chan BuildResult, 100)
go buildWorker(results)
```

Почему Go? Потому что:
- **Быстрый** (относительно Python): Компилируемый язык с GC
- **Простой в деплое**: Один статически скомпонованный бинарный файл
- **Хорошо работает с JSON**: Встроенная поддержка, struct tags
- **Отличная стандартная библиотека**: HTTP, JSON, crypto — всё есть
- **Горутины**: M:N threading model, миллионы конкурентных соединений
- **Стабильность экосистемы**: Сообщество Go не создаёт новый фреймворк каждую неделю

Основные зависимости:
```go
// Веб-сокеты для MCP
github.com/gorilla/websocket v1.5.1

// Структурированное логирование
go.uber.org/zap v1.26.0

// Метрики
github.com/prometheus/client_golang v1.18.0

// Тестирование
github.com/stretchr/testify v1.8.4
```

Никаких тяжёлых фреймворков, никаких ORM, никаких "enterprise-решений". Просто Go, стандартная библиотека и несколько проверенных временем пакетов.

### Кэширование: Потому что TeamCity не резиновый

#### Теория кэширования в распределённых системах

Кэширование — это классический trade-off между консистентностью и производительностью. В контексте MCP-сервера у нас есть несколько стратегий:

**1. No Cache** — всегда актуальные данные, медленно
**2. Write-Through** — пишем в кэш и БД одновременно
**3. Write-Behind** — пишем в кэш, в БД асинхронно
**4. TTL-based** — данные устаревают через время

#### Наша реализация: TTL + Read-Through

```go
type Cache struct {
    data map[string]cacheEntry
    mu   sync.RWMutex  // Читатели не блокируют друг друга
    ttl  time.Duration
}

type cacheEntry struct {
    value     interface{}
    expiresAt time.Time
    mutex     sync.Mutex  // Per-entry lock для cache stampede protection
}

func (c *Cache) GetOrFetch(key string, fetcher func() (interface{}, error)) (interface{}, error) {
    // Cache stampede protection: если несколько горутин запрашивают один ключ,
    // только одна будет делать fetch, остальные подождут
    c.mu.RLock()
    entry, exists := c.data[key]
    c.mu.RUnlock()
    
    if exists && time.Now().Before(entry.expiresAt) {
        metrics.RecordCacheHit()
        return entry.value, nil
    }
    
    // Только одна горутина на ключ будет делать fetch
    entry.mutex.Lock()
    defer entry.mutex.Unlock()
    
    // Double-check после получения lock
    if time.Now().Before(entry.expiresAt) {
        return entry.value, nil
    }
    
    value, err := fetcher()
    if err != nil {
        return nil, err
    }
    
    c.set(key, value)
    metrics.RecordCacheMiss()
    return value, nil
}
```

#### Почему не Redis?

**Redis плюсы:**
- Персистентность
- Распределённость  
- Богатые структуры данных
- Pub/Sub

**Redis минусы для нашего случая:**
- Сетевая задержка (даже localhost — это ~0.1ms)
- Дополнительная точка отказа
- Сериализация/десериализация данных
- Оверхед управления памятью

```
Latency comparison:
In-memory map:     ~10ns
Redis localhost:   ~100μs (в 10,000 раз медленнее!)
Redis network:     ~1ms   (в 100,000 раз медленнее!)
```

Простой in-memory кэш с TTL в 10 секунд. Почему не Redis? Потому что для этой задачи Redis — это как использовать танк для похода в магазин. Иногда простые решения работают лучше сложных.

#### Cache Eviction Policy

```go
// LRU eviction при достижении лимита памяти
func (c *Cache) cleanup() {
    ticker := time.NewTicker(c.ttl / 2)
    for range ticker.C {
        c.mu.Lock()
        now := time.Now()
        for key, entry := range c.data {
            if now.After(entry.expiresAt) {
                delete(c.data, key)
            }
        }
        c.mu.Unlock()
    }
}
```

### Логирование: Структурированное и осмысленное

```go
logger.Info("Triggering build",
    "buildTypeId", buildTypeId,
    "branchName", branchName,
    "requestId", requestId,
    "duration", duration.Seconds())
```

Используем Zap для структурированного логирования. Каждый запрос получает уникальный ID, каждая операция логируется с контекстом. Потому что дебажить логи в стиле "что-то пошло не так" — это прошлый век.

### Метрики: Наблюдаемость превыше всего

```go
// Количество MCP запросов
mcp_requests_total{method="trigger_build",status="success"} 42

// Время ответа TeamCity API
teamcity_api_duration_seconds{endpoint="/app/rest/builds"} 0.123

// Статистика кэша
cache_hits_total 156
cache_misses_total 23
```

Потому что если вы не можете это измерить, то вы не можете это улучшить. И потому что красивые графики в Grafana — это наш способ показать боссу, что мы не просто так зарплату получаем.

## Деплой: От localhost до production

### Docker: Потому что "у меня работает" — не аргумент

```dockerfile
FROM golang:1.23-alpine AS builder
# ... сборка приложения
FROM scratch
COPY --from=builder /app/teamcity-mcp /teamcity-mcp
ENTRYPOINT ["/teamcity-mcp"]
```

Мультистейдж сборка с финальным образом на `scratch`. Итоговый образ весит меньше 20MB. Потому что в 2024 году Docker образы размером в гигабайт — это моветон.

### Kubernetes: Потому что у нас есть YAML-зависимость

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: teamcity-mcp
spec:
  replicas: 1  # Потому что stateful сервис
  # ... остальная конфигурация
```

Да, мы используем Kubernetes для одного Pod-а. Да, это может показаться избыточным. Но когда вам нужно будет масштабировать, обновлять или мониторить, вы скажете спасибо.

### Переменные окружения: Конфигурация без конфигов

```bash
# Обязательные
export TC_URL="https://your-teamcity-server.com"
export TC_TOKEN="your-api-token"

# Опциональные
export SERVER_SECRET="your-hmac-secret"
export LOG_LEVEL="info"
export CACHE_TTL="10s"
```

Никаких конфигурационных файлов. Всё через переменные окружения. Потому что 12-factor app — это не просто модное слово, это способ жизни.

## Тестирование: Потому что баги в production — это стыдно

### Теория тестирования: Пирамида тестов vs Трофей

#### Классическая пирамида тестов

```
        /\
       /  \     E2E Tests (медленные, хрупкие, дорогие)
      /____\
     /      \
    /        \   Integration Tests
   /__________\
  /            \
 /              \ Unit Tests (быстрые, стабильные, дешёвые)
/________________\
```

#### Современный подход: Testing Trophy

```
      /\
     /  \    E2E
    /____\
   /      \
  /        \  Integration (больший фокус)
 /          \
/____________\
/            \
\    Unit    / Static Analysis (линтеры, типы)
 \__________/
```

### Unit тесты: Моки, моки везде

#### Принципы хорошего мокинга

```go
// ❌ Плохо: мокаем всё подряд
func TestBadExample(t *testing.T) {
    mockTime := &MockTime{}
    mockLogger := &MockLogger{}
    mockMetrics := &MockMetrics{}
    mockCache := &MockCache{}
    mockTC := &MockTeamCityClient{}
    
    // Тест превращается в проверку взаимодействия моков
}

// ✅ Хорошо: мокаем только внешние зависимости
func TestTriggerBuild(t *testing.T) {
    mockTC := &MockTeamCityClient{}
    mockTC.On("TriggerBuild", mock.MatchedBy(func(req *TriggerRequest) bool {
        return req.BuildTypeID == "MyProject_Build" && 
               req.BranchName == "main"
    })).Return(&BuildResponse{ID: "12345"}, nil)
    
    // Реальные объекты для бизнес-логики
    cache := cache.New(10 * time.Second)
    logger := logging.NewTest()
    
    handler := NewHandler(mockTC, cache, logger)
    
    result, err := handler.TriggerBuild(context.Background(), &TriggerBuildRequest{
        BuildTypeID: "MyProject_Build",
        BranchName:  "main",
    })
    
    assert.NoError(t, err)
    assert.Equal(t, "12345", result.BuildID)
    mockTC.AssertExpectations(t)
}
```

#### Test Doubles: Классификация

```go
// Dummy — заглушка, не используется в тесте
type DummyLogger struct{}
func (d *DummyLogger) Info(msg string, fields ...interface{}) {}

// Stub — возвращает заранее определённые данные
type StubTeamCity struct{}
func (s *StubTeamCity) GetBuilds() []Build { 
    return []Build{{ID: "123", Status: "SUCCESS"}} 
}

// Mock — проверяет взаимодействие
type MockTeamCity struct {
    mock.Mock
}
func (m *MockTeamCity) TriggerBuild(req *TriggerRequest) (*BuildResponse, error) {
    args := m.Called(req)
    return args.Get(0).(*BuildResponse), args.Error(1)
}

// Spy — записывает вызовы для последующей проверки
type SpyMetrics struct {
    calls []string
}
func (s *SpyMetrics) RecordRequest(method string) {
    s.calls = append(s.calls, method)
}
```

Мокаем TeamCity клиент, потому что unit тесты должны быть быстрыми и независимыми. Никто не хочет поднимать весь TeamCity для тестирования парсинга JSON.

### Integration тесты: Где реальность встречается с кодом

```go
func TestMCPProtocol(t *testing.T) {
    // Запускаем реальный сервер
    server := setupTestServer(t)
    defer server.Close()
    
    // Тестируем MCP протокол
    response := sendMCPRequest(server.URL, initializeRequest)
    assert.Equal(t, "2024-11-05", response.ProtocolVersion)
}
```

Интеграционные тесты запускают настоящий HTTP сервер и тестируют полный MCP протокол. Потому что unit тесты не покажут вам, что вы забыли правильно сериализовать JSON.

## GitHub Actions: Потому что CI/CD — это наша ДНК

```yaml
name: CI/CD Pipeline
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v4
      with:
        go-version: 1.23
    - run: make test
```

Простой пайплайн: тестируем, линтим, собираем, деплоим. Ничего лишнего, всё по делу. Потому что сложные пайплайны — это как сложные конфигурации: они ломаются в самый неподходящий момент.

## Безопасность: Потому что хакеры не спят

### Trivy сканирование

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    format: 'sarif'
```

Сканируем код и зависимости на уязвимости. Потому что в 2024 году безопасность — это не опция, это необходимость.

### Принципы безопасности

1. **Минимальные права** — TeamCity токен должен иметь только необходимые разрешения
2. **Шифрование** — TLS 1.3 для всех соединений
3. **Аудит** — все действия логируются с полным контекстом
4. **Изоляция** — сервер работает в контейнере без root прав

## Мониторинг: Потому что "работает" !== "работает хорошо"

### Теория observability: Три столпа

Современный мониторинг строится на трёх столпах:

#### 1. Metrics (Метрики) — "Что происходит?"

```go
// Counter — монотонно растущее значение
mcp_requests_total{method="trigger_build",status="success"} 42

// Gauge — текущее значение
active_connections 15
memory_usage_bytes 1048576

// Histogram — распределение значений
http_request_duration_seconds_bucket{le="0.1"} 100
http_request_duration_seconds_bucket{le="0.5"} 200
http_request_duration_seconds_bucket{le="1.0"} 250
```

#### 2. Logs (Логи) — "Что пошло не так?"

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "error",
  "message": "Failed to trigger build",
  "buildTypeId": "MyProject_Build",
  "error": "connection timeout",
  "trace_id": "abc123",
  "duration_ms": 5000
}
```

#### 3. Traces (Трассировка) — "Где узкое место?"

```
Request: trigger_build [total: 1.2s]
├── validate_request [50ms]
├── check_cache [5ms] ❌ cache miss
├── teamcity_api_call [1.1s] ⚠️ slow!
│   ├── auth [100ms]
│   ├── http_request [900ms] 🐌
│   └── parse_response [100ms]
└── update_cache [45ms]
```

### Prometheus метрики в деталях

```go
// SLI (Service Level Indicators) — что измеряем
var (
    // Latency — как быстро отвечаем
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "mcp_request_duration_seconds",
            Help: "Time spent processing MCP requests",
            Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1.0, 2.0, 5.0},
        },
        []string{"method", "status"},
    )
    
    // Throughput — сколько обрабатываем
    requestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "mcp_requests_total",
            Help: "Total number of MCP requests",
        },
        []string{"method", "status"},
    )
    
    // Error Rate — сколько падает
    errorsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "mcp_errors_total", 
            Help: "Total number of MCP errors",
        },
        []string{"method", "error_type"},
    )
    
    // Saturation — насколько загружены
    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "mcp_active_connections",
            Help: "Number of active MCP connections",
        },
    )
)
```

#### SLO (Service Level Objectives)

```yaml
# Что обещаем пользователям
slos:
  availability: 99.9%  # Uptime
  latency_p95: 500ms   # 95% запросов быстрее 500ms
  latency_p99: 1000ms  # 99% запросов быстрее 1s
  error_rate: <1%      # Меньше 1% ошибок
```

```
# Сколько запросов обрабатываем
mcp_requests_total{method="trigger_build",status="success"} 42

# Как быстро отвечает TeamCity
teamcity_api_duration_seconds{endpoint="/builds"} 0.123

# Насколько эффективен кэш
cache_hit_ratio 0.87
```

### Health checks

```go
func (h *HealthHandler) ReadinessCheck() error {
    // Проверяем доступность TeamCity
    if err := h.teamcity.Ping(); err != nil {
        return fmt.Errorf("TeamCity unavailable: %w", err)
    }
    return nil
}
```

Liveness и readiness пробы. Потому что Kubernetes должен знать, когда ваш сервис готов к работе, а когда его нужно перезапустить.

## Реальные примеры использования

### Сценарий 1: "Умный" код-ревью

```
Human: Посмотри на этот PR и запусти соответствующие тесты

AI: Анализирую изменения в PR #123...
    Обнаружены изменения в модуле `auth`
    Запускаю билд Auth_UnitTests...
    Билд #456 запущен, статус: RUNNING
    Ожидаемое время выполнения: 3 минуты
```

### Сценарий 2: Анализ проблем с билдами

```
Human: Почему билды Auth_Service падают последние 2 дня?

AI: Анализирую историю сборок Auth_Service...
    Найдено 23 неудачных билда за последние 48 часов
    Основные причины:
    - 67% падений из-за "connection timeout to database"
    - 21% из-за "test_user_permissions failed"
    - 12% из-за "insufficient memory"
    
    Рекомендую проверить подключение к БД и увеличить memory limit
```

### Сценарий 3: Автоматизация релизов

```
Human: Подготовь релиз версии 1.2.3

AI: Проверяю готовность к релизу...
    ✅ Все тесты прошли успешно
    ✅ Билд main ветки зелёный
    ✅ Артефакты собраны
    
    Запускаю билд Release_Deploy с параметром VERSION=1.2.3...
    Билд #789 запущен. Артефакты будут доступны через 5 минут.
```

## Что дальше: Планы на будущее

### Краткосрочные планы

#### 1. Расширение функциональности

**Больше инструментов:**
```go
// Управление агентами
type AgentManager interface {
    EnableAgent(agentID string) error
    DisableAgent(agentID string) error  
    RestartAgent(agentID string) error
}

// Управление проектами  
type ProjectManager interface {
    CreateProject(spec *ProjectSpec) (*Project, error)
    UpdateProject(id string, spec *ProjectSpec) error
    ArchiveProject(id string) error
}
```

#### 2. Real-time уведомления

**WebHooks интеграция:**
```go
// TeamCity WebHook -> MCP Event Stream
type EventStream struct {
    subscribers map[string]chan Event
    mu          sync.RWMutex
}

func (e *EventStream) Subscribe(clientID string) <-chan Event {
    ch := make(chan Event, 100)
    e.mu.Lock()
    e.subscribers[clientID] = ch
    e.mu.Unlock()
    return ch
}

// MCP Event для AI агента
type BuildEvent struct {
    Type      string `json:"type"`      // "build.started", "build.finished"
    BuildID   string `json:"buildId"`
    Status    string `json:"status"`
    Duration  int    `json:"duration"`
    Timestamp string `json:"timestamp"`
}
```

#### 3. Batch операции

**Параллельное выполнение:**
```go
type BatchRequest struct {
    Operations []Operation `json:"operations"`
    MaxConcurrency int     `json:"maxConcurrency"`
}

func (h *Handler) ExecuteBatch(ctx context.Context, req *BatchRequest) (*BatchResponse, error) {
    semaphore := make(chan struct{}, req.MaxConcurrency)
    results := make([]OperationResult, len(req.Operations))
    
    var wg sync.WaitGroup
    for i, op := range req.Operations {
        wg.Add(1)
        go func(idx int, operation Operation) {
            defer wg.Done()
            semaphore <- struct{}{}        // Acquire
            defer func() { <-semaphore }() // Release
            
            results[idx] = h.executeOperation(ctx, operation)
        }(i, op)
    }
    
    wg.Wait()
    return &BatchResponse{Results: results}, nil
}
```

### Долгосрочные планы

#### 1. AI-powered анализ

**Машинное обучение для DevOps:**

```python
# Анализ паттернов падений билдов
class BuildFailurePredictor:
    def __init__(self):
        self.model = RandomForestClassifier()
        
    def extract_features(self, build_history):
        return {
            'time_since_last_commit': ...,
            'changed_files_count': ...,
            'test_count_delta': ...,
            'branch_age_days': ...,
            'author_failure_rate': ...,
        }
    
    def predict_failure_probability(self, build_context):
        features = self.extract_features(build_context)
        return self.model.predict_proba([features])[0][1]

# Интеграция с MCP
{
    "name": "predict_build_failure",
    "description": "Predict build failure probability",
    "inputSchema": {
        "type": "object", 
        "properties": {
            "buildTypeId": {"type": "string"},
            "branchName": {"type": "string"}
        }
    }
}
```

#### 2. Multi-CI/CD поддержка

**Унифицированный интерфейс:**
```go
// Абстракция над разными CI/CD системами
type CIProvider interface {
    ListProjects() ([]Project, error)
    TriggerBuild(req *BuildRequest) (*Build, error)
    GetBuildStatus(buildID string) (*BuildStatus, error)
}

// Конкретные реализации
type TeamCityProvider struct { /* ... */ }
type JenkinsProvider struct { /* ... */ }
type GitLabCIProvider struct { /* ... */ }
type GitHubActionsProvider struct { /* ... */ }

// MCP сервер работает с любым провайдером
func NewMCPServer(provider CIProvider) *Server {
    return &Server{provider: provider}
}
```

#### 3. Observability++

**Grafana интерфейс для MCP:**
```typescript
// React компонент для мониторинга MCP сессий
interface MCPSession {
    id: string;
    clientId: string;
    startTime: Date;
    requestCount: number;
    activeTools: string[];
    lastActivity: Date;
}

const MCPDashboard: React.FC = () => {
    const [sessions, setSessions] = useState<MCPSession[]>([]);
    
    return (
        <div>
            <h2>Active MCP Sessions</h2>
            {sessions.map(session => (
                <SessionCard key={session.id} session={session} />
            ))}
        </div>
    );
};
```

## Заключение: Что мы получили

Мы создали мост между миром ИИ и миром DevOps. Теперь ваш AI-ассистент может:

- 🔍 **Анализировать** состояние билдов и найти причины проблем
- 🚀 **Автоматизировать** запуск билдов и деплоев
- 📊 **Мониторить** состояние CI/CD процессов
- 🔧 **Управлять** жизненным циклом билдов

Но самое главное — мы сделали это правильно:
- ✅ Следуя принципам 12-factor app
- ✅ С proper observability
- ✅ С нормальными тестами
- ✅ С security-first подходом
- ✅ С production-ready архитектурой

## Полезные ссылки

- [Репозиторий проекта](https://github.com/itcaat/teamcity-mcp)
- [Документация MCP](https://modelcontextprotocol.io/)
- [TeamCity REST API](https://www.jetbrains.com/help/teamcity/rest-api.html)
- [Prometheus метрики](https://prometheus.io/docs/concepts/metric_types/)

---

*P.S. Если вы дочитали до сюда — вы настоящий техлид. Остальные прочитали только TL;DR и сразу перешли к копипасте команд из README.*

*P.P.S. Да, я знаю, что можно было использовать готовый TeamCity Go SDK. Но где в этом фан? Иногда написать своё велосипедостроение — это не только способ лучше понять проблему, но и возможность сделать именно то, что нужно, без лишних зависимостей.*

*P.P.P.S. Если вы нашли баги — welcome to issues. Если хотите что-то улучшить — welcome to pull requests. Если хотите просто поругать код — welcome to Twitter, но там я вас не читаю.*

---

## Дополнительные материалы для изучения

### Теоретические основы

#### Distributed Systems
- **CAP Theorem** — почему нельзя иметь Consistency, Availability и Partition tolerance одновременно
- **ACID vs BASE** — транзакционные модели в распределённых системах
- **Event Sourcing** — хранение состояния как последовательность событий
- **CQRS** — разделение команд и запросов (применяется в MCP)

#### Concurrency Patterns
- **Actor Model** — изолированные процессы с message passing (Erlang, Akka)
- **CSP** — Communicating Sequential Processes (Go channels)
- **Reactor Pattern** — event-driven архитектура (Node.js, Netty)
- **Worker Pool** — ограничение конкурентности через семафоры

#### API Design
- **Richardson Maturity Model** — уровни RESTfulness
- **GraphQL vs REST** — когда использовать что
- **gRPC** — высокопроизводительный RPC
- **JSON-RPC 2.0** — простой протокол удалённых вызовов (используется в MCP)

### Практические инструменты

#### Observability Stack
```yaml
# Prometheus + Grafana + Jaeger
monitoring:
  metrics: prometheus      # Временные ряды
  visualization: grafana   # Дашборды  
  tracing: jaeger         # Распределённая трассировка
  logs: loki              # Логи как метрики
  alerting: alertmanager  # Уведомления
```

#### Testing Tools
```bash
# Статический анализ
golangci-lint run

# Fuzzing тестирование  
go test -fuzz=FuzzMCPHandler

# Load testing
k6 run load_test.js

# Chaos engineering
chaos-monkey --target=teamcity-mcp
```

#### Security Tools
```bash
# Vulnerability scanning
trivy fs .
snyk test

# SAST (Static Application Security Testing)
semgrep --config=auto .
gosec ./...

# Container security
docker scout cves teamcity-mcp:latest
```
