# Unlimited-LLMs
RVFreeLLM API Endpoints - Полное руководство по использованию.
Данные эндпойнты предоставляют безлимитный доступ к AI-моделям (text, image, audio, video).
Список актуальных провайдеров и их моделей обновляется каждые сутки в 02:00 по МСК. (проверяйте актуальность).

Нет ограничений на количество запросов и количество токенов (полный безлимит). 
Единственное ограничение - не более 30 запросов в минуту.

**Версия API:** 1.0  
**Дата обновления:** 17 ноября 2025  
**Base URL:** `https://rvlautoai.ru/webhook`

---

## 🔗 Полезные ссылки

- **Base URL:** https://rvlautoai.ru/webhook
- **Получить Api-Key в Telegram Bot:** @FreeApiLLMbot
- **Техническая поддержка:** Через Telegram

---

## 📋 Содержание

1. [Обзор API](#обзор-api)
2. [Базовая информация](#базовая-информация)
3. [Аутентификация](#аутентификация)
4. [Эндпойнт: POST /v1/chat/completions](#эндпойнт-post-v1chatcompletions)
5. [Эндпойнт: GET /v1/models](#эндпойнт-get-v1models)
6. [Эндпойнт: GET /v1/models/list](#эндпойнт-get-v1modelslist)
7. [Эндпойнт: GET /v1/providers](#эндпойнт-get-v1providers)
8. [Примеры использования в коде](#примеры-использования-в-коде)
9. [Обработка ошибок](#обработка-ошибок)
10. [Лучшие практики](#лучшие-практики)
11. [Rate Limiting](#rate-limiting)
12. [Fallback механизм](#fallback-механизм)

---

## 🎯 Обзор API

RVFreeLLM API предоставляет доступ к множеству AI моделей через унифицированный OpenAI-совместимый интерфейс. API поддерживает текстовые, изображения, аудио и видео модели от различных провайдеров (Capi, HuggingSpace, Gemini, OpenAIFM и других).

### Основные возможности:

- **Генерация текстовых ответов** (chat completions) с поддержкой streaming
- **Генерация изображений, аудио и видео контента**
- **Автоматический fallback между моделями и провайдерами**
- **Rate limiting** для защиты от злоупотреблений
- **Получение списка доступных моделей и провайдеров**
- **OpenAI API совместимость** для легкой интеграции

---

## 📡 Базовая информация

### Base URL

```
https://rvlautoai.ru/webhook
```

### Формат данных

Все эндпойнты используют JSON формат:

```http
Content-Type: application/json
```

### HTTP заголовки

```http
Content-Type: application/json
Access-Control-Allow-Origin: *
X-API-Version: 1.0
```

### Типы моделей

| Тип | Описание | Примеры моделей |
|-----|----------|----------------|
| `text` | Текстовые модели (чат, генерация текста) | gpt-4o, claude-3, gemini-2.5-flash |
| `image` | Генерация изображений | dall-e-3, flux, stable-diffusion |
| `audio` | Аудио модели (TTS, голосовые ассистенты) | tts-1, whisper, alloy, echo |
| `video` | Видео генерация | sora, cogvideo, runway |

---

## 🔐 Аутентификация

### Bearer Token

Все запросы к API (кроме информационных эндпойнтов `/v1/models` и `/v1/providers`) требуют аутентификации с помощью Bearer Token.

### Формат заголовка

```http
Authorization: Bearer YOUR_API_KEY
```

### Формат API ключа

- **Префикс:** `rvf_`
- **Длина:** 75 символов
- **Формат:** `rvf_` + 71 буквенно-цифровой символ

### Пример

```bash
curl -X POST "https://rvlautoai.ru/webhook/v1/chat/completions" \
  -H "Authorization: Bearer rvf_test1234567890abcdef..." \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-4o", "provider": "Capi", "messages": [...]}'
```

### Типы API ключей

| Тип | Лимит запросов | Срок действия | Описание |
|-----|----------------|---------------|----------|
| `test` | 30 запросов/час | 1 час | Тестовый ключ для ознакомления |
| `full` | 30 запросов/минуту | 30 дней | Полноценный доступ |
| `admin` | Без ограничений | Бессрочно | Административный доступ |

### Ошибки аутентификации

| Код | Описание | Причина |
|-----|----------|---------|
| `401` | Unauthorized | Неверный или отсутствующий API ключ |
| `401` | Key Expired | Ключ истек (test после 1 часа, full после 30 дней) |
| `403` | Key Revoked | Ключ аннулирован (возврат платежа) |
| `429` | Rate Limit Exceeded | Превышен лимит запросов |

---

## 📌 Эндпойнт: POST /v1/chat/completions

Основной эндпойнт для генерации текстовых ответов, изображений, аудио и видео контента.

### URL

```
POST https://rvlautoai.ru/webhook/v1/chat/completions
```

### Аутентификация

**Обязательна.** Требуется Bearer Token в заголовке `Authorization`.

### Request Body (обязательные параметры)

| Параметр | Тип | Описание |
|----------|-----|----------|
| `model` | string | **Обязательно.** Название модели (например, `gpt-4o`, `gemini-2.5-flash`) |
| `provider` | string | **Обязательно.** Название провайдера (`Capi`, `HuggingSpace`, `Gemini`, и т.д.) |
| `messages` | array | **Обязательно.** Массив сообщений в OpenAI формате |

### Request Body (опциональные параметры)

| Параметр | Тип | По умолчанию | Описание |
|----------|-----|--------------|----------|
| `stream` | boolean | `false` | Включить потоковую передачу ответа |
| `temperature` | number | `0.7` | Температура генерации (0.0-2.0) |
| `max_tokens` | integer | `2000` | Максимальное количество токенов в ответе |
| `top_p` | number | `1.0` | Nucleus sampling (0.0-1.0) |
| `frequency_penalty` | number | `0` | Штраф за повторения (-2.0 до 2.0) |
| `presence_penalty` | number | `0` | Штраф за присутствие (-2.0 до 2.0) |
| `web_search` | boolean | `false` | Включить веб-поиск (поддерживается не всеми моделями) |

### Формат массива messages

```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "Привет, как дела?"
    }
  ]
}
```

**Допустимые значения `role`:**
- `system` — системная инструкция для модели
- `user` — сообщение от пользователя
- `assistant` — ответ ассистента (для контекста диалога)

### Response Format (успешный запрос)

```json
{
  "id": "chatcmpl-AbCd1234567890",
  "object": "chat.completion",
  "created": 1731654208,
  "model": "gpt-4o",
  "provider": "Capi",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Привет! У меня все хорошо, спасибо. Чем могу помочь?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 15,
    "completion_tokens": 12,
    "total_tokens": 27
  }
}
```

### Примеры запросов

#### Пример 1: Простой текстовый запрос (PowerShell)

```powershell
$headers = @{
    "Authorization" = "Bearer YOUR_API_KEY"
    "Content-Type" = "application/json"
}

$body = @{
    model = "gpt-4o"
    provider = "Capi"
    messages = @(
        @{
            role = "user"
            content = "Привет, это тест"
        }
    )
    temperature = 0.7
    max_tokens = 100
} | ConvertTo-Json -Depth 10

$response = Invoke-RestMethod -Uri "https://rvlautoai.ru/webhook/v1/chat/completions" `
    -Method POST `
    -Headers $headers `
    -Body $body

Write-Host "Ответ: $($response.choices[0].message.content)"
```

#### Пример 2: Запрос с веб-поиском (PowerShell)

```powershell
$headers = @{
    "Authorization" = "Bearer YOUR_API_KEY"
    "Content-Type" = "application/json"
}

$body = @{
    model = "command-r-plus-08-2024"
    provider = "HuggingSpace"
    messages = @(
        @{
            role = "user"
            content = "Какая погода в Москве сегодня?"
        }
    )
    web_search = $true
} | ConvertTo-Json -Depth 10

$response = Invoke-RestMethod -Uri "https://rvlautoai.ru/webhook/v1/chat/completions" `
    -Method POST `
    -Headers $headers `
    -Body $body

Write-Host "Ответ: $($response.choices[0].message.content)"
```

#### Пример 3: Диалог с контекстом (Python)

```python
import requests

url = "https://rvlautoai.ru/webhook/v1/chat/completions"
headers = {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
}

body = {
    "model": "gpt-4o",
    "provider": "Capi",
    "messages": [
        {
            "role": "system",
            "content": "Ты полезный ассистент, отвечающий кратко и по делу."
        },
        {
            "role": "user",
            "content": "Сколько будет 15 + 27?"
        },
        {
            "role": "assistant",
            "content": "15 + 27 = 42"
        },
        {
            "role": "user",
            "content": "Умножь этот результат на 3"
        }
    ],
    "temperature": 0.3,
    "max_tokens": 50
}

response = requests.post(url, headers=headers, json=body)
result = response.json()

print("Ответ:", result['choices'][0]['message']['content'])
# Вывод: "42 × 3 = 126"
```

#### Пример 4: Генерация изображения (cURL)

```bash
curl -X POST "https://rvlautoai.ru/webhook/v1/chat/completions" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "flux",
    "provider": "PollinationsImage",
    "messages": [
      {
        "role": "user",
        "content": "A beautiful sunset over mountains, photorealistic"
      }
    ],
    "max_tokens": 100
  }'
```

#### Пример 5: Streaming ответ (JavaScript)

```javascript
const fetch = require('node-fetch');

const url = 'https://rvlautoai.ru/webhook/v1/chat/completions';
const headers = {
  'Authorization': 'Bearer YOUR_API_KEY',
  'Content-Type': 'application/json'
};

const body = {
  model: 'gpt-4o',
  provider: 'Capi',
  messages: [
    { role: 'user', content: 'Напиши короткую историю про робота' }
  ],
  stream: true
};

async function streamCompletion() {
  const response = await fetch(url, {
    method: 'POST',
    headers: headers,
    body: JSON.stringify(body)
  });

  const reader = response.body;
  reader.on('data', (chunk) => {
    const text = chunk.toString();
    console.log('Chunk:', text);
  });
}

streamCompletion();
```

### Поддерживаемые провайдеры

| Провайдер | Типы моделей | Особенности |
|-----------|-------------|-------------|
| `Capi` | text, image | Быстрые модели GPT-4, Claude, Gemini |
| `HuggingSpace` | text | Llama, Command-R, Mistral с поддержкой веб-поиска |
| `Gemini` | text | Google Gemini модели |
| `OpenAIFM` | audio | TTS голоса (alloy, echo, fable, nova, и т.д.) |
| `PollinationsImage` | image | DALL-E, Flux, Stable Diffusion |

**Полный список провайдеров:** Используйте эндпойнт `GET /v1/providers` для получения актуального списка.

Пример для просмотра актуального списка в PowerShell:
```powershell
Invoke-RestMethod -Uri "https://rvlautoai.ru/webhook/v1/models/list" -Method Get | Select-Object -ExpandProperty text
```
### Автоматический Fallback

API автоматически переключается на резервные модели в случае ошибок:

1. **Same-Provider Fallback:** Сначала пробуются другие модели того же провайдера
2. **Cross-Provider Fallback:** Затем переключение на модели других провайдеров
3. **До 10 попыток:** Максимум 10 попыток перед возвратом ошибки 503

**Пример:** Запрос к `gpt-4o (Capi)` → ошибка → автоматический retry с `gemini-2.5-flash (Capi)` → ошибка → retry с `gpt-4 (HuggingSpace)` → успех.

---

## 📌 Эндпойнт: GET /v1/models

Возвращает список всех доступных моделей в формате, совместимом с OpenAI API.

### URL

```
GET https://rvlautoai.ru/webhook/v1/models
```

### Аутентификация

**Не требуется.**

### Query параметры

| Параметр | Тип | Обязательный | Описание |
|----------|-----|--------------|----------|
| `provider` | string | Нет | Фильтрация по провайдеру (например, `Capi`, `OpenAIFM`) |

### Формат ответа

```json
{
  "object": "list",
  "data": [
    {
      "id": "gpt-4o-mini",
      "object": "model",
      "created": 1731654208,
      "owned_by": "Capi",
      "permission": [
        {
          "id": "modelperm-gpt-4o-mini-0",
          "object": "model_permission",
          "created": 1731654208,
          "allow_create_engine": false,
          "allow_sampling": true,
          "allow_logprobs": true,
          "allow_search_indices": false,
          "allow_view": true,
          "allow_fine_tuning": false,
          "organization": "*",
          "group": null,
          "is_blocking": false
        }
      ],
      "provider": "Capi",
      "category": "text",
      "latency_ms": 1500,
      "quality_score": 8.5,
      "last_verified": "2025-11-15 08:23:28"
    }
  ],
  "meta": {
    "total_models": 72,
    "provider_filter": null,
    "generated_at": "2025-11-17T10:30:00.000Z",
    "data_window": "active_only"
  }
}
```

### Примеры запросов

#### Все модели

```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models"
```

#### Фильтрация по провайдеру

```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models?provider=Capi"
```

#### PowerShell

```powershell
$response = Invoke-RestMethod -Uri "https://rvlautoai.ru/webhook/v1/models"
Write-Host "Всего моделей: $($response.meta.total_models)"

foreach ($model in $response.data) {
    Write-Host "$($model.id) - $($model.provider) ($($model.category))"
}
```

---

## 📌 Эндпойнт: GET /v1/models/list

Возвращает человекочитаемый текстовый список моделей, сгруппированных по типу и провайдеру.

### URL

```
GET https://rvlautoai.ru/webhook/v1/models/list
```

### Аутентификация

**Не требуется.**

### Query параметры

| Параметр | Тип | Обязательный | Описание | Допустимые значения |
|----------|-----|--------------|----------|---------------------|
| `provider` | string | Нет | Фильтрация по провайдеру | `Capi`, `OpenAIFM`, `HuggingSpace`, и т.д. |
| `type` | string | Нет | Фильтрация по типу модели | `text`, `image`, `audio`, `video` |

### Формат ответа

```json
{
  "text": "🤖 Доступные модели G4F API:\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n💬 ТЕКСТОВЫЕ МОДЕЛИ (50)\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n1. 🏢 Capi (18 моделей)\n   • gpt-4o — ⚡ 1.5s\n   • gemini-2.5-flash — ⚡ 1.6s\n..."
}
```

### Примеры запросов

#### Все модели

```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models/list"
```

#### Только текстовые модели

```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models/list?type=text"
```

#### Только модели провайдера Capi

```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/models/list?provider=Capi"
```

---

## 📌 Эндпойнт: GET /v1/providers

Возвращает список всех провайдеров с их статистикой.

### URL

```
GET https://rvlautoai.ru/webhook/v1/providers
```

### Аутентификация

**Не требуется.**

### Query параметры

| Параметр | Тип | Обязательный | Описание |
|----------|-----|--------------|----------|
| `name` | string | Нет | Получить информацию о конкретном провайдере |

### Формат ответа

```json
{
  "object": "list",
  "data": [
    {
      "id": "Capi",
      "object": "provider",
      "created": 1731654208,
      "name": "Capi",
      "capabilities": {
        "text": true,
        "image": true,
        "audio": false,
        "video": false
      },
      "models": {
        "total": 18,
        "text": 18,
        "image": 0,
        "audio": 0,
        "video": 0
      },
      "performance": {
        "avg_latency_ms": 1552,
        "avg_quality_score": 8.2
      },
      "last_verified": "2025-11-17 08:23:28"
    }
  ],
  "meta": {
    "total_providers": 10,
    "provider_filter": null,
    "generated_at": "2025-11-17T10:30:00.000Z",
    "data_window": "active_only"
  }
}
```

### Примеры запросов

#### Все провайдеры

```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/providers"
```

#### Конкретный провайдер

```bash
curl -X GET "https://rvlautoai.ru/webhook/v1/providers?name=Capi"
```

---

## 💻 Примеры использования в коде

### Python Client

```python
import requests
from typing import List, Dict, Any, Optional

class RVFreeLLMClient:
    def __init__(self, api_key: str, base_url: str = "https://rvlautoai.ru/webhook"):
        self.api_key = api_key
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        })
    
    def chat_completion(
        self,
        model: str,
        provider: str,
        messages: List[Dict[str, str]],
        temperature: float = 0.7,
        max_tokens: int = 2000,
        stream: bool = False,
        web_search: bool = False
    ) -> Dict[str, Any]:
        """
        Создать chat completion запрос.
        
        Args:
            model: Название модели (например, 'gpt-4o')
            provider: Название провайдера (например, 'Capi')
            messages: Список сообщений в формате [{"role": "user", "content": "..."}]
            temperature: Температура генерации (0.0-2.0)
            max_tokens: Максимальное количество токенов
            stream: Включить потоковую передачу
            web_search: Включить веб-поиск
            
        Returns:
            Dict с ответом от API
        """
        url = f"{self.base_url}/v1/chat/completions"
        
        body = {
            "model": model,
            "provider": provider,
            "messages": messages,
            "temperature": temperature,
            "max_tokens": max_tokens,
            "stream": stream,
            "web_search": web_search
        }
        
        try:
            response = self.session.post(url, json=body, timeout=180)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            error_data = e.response.json() if e.response else {}
            raise Exception(f"API Error: {error_data.get('error', {}).get('message', str(e))}")
    
    def get_models(self, provider: Optional[str] = None) -> Dict[str, Any]:
        """Получить список моделей."""
        url = f"{self.base_url}/v1/models"
        params = {"provider": provider} if provider else {}
        
        response = self.session.get(url, params=params, timeout=10)
        response.raise_for_status()
        return response.json()
    
    def get_providers(self, name: Optional[str] = None) -> Dict[str, Any]:
        """Получить список провайдеров."""
        url = f"{self.base_url}/v1/providers"
        params = {"name": name} if name else {}
        
        response = self.session.get(url, params=params, timeout=10)
        response.raise_for_status()
        return response.json()

# Использование
client = RVFreeLLMClient(api_key="YOUR_API_KEY")

# Простой запрос
response = client.chat_completion(
    model="gpt-4o",
    provider="Capi",
    messages=[
        {"role": "user", "content": "Привет, как дела?"}
    ]
)
print(response['choices'][0]['message']['content'])

# Запрос с веб-поиском
response = client.chat_completion(
    model="command-r-plus-08-2024",
    provider="HuggingSpace",
    messages=[
        {"role": "user", "content": "Какая погода в Москве?"}
    ],
    web_search=True
)
print(response['choices'][0]['message']['content'])
```

### JavaScript/Node.js Client

```javascript
const axios = require('axios');

class RVFreeLLMClient {
  constructor(apiKey, baseURL = 'https://rvlautoai.ru/webhook') {
    this.apiKey = apiKey;
    this.baseURL = baseURL;
    this.client = axios.create({
      baseURL,
      timeout: 180000,
      headers: {
        'Authorization': `Bearer ${apiKey}`,
        'Content-Type': 'application/json'
      }
    });
  }

  async chatCompletion({
    model,
    provider,
    messages,
    temperature = 0.7,
    maxTokens = 2000,
    stream = false,
    webSearch = false
  }) {
    try {
      const response = await this.client.post('/v1/chat/completions', {
        model,
        provider,
        messages,
        temperature,
        max_tokens: maxTokens,
        stream,
        web_search: webSearch
      });
      return response.data;
    } catch (error) {
      const errorMessage = error.response?.data?.error?.message || error.message;
      throw new Error(`API Error: ${errorMessage}`);
    }
  }

  async getModels(provider = null) {
    const params = provider ? { provider } : {};
    const response = await this.client.get('/v1/models', { params });
    return response.data;
  }

  async getProviders(name = null) {
    const params = name ? { name } : {};
    const response = await this.client.get('/v1/providers', { params });
    return response.data;
  }
}

// Использование
const client = new RVFreeLLMClient('YOUR_API_KEY');

(async () => {
  // Простой запрос
  const response = await client.chatCompletion({
    model: 'gpt-4o',
    provider: 'Capi',
    messages: [
      { role: 'user', content: 'Привет, как дела?' }
    ]
  });
  console.log(response.choices[0].message.content);

  // Запрос с веб-поиском
  const response2 = await client.chatCompletion({
    model: 'command-r-plus-08-2024',
    provider: 'HuggingSpace',
    messages: [
      { role: 'user', content: 'Какая погода в Москве?' }
    ],
    webSearch: true
  });
  console.log(response2.choices[0].message.content);
})();
```

---

## ⚠️ Обработка ошибок

### Коды HTTP ответов

| Код | Описание | Причина |
|-----|----------|---------|
| `200` | OK | Успешный запрос |
| `400` | Bad Request | Невалидные параметры запроса |
| `401` | Unauthorized | Неверный или отсутствующий API ключ |
| `403` | Forbidden | API ключ аннулирован |
| `404` | Not Found | Модель или провайдер не найдены |
| `429` | Too Many Requests | Превышен rate limit |
| `500` | Internal Server Error | Внутренняя ошибка сервера |
| `503` | Service Unavailable | Все модели недоступны (exhausted fallback) |

### Формат ошибки

```json
{
  "error": {
    "message": "Invalid API key",
    "type": "invalid_request_error",
    "code": "invalid_api_key",
    "param": null
  }
}
```

### Типы ошибок

| Type | Описание |
|------|----------|
| `invalid_request_error` | Неверные параметры запроса |
| `authentication_error` | Ошибка аутентификации (401/403) |
| `rate_limit_error` | Превышен rate limit (429) |
| `service_unavailable_error` | Все модели недоступны (503) |
| `api_error` | Внутренняя ошибка API (500) |

### Примеры обработки ошибок

#### Python

```python
import requests

def safe_chat_completion(client, model, provider, messages):
    try:
        response = client.chat_completion(
            model=model,
            provider=provider,
            messages=messages
        )
        return response['choices'][0]['message']['content']
    except requests.exceptions.HTTPError as e:
        status_code = e.response.status_code
        error_data = e.response.json()
        
        if status_code == 401:
            print("❌ Ошибка аутентификации:", error_data['error']['message'])
        elif status_code == 429:
            print("⏳ Rate limit превышен. Попробуйте позже.")
        elif status_code == 503:
            print("🔴 Все модели недоступны. Попробуйте позже.")
        else:
            print(f"❌ Ошибка {status_code}:", error_data['error']['message'])
        
        return None
    except requests.exceptions.Timeout:
        print("⏱️ Превышено время ожидания")
        return None
    except Exception as e:
        print(f"❌ Неизвестная ошибка: {str(e)}")
        return None
```

---

## 🎯 Лучшие практики

### 1. Используйте правильные модели и провайдеры

Проверяйте доступность модели для конкретного провайдера перед запросом:

```python
# Получить список моделей провайдера
models = client.get_models(provider="Capi")
available_models = [m['id'] for m in models['data']]

if "gpt-4o" in available_models:
    response = client.chat_completion(
        model="gpt-4o",
        provider="Capi",
        messages=[...]
    )
```

### 2. Обрабатывайте ошибки gracefully

Всегда используйте try-except блоки и проверяйте HTTP статус коды.

### 3. Используйте разумные timeout'ы

```python
response = requests.post(url, json=body, timeout=180)  # 3 минуты для сложных запросов
```

### 4. Кэшируйте список моделей

API эндпойнты `/v1/models` и `/v1/providers` имеют кэширование на 10 минут. Кэшируйте результаты на клиенте:

```python
import time

class CachedClient:
    def __init__(self, client):
        self.client = client
        self.models_cache = None
        self.models_cache_time = 0
    
    def get_models_cached(self, provider=None):
        now = time.time()
        if self.models_cache is None or (now - self.models_cache_time) > 600:
            self.models_cache = self.client.get_models(provider)
            self.models_cache_time = now
        return self.models_cache
```

### 5. Мониторьте использование токенов

```python
response = client.chat_completion(...)
usage = response['usage']
print(f"Использовано токенов: {usage['total_tokens']}")
print(f"Prompt: {usage['prompt_tokens']}, Completion: {usage['completion_tokens']}")
```

### 6. Используйте системные инструкции

```python
messages = [
    {"role": "system", "content": "Ты полезный ассистент, отвечающий кратко и по делу."},
    {"role": "user", "content": "Что такое API?"}
]
```

---

## 🚦 Rate Limiting

### Лимиты по типам ключей

| Тип ключа | Лимит | Окно |
|-----------|-------|------|
| `test` | 30 запросов | 1 час |
| `full` | 30 запросов | 1 минута |
| `admin` | Без ограничений | — |

### Заголовки Rate Limit

API возвращает заголовки для мониторинга rate limit:

```http
X-RateLimit-Limit: 30
X-RateLimit-Remaining: 25
X-RateLimit-Reset: 1731654208
```

### Обработка 429 ошибки

```python
import time

def request_with_retry(client, model, provider, messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.chat_completion(model, provider, messages)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                retry_after = int(e.response.headers.get('Retry-After', 60))
                print(f"⏳ Rate limit. Ожидание {retry_after} секунд...")
                time.sleep(retry_after)
            else:
                raise
    
    raise Exception("Превышен лимит попыток")
```

---

## 🔄 Fallback механизм

### Как работает автоматический fallback

API автоматически переключается между моделями в случае ошибок:

1. **Первая попытка:** Запрос к указанной модели и провайдеру
2. **При ошибке:** Автоматический выбор следующей модели из очереди
3. **Same-Provider приоритет:** Сначала пробуются модели того же провайдера
4. **Cross-Provider fallback:** Затем модели других провайдеров
5. **До 10 попыток:** Максимум 10 попыток перед ошибкой 503

### Пример fallback цепочки

```
Запрос: gpt-4o (Capi)
  ↓ Ошибка 500
Retry 1: gemini-2.5-flash (Capi)  ← Same provider
  ↓ Ошибка 503
Retry 2: claude-3-opus (Capi)     ← Same provider
  ↓ Ошибка 503
Retry 3: gpt-4 (HuggingSpace)     ← Cross-provider
  ↓ Успех ✅
Ответ клиенту
```

### Информация о fallback в ответе

При успешном fallback API возвращает информацию о финальной модели:

```json
{
  "model": "gpt-4",
  "provider": "HuggingSpace",
  "choices": [...],
  "_metadata": {
    "original_model": "gpt-4o",
    "original_provider": "Capi",
    "fallback_attempts": 3
  }
}
```

### Ошибка 503 (все модели exhausted)

Если все fallback модели недоступны:

```json
{
  "error": {
    "message": "Service temporarily unavailable. All 10 model(s) failed to respond.\n\nTried models:\ngpt-4o (Capi)\ngemini-2.5-flash (Capi)\ngpt-4 (HuggingSpace)\n...",
    "type": "service_unavailable_error",
    "code": "all_models_failed",
    "details": {
      "total_attempts": 10,
      "tried_models": ["gpt-4o (Capi)", "gemini-2.5-flash (Capi)", ...],
      "error_history": [...]
    }
  }
}
```

---

## 📊 Мониторинг и логирование

### Usage Stats

API логирует все запросы в таблицу `usage_stats` для мониторинга:

- API key ID и user ID
- Использованная модель и провайдер
- Количество токенов (prompt + completion)
- Время ответа (response time в мс)
- HTTP status code
- Сообщения об ошибках

### Проверка статистики

Администраторы могут запросить статистику использования через отдельный эндпойнт (требуется admin ключ).

---

## 🔗 Полезные ссылки

- **Base URL:** https://rvlautoai.ru/webhook
- **Telegram Bot: получить Api-Key** @FreeApiLLMbot
- **Техническая поддержка:** Через Telegram

---

## 📝 Changelog

| Версия | Дата | Изменения |
|--------|------|-----------|
| 1.0 | 17.11.2025 | Добавлен эндпойнт POST /v1/chat/completions с полной документацией |
| 1.0 | 17.11.2025 | Добавлена документация по аутентификации и rate limiting |
| 1.0 | 17.11.2025 | Документирован автоматический fallback механизм |
| 1.0 | 15.11.2025 | Первая версия API документации (GET эндпойнты) |

---

**Дата обновления:** 17 ноября 2025  
**Автор:** RVFreeLLM Team  
**Статус:** ✅ Production Ready
