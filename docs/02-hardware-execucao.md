# 02 — O que roda na minha máquina + execução

## Realidade da minha máquina
- Desktop Ryzen 5 5600X, 32 GB RAM, RTX 3050 8 GB.
- **Rede cabeada (Realtek GbE). Sem WiFi ativo** (`wlansvc` parado).
- Consequência: o modo "custo zero" (RSSI via `netsh`) **não funciona sem um rádio WiFi**.

## Matriz de execução

| Modo | Precisa de | Roda aqui hoje? | O que entrega |
|---|---|---|---|
| **Docker `simulated`** | Docker (tem) | ✅ Sim | Stack full (REST/WS/dashboard) com CSI sintético |
| **`make verify`** | Python + numpy/scipy | ✅ Sim | Prova determinística do pipeline de sinal |
| **RSSI Windows** | Adaptador WiFi ativo | ❌ Não (sem WiFi) | Presença/movimento reais |
| **ESP32 CSI** | 1–3× ESP32-S3 (~US$9) | ⛔ Falta HW | Respiração/batimento/queda/pose |

## GOTCHA importante (confirmado no docker-compose)
O `CSI_SOURCE=auto` **falha com exit 78** se não detectar ESP32 nem WiFi
(mudança do issue #937 — não há mais fallback sintético silencioso).
**Para demo sem hardware, force `CSI_SOURCE=simulated`.**

---

## Fase 0 — Rodar agora, sem hardware

### A) Verificação do pipeline (mais rápido, prova que tudo bate)
```powershell
cd "E:\Teleperformance CRM SA\Arquitetura\Fontes\WifiSense\RuView"
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt        # inclui torch — pesado; ver nota GPU abaixo
make verify                            # ou: python archive/v1/data/proof/verify.py
```

### B) Docker simulation (stack completa + dashboard)
```powershell
cd "E:\Teleperformance CRM SA\Arquitetura\Fontes\WifiSense\RuView"
$env:CSI_SOURCE = "simulated"          # OBRIGATÓRIO sem hardware
docker compose -f docker/docker-compose.yml up sensing-server
# REST:  http://localhost:3000
# WS:    ws://localhost:3001
```
> Primeira build compila Rust dentro do container — pode demorar. Alternativa mais leve:
> `docker pull ruvnet/wifi-densepose:latest` (imagem pré-buildada).

---

## Nota sobre a RTX 3050 (8 GB) e PyTorch
- `requirements.txt` pede `torch>=1.12`. O `pip install torch` padrão no Windows costuma
  vir **CPU-only**. Para usar a 3050, instalar a build CUDA:
  ```powershell
  pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124
  ```
- 8 GB de VRAM: ótimo para **inferência** e **fine-tune leve** (LoRA/contrastive). Treino
  do zero de modelos grandes de pose provavelmente não cabe — mirar em fine-tune/adaptação.
- O runtime de inferência em produção do RuView usa **Candle** (Rust ML), não PyTorch;
  PyTorch é mais para os fluxos de treino/experimentação.

---

## Fase 1 — Habilitar sensing real barato (próximo passo lógico)
Duas opções para ganhar um rádio WiFi neste desktop:
1. **Dongle USB WiFi** (~R$40–120). Depois: iniciar o serviço e validar:
   ```powershell
   Start-Service wlansvc
   netsh wlan show interfaces      # precisa aparecer State: connected
   ```
2. **Ativar o WiFi onboard** da placa-mãe (se houver módulo M.2 CNVi/PCIe).

Com WiFi ativo → roda o modo RSSI (presença/movimento reais) do `archive/v1/src/sensing`.

## Fase 2 — CSI de verdade (ESP32-S3)
- 1 placa já dá respiração/presença; 3 placas melhoram pose/contagem.
- Firmware pré-buildado em `RuView/firmware/esp32-csi-node/`.
- Flashar com `esptool` (a compilação do firmware no Windows pede WSL2 ou PowerShell puro —
  Git Bash conflita com o ESP-IDF).
- Aqui fecha o ciclo: ESP32 → coletor → modelo (RTX 3050) → métricas (Grafana/OTel).
