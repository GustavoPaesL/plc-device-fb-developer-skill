# Padrões de Lógica Ladder IEC 61131-3 para Function Blocks

Este guia de referência apresenta os modelos esquemáticos em ASCII Ladder para cada uma das 6 Networks obrigatórias do `PLC_Device_FB_Developer`, além das equivalências de blocos nativos entre os principais fabricantes de CLP.

---

## 1. Equivalência de Instruções por Plataforma / IDE

| Função | IEC 61131-3 / Codesys / WEG | Siemens TIA Portal (S7-1200/1500) | Rockwell Studio 5000 (AOI) |
| :--- | :--- | :--- | :--- |
| **Borda de Subida** | `R_TRIG` | `P_TRIG` / `--\|P\|--` | `ONS` (One Shot) |
| **Timer On-Delay** | `TON` | `TON` (IEC_Timer) | `TON` (Timer On Delay) |
| **Limitação de Faixa** | `LIMIT(MN, IN, MX)` | `LIMIT` | `LIM` + `MOV` ou `CPT` |
| **Escalonamento** | `LIN_TRAFO` ou `MUL`/`DIV` | `NORM_X` + `SCALE_X` | `CPT` (Compute) ou `SCL` |
| **Acesso a Bits de Word** | `wWord.0`, `wWord.1` | `wWord.%X0`, `wWord.%X1` (Slice) | `wWord.0`, `wWord.1` |
| **I/O Cíclico Consistente** | Mapeamento Direto / `MEMCPY` | `DPRD_DAT` / `DPWR_DAT` | `CPS` (Copy Synchronous) |
| **Modbus RTU / TCP** | `ModbusRequest` / `MB_Master` | `MB_MASTER` / `MB_CLIENT` | `MSG` (Message Instruction) |

---

## 2. Modelos de Networks em ASCII Ladder

### Network 1 — Inicialização e Validação (`xEnable`)
Quando `xEnable` está desativado (`FALSE`), o bloco zera todas as saídas de controle, limpa estados de execução e reinicia a máquina de estados para garantir partida segura.

```text
Network 1: Inicialização e Estado Seguro quando xEnable = FALSE
|
|   xEnable                                            xInternalRunCmd
|----]/[----+------------------------------------------------(R)----|
|           |
|           |                     +----------+
|           +---------------------|   MOVE   |----------------------|
|           |              16#0000|IN     OUT|-wRawControlWord
|           |                     +----------+
|           |
|           |                     +----------+
|           +---------------------|   MOVE   |----------------------|
|           |                    0|IN     OUT|-iState
|           |                     +----------+
|           |
|           |                                             xRunning
|           +------------------------------------------------(R)----|
|           |
|           |                                              xReady
|           +------------------------------------------------(R)----|
```

---

### Network 2 — Desempacotamento da Status Word e Feedbacks (*Unpacking & Descaling*)
Extrai os bits da `wRawStatusWord` recebida do equipamento e converte os registradores brutos para unidades de engenharia (`REAL`).

```text
Network 2: Extração de Bits da Status Word e Escalonamento de Leituras
|
|   xEnable   wRawStatusWord.%X0                            xReady
|----] [------------] [--------------------------------------( )----|
|
|   xEnable   wRawStatusWord.%X1                           xRunning
|----] [------------] [--------------------------------------( )----|
|
|   xEnable   wRawStatusWord.%X3                            xFault
|----] [------------] [--------------------------------------( )----|
|
|   xEnable         +----------------+      +---------------+
|----] [------------|  INT_TO_REAL   |------|      DIV      |-------|
|                   |                |      |               |
|    iRawActualSpeed|IN           OUT|-rTemp|IN1         OUT|-rActualSpeed_Hz
|                   +----------------+ 100.0|IN2            |
|                                           +---------------+
```

---

### Network 3 — Intertravamentos e Lógica de Comando (*Interlocks*)
Garante prioridade de parada (`xCmdStop`), bloqueia partida em caso de falha (`xFault` ou `xCommError`) e gera pulso de borda de subida para `xCmdResetFault`.

```text
Network 3: Intertravamento Operacional e Borda de Subida de Reset
|
|   xEnable   xCmdStart   xCmdStop   xFault   xCommError  xInternalRunCmd
|----] [----+----] [----+----]/[-------]/[-------]/[------------( )----|
|           |           |
|           | xInternal |
|           |  RunCmd   |
|           +----] [----+
|
|   xEnable   xCmdResetFault    +--------------+   wRawControlWord.%X7
|----] [------------] [---------|    R_TRIG    |-----------( )---------|
|                               |  rtResetFault|
|                               +--------------+
```

