# 04 — Mapa completo de capacidades (o que dá pra fazer)

Fonte: `RuView/docs/edge-modules/` — **65 módulos WASM** em 12 categorias, cada um roda
localmente no ESP32 (<10ms, sem nuvem). Aqui está o mapa inteiro, marcado por **tier de
hardware** e por **relevância aos meus 4 alvos**.

Legenda de HW:
- 🟢 **RSSI** — WiFi comum (funciona no notebook Fedora, sem hardware extra)
- 🔵 **CSI** — exige ESP32-S3 (52 subportadoras) ou NIC Intel 5300/Atheros
- 🟣 **Multi-nó/modelo** — CSI de várias placas + inferência (RTX 3050 ajuda)

Alvos escolhidos: **[P]**resença/home-office · **[S]**ono/respiração · **[Q]**ueda · **[O]**bs/OTLP

---

## As 12 categorias (catálogo real)

| # | Categoria | Módulos | HW | Exemplos (Event IDs) |
|---|---|---|---|---|
| 1 | **Core** | 7 | 🟢🔵 | presença, movimento, gesto, coerência, anomalia (0–99) |
| 2 | **Medical & Health** | 5 | 🔵 | apneia, bradicardia, taquicardia, convulsão (100–199) **[S]** |
| 3 | **Security & Safety** | 6 | 🔵 | intrusão, perímetro, loitering, pânico, **queda** (200–299) **[Q]** |
| 4 | **Smart Building** | 5 | 🔵 | zona ocupada, HVAC, luz, elevador, reunião (300–399) **[P]** |
| 5 | **Retail & Hospitality** | 5 | 🔵🟣 | fila, dwell zone, fluxo, turnover (400–499) |
| 6 | **Industrial** | 5 | 🔵 | proximidade, espaço confinado, vibração (500–599) |
| 7 | **Exotic & Research** | 10 | 🔵🟣 | estágio do sono, emoção, língua de sinais, chuva (600–699) **[S]** |
| 8 | **Signal Intelligence** | 6 | 🔵 | atenção, coherence gate, compressão, recuperação (700–729) |
| 9 | **Adaptive Learning** | 4 | 🟣 | gesto aprendido, atrator, adaptação, EWC (730–759) |
| 10 | **Spatial & Temporal** | 6 | 🟣 | influência, match HNSW, tracking spiking, LTL, GOAP (760–819) |
| 11 | **AI Security** | 2 | 🔵 | replay attack, injeção, jamming, comportamento (820–849) |
| 12 | **Quantum & Autonomous** | 4 | 🟣 | inferência, regra disparada, mesh reconfig (850–899) |

> São primitivas componíveis: cada módulo lê CSI e emite eventos. Você combina vários
> num "app" (ex.: quarto = presença + respiração + queda).

---

## As 12 primitivas do Host API (a "linguagem" que você programa)
Todo módulo conversa com o ESP32 por 12 funções — é o seu kit de LEGO:

| Função | Retorna | Uso |
|---|---|---|
| `csi_get_presence()` | 0/1 | tem alguém? **[P]** |
| `csi_get_motion_energy()` | f32 | nível de movimento **[P][Q]** |
| `csi_get_bpm_breathing()` | f32 | respiração **[S]** |
| `csi_get_bpm_heartrate()` | f32 | batimento **[S]** |
| `csi_get_n_persons()` | int | contagem de pessoas **[P]** |
| `csi_get_phase(i)` / `_amplitude(i)` / `_variance(i)` | f32 | sinal cru por subportadora |
| `csi_get_phase_history(buf,max)` | int | histórico p/ tendência (queda, sono) **[S][Q]** |
| `csi_emit_event(id,val)` | — | dispara detecção → agregador **[O]** |
| `csi_log` / `csi_get_timestamp` | — | serial / relógio |

---

## Meus 4 alvos → pipeline concreto

### [P] Presença / produtividade home-office
- **Menor caminho (🟢 no notebook Fedora HOJE):** `rssi_collector` → `feature_extractor`
  → `classifier` (absent/still/active). Sem hardware extra.
- **Ação:** presente >X min → cena (luz/música/"on air"); ausente >Y min → pausa timer.
- **Upgrade 🔵:** módulo Smart Building "zone occupied" com contagem de pessoas.

### [S] Sono / respiração
- **🔵 exige ESP32** (RSSI não tem resolução p/ respiração — confirmado no doc 02).
- Pipeline: CSI → `csi_get_bpm_breathing()` + `phase_history` → estágio de sono (cat. Exotic).
- **Ação:** série temporal de rpm a noite → Grafana; alerta de apneia (cat. Medical).

### [Q] Alerta de queda
- **🔵 exige ESP32.** Módulo Security "fall" = impacto (pico de motion_energy) + imobilidade
  (motion→0 sustentado). Resposta <200ms.
- **Ação:** evento 2xx → notificação Telegram/Home Assistant.

### [O] Exporter OTLP + dashboards (independe do HW final)
- **Ponte:** eventos `csi_emit_event(id,val)` chegam via UDP ao agregador → você escreve um
  **exporter OTLP** (Rust ou .NET) que vira métricas: `wifi_presence`, `wifi_breathing_bpm`,
  `wifi_fall_total`, `wifi_persons`. Dashboards Grafana/LGTM — teu forte.
- Já tem `prometheus-client` no `requirements.txt` → caminho curto p/ /metrics.

---

## "O que mais dá pra fazer" (além dos 4)
- **Gestos no ar** (cat. Core/Adaptive): controlar luz/mídia com a mão, sem toque.
- **Contagem de pessoas por cômodo** → automação HVAC/energia (cat. Smart Building).
- **Segurança sem câmera** (cat. Security): movimento em área que deveria estar vazia,
  detecção de loitering, "panic pattern".
- **Emoção/atenção** (cat. Exotic/Signal Intelligence) — experimental, mas curioso.
- **Detecção de jamming/replay/injeção** (cat. AI Security) — cruzamento com teu lado
  segurança/pentest.
- **Self-learning do ambiente** (cat. Adaptive): calibra ~30s e adapta ao seu cômodo
  (contrastive embeddings) — a RTX 3050 entra aqui p/ fine-tune.
- **Language of signs / detecção de chuva / sleep staging** — os "exotic", pra brincar.
- **Home Assistant / Matter / Apple/Google/Alexa**: o RuView já fala; dá pra expor os
  sensores como entidades nativas do HA.

## Realidade / limites (não iludir)
- Respiração/batimento/pose exigem CSI + ambiente controlado; não é grau médico.
- "Através da parede" é condicional, não raio-X.
- Multipath, mobília e várias pessoas degradam sinal — é engenharia de sinal.
