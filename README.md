# wifi-sense — Sensoriamento Espacial via WiFi (RuView)

Projeto pessoal baseado no [RuView](https://github.com/ruvnet/RuView) (MIT):
transformar sinal WiFi em **inteligência espacial** — presença, movimento, contagem
de pessoas, sinais vitais (respiração/batimento), detecção de queda e pose — **sem câmera**.

> A técnica se chama **WiFi Sensing / CSI (Channel State Information)**: o corpo humano
> reflete e perturba as ondas de rádio; ao ler essas perturbações dá pra inferir o que
> acontece no ambiente. É a mesma família de ideias do "DensePose from WiFi" (CMU, 2022).

## Estrutura do workspace

```
wifi-sense/
├─ README.md                  <- este arquivo
├─ RuView/                    <- upstream como submodule (git submodule update --init)
└─ docs/
   ├─ 01-possibilidades.md         <- o que dá pra fazer com isso (casos de uso)
   ├─ 02-hardware-execucao.md      <- o que roda NA MINHA máquina + roadmap de hardware
   ├─ 03-setup-notebook-e-github.md <- retomar no notebook (Fedora) + criar o repo
   └─ 04-mapa-capacidades.md       <- MAPA completo: 65 módulos/12 categorias + alvos
```

> **Nome sugerido do repo GitHub:** `wifi-sense` (ver `docs/03`). Continuação será no
> notebook **Fedora** (WiFi ativo → modo RSSI real funciona lá, nativo via `LinuxWifiCollector`).

> **Vai continuar no notebook?** Comece por `docs/03-setup-notebook-e-github.md`.
> O notebook tem WiFi ativo → o modo RSSI (presença/movimento real, sem hardware)
> funciona lá, ao contrário deste desktop cabeado.

## Meu hardware

| Recurso | Valor | Implicação |
|---|---|---|
| CPU | Ryzen 5 5600X (6c/12t) | Sobra pra pipeline de sinal e inferência |
| RAM | 32 GB | Muito confortável |
| GPU | RTX 3050 (8 GB) | Serve p/ treino/fine-tune leve e inferência (Candle/PyTorch) |
| Rede | **Ethernet cabeada** (Realtek GbE) | ⚠️ **Sem rádio WiFi ativo** (`wlansvc` desligado) |

### O ponto crítico
O modo "custo zero" do RuView (ler RSSI do próprio WiFi via `netsh`) **não roda aqui como
está**, porque este é um desktop no cabo, sem adaptador WiFi ativo. Isso define por onde
começar — ver `docs/02-hardware-execucao.md`.

## Caminho recomendado (3 fases)

1. **Fase 0 — Explorar sem hardware (hoje):** Docker simulation + `make verify`.
   Roda a stack completa (API/WebSocket/dashboard) com dados simulados.
2. **Fase 1 — Sensing real barato (~R$50–120):** um dongle USB WiFi **ou** ativar o
   WiFi da placa-mãe → habilita presença/movimento reais via RSSI.
3. **Fase 2 — O "de verdade" (CSI):** 1–3 placas **ESP32-S3** (~US$9 cada) → respiração,
   batimento, pose. Aqui a RTX 3050 entra pra treino/inferência dos modelos.

## Licença
Upstream é MIT. Este workspace é uso pessoal/estudo.
