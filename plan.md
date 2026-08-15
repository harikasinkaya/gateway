# OpenAI Compatible AI Gateway - Yol Haritası

## Proje Amacı

OpenAI API standartlarına tam uyumlu bir AI Gateway geliştirmek.

Amaçlar:

- OpenAI SDK'ları ile doğrudan çalışabilmek
- Kendi sanal modellerimizi oluşturabilmek
- Her modele özel sistem promptu tanımlayabilmek
- Hafıza (Memory) desteği ekleyebilmek
- Tool Calling desteği sunabilmek
- Birden fazla sağlayıcıyı (OpenAI, Gemini, DeepSeek, Ollama vb.) tek API altında birleştirebilmek
- Kullanıcıların sadece model adı değiştirerek farklı davranışlar elde etmesini sağlamak

Örnek:

```json
{
  "model": "barlas-v1",
  "messages": [
    {
      "role": "user",
      "content": "Merhaba"
    }
  ]
}
```

Gateway arka planda:

- Sistem promptu ekler
- Hafıza yükler
- Toolları bağlar
- Sağlayıcı seçer
- Sonucu OpenAI formatında döndürür

---

# Faz 1 - OpenAI Uyumluluğu

## Endpointler

### GET /v1/models

Desteklenen sanal modelleri listeler.

Örnek:

```json
{
  "data": [
    {
      "id": "barlas-v1"
    },
    {
      "id": "barlas-v2"
    }
  ]
}
```

### POST /v1/chat/completions

Ana endpoint.

### POST /v1/embeddings

Embedding desteği.

### POST /v1/responses

Yeni OpenAI standardı için hazırlık.

---

# Faz 2 - Model Registry

Her model veritabanında kayıtlı olacak.

Örnek:

```json
{
  "id": "barlas-v1",
  "provider": "deepseek",
  "provider_model": "deepseek-chat",
  "temperature": 0.7,
  "memory": true,
  "tools": true
}
```

Örnek:

```json
{
  "id": "barlas-v2",
  "provider": "gemini",
  "provider_model": "gemini-2.5-pro"
}
```

Amaç:

Kullanıcının gerçek modeli bilmesine gerek kalmaması.

---

# Faz 3 - Prompt Engine

Her modelin kendi davranışı olacak.

Örnek:

## barlas-v1

```text
Sen Barlas V1'sin.
Node.js konusunda uzmansın.
```

## barlas-v2

```text
Sen Barlas V2'sin.
Kod üretirken açıklamalı cevap ver.
```

İş Akışı:

```text
Model Seç
↓
Sistem Promptunu Yükle
↓
Memory Yükle
↓
Kullanıcı Mesajını Ekle
↓
Provider'a Gönder
```

---

# Faz 4 - Memory Sistemi

## Kısa Süreli Hafıza

Son konuşmalar.

```sql
messages
```

---

## Uzun Süreli Hafıza

Önemli kullanıcı bilgileri.

```sql
memories
```

Örnek:

```json
{
  "user_id": "123",
  "content": "Node.js kullanıyor"
}
```

---

## Vektör Hafıza

RAG sistemi.

Alternatifler:

- Qdrant
- PgVector
- Weaviate

Amaç:

Geçmiş konuşmaları geri çağırabilmek.

---

# Faz 5 - Provider Sistemi

Desteklenecek sağlayıcılar:

- OpenAI
- DeepSeek
- Gemini
- Claude
- Ollama
- OpenRouter

Örnek:

```text
barlas-v1
↓
DeepSeek
↓
deepseek-chat
```

```text
barlas-v2
↓
Gemini
↓
gemini-pro
```

---

# Faz 6 - Tool Calling

İlk sürüm:

- Web Search
- Memory Search
- Calculator
- Code Execution

İkinci sürüm:

- Browser Automation
- Playwright
- MCP
- File Search

---

# Faz 7 - Streaming

Tam OpenAI uyumlu SSE desteği.

Örnek:

```text
data: {...}

data: {...}

data: [DONE]
```

OpenAI SDK'larının sorunsuz çalışması hedeflenir.

---

# Faz 8 - API Key Sistemi

Tablo:

```sql
api_keys
```

Alanlar:

- id
- key
- owner
- created_at
- disabled

---

# Faz 9 - Rate Limiting

Destek:

- RPM
- TPM
- Günlük Limit
- Aylık Limit

Örnek:

```json
{
  "rpm": 60,
  "tpm": 100000
}
```

---

# Faz 10 - Kullanım Takibi

Tablo:

```sql
usage_logs
```

Alanlar:

- request_id
- model
- provider
- input_tokens
- output_tokens
- latency
- user_id

---

# Faz 11 - Yönetim Paneli

## Model Yönetimi

- Model Ekle
- Model Sil
- Prompt Düzenle
- Sağlayıcı Değiştir

## API Key Yönetimi

- Oluştur
- Sil
- Devre Dışı Bırak

## İstatistikler

- Toplam İstek
- Token Kullanımı
- Hata Oranları
- Sağlayıcı Performansı

---

# Faz 12 - Enterprise Özellikleri

- Prompt Versiyonlama
- Model Fallback
- Multi Provider Routing
- Auto Model Selection
- A/B Testing
- Agent Framework
- Workflow Builder
- MCP Marketplace
- Fine-Tuning Yönetimi
- Vision Modelleri
- Ses Modelleri
- Görsel Üretim Modelleri

---

# Veritabanı Tasarımı

## models

```sql
id
name
provider
provider_model
system_prompt
temperature
memory_enabled
tools_enabled
created_at
```

## api_keys

```sql
id
key
owner
created_at
```

## conversations

```sql
id
user_id
model
created_at
```

## messages

```sql
id
conversation_id
role
content
created_at
```

## memories

```sql
id
user_id
content
embedding
created_at
```

## usage_logs

```sql
id
request_id
user_id
model
provider
input_tokens
output_tokens
latency
created_at
```

---

# Teknoloji Seçimi

Backend:

- Node.js
- TypeScript
- Fastify

Database:

- PostgreSQL

ORM:

- Prisma

Cache:

- Redis

Queue:

- BullMQ

Vector Database:

- Qdrant

Monitoring:

- Prometheus
- Grafana

Container:

- Docker

Deployment:

- Coolify
- Dokploy
- Kubernetes (ileride)

---

# İlk MVP Hedefi

- OpenAI uyumlu API
- /v1/models
- /v1/chat/completions
- Streaming
- Model Registry
- Sistem Promptları
- Hafıza Sistemi
- DeepSeek Desteği
- Gemini Desteği
- Ollama Desteği
- API Key Sistemi
- Basit Yönetim Paneli

