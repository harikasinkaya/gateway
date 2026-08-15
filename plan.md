Node.js ve Fastify kullanarak OpenAI API uyumlu bir AI Gateway geliştir.

Amaç:
Kullanıcıların OpenAI standartlarında istek atabildiği, model alias sistemi bulunan ve her alias için farklı system prompt tanımlanabilen hafif bir gateway oluşturmak.

Teknolojiler:

* Node.js
* Fastify
* Native fetch
* JSON tabanlı konfigürasyon
* Veritabanı kullanma

Dosya yapısı:

```text
/
├── package.json
├── server.js
├── models.json
└── prompts/
    ├── barlas-v1.md
    ├── barlas-v2.md
    └── barlas-coder.md
```

Gereksinimler:

1. OpenAI uyumlu endpoint oluştur:

```http
POST /v1/chat/completions
```

2. models.json dosyasından model konfigürasyonlarını oku.

Örnek:

```json
{
  "barlas-v1": {
    "provider": "deepseek",
    "model": "deepseek-chat",
    "prompt": "./prompts/barlas-v1.md"
  },
  "barlas-coder": {
    "provider": "openai",
    "model": "qwen3-coder",
    "prompt": "./prompts/barlas-coder.md"
  }
}
```

3. İstek geldiğinde:

* model alanını kontrol et
* alias yapılandırmasını bul
* prompt dosyasını oku
* system mesajı olarak messages dizisinin başına ekle

Örnek:

Kullanıcı:

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

Gateway:

```json
{
  "model": "deepseek-chat",
  "messages": [
    {
      "role": "system",
      "content": "...prompt dosyası..."
    },
    {
      "role": "user",
      "content": "Merhaba"
    }
  ]
}
```

4. Provider sistemi oluştur.

Şimdilik desteklenecek providerlar:

* deepseek
* openai

Provider mantığı ayrı fonksiyonlar halinde yazılsın.

5. Ortam değişkenleri:

```env
DEEPSEEK_API_KEY=
OPENAI_API_KEY=
PORT=3000
```

6. Gerçek sağlayıcıya istek gönder.

DeepSeek:

```text
https://api.deepseek.com/v1/chat/completions
```

OpenAI:

```text
https://api.openai.com/v1/chat/completions
```

7. Dönen cevabı mümkün olduğunca OpenAI formatını bozmadan kullanıcıya ilet.

8. Ek endpointler:

```http
GET /v1/models
GET /health
```

9. Hata yönetimi:

* model bulunamadı
* prompt dosyası yok
* provider desteklenmiyor
* API hataları

için düzgün JSON response döndür.

10. Kod kalitesi:

* Modüler yapı
* Açıklayıcı yorumlar
* any kullanma
* Gereksiz bağımlılık ekleme
* Production-ready kod yaz

11. Bonus:

Streaming desteği için altyapıyı hazır bırak ancak ilk sürümde implement etmek zorunlu değil.

Görevi tamamladıktan sonra:

* Tüm dosyaları oluştur
* package.json hazırla
* npm install sonrası çalışabilir hale getir
* örnek models.json ve prompt dosyalarını ekle
* çalıştırma talimatlarını README.md içinde açıkla
