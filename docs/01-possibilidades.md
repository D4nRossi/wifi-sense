# 01 — Possibilidades e casos de uso

O RuView expõe ~105 "edge modules" sobre um backbone de sensing. As capacidades-base:

| Capacidade | Requer | Confiabilidade |
|---|---|---|
| Presença (tem alguém no cômodo?) | RSSI (WiFi comum) | Alta |
| Movimento / atividade | RSSI (variância espectral) | Alta |
| Contagem de pessoas | CSI (ESP32) | Média |
| Respiração (6–30 rpm) | CSI (ESP32) | Média/Alta em repouso |
| Batimento (40–120 bpm) | CSI (ESP32) | Média (ambiente controlado) |
| Detecção de queda (<200 ms) | CSI (ESP32) | Média/Alta |
| Pose (17 keypoints, esqueleto) | CSI multi-nó + modelo | Experimental (~82% torso-PCK) |
| "Through-wall" (atravessa parede) | CSI | Depende muito do ambiente |

## Ideias de projeto pessoal (ordenadas por esforço)

### Nível 1 — Zero/baixo hardware
- **Sensor de presença do home-office:** liga/desliga cena (luz, "on air", música) quando
  detecta que você sentou na cadeira — sem câmera, respeitando privacidade.
- **Detector de movimento noturno:** alerta se há atividade num cômodo depois de X horas.
- **"Foco/ausência" para produtividade:** loga tempo presente na mesa vs. ausente.

### Nível 2 — ESP32 (CSI real)
- **Monitor de sono/respiração:** taxa respiratória durante a noite → gráfico no Grafana.
  (Ótimo casamento com tua stack de observabilidade — OTel/Grafana.)
- **Alerta de queda para idoso/pessoa sozinha:** notificação (Telegram/Home Assistant)
  quando detecta impacto + imobilidade. Alto valor social, sem invadir privacidade.
- **Ocupação de salas:** quantas pessoas em cada ambiente → automação de HVAC/luz.
- **Segurança perimetral sem câmera:** movimento em área que deveria estar vazia.

### Nível 3 — Integração/plataforma (onde teu perfil brilha)
- **Bridge para Home Assistant / Matter / MQTT:** o RuView já fala esses protocolos.
- **Exporter OTLP → Grafana/LGTM:** transformar respiração/presença/quedas em métricas e
  dashboards (exatamente o padrão que você já domina no trabalho).
- **API própria + widget web:** o RuView expõe REST (`:3000`) e WebSocket (`:3001`).
- **Fine-tune do modelo no seu ambiente:** contrastive learning p/ adaptar ao seu cômodo
  (self-learning ~30s de calibração). Aqui a RTX 3050 é útil.

## Onde NÃO se iludir
- Batimento/pose exigem ambiente controlado e calibração; não é grau médico.
- "Ver através da parede" funciona em condições específicas, não é raio-X.
- Multipath/mobília/outras pessoas degradam o sinal — é ciência de sinal, não mágica.

## Ganchos com o que você já faz
- **Observabilidade:** métricas de sensing → OTel → Grafana/LGTM (você já tem pipeline).
- **Notificação:** quedas/presença → Telegram/n8n (você já opera n8n na Lapa).
- **Edge/IoT:** ESP32 como fonte, .NET/Rust como coletor.
