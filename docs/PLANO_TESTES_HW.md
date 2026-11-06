# Plano de testes do hardware — V-GUARD

Como provar que o hardware funciona, ensaio por ensaio: o que montar, o que fazer e o
que precisa passar. Cobre o **sistema** — Raspberry Pi, câmera, alimentação, 4G, GNSS,
IMU e a integração. Os ensaios do ESP32 e do CAN na bancada já estão no
[`TESTES.md` do firmware](../firmware_telemetria/docs/TESTES.md) e aqui só são citados.

Cada ensaio executado vira um registro em [`ensaios/`](ensaios/), a partir do
[modelo](ensaios/MODELO.md).

**Os números dos critérios são valores iniciais**, tirados dos requisitos, das folhas
de dados e da configuração do software. Se um ensaio mostrar que um valor não faz
sentido, ele muda aqui, com o motivo, antes do ensaio seguinte.

## Resumo

| ID | Ensaio | Precisa de | Tarefa | Requisitos | Quando |
|---|---|---|---|---|---|
| H01 | UART5 da Pi em laço | Pi e um jumper | T24.2 | RNF07 | **agora** |
| H02 | Desempenho da visão na Pi | Pi e webcam | T25.3 | RNF16 | **agora** |
| H03 | Corte de energia com gravação | Pi e webcam | T31.2 | RNF06, RNF25 | **agora** |
| H04 | Câmera OV5647 e LEDs IR | câmera | T25.2 | RF06, RF07 | peça |
| H05 | Alimentação em bancada | fonte de 12 V, conversores, 4G | T18.3 | RNF09 | peça |
| H06 | Consumo e temperatura | H05 montado | T20 | RNF09, RNF16 | peça |
| H07 | Módulo 4G | A7670E, chip de dados | T36.2 | RF19–RF21, RF30, RNF15 | peça |
| H08 | GNSS e PPS | receptor GNSS | T16.3 | RF02 | peça |
| H09 | IMU | IMU | T15.3 | RF01 | peça |
| H10 | CAN e OBD-II na bancada | ESP32, TJA1050 | T17.2 | RNF08, RNF10 | [`TESTES.md` §10](../firmware_telemetria/docs/TESTES.md) |
| H11 | Pi e ESP32 pelos fios | ESP32 | T40 | RNF07 | [`TESTES.md` §12](../firmware_telemetria/docs/TESTES.md) |
| H12 | No carro, parado | tudo montado | T43.3 | RNF09, RNF10, RNF19 | por último |

**Ensaios já feitos, a registrar** com o modelo, mesmo que antes deste plano:
- Pi 4B com webcam rodando visão, eventos e banco (Pedro, 23/10) — conta como parte do H02.
- OBD-II do Voyage 2015 lido pelo sketch com MCP2515 (Nicolas) — conta como prova de conceito do H10.

## Equipamento

| Item | Para | Onde |
|---|---|---|
| Multímetro | tensões e corrente média | da equipe |
| Fonte de bancada ajustável (0–15 V, limite de corrente) | H05, H06 | laboratório |
| Osciloscópio | quedas de tensão do 4G, PPS | laboratório, numa ida só junto com a do [`TESTES.md` §10.4](../firmware_telemetria/docs/TESTES.md) |
| Jumper fêmea-fêmea | H01 | da equipe |
| Celular com mapa | referência de posição no H08 | da equipe |

## Decisões que travam ensaios

- **GNSS:** o A7670E da lista de componentes vem com antena "LTE/GNSS". Se o GNSS for o
  dele, a posição sai pela USB do módulo (comandos AT ou porta NMEA), não pela UART3, e o
  PPS pode não existir. Decidir antes do H08 e do esquemático.
- **OBD-II:** só o TJA1050 (ADR-0010, firmware oficial) ou o MCP2515 (sketch do Nicolas).
  Decide o H10 e o esquemático.
- **IMU:** o modelo não está na lista de componentes. Decide o H09.

