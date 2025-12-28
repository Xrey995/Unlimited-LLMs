# Unlimited-LLMs (RVFreeLLM API)

OpenAI-compatible API-шлюз с мульти-провайдерной маршрутизацией, автоматическим fallback и единым эндпойнтом `/v1/chat/completions` для текста и изображений.

---

## 🔗 Полезные ссылки

- **Base URL:** `https://rvlautoai.ru/webhook`
- **Telegram бот:** `@FreeApiLLMbot`
- **GitHub:** `https://github.com/Xrey995/Unlimited-LLMs`

---

## 📋 Содержание

1. [🎯 Обзор API](#-обзор-api)
2. [📡 Базовая информация](#-базовая-информация)
3. [🔐 Аутентификация](#-аутентификация)
4. [📌 POST /v1/chat/completions](#-эндпойнт-post-v1chatcompletions)
5. [🖼️ Генерация изображений: Формат Markdown](#️-генерация-изображений-формат-markdown)
6. [📌 GET /v1/models](#-эндпойнт-get-v1models)
7. [📌 GET /v1/models/list](#-эндпойнт-get-v1modelslist)
8. [📌 GET /v1/providers](#-эндпойнт-get-v1providers)
9. [⏱ Rate limiting](#-rate-limiting)
10. [🧯 Fallback / Retry](#-fallback--retry)
11. [🧪 Быстрые примеры](#-быстрые-примеры)
12. [🧾 Changelog](#-changelog)

---

## 🎯 Обзор API

**Основные возможности:**
- OpenAI-compatible формат запросов/ответов (chat completions)
- Несколько провайдеров (text/image/audio/video), единый формат
- Автоматический fallback (в т.ч. cross-provider) при ошибках 5xx/пустом ответе
- Логи usage-статистики на стороне сервиса
- **ВАЖНО:** Генерация изображений возвращает Markdown со ссылками на `pollinations.ai`

---

## 📡 Базовая информация

### Base URL

```
https://rvlautoai.ru/webhook
```

### Формат данных

- Request/Response: `application/json`
- Все запросы — по HTTPS

### HTTP заголовки

**Обязательные:**
- `Authorization: Bearer YOUR_API_KEY` (формат: `rvf_full...`, 75 символов)
- `Content-Type: application/json`

---

## 🔐 Аутентификация

### Bearer Token

Передавайте ключ в заголовке:

```
Authorization: Bearer rvf_full...
```

### Формат API ключа

- **Префикс:** `rvf`
- **Длина:** 75 символов (включая префикс)
- **Символы:** латинские буквы a-z, A-Z и цифры 0-9
- **Пример:** `rvf_full...`

### Типы ключей

| Тип | Лимит запросов | Длительность | Описание |
|-----|---|---|---|
| `test` | 30 | 1 час | Тестовый доступ, ограниченный |
| `full` | Unlimited | 30 дней | Полный доступ, стандартный ключ |
| `admin` | Unlimited | Unlimited | Служебный ключ для администрирования |

### Ошибки аутентификации

| Код | Значение | Причина |
|-----|----------|---------|
| `401` | Unauthorized | Неверный/истекший ключ |
| `401` | Key Expired | Ключ истёк (test через 1ч, full через 30дн) |
| `403` | Key Revoked | Ключ отозван администратором |
| `429` | Rate Limit Exceeded | Превышен лимит запросов |

---

## 📌 Эндпойнт: POST /v1/chat/completions

### URL

```
POST https://rvlautoai.ru/webhook/v1/chat/completions
```

### Аутентификация

Bearer Token в заголовке `Authorization`.

### Request Body (обязательные параметры)

| Поле | Тип | Обязательно | Описание |
|------|-----|:----:|----------|
| `model` | string | ✅ | ID модели: `gpt-4o`, `flux`, `command-r-plus-08-2024`, и т.д. |
| `provider` | string | ✅ | Провайдер: `Capi`, `HuggingSpace`, `Gemini`, `PollinationsImage`, `BlackForestLabs_Flux1Dev` и т.д. |
| `messages` | array | ✅ | Массив сообщений OpenAI-формата |

### Request Body (опциональные параметры)

| Поле | Тип | Значение по умолчанию | Диапазон | Примечание |
|------|-----|:----:|:----:|----------|
| `stream` | boolean | `false` | - | SSE-стриминг (зависит от провайдера) |
| `temperature` | number | `0.7` | 0.0–2.0 | Креативность ответа |
| `max_tokens` | integer | `2000` | 1–8192 | Максимальная длина ответа |
| `top_p` | number | `1.0` | 0.0–1.0 | Nucleus sampling |
| `frequency_penalty` | number | `0` | -2.0–2.0 | Штраф за повторения |
| `presence_penalty` | number | `0` | -2.0–2.0 | Штраф за известные слова |
| `websearch` | boolean | `false` | - | Веб-поиск (частичная поддержка) |

### Формат массива `messages`

```json
[
  {
    "role": "system",
    "content": "You are a helpful assistant."
  },
  {
    "role": "user",
    "content": "Объясни, что такое API."
  },
  {
    "role": "assistant",
    "content": "API (Application Programming Interface) — это интерфейс..."
  }
]
```

**Роли:**
- `system` — системный промпт
- `user` — сообщение пользователя
- `assistant` — ответ ассистента

### Response Format (успешный запрос)

OpenAI-compatible:

```json
{
  "id": "chatcmpl-AbCd1234567890",
  "object": "chat.completion",
  "created": 1731654208,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "API (Application Programming Interface) — это интерфейс для взаимодействия приложений..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 15,
    "completion_tokens": 120,
    "total_tokens": 135
  }
}
```

### Ошибки

OpenAI-compatible error format:

```json
{
  "error": {
    "message": "Invalid authentication credentials",
    "type": "invalid_request_error",
    "code": "invalid_api_key",
    "param": null
  }
}
```

---

## 🖼️ Генерация изображений: Формат Markdown

### ⚠️ КРИТИЧЕСКАЯ ИНФОРМАЦИЯ

**Провайдер `PollinationsImage` (и связанные) возвращают результат в виде Markdown-ссылки**, а не чистого URL или base64.

Это выглядит как:
```
[![alt_text](https://image.pollinations.ai/prompt/...)](https://image.pollinations.ai/prompt/...)
```

**Параметр `response_format` игнорируется** для изображений.

Чтобы получить URL изображения, нужно **извлечь его из Markdown** с помощью regex.

---

### Пример запроса: Генерация изображения (cURL)

```bash
curl -X POST "https://rvlautoai.ru/webhook/v1/chat/completions" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "flux-dev",
    "provider": "BlackForestLabs_Flux1Dev",
    "messages": [
      {
        "role": "user",
        "content": "A dramatic close-up portrait of a young woman, cinematic lighting, golden hour"
      }
    ]
  }'
```

### Пример ответа от API

```json
{
  "id": "chatcmpl-7eOR87ooCarBS6TwDOv9AOzIdk25",
  "object": "chat.completion",
  "created": 1766902699,
  "model": "flux-dev",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "[![A dramatic close-up portrait...](https://image.pollinations.ai/prompt/A%2Bdramatic%2Bclose-up%2Bportrait%2Bof%2Ba%2Byoung%2Bwoman%2C%2Bcinematic%2Blighting%2C%2Bgolden%2Bhour...?width=1024&height=1024&model=flux&seed=4220590749)](https://image.pollinations.ai/prompt/A%2Bdramatic%2Bclose-up%2Bportrait%2Bof%2Ba%2Byoung%2Bwoman%2C%2Bcinematic%2Blighting%2C%2Bgolden%2Bhour...?width=1024&height=1024&model=flux&seed=4220590749)"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 0,
    "completion_tokens": 4,
    "total_tokens": 4
  }
}
```

---

### ✅ РАБОЧИЙ ПРИМЕР: Извлечение URL и скачивание (PowerShell)

```powershell
# 1. ПОДГОТОВКА: Установите заголовки и тело запроса
$headers = @{
    "Authorization" = "Bearer YOUR_API_KEY"
    "Content-Type"  = "application/json"
}

$body = @{
    model       = "flux-dev"
    provider    = "BlackForestLabs_Flux1Dev"
    messages    = @(
        @{
            role    = "user"
            content = "A dramatic close-up portrait of a young woman, cinematic lighting, golden hour, shallow depth of field, DSLR, 50 mm lens, soft bokeh background, ultra-realistic, photorealistic"
        }
    )
} | ConvertTo-Json -Depth 10

# 2. ЗАПРОС
$response = Invoke-RestMethod -Uri "https://rvlautoai.ru/webhook/v1/chat/completions" `
    -Method Post `
    -Headers $headers `
    -Body $body

# 3. ИЗВЛЕЧЕНИЕ Markdown-контента
$markdownContent = $response.choices[0].message.content

Write-Host "=== ПОЛУЧЕННЫЙ Markdown ===" -ForegroundColor Cyan
Write-Host $markdownContent

# 4. ИЗВЛЕЧЕНИЕ URL с помощью regex
# Шаблон: \(https://[^\)]+\)
# Извлекает первый URL в круглых скобках
if ($markdownContent -match '\(https://[^\)]+\)') {
    # $matches[0] содержит "(https://...)"
    # Убираем скобки с помощью -replace
    $imageUrl = $matches[0] -replace '[()]', ''
    
    Write-Host "`n=== ИЗВЛЕЧЁННЫЙ URL ===" -ForegroundColor Green
    Write-Host $imageUrl
    
    # 5. СКАЧИВАНИЕ изображения
    $outputPath = "$env:USERPROFILE\Downloads\generated_image_$(Get-Date -Format 'yyyyMMdd_HHmmss').jpg"
    Invoke-WebRequest -Uri $imageUrl -OutFile $outputPath
    
    Write-Host "`n=== ИЗОБРАЖЕНИЕ СОХРАНЕНО ===" -ForegroundColor Magenta
    Write-Host "Путь: $outputPath"
    
    # 6. ОТКРЫТЬ изображение в браузере/просмотрщике
    Start-Process $outputPath
    Write-Host "`n✅ Готово!" -ForegroundColor Green
} else {
    Write-Host "`n❌ ОШИБКА: Не удалось извлечь URL из Markdown-ответа" -ForegroundColor Red
    Write-Host "Сырой контент: $markdownContent" -ForegroundColor Yellow
}
```

**Что происходит в этом скрипте:**
1. Отправляет POST-запрос к API
2. Получает JSON-ответ
3. Извлекает `content` из первого `choice`
4. **Используя regex `\(https://[^\)]+\)`** извлекает URL из Markdown
5. Скачивает изображение в папку `Downloads`
6. Открывает его в просмотрщике

---

### ✅ РАБОЧИЙ ПРИМЕР: Извлечение URL (Python)

```python
import re
import requests
from datetime import datetime

# Подготовка
BASE_URL = "https://rvlautoai.ru/webhook"
API_KEY = "YOUR_API_KEY"

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}

body = {
    "model": "flux-dev",
    "provider": "BlackForestLabs_Flux1Dev",
    "messages": [
        {
            "role": "user",
            "content": "A dramatic close-up portrait of a young woman, cinematic lighting, golden hour, shallow depth of field"
        }
    ]
}

# Запрос
try:
    response = requests.post(
        f"{BASE_URL}/v1/chat/completions",
        headers=headers,
        json=body,
        timeout=180
    )
    response.raise_for_status()
except requests.exceptions.RequestException as e:
    print(f"❌ Ошибка запроса: {e}")
    exit(1)

# Извлечение Markdown-контента
result = response.json()
markdown_content = result["choices"][0]["message"]["content"]

print("=== ПОЛУЧЕННЫЙ Markdown ===")
print(markdown_content)

# Извлечение URL
# Regex: \((https://[^\)]+)\)
url_match = re.search(r"\((https://[^\)]+)\)", markdown_content)

if not url_match:
    print("\n❌ ОШИБКА: Не удалось извлечь URL")
    exit(1)

image_url = url_match.group(1)
print(f"\n=== ИЗВЛЕЧЁННЫЙ URL ===")
print(image_url)

# Скачивание изображения
try:
    img_response = requests.get(image_url, timeout=180)
    img_response.raise_for_status()
    
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    output_path = f"generated_image_{timestamp}.jpg"
    
    with open(output_path, "wb") as f:
        f.write(img_response.content)
    
    print(f"\n=== ИЗОБРАЖЕНИЕ СОХРАНЕНО ===")
    print(f"Путь: {output_path}")
    print(f"\n✅ Готово!")
    
except requests.exceptions.RequestException as e:
    print(f"\n❌ Ошибка скачивания: {e}")
    exit(1)
```

---

### ✅ РАБОЧИЙ ПРИМЕР: Извлечение URL (JavaScript/Node.js)

```javascript
const axios = require('axios');
const fs = require('fs');
const path = require('path');

const API_KEY = "YOUR_API_KEY";
const BASE_URL = "https://rvlautoai.ru/webhook";

async function generateAndDownloadImage() {
    try {
        // 1. Запрос к API
        const response = await axios.post(
            `${BASE_URL}/v1/chat/completions`,
            {
                model: "flux-dev",
                provider: "BlackForestLabs_Flux1Dev",
                messages: [
                    {
                        role: "user",
                        content: "A dramatic close-up portrait of a young woman, cinematic lighting, golden hour, shallow depth of field"
                    }
                ]
            },
            {
                headers: {
                    "Authorization": `Bearer ${API_KEY}`,
                    "Content-Type": "application/json"
                },
                timeout: 180000
            }
        );

        // 2. Извлечение Markdown
        const markdownContent = response.data.choices[0].message.content;
        console.log("=== ПОЛУЧЕННЫЙ Markdown ===");
        console.log(markdownContent);

        // 3. Извлечение URL с помощью regex
        // Шаблон: \((https://[^\)]+)\)
        const urlMatch = markdownContent.match(/\((https:\/\/[^\)]+)\)/);
        
        if (!urlMatch) {
            throw new Error("Не удалось извлечь URL из Markdown-ответа");
        }

        const imageUrl = urlMatch[1];
        console.log("\n=== ИЗВЛЕЧЁННЫЙ URL ===");
        console.log(imageUrl);

        // 4. Скачивание изображения
        const imageResponse = await axios.get(imageUrl, {
            responseType: 'arraybuffer',
            timeout: 180000
        });

        // 5. Сохранение в файл
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, -5);
        const outputPath = `generated_image_${timestamp}.jpg`;
        
        fs.writeFileSync(outputPath, imageResponse.data);

        console.log("\n=== ИЗОБРАЖЕНИЕ СОХРАНЕНО ===");
        console.log(`Путь: ${path.resolve(outputPath)}`);
        console.log("\n✅ Готово!");

    } catch (error) {
        console.error(`\n❌ Ошибка:`, error.message);
        process.exit(1);
    }
}

generateAndDownloadImage();
```

---

## 📌 Эндпойнт: GET /v1/models

### URL

```
GET https://rvlautoai.ru/webhook/v1/models
```

### Аутентификация

Bearer Token в заголовке `Authorization`.

### Query параметры

| Параметр | Тип | Описание |
|----------|-----|----------|
| `provider` | string | Фильтр по провайдеру (например `Capi`, `HuggingSpace`) |

### Примеры

**Все модели:**
```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models" \
  -H "Authorization: Bearer rvf_admin..."
```

**Фильтр по провайдеру:**
```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models?provider=Capi" \
  -H "Authorization: Bearer rvf_admin..."
```

---

## 📌 Эндпойнт: GET /v1/models/list

### URL

```
GET https://rvlautoai.ru/webhook/v1/models/list
```

### Аутентификация

Bearer Token в заголовке `Authorization`.

### Query параметры

| Параметр | Тип | Описание |
|----------|-----|----------|
| `provider` | string | Фильтр по провайдеру (например `Capi`, `HuggingSpace`) |
| `type` | string | Фильтр по типу: `text`, `image`, `audio`, `video` |

### Примеры

**Все модели:**
```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models/list" \
  -H "Authorization: Bearer rvf_admin..."
```

**Только текстовые модели Capi:**
```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models/list?type=text&provider=Capi" \
  -H "Authorization: Bearer rvf_admin..."
```

**Только модели для генерации изображений:**
```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models/list?type=image" \
  -H "Authorization: Bearer rvf_admin..."
```

---

## 📌 Эндпойнт: GET /v1/providers

### URL

```
GET https://rvlautoai.ru/webhook/v1/providers
```

### Аутентификация

Bearer Token в заголовке `Authorization`.

### Query параметры

| Параметр | Тип | Описание |
|----------|-----|----------|
| `name` | string | Фильтр по имени провайдера (например `Capi`) |

### Примеры

**Все провайдеры:**
```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/providers" \
  -H "Authorization: Bearer rvf_admin..."
```

**Информация о конкретном провайдере:**
```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/providers?name=Capi" \
  -H "Authorization: Bearer rvf_admin..."
```

---

## ⏱ Rate limiting

### Заголовки ответа

Сервис может возвращать заголовки лимитирования:

```
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 25
X-RateLimit-Reset: 1731654208
```

### Типы ключей и лимиты

| Тип ключа | Запросов в период | Период | Описание |
|-----------|:----:|:----:|----------|
| `test` | 30 | 1 час | Тестовый доступ, автоматически истекает через 1 час |
| `full` | Unlimited | 30 дней | Стандартный доступ, автоматически истекает через 30 дней |

### Обработка ошибок лимитирования

```powershell
if ($response.StatusCode -eq 429) {
    $retryAfter = [int]($response.Headers.'Retry-After' ?? 60)
    Write-Host "⏱ Rate limit. Ожидание $retryAfter секунд..."
    Start-Sleep -Seconds $retryAfter
    # Повторить запрос
}
```

---

## 🧯 Fallback / Retry

### Как это работает

1. Отправляется запрос к основной модели провайдера
2. Если ошибка 5xx (500, 502, 503) или пустой ответ — **автоматический retry**
3. Формируется очередь fallback-моделей:
   - **Same-provider fallback** — другие модели того же провайдера (быстрее)
   - **Cross-provider fallback** — модели других провайдеров (если основной недоступен)
4. **Максимум 5 попыток** (включая основную)
5. Если все не удалось — возвращается ошибка `503 Service Unavailable`

### Пример: Fallback в действии

```
Запрос: model=gpt-4o, provider=Capi
↓
Попытка 1: gpt-4o @ Capi → HTTP 500
↓
Попытка 2: gemini-2.5-flash @ Capi → HTTP 503
↓
Попытка 3: claude-3-opus @ Capi → HTTP 503
↓
Попытка 4: gpt-4 @ HuggingSpace → ✅ 200 OK
↓
Ответ с указанием fallback в метаданных
```

### Ошибка при исчерпании всех попыток

```json
{
  "error": {
    "message": "Service temporarily unavailable. All 5 models failed to respond.",
    "type": "service_unavailable_error",
    "code": "all_models_failed",
    "param": null,
    "details": {
      "total_attempts": 5,
      "tried_models": ["gpt-4o@Capi", "gemini-2.5-flash@Capi", "claude-3-opus@Capi", "gpt-4@HuggingSpace"],
      "error_history": [...]
    }
  }
}
```

---

## 🧪 Быстрые примеры

### 1) Простой текстовый запрос (PowerShell)

```powershell
$headers = @{
    "Authorization" = "Bearer rvf_full..."
    "Content-Type"  = "application/json"
}

$body = @{
    model       = "gpt-4o"
    provider    = "Capi"
    messages    = @(
        @{
            role    = "user"
            content = "Что такое REST API? Ответь кратко."
        }
    )
    temperature = 0.7
    max_tokens  = 300
} | ConvertTo-Json -Depth 10

$response = Invoke-RestMethod -Uri "https://rvlautoai.ru/webhook/v1/chat/completions" `
    -Method Post `
    -Headers $headers `
    -Body $body

Write-Host $response.choices[0].message.content
```

---

### 2) Диалог с контекстом (Python)

```python
import requests

BASE_URL = "https://rvlautoai.ru/webhook"
API_KEY = "rvf_full..."

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}

messages = [
    {"role": "system", "content": "Ты краткий и точный ассистент."},
    {"role": "user", "content": "15 + 27?"},
    {"role": "assistant", "content": "42"},
    {"role": "user", "content": "Умножь результат на 3"}
]

response = requests.post(
    f"{BASE_URL}/v1/chat/completions",
    headers=headers,
    json={
        "model": "gpt-4o",
        "provider": "Capi",
        "messages": messages,
        "temperature": 0.3,
        "max_tokens": 50
    },
    timeout=180
)

response.raise_for_status()
print(response.json()["choices"][0]["message"]["content"])
```

---

### 3) Веб-поиск (если поддерживается провайдером)

```bash
curl -X POST "https://rvlautoai.ru/webhook/v1/chat/completions" \
  -H "Authorization: Bearer rvf_full..." \
  -H "Content-Type: application/json" \
  -d '{
    "model": "command-r-plus-08-2024",
    "provider": "HuggingSpace",
    "messages": [
      {"role": "user", "content": "Какие последние новости в области AI?"}
    ],
    "websearch": true
  }'
```

---

### 4) Обработка ошибок

```python
import requests
import time

def safe_chat_completion(model, provider, messages, max_retries=3):
    """Безопасный запрос с обработкой ошибок и retry"""
    
    headers = {
        "Authorization": "Bearer rvf_full...",
        "Content-Type": "application/json",
    }
    
    for attempt in range(max_retries):
        try:
            response = requests.post(
                "https://rvlautoai.ru/webhook/v1/chat/completions",
                headers=headers,
                json={"model": model, "provider": provider, "messages": messages},
                timeout=180
            )
            
            if response.status_code == 200:
                return response.json()["choices"][0]["message"]["content"]
            
            elif response.status_code == 401:
                print("❌ Ошибка аутентификации: неверный API ключ")
                return None
            
            elif response.status_code == 429:
                retry_after = int(response.headers.get("Retry-After", 60))
                print(f"⏱ Rate limit. Ожидание {retry_after}с...")
                time.sleep(retry_after)
            
            elif response.status_code == 503:
                print(f"⚠️ Попытка {attempt+1}/{max_retries}: Service unavailable")
                if attempt < max_retries - 1:
                    time.sleep(2 ** attempt)  # Экспоненциальная задержка
            
            else:
                print(f"❌ HTTP {response.status_code}: {response.text}")
                return None
        
        except requests.exceptions.Timeout:
            print(f"⚠️ Timeout. Попытка {attempt+1}/{max_retries}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
        
        except Exception as e:
            print(f"❌ Ошибка: {e}")
            return None
    
    print("❌ Все попытки исчерпаны")
    return None

# Использование
result = safe_chat_completion(
    model="gpt-4o",
    provider="Capi",
    messages=[{"role": "user", "content": "Привет!"}]
)
print(result)
```

---

## 🧾 Changelog

| Версия | Дата | Изменения |
|--------|------|----------|
| **1.2** | **28.12.2025** | **🔥 README полностью актуализирован:** Добавлены рабочие примеры извлечения URL из Markdown (PowerShell, Python, JS), описано реальное поведение `PollinationsImage` и `BlackForestLabs_Flux1Dev`, уточнены типы ключей и лимиты, добавлена обработка ошибок |
| 1.1 | 22.12.2025 | Расширена документация GET endpoints |
| 1.0 | 17.11.2025 | Добавлен POST `/v1/chat/completions` с документацией |
| 1.0 | 17.11.2025 | Добавлена документация по аутентификации и rate limiting |
| 1.0 | 17.11.2025 | Документирован автоматический fallback механизм |
| 1.0 | 15.11.2025 | Первая версия API документации (GET эндпойнты) |

---

## 📝 Дополнительная информация

**Дата обновления:** 28 декабря 2025  
**Автор:** RVFreeLLM Team  
**Статус:** ✅ Production Ready  
**Версия API:** 1.2  

---

## 🤝 Поддержка

- **Telegram:** `@FreeApiLLMbot`
- **GitHub Issues:** https://github.com/Xrey995/Unlimited-LLMs/issues
- **Email:** support@rvlautoai.ru (при наличии)
