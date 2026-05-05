# sentinel-coming

**Sentinel**, MicroK8s + Juju + COS Lite altyapısı üzerinde çalışan, LLM destekli bir gözlemlenebilirlik (observability) ve altyapı yönetim projesidir. Bu repo; ana CLI aracını, altyapı kurulum scriptlerini, skill dokümantasyonlarını ve bağımsız agentic ürünleri bir arada barındıran bir monorepo olarak yapılandırılmıştır.

---

## Repo Yapısı

```
sentinel-coming/
├── cli/                  → Ana ürün: Sentinel CLI
├── observability-gateway/→ CLI icin Grafana'siz observability read-only gateway
├── agentic/              → ⚠️ Bağımsız 3 harici ürün (bkz. aşağıdaki not)
│   ├── Pywen-dev/
│   ├── codex-main/
│   └── cli-claude/
├── skills/               → COS / Juju kurulum skill kılavuzları (Faz 1)
├── cli/skills/           → CLI agent skill dokümanları (Faz 2+, 65+ skill)
├── documantations/       → Mimari, implementasyon planları, faz dokümanları
├── scripts/              → Operasyonel yardımcı scriptler
├── for-download/         → Hazır deployment paketleri ve kurulum scriptleri
└── .github/workflows/    → CI/CD (GitHub Actions)
```

---

## Bileşenler

### `/cli` — Sentinel CLI *(Ana Ürün)*

Python tabanlı, agentic mimariye sahip terminal aracı. Doğal dil komutlarıyla altyapı gözlemleme ve yönetim işlemleri yapılmasını sağlar.

**Teknoloji:** Python 3.11+, pydantic, httpx, PyYAML, rich