---

## H01 · UART5 da Pi em laço

**Objetivo:** provar a UART que o ESP32 vai usar (`dtoverlay=uart5`, 460800 baud) antes
de ligar o ESP32.

**Montagem:** jumper do pino 32 (GPIO12, TX) ao pino 33 (GPIO13, RX) da Pi. Console
serial desligado e `dtoverlay=uart5` no `config.txt` ([`BANCADA.md` §6](../firmware_telemetria/docs/BANCADA.md)).

**Procedimento:**
1. Conferir que `/dev/ttyAMA5` existe.
2. Enviar 1 MB de bytes aleatórios a 460800 baud e comparar com o que voltou.
   O `host/link_test.py --laco` previsto no firmware ainda não existe; até lá, um script
   com o `open_port` do `integracao/events/uart_reader.py` serve.
3. Repetir com o jumper solto: o script precisa acusar falta de dados, não travar.

| Critério | Esperado |
|---|---|
| Bytes recebidos iguais aos enviados | 100%, 0 erros em 1 MB |
| Jumper solto | erro claro em até 2 s |

## H02 · Desempenho da visão na Pi

**Objetivo:** mostrar que a Pi 4 roda a visão rápido o bastante, sem esquentar nem
vazar memória (gate de desempenho do T25.3).

**Por que o FPS mínimo é 10:** o detector tolera interrupções de até 0,15 s
(`GRACE_PERIOD_S`). Com menos de ~7 quadros/s, o intervalo entre quadros passa disso, e
uma piscada vira "olhos abertos" no meio da contagem. 10 quadros/s dá folga.

**Montagem:** Pi 4B com a fonte oficial, webcam USB (a OV5647 entra no H04), sem
gabinete, sala a ~25 °C. Pi sem outros programas pesados abertos.

**Procedimento:**
1. Num terminal, registrar a cada 10 s temperatura, limitação e memória:
   ```bash
   while true; do echo "$(date +%T) $(vcgencmd measure_temp) $(vcgencmd get_throttled) mem=$(free -m | awk '/Mem/{print $3}')MB"; sleep 10; done | tee h02-saude.log
   ```
2. No outro, rodar a visão por cerca de 30 min, contando o tempo:
   ```bash
   time python -B vguard.py --headless --source 0 --no-uart --no-gps --no-sync --max-frames 18000
   ```
3. FPS médio = 18000 ÷ o tempo `real`. Se terminar antes de 30 min, aumentar `--max-frames`.
4. Durante o ensaio, provocar alguns eventos (fechar os olhos 2 s, virar a cabeça 3 s)
   e conferir que eles aparecem no banco com foto.

| Critério | Esperado |
|---|---|
| FPS médio | ≥ 10 |
| Temperatura máxima | < 80 °C (a Pi 4 começa a reduzir a velocidade aos 80 °C) |
| `get_throttled` | `0x0` o tempo todo (sem subtensão nem limitação) |
| Memória usada | não cresce mais que 10% entre o minuto 5 e o 30 |
| Eventos provocados | todos gravados, com foto |

## H03 · Corte de energia com gravação

**Objetivo:** provar na Pi de verdade o que o `test_event_manager.py` já prova
simulando: um corte de energia no meio da gravação não corrompe o banco nem perde o que
já foi gravado (RNF06, RNF25).

**Montagem:** a do H02, com `VGUARD_DATA` numa pasta nova só para o ensaio.

**Procedimento:**
1. Rodar o `vguard.py` e provocar eventos.
2. Tirar o cabo de energia da Pi com o programa rodando. Esperar 10 s, religar.
3. Repetir 10 vezes, variando o momento (logo depois de um evento, durante a janela de
   10 s, parado).
4. No fim, conferir o banco e as fotos:
   ```bash
   sqlite3 dados/vguard.db "PRAGMA integrity_check;"
   ```
   e que todo evento com `photo` tem o arquivo em `photos/`, e não sobrou `.tmp`.
