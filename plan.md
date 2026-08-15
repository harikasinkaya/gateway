# OpenAI-Compatible API Gateway — Build Plan

## Proje Amacı
OpenAI API şemasına (v1) %100 uyumlu bir gateway inşa et. Gateway, `model` alanına göre
kayıtlı "custom model"leri (örn. `barlas-v1`, `barlas-v2`) çözer, her birine özel system
prompt + memory (konuşma hafızası) enjekte eder, gerçek bir backend LLM sağlayıcısına
(OpenAI/Anthropic/local) yönlendirir ve cevabı yine OpenAI şemasında döner.

Stack: **FastAPI (Python 3.11+) + Pydantic v2 + PostgreSQL + Redis + Docker**

---

## FAZ 0 — Proje İskeleti

- [ ] `pyproject.toml` / `requirements.txt` oluştur: `fastapi`, `uvicorn[standard]`, `pydantic`,
      `sqlalchemy`, `asyncpg`, `redis`, `httpx`, `python-dotenv`, `alembic`
- [ ] Klasör yapısını kur:
  ```
  app/
    main.py
    core/
      config.py          # env vars, settings
      security.py        # API key auth
    api/
      v1/
        chat.py           # /v1/chat/completions
        completions.py    # /v1/completions (legacy)
        models.py         # /v1/models, /v1/models/{id}
        embeddings.py      # /v1/embeddings
    schemas/
      openai_chat.py       # Pydantic: ChatCompletionRequest/Response/Chunk
      openai_common.py      # Error şeması, ortak tipler
    registry/
      models.py             # Model config CRUD (Postgres)
      db.py
    providers/
      base.py                # Provider interface (generate, stream)
      openai_provider.py
      anthropic_provider.py
      local_provider.py       # vLLM/Ollama vb.
    memory/
      store.py                 # Redis tabanlı conversation memory
    middleware/
      auth.py
      rate_limit.py
      logging.py
  docker-compose.yml   # api + postgres + redis
  Dockerfile
  .env.example
  tests/
  ```
- [ ] `.env.example` hazırla: `DATABASE_URL`, `REDIS_URL`, `OPENAI_API_KEY`,
      `ANTHROPIC_API_KEY`, `GATEWAY_MASTER_KEY`
- [ ] `docker-compose.yml`: postgres:16, redis:7, api servisi (hot-reload ile)

**Kabul kriteri:** `docker compose up` sonrası `GET /health` 200 dönüyor.

---

## FAZ 1 — OpenAI Şemaları (Pydantic)

- [ ] `ChatMessage` modeli: `role` (`system|user|assistant|tool`), `content` (str veya
      multimodal list), `name` (opsiyonel), `tool_calls`, `tool_call_id`
- [ ] `ChatCompletionRequest`: `model`, `messages`, `temperature`, `top_p`, `n`, `stream`,
      `stop`, `max_tokens`, `presence_penalty`, `frequency_penalty`, `logit_bias`, `user`,
      `tools`, `tool_choice`, `response_format`
- [ ] `ChatCompletionResponse`: `id` (`chatcmpl-...`), `object: "chat.completion"`,
      `created`, `model`, `choices[]` (`index`, `message`, `finish_reason`), `usage`
      (`prompt_tokens`, `completion_tokens`, `total_tokens`)
- [ ] `ChatCompletionChunk` (streaming): `object: "chat.completion.chunk"`,
      `choices[].delta` (`role`/`content` parçalı), `finish_reason: null|"stop"`
- [ ] `ErrorResponse`: `{"error": {"message": str, "type": str, "param": str|null, "code": str|null}}`
- [ ] `ModelObject`: `id`, `object: "model"`, `created`, `owned_by`
- [ ] `EmbeddingRequest` / `EmbeddingResponse`

**Kabul kriteri:** Gerçek OpenAI Python SDK response objeleriyle field-by-field birebir eşleşiyor.

---

## FAZ 2 — Model Registry (barlas-v1, barlas-v2 mantığı)

- [ ] Postgres tablosu `model_configs`:
  ```sql
  id UUID PK
  model_id TEXT UNIQUE           -- "barlas-v1"
  backend_provider TEXT          -- "openai" | "anthropic" | "local"
  backend_model TEXT             -- "gpt-4o-mini"
  system_prompt TEXT
  default_params JSONB           -- {temperature, max_tokens, top_p...}
  memory_enabled BOOLEAN DEFAULT true
  is_active BOOLEAN DEFAULT true
  created_at, updated_at
  ```
- [ ] Alembic migration + seed script: `barlas-v1` ve `barlas-v2` örnek kayıtları
- [ ] `registry/models.py`: `get_model_config(model_id)`, `list_models()`,
      `upsert_model_config(...)` fonksiyonları
- [ ] (Opsiyonel ama önerilir) `/admin/models` CRUD endpoint'leri — yeni model eklemeyi
      koddan değil API'den yapabilmek için (auth: sadece master key)

**Kabul kriteri:** `barlas-v1` ve `barlas-v2` DB'de farklı `system_prompt` ile kayıtlı,
`GET /v1/models` ikisini de listeliyor.

---

## FAZ 3 — Provider Adapter Katmanı

- [ ] `providers/base.py`: soyut interface
  ```python
  class BaseProvider(ABC):
      async def generate(self, messages, params) -> dict: ...
      async def stream(self, messages, params) -> AsyncIterator[dict]: ...
  ```
- [ ] `openai_provider.py`: gerçek OpenAI API'sini `httpx.AsyncClient` ile çağırır
      (kendi gateway'in aslında OpenAI'yi proxy'liyor olabilir)
- [ ] `anthropic_provider.py`: Anthropic Messages API'sine çevirip çağırır (role mapping,
      system prompt ayrı parametre olduğu için dikkat)