**Temel özellikler:**
- `run`, `repl`, `config`, `doctor`, `version` komutları
- Çoklu LLM sağlayıcı desteği: Anthropic (Claude), OpenAI-compatible, yerel modeller (Ollama)
- MCP (Model Context Protocol) entegrasyonu
- Bash ve dosya sistemi araçları; onay/izin iş akışları
- Katmanlı konfigürasyon sistemi (YAML profilleri + env override)
- Oturum kalıcılığı ve trajectory kaydı (debugging için)
- Grafana entegrasyon kontrolleri
- **Proje bellek sistemi** (memory extract, dream konsolidasyonu, magic docs)
- **Gizli bilgi redaksiyonu** (tüm bellek yazımlarında otomatik)
- **Tur sonu pipeline** (hook'lar, away özeti, arka plan thread desteği)

**CI:** GitHub Actions → Python 3.12, ruff lint, pytest

---

#### Proje Bellek Sistemi

Her başarılı tur sonunda agent döngüsü bir **tur sonu pipeline** çalıştırır. Bu pipeline sırasıyla şunları yapar:

1. **`post_turn` hook'u** çalıştırır.
2. **Bellek çıkarma (extract):** Tur mesajlarının özeti `.sentinel/memory/extract.jsonl` veya `~/.sentinel/projects/<hash>/memory/extract.jsonl` dosyasına eklenir.
3. **Away özeti:** `repl` oturumlarında son kullanıcı + asistan mesajından tek satırlık özet üretilir.
4. **Magic Docs güncelleme:** `# MAGIC DOC` başlıklı Markdown dosyalarındaki `<!-- SENTINEL_MAGIC_DOC:BEGIN -->` ... `<!-- SENTINEL_MAGIC_DOC:END -->` bloğu güncel özetle değiştirilir (varsayılan kökler: `skills/`, `documantations/`).
5. **Dream konsolidasyonu:** Süre (min 24 saat) + oturum sayısı (min 5) eşiği sağlandığında LLM ile `index.md` güncellenir; dosya kilidi ile paralel çalışma önlenir.

Tüm bellek yazımları **redaksiyon filtresi**nden geçer; email, API token, Bearer header, export ifadeleri, parola atamaları, kubeconfig kimlik bilgileri, AWS AKIA anahtarları, JWT ve PEM blokları otomatik olarak temizlenir.

**Bellek politikası:**
- `policy: project` → `<proje_dizini>/.sentinel/memory/`
- `policy: user` → `~/.sentinel/projects/<cwd_hash>/memory/`
- `enforce_write_jail: true` → dosya yazma aracı yalnızca bellek köküne izin verir
- `allow_non_interactive: false` → `--bare` / pipe modunda bellek yazımları kapalı

**Konfigürasyon (sentinel.yaml):**
```yaml
memory:
  enabled: false            # bellek sistemini etkinleştirir
  directory: null           # null ise policy ile belirlenir
  policy: project           # "project" | "user"
  extract_on_turn_end: true
  allow_non_interactive: false
  enforce_write_jail: false

dream:
  enabled: false
  min_hours_between: 24
  min_sessions: 5
  lock_stale_sec: 3600

turn_pipeline:
  enabled: true
  run_in_background: false  # true ise arka plan thread'inde çalışır

semantic_compaction:
  inject_trim_notice: true
  persist_session_memory_path: true

magic_docs:
  enabled: false
  title_marker: MAGIC DOC
  begin_marker: "<!-- SENTINEL_MAGIC_DOC:BEGIN -->"
  end_marker: "<!-- SENTINEL_MAGIC_DOC:END -->"
  roots:
    - skills
    - documantations
  max_files: 32

away:
  enabled: true
  max_chars: 200
```

**Ortam değişkenleri ile hızlı etkinleştirme:**
```bash
SENTINEL_MEMORY_ENABLED=1
SENTINEL_DREAM_ENABLED=1
SENTINEL_DREAM_MIN_HOURS=24
SENTINEL_DREAM_MIN_SESSIONS=5
SENTINEL_MEMORY_DIR=~/.sentinel/my-project/memory
```

---

#### Araç Güvenliği İyileştirmeleri

**Bash read-only modu:** `tools.bash_read_only: true` ile bash aracı yalnızca güvenli okuma komutlarına (`ls`, `cat`, `head`, `tail`, `grep`, `find`, `pwd`, `echo`, `wc`, `stat`, `sort`, `uniq`, `git`) izin verir; yönlendirme, pipe, komut substitution ve arka plan çalıştırma engellenir.

**Memory write jail:** `memory.enforce_write_jail: true` ile `write_file` aracı yalnızca bellek kök dizinine yazabilir; bu dizin dışındaki tüm yazma girişimleri reddedilir.

**Yeni hook fazları:** `pre_memory` ve `post_memory` hook fazları eklendi. Bellek extract/dream işlemleri öncesi ve sonrası özel komutlar çalıştırılabilir.

```yaml
hooks:
  enabled: true
  # phase: pre_tool | post_tool | post_turn | pre_memory | post_memory
  commands:
    - phase: pre_memory
      command: ["echo", "bellek yazılıyor"]
      on_error: warn
```

---

### `/observability-gateway` — Sentinel Observability Gateway

FastAPI tabanli bagimsiz bir Python servisi. Sentinel CLI icin Prometheus, Loki ve Tempo'ya yonelik tek bir read-only HTTP giris noktasi saglar. Grafana veri kaynagi veya Grafana API bagimliligi olmadan calisir.

**Teknoloji:** Python 3.11+, FastAPI, httpx, pydantic, pydantic-settings, PyYAML

**Temel ozellikler:**
- `GET /health` ve `GET /api/v1/status` ile servis ve backend durumu ozeti
- Prometheus icin anlik metric sorgusu adapter'i
- Loki icin `query_range` log sorgu adapter'i
- Tempo icin trace arama ve trace detay alma adapter'i
- Secret-safe hata modeli ve backend bagimsiz yanit sekli
- Ortak timeout ve retry ayarlari

---

### `/skills` — COS/Juju Kurulum Skill'leri *(Faz 1)*

MicroK8s, Juju ve COS Lite bileşenlerini adım adım kurmak için hazırlanmış modüler kılavuzlar. Her skill; ön koşulları, uygulama adımlarını ve başarı kriterlerini tanımlar.

Kapsadığı alanlar: MicroK8s kurulumu, addon aktivasyonu, Juju bootstrap, COS charm deploy (Prometheus, Grafana, Loki, Alertmanager, Traefik), relation tanımları, ingress konfigürasyonu.

---

### `/cli/skills` — CLI Agent Skill Dokümanları *(Faz 2+)*

Sentinel CLI'nin agent döngüsüne entegre edilmiş 65+ SKILL.md dosyası. Her biri belirli bir agentic davranışı veya teknik konuyu kapsar:

- LLM provider kontratları (Anthropic, OpenAI-compatible, streaming, retry)
- Araç sistemi (bash, dosya okuma/yazma, web fetch)
- Konfigürasyon katmanları ve profiller
- MCP client/tool mapping
- Test ve entegrasyon kalıpları
- Gözlemlenebilirlik sorun giderme (Loki, Prometheus, Grafana, Traefik, Alertmanager)
- Güvenlik: sandbox hardening, prompt injection guardrail'leri, secrets yönetimi

---

### `/documantations` — Proje Dokümantasyonu

| Dosya | İçerik |
|---|---|
| `PROJECT_ROOT.md` | Monorepo vizyonu, fazlar (0–5), COS Lite mimari özeti, AI otomasyon kuralları |
| `ARCHITECTURE_COS.md` | MicroK8s → Kubernetes → Juju → COS bileşenleri arası ilişki haritası |
| `IMPLEMENTATION_PLAN*.md` | Faz bazlı uygulama adımları, hata kurtarma yolları |
| `cli/documantations/OBSERVABILITY_GATEWAY_AND_AGENT_PLAN.md` | Gateway ile tamamlanan son entegrasyon ve agent/CLI icin sonraki adim |
| `PHASE*_SKILL_AND_DOC_INDEX.md` | Her faza ait skill ve doküman kataloğu |
| `ENV_FLAGS_PHASE*.md` | Faz bazlı environment değişkenleri ve feature flag'ler |
| `GRAFANA_AI_PLATFORM_RESEARCH.md` | Grafana LLM Platform araştırması |
| `INTEGRATION_SENTINEL_CLI_FROM_CLI_CLAUDE.md` | cli-claude'dan Sentinel'e taşınan bellek/güvenlik davranışları referans haritası |

---

### `/scripts` — Operasyonel Scriptler

- **`cos-microk8s-start.sh`** — MicroK8s'i güvenli şekilde başlatır; Kubernetes API, node hazırlığı ve Juju agent stabilizasyonunu bekler. Erken hook çalışmasını önlemek için 45 saniyelik stabilizasyon gecikmesi içerir.
- **`cos-microk8s-heal.sh`** — Host IP değişince Juju kubeconfig drift'ini ve `kubelet.crt` SAN uyumsuzluğunu onarır; gerekirse MicroK8s'i yeniden başlatır.
- **`auto-push-watch.sh`** — Repo değişikliklerini izleyerek otomatik commit + push yapar.

---

### `/for-download` — Hazır Kurulum Paketleri

Doğrudan kullanıma hazır deployment template ve scriptleri:

- **`prepare-env.sh`** — MicroK8s, MetalLB ve Juju'yu otomatik kurar; host IP'ye göre MetalLB IP aralığını hesaplar.
  Ayrıca Juju classic snap altındaki `microk8s` kubeconfig kopyalarını da güncel tutar.
- **`my-product-bundle.yaml`** — Prometheus, Loki, Alertmanager, Grafana, Traefik, Catalogue, Tempo ve OpenTelemetry Collector içeren tam COS Lite Kubernetes bundle'ı.
- **`faz1-telemetry.sh`** — Mevcut COS'a Tempo ve OTEL Collector ekler; metrics/logs/traces pipeline'larını kurar.
- **`faz4-5.sh`** — Deploy sonrası doğrulama: Traefik endpoint'leri, Grafana credential'ları ve OTLP Gateway IP'sini getirir.

---

## Altyapı Mimarisi

```
Host OS
└── MicroK8s (snap)
    └── Kubernetes
        ├── DNS, hostpath-storage, MetalLB
        └── Juju (OLM)
            └── cos model
                ├── Prometheus   ← metrics
                ├── Loki         ← logs
                ├── Tempo        ← traces
                ├── Alertmanager ← alerting
                ├── Grafana      ← visualization
                ├── Traefik      ← ingress
                ├── Catalogue    ← service discovery
                └── OTEL Collector ← telemetry gateway
```

**Juju**, tüm charm'ların yaşam döngüsünü yöneten evrensel bir Operator Lifecycle Manager olarak görev yapar. Bileşenler arası bağlantılar Juju relation'ları üzerinden kurulur (`grafana-source`, `metrics-endpoint`, `ingress`, vb.).

---

## LLM Sağlayıcılar

| Sağlayıcı | Model |
|---|---|
| Anthropic | Claude (Opus, Sonnet, Haiku) |
| OpenAI-compatible | GPT-4, GPT-3.5, Ollama (yerel) |
| Google | Gemini (varsayılan cloud profili: gemini-2.5-flash) |
| Yerel | Gemma (Ollama, yerel profil: gemma4:latest) |

---

## Geliştirme

```bash
# CLI kurulumu
cd cli
pip install -e ".[dev]"

# Linting
ruff check src/

# Test
pytest

# Wheel build
python -m build
```

**Konfigürasyon:**
```bash
cp cli/config/sentinel.example.yaml cli/config/sentinel.yaml
# sentinel.yaml dosyasını düzenle (API key env, LLM ayarları, memory vb.)
```

---

---

> ⚠️ **`/agentic` Klasörü Hakkında Önemli Not**
>
> `agentic/` dizini altındaki **3 klasörün her biri bağımsız bir üründür** ve bu projeyle doğrudan ilgisi yoktur:
>
> | Klasör | Ürün | Teknoloji |
> |---|---|---|
> | `agentic/Pywen-dev/` | **Pywen** — Çok modelli Python araştırma agent framework'ü (Qwen3, Claude, Codex, Gemini destekli) | Python 3.10–3.12, openai, anthropic, tree-sitter, textual |
> | `agentic/codex-main/` | **Codex CLI** — OpenAI'nin kaynak kodlu TypeScript/Rust coding agent'ı | TypeScript, Rust, Bazel, React/Ink |
> | `agentic/cli-claude/` | **Claude Code** — Anthropic'in TypeScript tabanlı kod asistanı | TypeScript, Node.js |
>
> Bu ürünler; ihtiyaç duyulduğunda referans alınmak, belirli bileşenler ödünç alınarak Sentinel'e entegre edilmek üzere burada saklanmaktadır. Özellikle `cli-claude/` içindeki bellek (auto-dream, extract memories, magic docs, away summary) ve güvenlik (tool isolation, hook pipeline) davranışları Sentinel CLI'ye Python'da yeniden yazılarak aktarılmıştır. Sentinel'in ana geliştirme akışından bağımsız olarak değerlendirilmelidir.

---

## Lisans

MIT License — © 2026 Ali Çağlar Koçer
