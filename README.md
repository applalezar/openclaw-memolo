# 🧠 Memolo — Çok Ajanlı Yapay Zeka için Akıllı Bellek Sistemi

> Kalıcı, kendini organize eden bellek için AI ajanlar — atomik gerçek çıkarımı, bilgi grafiği, LLM deduplication ve LAN içindeki anlamsal hatırlama

## Hızlı Başlat

### Ön Koşullar

- **[Docker Desktop](https://www.docker.com/products/docker-desktop/)** — PostgreSQL, Qdrant, Sunucu ve Dashboard çalıştırır
- **[Ollama](https://ollama.com)** — embedding üretimi için host makinesinde çalışır (GPU gerektirir)
- **[MiniMax API Anahtarı](https://platform.minimax.io)** — gerçek çıkarımı ve deduplication için cloud LLM

### Kurulum

```bash
# 1. Klonla
git clone https://github.com/thdeptrai/memolo.git
cd memolo

# 2. Embedding modelini çek
ollama pull qwen3-embedding:8b

# 3. Yapılandır
cp .env.example .env
# .env içinde şunları ayarla:
#   MEMOLO_MASTER_KEY=<rastgele string>
#   MINIMAX_API_KEY=<MiniMax API anahtarın>

# 4. Her şeyi başlat
docker compose up -d

# ✅ Hazır!
# API:       http://localhost:7437
# Dashboard: http://localhost:3001
```

### En Son Sürüme Güncelleme

```bash
bash update.sh
# veya: git pull && docker compose up -d --build memolo
```

---

## Memolo Nasıl Çalışır

Memolo, ham ajan konuşmalarını yapılandırılmış, aranabilir ve kendini sürdüren bir bilgiye dönüştüren **yerel-öncelikli bir AI bellek sistemidir**. Bir ajan mesaj gönderdiğinde ne olur:

### 📥 Depolama Akışı — `POST /api/memory/store` çağrıldığında ne olur

```text
KULLANICI MESAJI + AJAN CEVABI
             │
             ▼
     ┌───────────────────┐
     │ 1. Ham Veriyi Kaydet │  Exchange PostgreSQL'e sıra numarasıyla kaydedilir
     └───────┬───────────┘
             │
             ▼
     ┌───────────────────┐
     │ 2. Toplu Kuyruk   │  Exchange, bellek tamponuna eklenir
     │ (batch enabled)   │  (cloud sağlayıcılar için LLM çağrılarını azaltır)
     └───────┬───────────┘
             │
             ├──── (her 10s veya batch dolunca boşaltılır) ────┐
             │                                               │
             ▼                                               ▼
     ┌───────────────────┐                           ┌──────────────────┐
     │ 3. Gerçekleri Çıkar │                           │ 5. Grafiği Güncelle │
     │ + Dedup (birleşik)│  1 LLM çağrısı / batch:      │                  │
     │                   │   • Atomik gerçekler çıkar   │ Varlıklar → DB   │
     │                   │   • EKLE/GÜNCELLE/SİL/YOK │ Birlikte geçenler │
     │                   │   • Kullanıcı + Ajan gerçekleri │ ilişkiler        │
     └───────┬───────────┘                           └──────────────────┘
             │
             ▼
     ┌───────────────────┐
     │ 4. Embed & Kaydet │  4096 boyutlu embedding üret, Qdrant'a kaydet
     │                   │  anlamsal arama için
     └───────────────────┘
```

### 📤 Hatırlama Akışı — `POST /api/memory/recall` çağrıldığında ne olur

```text
SORGULAR: "Kullanıcı hangi araçları tercih ediyor?"
             │
             ▼
     ┌───────────────────┐
     │ 1. Sorgu Zenginleştirme │  Konuşma profili kullanılarak soru zenginleştirilir
     └───────┬───────────┘  (aktif konu, tercih, bağlam)
             │
             ├────────────────────┬──────────────────┐
             ▼                    ▼                   ▼
     ┌──────────────┐   ┌──────────────┐    ┌──────────────┐
     │ 2. Anlamsal   │   │ 3. Çapraz-   │    │ 4. Bilgi     │
     │ Arama        │   │ Ajan Arama  │    │ Grafiği     │
     │ (Qdrant,     │   │ (paylaşımlı KB) │   │ Sorgula     │
     │ konu farkındalı)│ (ortak bilgi) │  (komşular) │
     └──────────────┘   └──────────────┘    └──────────────┘
```

---

## Özellikler

- **Atomik gerçek çıkarımı**: konuşmalardan anlamlı olgular çıkarır
- **Bilgi grafiği**: varlıklar ve ilişkiler oluşturur
- **LLM deduplication**: benzer ve tekrarlanan hafızaları birleştirir
- **Anlamsal geri çağırma**: belleği doğal dil sorularıyla bulur
- **Ağ içinde paylaşımlı bellek**: aynı LAN içindeki diğer ajanlar erişebilir
- **Ayrı ajana özel bellek**: her ajan kendi hafızasına sahip olabilir
- **Dahili dashboard**: canlı loglar, hafıza araştırması, bilgi grafiği ve çalışma zamanı ayarları

## Temel Bileşenler

- **Server**: Express.js API + akıllı hafıza katmanı
- **PostgreSQL**: konuşma, exchange, hafıza, varlıklar, ilişkiler, bilgi tabanı, geçici geçmiş,
  ajanlar ve konuşma-hafıza eşlemeleri için 9 tablo
- **Qdrant**: 4096 boyutlu vektör embeddingleri ve anlamsal arama
- **MiniMax M2.5**: gerçek çıkarımı, deduplication ve rerank için cloud LLM
- **Ollama**: yerel embedding oluşturma (`qwen3-embedding:8b`)
- **Dashboard**: Next.js arayüzü ile canlı izleme ve hafıza keşfi

## Hızlı referans

### API endpointleri

- `POST /api/memory/store` — konuşmayı kaydet
- `POST /api/memory/recall` — benzer hafızaları bulup getir
- `POST /api/memory/agents/register` — ajan kaydı oluştur
- `POST /api/memory/agents/:id/regenerate-key` — ajan API anahtarını yenile
- `GET /api/health` — sistem sağlığını kontrol et

### Ana ortam değişkenleri

```env
MEMOLO_MASTER_KEY=your-random-secret-key
MINIMAX_API_KEY=your-minimax-api-key
MINIMAX_MODEL=MiniMax-M2.5
MINIMAX_BASE_URL=https://api.minimax.io/anthropic/v1/messages
PORT=7437
OLLAMA_BASE_URL=http://host.docker.internal:11434
OLLAMA_EMBED_MODEL=qwen3-embedding:8b
EMBEDDING_DIMENSIONS=4096
```

---

Bu dosya, Memolo projesinin ana kavramlarını ve kullanım akışını özet şekilde Türkçeleştirir.