- [ ] `local_provider.py`: vLLM/Ollama gibi OpenAI-uyumlu local sunuculara passthrough
- [ ] Provider factory: `get_provider(backend_provider: str) -> BaseProvider`

**Kabul kriteri:** Aynı `messages` listesi 3 farklı provider'a da gönderilip normalize
edilmiş aynı formatta cevap dönebiliyor.

---

## FAZ 4 — Prompt Injection + Memory

- [ ] `memory/store.py`: Redis key şeması `memory:{model_id}:{session_id}` → mesaj listesi
      (JSON, TTL ile, örn. 24 saat)
- [ ] `session_id` çözümü: `user` alanından veya custom header `X-Session-Id`'den al
- [ ] İstek akışı (`chat.py` içinde):
  1. `model_id` → registry'den config çek (404 ise OpenAI-uyumlu error dön:
     `model_not_found`)
  2. `memory_enabled` ise Redis'ten geçmişi çek, mesaj dizisinin başına ekle
  3. Config'teki `system_prompt`'u en başa inject et (kullanıcı `system` mesajı
     gönderdiyse: **birleştirme stratejisine karar ver** — örn. kullanıcı system'i
     gateway system'inin altına ek not olarak eklenir)
  4. `default_params` + kullanıcı override'larını merge et (kullanıcı isteği öncelikli)
  5. Provider'ı çağır
  6. Cevabı memory'e yaz (user mesajı + assistant cevabı)

**Kabul kriteri:** Aynı `session_id` ile art arda 2 istek atıldığında model önceki
mesajı hatırlıyor; `barlas-v1` ve `barlas-v2` aynı input'a farklı persona ile cevap veriyor.

---

## FAZ 5 — Endpoint'ler

- [ ] `POST /v1/chat/completions` — non-streaming + streaming (SSE) ikisi de
  - SSE format: `data: {...}\n\n` ... son chunk `data: [DONE]\n\n`
  - `Content-Type: text/event-stream`
- [ ] `GET /v1/models` — registry'deki tüm aktif modelleri OpenAI `ModelObject` formatında listele
- [ ] `GET /v1/models/{model_id}` — tekil model detay, yoksa 404 (OpenAI error formatında)
- [ ] `POST /v1/completions` — legacy text completion (chat.completions'a wrap edilebilir)
- [ ] `POST /v1/embeddings` — memory/RAG için gerekiyorsa gerçek embedding provider'a passthrough
- [ ] Global exception handler: her hata OpenAI error şemasına sarılıp doğru HTTP status
      ile dönsün (401, 404, 429, 500)

**Kabul kriteri:** Gerçek `openai` Python SDK'sı `base_url` değiştirilerek kullanılabiliyor,
hiçbir parse hatası vermiyor.

---

## FAZ 6 — Auth & Rate Limiting

- [ ] `Authorization: Bearer sk-...` header validasyonu (kendi key sistemin — Postgres'te
      `api_keys` tablosu: `key_hash`, `owner`, `is_active`, `rate_limit_rpm`)
- [ ] Redis token-bucket ile rate limit; response header'larına
      `x-ratelimit-limit-requests`, `x-ratelimit-remaining-requests` ekle
- [ ] Key yoksa/expired ise `401` + `{"error": {"type": "invalid_request_error", "code": "invalid_api_key"}}`

**Kabul kriteri:** Geçersiz key ile istek atınca OpenAI'nin döndüğü hata formatının birebir aynısı dönüyor.

---

## FAZ 7 — Observability & Test

- [ ] Structured logging (request id, model_id, latency, token usage) — `structlog` veya benzeri
- [ ] `tests/test_chat_completions.py`: gerçek `openai` SDK'sını kullanarak entegrasyon testi
  ```python
  client = OpenAI(base_url="http://localhost:8000/v1", api_key="sk-test")
  resp = client.chat.completions.create(model="barlas-v1", messages=[...])
  ```
- [ ] Streaming testi (chunk'ların sırayla geldiğini ve `[DONE]` ile bittiğini doğrula)
- [ ] Memory testi (aynı session_id ile 2. mesajda önceki context'in geldiğini doğrula)
- [ ] Load test (opsiyonel): `locust` veya `k6` ile

**Kabul kriteri:** `pytest` tüm testler yeşil, CI pipeline'da (GitHub Actions) otomatik çalışıyor.

---

## FAZ 8 — (Opsiyonel) İleri Seviye

- [ ] Function calling / tool use desteği (`tools`, `tool_choice` parametrelerini backend'e
      doğru map et)
- [ ] `response_format: {"type": "json_object"}` desteği
- [ ] Usage/billing tracking (kullanıcı bazlı token sayacı, Postgres'e yaz)
- [ ] Admin dashboard (basit bir Next.js/React panel — model ekle/düzenle, key yönetimi)
- [ ] Çoklu model versiyonlama (`barlas-v1@2024-06-01` gibi tarih pinli sürümler)

---

## Notlar / Kararlar (opencode'un sorması muhtemel noktalar)

1. Backend olarak gerçek bir LLM API'si mi (OpenAI/Anthropic) yoksa local model mi
   kullanılacak? (Provider seçimini buna göre önceliklendir.)
2. `system_prompt` ile kullanıcının gönderdiği `system` mesajı çakışırsa hangisi kazanır?
3. Memory scope: kullanıcı bazlı mı (`user` alanı), yoksa session bazlı mı (`X-Session-Id`)?
4. Rate limit ve auth production'da mı yoksa ilk fazda skip mi edilecek?

Bu planı Faz 0'dan başlayarak sırayla uygula; her faz sonunda "Kabul kriteri"ni
karşılayan bir commit at.
