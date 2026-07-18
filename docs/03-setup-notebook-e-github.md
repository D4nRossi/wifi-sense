# 03 — Setup no notebook (Fedora) + criar o repo no GitHub

Guia para retomar o projeto no **notebook Fedora** (tem WiFi ativo → destrava o modo RSSI
de graça) e publicar como **repo próprio no GitHub**.

> ✅ Confirmado no código: `archive/v1/src/sensing/rssi_collector.py` tem um
> **`LinuxWifiCollector`** que lê `/proc/net/wireless` + `iw dev <iface> station dump`, e um
> factory que **auto-seleciona o coletor por plataforma**. No Fedora roda nativo e sem a
> latência do `netsh` do Windows.

---

## Nome do repo
**Recomendado:** `wifi-sense`  (claro, descreve "sensoriamento por WiFi", não confunde com
scanner de redes). Alternativas com personalidade: `casa-sense`, `rf-presence`, `wispr`
(WiFi Sensing Presence Recognition). Os docs abaixo usam `wifi-sense` — troque à vontade.

---

## Parte A — Criar o repo no GitHub

O workspace já está pronto pra virar repo: tem `README.md`, `docs/`, `.gitignore`.
Cuidado único: **não versionar o clone do upstream** (`RuView/`, 150 MB, tem `.git` próprio).

### Opção 1 (recomendada) — Upstream como submodule
```bash
cd ~/projs/wifi-sense          # onde você montar o workspace no Fedora
rm -rf RuView                  # remove clone solto; submodule recria apontando pro upstream

git init
git add README.md docs .gitignore
git commit -m "docs: workspace inicial wifi-sense (RuView / WiFi sensing)"

git submodule add https://github.com/ruvnet/RuView.git RuView
git commit -m "chore: RuView upstream como submodule"

gh repo create wifi-sense --private --source=. --push   # precisa: gh auth login
```

### Opção 2 (mais simples) — só teus docs, ignora o RuView
`.gitignore` já exclui `/RuView/`.
```bash
git init && git add . && git commit -m "docs: workspace inicial wifi-sense"
gh repo create wifi-sense --private --source=. --push
```

### Licença
RuView é **MIT**. Se reusar código dele, mantenha o aviso MIT. Teus docs próprios podem
levar a licença que preferir (MIT também é uma boa).

---

## Parte B — Preparar o Fedora

```bash
# ferramentas de rede WiFi (o iw é usado pelo LinuxWifiCollector)
sudo dnf install -y iw python3 python3-pip git make

# confirmar que o WiFi está conectado (destrava o modo RSSI):
nmcli device status                 # wlan0/wlp* deve estar "connected"
cat /proc/net/wireless              # tem que listar sua interface + nível de sinal
iw dev                              # nome da interface (ex.: wlp2s0)
```
Se `/proc/net/wireless` listar a interface, o **modo RSSI (presença/movimento real, sem
hardware extra)** funciona.

---

## Parte C — Ordem de execução no notebook

```bash
# 0. Clonar TEU repo (e o submodule, se usou Opção 1)
git clone https://github.com/<seu-usuario>/wifi-sense.git
cd wifi-sense
git submodule update --init --recursive      # se Opção 1
# (Opção 2: git clone --depth 1 https://github.com/ruvnet/RuView.git RuView)

cd RuView
python3 -m venv .venv && source .venv/bin/activate
```

1. **Sanity check do pipeline (leve):**
   ```bash
   pip install numpy scipy
   make verify
   ```

2. **Modo RSSI real (nativo no Fedora, com WiFi):**
   ```bash
   export PYTHONPATH=archive/v1
   python archive/v1/tests/integration/live_sense_monitor.py
   # Dashboard a cada ~3s: absent / still / active + confiança
   ```
   > Se pedir a interface, use o nome do `iw dev` (ex.: `wlp2s0`). O factory detecta Linux
   > e usa o `LinuxWifiCollector` automaticamente.

3. **Docker simulation (stack completa):**
   ```bash
   export CSI_SOURCE=simulated          # OBRIGATÓRIO sem ESP32/WiFi (senão exit 78)
   docker compose -f docker/docker-compose.yml up sensing-server
   # REST http://localhost:3000  |  WS ws://localhost:3001
   ```

4. **(Opcional) GPU do notebook no PyTorch:** instalar build CUDA
   (`pip install torch --index-url https://download.pytorch.org/whl/cu124`) — ver doc 02.

---

## Mapear o hardware do notebook (fazer quando estiver nele)
```bash
lspci | grep -i network ; lsusb | grep -i wireless   # chip WiFi
iw list | grep -A8 "Supported interface modes"        # suporta monitor mode?
nvidia-smi                                             # GPU (se tiver)
free -h ; nproc                                        # RAM / cores
```
Cola a saída aqui e a gente completa o doc 02 com o perfil do notebook.

---

## Checklist de retomada
- [ ] Repo `wifi-sense` no GitHub e clonado no Fedora
- [ ] Upstream RuView disponível (submodule ou clone)
- [ ] `cat /proc/net/wireless` lista a interface
- [ ] `make verify` passou
- [ ] `live_sense_monitor.py` classificando presença/movimento (RSSI real)
- [ ] Docker `simulated` subindo REST/WS
- [ ] Próximo marco: exporter OTLP→Grafana **ou** comprar ESP32-S3 (Fase 2)
