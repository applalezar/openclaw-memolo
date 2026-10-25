# Hướng dẫn Setup & Vận hành — Memolo Memory System

## İçindekiler
1. [Quick Setup (Docker)](#1-quick-setup-docker)
2. [Manuel kurulum (dev)](#2-manuel-kurulum-dev)
3. [Yapılandırma](#3-yapılandırma)
4. [Kimlik Doğrulama & Ajan Anahtarları](#4-kimlik-doğrulama--ajan-anahtarları)
5. [LAN Erişimi](#5-lan-erişimi)
6. [Ajan Entegrasyonu](#6-ajen-entegreasyonu)
7. [Dashboard](#7-dashboard)
8. [Çalışma Zamanı Yapılandırması](#8-çalışma-zamanı-yapılandırması)
9. [Çalıştırma & Bakım](#9-çalıştırma--bakım)
10. [Hata Yönetimi](#10-hata-yönetimi)

---

## 1. Quick Setup (Docker)

### Ön Koşullar
- Docker Desktop ≥ 4.0
- Ollama ([ollama.com](https://ollama.com)) host üzerinde çalışmalı (yalnızca embedding için kullanılır)
- MiniMax API key ([platform.minimax.io](https://platform.minimax.io))

```bash
# 1. Ollama embedding modelini çek (host üzerinde çalıştır, Docker içinde değil)
ollama pull qwen3-embedding:8b

# 2. Yapılandır
cd memolo
cp .env.example .env
# .env içinde düzenle:
#   → MEMOLO_MASTER_KEY: rastgele bir diziye değiştir
#   → MINIMAX_API_KEY: MiniMax API anahtarını ekle
# Master key üret: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# 3. Başlat (PG + Qdrant + Server)
docker compose up -d

# 4. Doğrula
curl http://localhost:7437/api/health
# → { "status": "healthy", "checks": { "server": "ok", "postgres": "ok", "qdrant": "ok", "embedding": "ok" } }
```

**Tamam!** Sunucu 7437 portunda çalışır, dashboard aynı URL ile açılır.

---

## 2. Manuel kurulum (dev)

Eğer Docker üzerinden değil, doğrudan geliştirme modunda çalışmak isterseniz:

```bash
# Altyapıyı başlat
docker compose up -d postgres qdrant   # Sadece PG + Qdrant

# Sunucuyu doğrudan çalıştır
docker compose up -d postgres qdrant
cd server
cp ../.env.example .env
# .env dosyasını düzenle: PG_HOST=localhost, QDRANT_HOST=localhost, OLLAMA_BASE_URL=http://localhost:11434
npm install
npm run migrate
npm run dev    # Otomatik yeniden yükleme (node --watch)
```

---

## 3. Yapılandırma

Proje kökünde `.env` dosyası bulunur:

```env
# === ZORUNLU ===
MEMOLO_MASTER_KEY=your-random-secret-key    # Master API anahtarı

# === LLM: MiniMax M2.5 (ZORUNLU) ===
MINIMAX_API_KEY=your-minimax-api-key        # platform.minimax.io'dan alın
MINIMAX_MODEL=MiniMax-M2.5                  # Model adı
MINIMAX_BASE_URL=https://api.minimax.io/anthropic/v1/messages

# === SUNUCU ===
PORT=7437                                    # Sunucu portu

# === EMBEDDING: Ollama (yalnızca embedding için) ===
OLLAMA_BASE_URL=http://host.docker.internal:11434  # Ollama URL'si (Docker)
OLLAMA_EMBED_MODEL=qwen3-embedding:8b
EMBEDDING_DIMENSIONS=4096                    # Vektör boyutları

# === ÖZELLİK BAYRAKLARI (varsayılan: true) ===
ENABLE_FACT_EXTRACTION=true                  # Her etkileşimden atomik gerçek çıkar
ENABLE_RERANKING=true                       # LLM rerank işlemi
ENABLE_GRAPH=true                            # Bilgi grafiği oluştur

# === ZAMAN AŞIMLARI ===
FACT_EXTRACT_TIMEOUT=60000                   # Gerçek çıkarma (ms)
DEDUP_TIMEOUT=60000                         # Deduplication (ms)
RERANK_TIMEOUT=30000                        # Reranking (ms)

# === VERİTABANI (Docker kullanıyorsanız değiştirmeyin) ===
PG_HOST=postgres        # Docker servis adı
PG_PORT=5432
PG_USER=memolo
PG_PASSWORD=memolo_secret
PG_DATABASE=memolo
QDRANT_HOST=qdrant      # Docker servis adı
QDRANT_PORT=6333
```

> ⚠️ **Env değişkeni isimleri**: `PG_HOST` (`POSTGRES_HOST` değil), `OLLAMA_BASE_URL` (`OLLAMA_URL` değil)

---

## 4. Kimlik Doğrulama & Ajan Anahtarları

### 2 katmanlı kimlik doğrulama sistemi

```text
Master Key (env: MEMOLO_MASTER_KEY)
  ├── Tam erişim: tüm endpoint'ler
  ├── Kullanım: ajan kaydı, zeka çalıştırma, yönetici işlemleri
  └── .env içinde tek seferde ayarlanır

Per-Agent Key (DB: agents.api_key)
  ├── Tam erişim: tüm endpoint'ler
  ├── Kayıt sırasında otomatik üretilir
  └── Her ajan için ayrı bir key bulunur
```

### Ajan oluştur + key al

```bash
curl -X POST http://localhost:7437/api/memory/agents/register \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_MASTER_KEY" \
  -d '{"id":"coding-agent","name":"Coding Assistant"}'
```

Cevap:
```json
{
  "success": true,
  "data": {
    "id": "coding-agent",
    "name": "Coding Assistant",
    "apiKey": "a1b2c3d4e5f6...64-char-hex-string",
    "created_at": "2026-02-19T..."
  }
}
```

**`apiKey` değerini ajan konfigürasyonunda saklayın** — tüm API çağrılarında kullanılır.

### Anahtarı yenileme (sızarsa)

```bash
curl -X POST http://localhost:7437/api/memory/agents/coding-agent/regenerate-key \
  -H "X-API-Key: YOUR_MASTER_KEY"
# → Yeni key, eski key geçersiz olur
```

### İsteklerde key gönderme

```text
X-API-Key: agent-api-key-here
```

### İstisnalar
- `GET /api/health` — kimlik doğrulama gerektirmez
- `MEMOLO_MASTER_KEY` tanımlanmamışsa — auth tamamen kapatılır (dev modu)

---

## 5. LAN Erişimi

Sunucu `0.0.0.0:7437` bağlanır → LAN içindeki tüm cihazlardan erişilebilir.

### Sunucu IP'sini bulma

Başlangıç loglarında şu satırlar görünür:

```text
📡 Access URLs:
   Local:     http://localhost:7437
   LAN (Wi-Fi): http://192.168.1.100:7437
   LAN (Ethernet): http://192.168.1.101:7437
```

### Başka cihazdan ajan kullanımı

```javascript
// Başka bir laptop üzerindeki ajan
const memory = new MemoryClient({
  agentId: 'remote-agent',
  apiKey: 'agent-specific-key',
  serverUrl: 'http://192.168.1.100:7437', // sunucu çalışan makinenin IP'si
});
```

### Windows Güvenlik Duvarı
Eğer diğer ajanlar bağlanamıyorsa şu portu açın:

```powershell
# PowerShell (Yönetici)
New-NetFirewallRule -DisplayName "Memolo TCP 7437" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 7437
```

---

## 6. Ajen Entegrasyonu

Memolo, ajanların konuşmalarını izleyerek belleği otomatik olarak yönetir. Ajanlar, doğrudan `memolo_search`, `memolo_store`, `memolo_list`, `memolo_get` ve `memolo_forget` araçlarını kullanabilir.

### Örnek yapılandırma (OpenClaw plugin)

```json
{
  "memolo": {
    "agentId": "coding-agent",
    "agentName": "Coding Assistant",
    "apiKey": "${MEMOLO_API_KEY}",
    "serverUrl": "http://192.168.1.100:7437",
    "autoRecall": true,
    "autoCapture": true,
    "topK": 10,
    "includeOtherAgents": true
  }
}
```

Bu özelliklerle:
- her cevap öncesinde ilgili hafıza aranır,
- her cevap sonrası konuşma hafızaya kaydedilir,
- diğer ajanların paylaşılan bilgilerine erişim verilebilir.

---

## 7. Dashboard

Dashboard, açık hafıza içeriğini izlemenizi, geçmiş konuşmaları aramanızı, bilgi grafiğini görmenizi ve çalışma zamanı ayarlarını yönetmenizi sağlar.

- `http://localhost:3001` adresinde erişilir
- canlı işlem loglarını gösterir
- bellekleri arar ve ilgili sonuçları listeler
- varlık/ilişki ağını görüntüler
- runtime ayarları değiştirir

---

## 8. Çalışma Zamanı Yapılandırması

Yapılandırma örnekleri için `.env.example` dosyasına bakın. Ana parametreleri burada özetledik:

- `MEMOLO_MASTER_KEY`: tüm API erişimi için ana güvenlik anahtarı
- `MINIMAX_API_KEY`: cloud LLM için erişim anahtarı
- `MINIMAX_MODEL`: kullanılacak model adı
- `MINIMAX_BASE_URL`: MiniMax Anthropic uyumlu API URL'si
- `PORT`: Sunucu dinleme portu (varsayılan 7437)
- `OLLAMA_BASE_URL`: embedding için Ollama adresi
- `OLLAMA_EMBED_MODEL`: embedding modeli
- `ENABLE_FACT_EXTRACTION`, `ENABLE_RERANKING`, `ENABLE_GRAPH`: açık/kapalı feature bayrakları

---

## 9. Çalıştırma & Bakım

### Sunucuyu başlatma

```bash
cd memolo
docker compose up -d
```

### Güncelleme

```bash
bash update.sh
# veya
git pull && docker compose up -d --build memolo
```

### Logları izleme

```bash
docker compose logs -f memolo
```

---

## 10. Hata Yönetimi

### Docker çalışmıyor
- Docker Desktop'ın açık olduğundan emin olun.
- `docker compose ps` ile servis durumunu kontrol edin.
- `docker compose logs` ile hataları inceleyin.

### Ollama bağlantısı yok
- Ollama host makinede çalışıyor olmalı.
- `ollama list` komutuyla embedding modelini doğrulayın.

### API cevap vermiyor
- `curl http://localhost:7437/api/health` ile canlılığı kontrol edin.
- `.env` dosyasındaki `MEMOLO_MASTER_KEY` ve `MINIMAX_API_KEY` değerlerini doğrulayın.

### Qdrant/PostgreSQL sorunları
- Docker konteynerlerinin sağlıklı çalıştığından emin olun.
- `docker compose ps` ve `docker compose logs postgres qdrant` kullanın.

---

Bu dosya, Memolo sisteminin kurulumu, kimlik doğrulama, ağ erişimi ve bakım süreçlerini Türkçeye çevrilmiş şekilde özetler.