---

### Network 4 — Empacotamento da Control Word e Setpoints (*Packing & Scaling*)
Limita o setpoint dentro da faixa segura (`LIMIT`), converte de unidade de engenharia para inteiro bruto do registrador e empacota os bits da `wRawControlWord`.

```text
Network 4: Limitação de Setpoint, Conversão de Escala e Montagem da Control Word
|
|   xEnable         +---------------+      +---------------+      +----------------+
|----] [------------|     LIMIT     |------|      MUL      |------|  REAL_TO_INT   |---|
|                   |               |      |               |      |                |
|                0.0|MN             | rTemp|IN1            | rTemp|IN           OUT|-iRawSetpointSpeed
|  rSetpointSpeed_Hz|IN          OUT|-Limit|            OUT|-Scale|                |
|               60.0|MX             | 100.0|IN2            |      +----------------+
|                   +---------------+      +---------------+
|
|   xEnable   xInternalRunCmd                              wRawControlWord.%X0
|----] [------------] [--------------------------------------------( )----|
|
|   xEnable   xCmdReverse                                  wRawControlWord.%X1
|----] [------------] [--------------------------------------------( )----|
```

---

### Network 5 — Máquina de Estados de Comunicação / Atualização de I/O

#### Opção A: I/O Cíclico (Profinet / EtherNet/IP)
```text
Network 5A: Transferência Cíclica de Dados (Periferia / Telegrama PZD)
|
|   xEnable         +------------------+           +------------------+
|----] [------------|     DPRD_DAT     |-----------|     DPWR_DAT     |---|
|                   |                  |           |                  |
| udiHwId_Or_NodeAdd|LADDR       RET_VA|-iRetVal   |LADDR       RET_VA|-iRetVal
|                   |            RECORD|-awRxBuffer|                  |
|                   +------------------+ awTxBuffer|RECORD            |
|                                                  +------------------+
```

#### Opção B: Comunicação Acíclica / Serial (Modbus RTU / TCP)
```text
Network 5B: Sequenciador Passo a Passo (iState = 0 -> 10 -> 20 -> 10)
|
|   xEnable     +---------+                        +----------+
|----] [--------|  EQ_INT |------------------------|   MOVE   |-------|
|         iState|IN1      |                      10|IN     OUT|-iState
|              0|IN2      |                        +----------+
|               +---------+
|
|   xEnable     +---------+     +---------------------+
|----] [--------|  EQ_INT |-----|   MB_MASTER (READ)  |---------------|
|         iState|IN1      |     |                     |
|             10|IN2      |    0|MODE             DONE|---[MOVE 20 -> iState]
|               +---------+     |               ERROR |---[MOVE 99 -> iState]
|                               +---------------------+
|
|   xEnable     +---------+     +---------------------+
|----] [--------|  EQ_INT |-----|  MB_MASTER (WRITE)  |---------------|
|         iState|IN1      |     |                     |
|             20|IN2      |    1|MODE             DONE|---[MOVE 10 -> iState]
|               +---------+     |               ERROR |---[MOVE 99 -> iState]
|                               +---------------------+
```

---

### Network 6 — Watchdog, Timeout e Tratamento de Falhas (*Safe State*)
Monitora o tempo máximo de resposta. Caso `tonCommWatchdog.Q` dispare, aciona `xCommError`, zera a `wRawControlWord` (Estado Seguro) e registra o código de falha.

```text
Network 6: Watchdog de Comunicação e Forçamento de Safe State
|
|   xEnable   xCommDonePulse     +-------------------+      xCommError
|----] [------------]/[----------|        TON        |-----+----( )----|
|                                |  tonCommWatchdog  |     |
|                    tTimeoutComm|PT               ET|-    |
|                                +-------------------+     |
|                                                          |
|                                      +----------+        |
|                                      |   MOVE   |--------+
|                               16#0000|IN     OUT|-wRawControlWord
|                                      +----------+        |
|                                                          |
|                                      +----------+        |
|                                      |   MOVE   |--------+
|                               16#FF01|IN     OUT|-wFaultCode
|                                      +----------+
```