5. Rodar o `vguard.py` com o `Sync` ligado a um backend e conferir que o pendente sobe.

| Critério | Esperado |
|---|---|
| `integrity_check` | `ok` nos 10 cortes |
| Eventos gravados antes do corte | todos presentes depois de religar |
| Foto de cada evento | arquivo presente, sem `.tmp` sobrando |
| Retomada | o `vguard.py` sobe e grava de novo sem intervenção |
| Pendentes | sobem para o backend na sincronização seguinte |

## H04 · Câmera OV5647 e LEDs IR

**Objetivo:** a câmera definitiva funciona na Pi, de dia e no escuro, com o
desempenho do H02.

**Montagem:** OV5647 pela fita CSI, LEDs IR ligados, a ~60 cm do rosto, como no carro.

**Procedimento:**
1. Repetir o H02 com a OV5647.
2. Em três condições de luz — sala clara, penumbra e escuro total só com o IR — gravar
   2 min em cada uma e contar os quadros com rosto detectado.
3. Provocar 5 eventos de fadiga e 5 de distração em cada condição.

| Critério | Esperado |
|---|---|
| FPS e temperatura | os mesmos do H02 |
| Quadros com rosto detectado | ≥ 90% em cada condição |
| Eventos provocados detectados | ≥ 4 de 5 em cada condição |
| Foto no escuro | rosto reconhecível a olho |

## H05 · Alimentação em bancada

**Objetivo:** o gate elétrico do T18.3. A partir dos 12 V, a alimentação segura a Pi,
a câmera e o 4G juntos, inclusive nos picos de corrente do 4G (RNF09).

**Montagem:** a fonte de bancada no lugar do carro, com limite de corrente de 5 A →
fusível → carregador USB (ramo da Pi) e LM2596 → capacitor (ramo do 4G), como no
diagrama 06.

**Procedimento:**
1. Sem carga: ajustar o LM2596 para 5,0 V e medir as duas saídas.
2. Com a Pi em carga máxima (visão rodando, H02) e o 4G transmitindo (envio de 1 MB
   em laço), medir as duas saídas com o multímetro e as quedas do ramo do 4G com o
   osciloscópio.
3. Variar a entrada de 11 V a 14,8 V (carro desligado a carro carregando bateria) e
   repetir o passo 2 nos extremos.
4. Baixar a entrada até a Pi reiniciar e anotar a tensão (simula a partida do motor).
5. **Polaridade invertida:** só se o esquemático tiver proteção (diodo ou MOSFET), e
   com a fonte limitada a 0,1 A e **sem a Pi e o 4G ligados**.

| Critério | Esperado |
|---|---|
| Saída da Pi em carga, de 11 a 14,8 V | ≥ 4,85 V |
| `get_throttled` durante todo o ensaio | sem subtensão (bit 0 e bit 16 em zero) |
| Picos do 4G ao transmitir | a Pi não reinicia, e o 4G não perde o registro na rede |
| Tensão em que a Pi reinicia | anotada (critério para o ensaio no carro) |
| Polaridade invertida | nada liga e nada esquenta |

## H06 · Consumo e temperatura

**Objetivo:** o T20. Quanto o sistema puxa do carro e se aguenta 1 h fechado no
gabinete.

**Procedimento:**
1. Com a montagem do H05 a 12,6 V, medir a corrente de entrada em quatro situações:
   ligando; parado; visão rodando; visão e 4G transmitindo.
2. Com tudo rodando, deixar 1 h fora do gabinete e 1 h dentro dele (quando existir),
   registrando a temperatura como no H02.

| Critério | Esperado |
|---|---|
| Corrente em cada situação | anotada, e abaixo do fusível escolhido com folga de 50% |
| Temperatura da Pi no gabinete | < 80 °C depois de 1 h |
| Temperatura dos conversores | dá para encostar o dedo (< ~60 °C) |

## H07 · Módulo 4G

**Objetivo:** o T36.2. A Pi envia pelo 4G com o módulo parado na bancada, e as regras
de envio funcionam (RF19–RF21, RF30).

**Montagem:** A7670E na USB da Pi, com chip de dados e antena, alimentado pelo seu
ramo (H05) ou por uma fonte de 5 V separada.

**Procedimento:**
1. Conferir que o módulo aparece como placa de rede (`usb0` ou `wwan0`) ou como modem
   no NetworkManager.
2. Com o Wi-Fi desligado, rodar o `vguard.py` com o `Sync` e provocar eventos: conferir
   `enlace: cellular` e os dados e as fotos chegando ao backend.
3. Ligar um Wi-Fi autorizado (`AUTHORIZED_WIFI`): o envio passa para o Wi-Fi sem
   perder nem repetir nada.
4. Deixar 1 h no 4G e anotar o `cellular_bytes` do dashboard.

| Critério | Esperado |
|---|---|
| Registro na rede | em até 60 s depois de ligar |
| Envio pelo 4G | telemetria, eventos e fotos chegam; foto que falha não atrasa os dados |
| Troca para o Wi-Fi | o envio continua sem intervenção, sem itens repetidos no backend |
| Dados em 1 h | anotados; projeção do mês dentro da franquia do chip |

## H08 · GNSS e PPS

**Objetivo:** o T16.3. Posição certa e um pulso por segundo para o relógio da Pi.
Depende da decisão do GNSS (acima).

**Montagem:** se for receptor separado: UART3 da Pi (GPIO4 e GPIO5) e PPS no GPIO18,
como no [`BANCADA.md`](../firmware_telemetria/docs/BANCADA.md). Antena com vista do céu.

**Procedimento:**
1. Partida a frio, ao ar livre: medir o tempo até o primeiro fix.
2. Comparar 10 posições com o mapa do celular no mesmo lugar.
3. Rodar o `vguard.py` e conferir a posição no evento e o mapa do dashboard.
4. PPS: `ppstest /dev/pps0` ou o osciloscópio no GPIO18.

| Critério | Esperado |
|---|---|
| Primeiro fix, a frio | ≤ 60 s |
| Erro de posição | ≤ 10 m |
| Eventos | levam `lat`, `lon` e `speed`; sem fix, levam só o horário |
| PPS | 1 pulso por segundo, sem falhas em 5 min |

## H09 · IMU

**Objetivo:** o T15.3. Leituras coerentes parada e em movimento.

**Procedimento:** com a IMU no ESP32 (I²C nos GPIO21 e GPIO22), registrar 1 min parada
em cada uma das 3 orientações (cada eixo para cima), depois girar e sacudir.

| Critério | Esperado |
|---|---|
| Aceleração parada | 9,81 ± 0,3 m/s² no eixo para cima, ~0 nos outros |
| Giro parado | 0 ± 1 °/s |
| Taxa de leitura | 100 por segundo, sem falhas |
| Sensor desligado e religado | a IMU volta sozinha, e o OBD continua (diagrama 02) |

## H10 a H12

- **H10, CAN e OBD-II na bancada:** a bancada de três placas e a sessão com transceptor
  do [`TESTES.md` §10](../firmware_telemetria/docs/TESTES.md), inclusive o ensaio do
  GPIO27 (transceptor mudo até o firmware liberar).
- **H11, Pi e ESP32 pelos fios:** o [`TESTES.md` §12](../firmware_telemetria/docs/TESTES.md),
  com o ensaio de 2 h (T24.2).
- **H12, no carro parado:** depois de H05 a H11. Motor ligado e desligado, partida com
  o sistema ligado, lista de PIDs suportados (T17.3) e nenhum quadro fora da lista
  ([`TESTES.md` §13](../firmware_telemetria/docs/TESTES.md)). Sem indução de fadiga no
  motorista (T43.3).
