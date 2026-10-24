# Documentação e Projeto do Function Block — `FB_FLUKE_8808A`
**Plataforma:** Siemens TIA Portal (S7-1200 / S7-1500) | **Linguagem:** IEC 61131-3 Ladder (LD) | **Protocolo:** RS-232 ASCII (Point-to-Point)
**Modos de Operação Suportados:** Tensão DC (`VDC`), Tensão AC (`VAC`), Resistência (`OHMS`) e Continuidade (`CONT`)

---

## Seção 1: Mapeamento Técnico e Fatores de Escala

O multímetro digital **Fluke 8808A** opera via interface serial **RS-232C** assíncrona com comandos ASCII terminados por `<CR><LF>` (`16#0D0A` / `'$R$L'`). O bloco `FB_FLUKE_8808A` encapsula a seleção dinâmica e a troca *on-the-fly* entre os **4 modos solicitados** (**Tensão DC**, **Tensão AC**, **Resistência** e **Continuidade**), ajustando automaticamente os comandos seriais, os limites de escala do Setpoint e a lógica de detecção de continuidade.

### 1.1 Mapeamento dos 4 Modos de Medição (Tabelas 1-9 a 1-13 e Tabela 4-9 do Manual)

| Código (`usiModeSelect`) | Modo de Operação | Comando RS-232 (`TX`) | Faixa de Medição / Limites de Setpoint | Unidade de Engenharia | Particularidades Técnicas do Manual Fluke 8808A |
| :---: | :--- | :--- | :--- | :---: | :--- |
| **`0`** | **Tensão DC (`VDC`)** | `'VDC$R$L'` | `-1000.0` a `+1000.0` | `V` (`VDC`) | 5 faixas (`200 mV` a `1000 Vdc`), sensibilidade de `1 µV`. |
| **`1`** | **Tensão AC (`VAC`)** | `'VAC$R$L'` | `0.0` a `+750.0` | `V` (`VAC`) | True-RMS acoplado em CA (`20 Hz` a `100 kHz`), máx. `750 Vac`. |
| **`2`** | **Resistência (`OHMS`)** | `'OHMS$R$L'` | `0.0` a `+1.0E+8` (`100 MΩ`) | `Ω` (`OHMS`) | 7 faixas (`200 Ω` a `100 MΩ`). Em circuito aberto retorna Overload (`+1.0E+9`). |
| **`3`** | **Continuidade (`CONT`)** | `'CONT$R$L'` | `0.0` a `+199.99` | `Ω` (`CONT`) | Limiar de continuidade de fábrica = **`20.0 Ω`** (corrente de teste `1 mA`, taxa fixa `Fast = 100 amostras/s`). **Atenção (Tabela 4-11):** Não aceita o comando `AUTO` (gera erro `!>`). |

### 1.2 Mapeamento Bit a Bit — Control Word Interna (`#wRawControlWord`) e Status Word (`#wRawStatusWord`)

| Bit | Control Word (`#wRawControlWord` — Comando Interno $\rightarrow$ Serial) | Status Word (`#wRawStatusWord` — Mapeado de `ESR` / `STB` / Prompts RS-232) |
| :--- | :--- | :--- |
| **`%X0`** | **Run / Measure:** Habilita ciclo de leitura contínua (`REMS` + `VAL1?`) | **OPC / Ready (`ESR.0` / Prompt `=>`):** Equipamento em modo remoto e pronto |
| **`%X1`** | **Mode VDC Active:** Seleciona modo `0` — Tensão DC (`'VDC$R$L'`) | **Running / Measuring:** Aquisição cíclica ativa recebendo leituras válidas |
| **`%X2`** | **Mode VAC Active:** Seleciona modo `1` — Tensão AC (`'VAC$R$L'`) | **Continuity Closed (`< 20.0 Ω`):** Circuito fechado detectado no modo Continuidade |
| **`%X3`** | **Mode OHMS Active:** Seleciona modo `2` — Resistência (`'OHMS$R$L'`) | **Fault Summary (`STB.5` - `ESB`):** Falha ativa de comando, execução ou comunicação |
| **`%X4`** | **Mode CONT Active:** Seleciona modo `3` — Continuidade (`'CONT$R$L'`) | **Execution Error (`ESR.4` - `EXE` / Prompt `!>`):** Comando não executável no modo atual |
| **`%X5`** | **Mode Change Req:** Solicita reconfiguração imediata de função na serial | **Command Error (`ESR.5` - `CME` / Prompt `?>`):** Erro de sintaxe no comando ASCII |
| **`%X6`** | **Update Setpoint:** Envia limite de comparação (`COMPHI`) | **Overload / Open Circuit (`0L` = `+/-1.0E+9`):** Acima do fundo de escala / Circuito aberto |
| **`%X7`** | **Fault Reset (`*CLS`):** Limpa registradores de erro `ESR`/`STB` (Borda de subida) | **Above Setpoint:** Medição atual superior ao `#rSetpointLimit` |

### 1.3 Equações de Conversão (Scaling / Descaling)

1. **Descaling da Leitura (`STRG_VAL` — Raw ASCII $\rightarrow$ Engenharia):**
   - No Formato 1 (`FORMAT 1`), o comando `'VAL1?$R$L'` retorna uma string científica ASCII terminada por `<CR><LF>`, ex.: `'+2.4015E+1\r\n=>\r\n'`.
   - A instrução nativa `STRG_VAL` do TIA Portal converte a string diretamente para `#rActualValue` (`REAL`) em `V` (modos `VDC`/`VAC`) ou `Ω` (modos `OHMS`/`CONT`).
2. **Tratamento de Overload / Circuito Aberto (`0L` = `+/-1.0E+9`):**
   - Quando o valor absoluto lido satisfaz $\lvert\text{#rTempParsedVal}\rvert \ge 9.99 \times 10^8$:
     - Nos modos **Tensão DC (`0`)** e **Tensão AC (`1`)**, Sobrecarga (`0L`) é tratada como **Falha de Sobrecarga** (`#xFault := TRUE`, `#wFaultCode := 16#8006`).
     - Nos modos **Resistência (`2`)** e **Continuidade (`3`)**, `0L` representa **Circuito Aberto** (ponteiras abertas): aciona a flag `#xOpenCircuit := TRUE`, mantém `#xContinuityOk := FALSE` e **não trava o bloco em falha**, permitindo testar continuidade/resistores continuamente.
3. **Detecção de Continuidade (`#xContinuityOk`):**
   $$\text{xContinuityOk} = (\text{usiModeSelect} = 3) \land \neg\text{xOpenCircuit} \land (\text{rActualValue} \le 20.0\ \Omega)$$

---

## Seção 2: Estrutura de Dados Auxiliar (`UDT` / `PLC Data Type`)

Crie a `UDT_Fluke8808A` em **PLC data types** no TIA Portal para integração direta com IHM/SCADA:

```iecst
TYPE "UDT_Fluke8808A"
VERSION : 0.2
   STRUCT
      // --- Comandos e Seleção de Modo (HMI / Processo -> FB) ---
      xCmdStart          : Bool := FALSE;   // Inicia ciclo contínuo de medição remota
      xCmdResetFault     : Bool := FALSE;   // Pulso para limpeza de falhas (*CLS)
      usiModeSelect      : USInt := 0;      // 0: Tensão DC (VDC), 1: Tensão AC (VAC), 2: Resistência (OHMS), 3: Continuidade (CONT)
      rSetpointLimit     : Real := 20.0;    // Setpoint / Limite de comparação em V ou Ohms
      
      // --- Diagnósticos e Leituras (FB -> HMI / Processo) ---
      xReady             : Bool := FALSE;   // Multímetro em modo remoto e pronto
      xRunning           : Bool := FALSE;   // Aquisição cíclica RS-232 em execução
      xFault             : Bool := FALSE;   // Falha ativa (Sintaxe, Execução, Overload V ou Comm)
      xCommError         : Bool := FALSE;   // Timeout ou falha no módulo serial CM 1241 / CM PtP
      xOpenCircuit       : Bool := FALSE;   // Indicador de sobrecarga / circuito aberto (0L = 1.0E+9)
      xContinuityOk      : Bool := FALSE;   // TRUE quando em Modo Continuidade (3) e R <= 20.0 Ohms
      xAboveSetpoint     : Bool := FALSE;   // TRUE quando leitura atual > rSetpointLimit
      rActualValue       : Real := 0.0;     // Valor medido em unidade de engenharia (V ou Ohms)
      wFaultCode         : Word := 16#0000; // Código de falha atual
      sActiveModeUnit    : String[8] := 'VDC'; // Texto do modo ativo: 'VDC', 'VAC', 'OHMS', 'CONT'
   END_STRUCT;
END_TYPE
```

---

## Seção 3: Tabela de Declaração de Variáveis do FB (`FB_FLUKE_8808A`)

### 3.1 `VAR_INPUT`
| Nome da Variável | Tipo de Dado | Valor Inicial | Comentário Técnico |
| :--- | :--- | :--- | :--- |
| `xEnable` | `Bool` | `FALSE` | Habilita o bloco; se `FALSE`, zera controles e coloca saídas em Safe State |
| `hwPortId` | `PORT` (`Word`) | `16#0000` | Identificador de Hardware (System Constant) do módulo RS-232 (CM 1241 / CM PtP) |
| `xCmdStart` | `Bool` | `FALSE` | `TRUE` = Inicia medição cíclica no modo selecionado |
| `xCmdResetFault` | `Bool` | `FALSE` | Limpa falhas ativas e envia `*CLS` ao instrumento (borda de subida) |
| `usiModeSelect` | `USInt` | `0` | Modo de operação: `0`=Tensão DC, `1`=Tensão AC, `2`=Resistência, `3`=Continuidade |
| `rSetpointLimit` | `Real` | `20.0` | Setpoint / Limite de comparação (`V` para modos 0/1; `Ω` para modos 2/3) |
| `tTimeoutComm` | `Time` | `T#2S` | Tempo máximo de espera de resposta serial RS-232 (Watchdog) |

### 3.2 `VAR_OUTPUT`
| Nome da Variável | Tipo de Dado | Valor Inicial | Comentário Técnico |
| :--- | :--- | :--- | :--- |
| `xReady` | `Bool` | `FALSE` | Equipamento inicializado em modo remoto e pronto |
| `xRunning` | `Bool` | `FALSE` | Ciclo de leitura ativo e recebendo dados válidos |
| `xFault` | `Bool` | `FALSE` | Falha ativa (Erro RS-232, comando inválido ou sobrecarga de tensão) |
| `xCommError` | `Bool` | `FALSE` | Erro ou Timeout de comunicação RS-232 |
| `xContinuityOk` | `Bool` | `FALSE` | `TRUE` quando em modo Continuidade (`3`) e resistência medida $\le 20.0\ \Omega$ |
| `xOpenCircuit` | `Bool` | `FALSE` | `TRUE` quando o display indica `0L` (`1.0E+9` — ponteiras abertas ou fora de faixa) |
| `rActualValue` | `Real` | `0.0` | Valor atual da medição principal (`V` em DC/AC ou `Ω` em Resistência/Continuidade) |
| `wFaultCode` | `Word` | `16#0000` | Código numérico de diagnóstico (`16#0000` = OK) |

### 3.3 `VAR_IN_OUT`
| Nome da Variável | Tipo de Dado | Comentário Técnico |
| :--- | :--- | :--- |
| `stHmi` | `"UDT_Fluke8808A"` | Estrutura de dados para integração com IHM/SCADA |

### 3.4 `VAR` (`STATIC` — Memória Interna de Instância)
| Nome da Variável | Tipo de Dado | Valor Inicial | Comentário Técnico |
| :--- | :--- | :--- | :--- |
| `iState` | `Int` | `0` | Passo da máquina de estados (`0`=Idle, `10`=Init, `20`=SendMode, `40`=TxVal, `50`=RxVal, `80`=Reset, `99`=Error) |
| `usiActiveMode` | `USInt` | `255` | Último modo enviado ao Fluke (`0..3`); inicializado em `255` para forçar configuração na partida |
| `wRawControlWord` | `Word` | `16#0000` | Palavra de controle interna empacotada |
| `wRawStatusWord` | `Word` | `16#0000` | Palavra de status interna desempacotada |
| `rClampedSetpoint` | `Real` | `0.0` | Setpoint limitado conforme os limites do modo ativo (`VDC`, `VAC`, `OHMS`, `CONT`) |
| `sModeCmdStr` | `String[20]` | `'VDC$R$L'` | Comando ASCII correspondente ao modo selecionado |
| `sTxBuffer` | `String[50]` | `''` | Buffer ASCII de envio para `Send_P2P` |
| `sRxBuffer` | `String[50]` | `''` | Buffer ASCII de recepção para `Receive_P2P` |
| `uiRxLength` | `UInt` | `0` | Comprimento em bytes do último pacote recebido |
| `xSendReq` | `Bool` | `FALSE` | Disparo de transmissão da instância `instSendP2P` |
| `xSendDone` | `Bool` | `FALSE` | Confirmação `DONE` da instrução `instSendP2P` |
| `xSendError` | `Bool` | `FALSE` | Flag `ERROR` da instrução `instSendP2P` |
| `wSendStatus` | `Word` | `16#0000` | Código `STATUS` da instrução `instSendP2P` |
| `xRecvNdr` | `Bool` | `FALSE` | Novo dado recebido (`NDR`) na instrução `instReceiveP2P` |
| `xRecvError` | `Bool` | `FALSE` | Flag `ERROR` da instrução `instReceiveP2P` |
| `wRecvStatus` | `Word` | `16#0000` | Código `STATUS` da instrução `instReceiveP2P` |
| `xResetEdgeMem` | `Bool` | `FALSE` | Memória auxiliar de borda de subida do Reset (`P_TRIG`) |
| `xResetPulse` | `Bool` | `FALSE` | Pulso de 1 scan de Reset de Falha |
| `xInternalStart` | `Bool` | `FALSE` | Comando interno de operação validado pelos intertravamentos |
| `instSendP2P` | `Send_P2P` | - | Multi-instância nativa do bloco de envio serial Siemens |
| `instReceiveP2P` | `Receive_P2P` | - | Multi-instância nativa do bloco de recepção serial Siemens |
| `tonCommWatchdog` | `TON_TIME` | - | Timer IEC de Watchdog para detecção de timeout serial RS-232 |

### 3.5 `VAR_TEMP`
| Nome da Variável | Tipo de Dado | Comentário Técnico |
| :--- | :--- | :--- |
| `rTempParsedVal` | `Real` | Valor temporário convertido de ASCII por `STRG_VAL` |
| `rTempAbsVal` | `Real` | Valor absoluto da leitura para detecção de `0L` (`1.0E+9`) |
| `rTempMinLimit` | `Real` | Limite mínimo dinâmico do Setpoint conforme modo (`0..3`) |
| `rTempMaxLimit` | `Real` | Limite máximo dinâmico do Setpoint conforme modo (`0..3`) |
| `iPosPromptOk` | `Int` | Posição do prompt `'=>'` na string recebida |
| `iPosPromptCmdErr` | `Int` | Posição do prompt `'?>'` na string recebida |
| `iPosPromptExecErr` | `Int` | Posição do prompt `'!>'` na string recebida |

### 3.6 `CONSTANT`
| Nome da Constante | Tipo de Dado | Valor | Comentário Técnico |
| :--- | :--- | :--- | :--- |
| `MODE_VDC` | `USInt` | `0` | Código do modo Tensão DC (`'VDC$R$L'`) |
| `MODE_VAC` | `USInt` | `1` | Código do modo Tensão AC (`'VAC$R$L'`) |
| `MODE_OHMS` | `USInt` | `2` | Código do modo Resistência 2 Fios (`'OHMS$R$L'`) |
| `MODE_CONT` | `USInt` | `3` | Código do modo Teste de Continuidade (`'CONT$R$L'`) |
| `OVERLOAD_THRESHOLD` | `Real` | `9.99E+8` | Limiar para detecção de `0L` (`+/-1.0E+9`) |
| `CONT_THRESHOLD_OHM` | `Real` | `20.0` | Limiar padrão de continuidade do Fluke 8808A (`20.0 Ω`, pág. 1-13) |
| `MAX_VDC_LIMIT` | `Real` | `1000.0` | Fundo de escala de Tensão DC (`1000 V`) |
| `MAX_VAC_LIMIT` | `Real` | `750.0` | Fundo de escala de Tensão AC (`750 V`) |
| `MAX_OHMS_LIMIT` | `Real` | `1.0E+8` | Fundo de escala de Resistência (`100 MΩ`) |
| `MAX_CONT_LIMIT` | `Real` | `199.99` | Fundo de escala de Continuidade (`199.99 Ω`) |
| `ERR_NONE` | `Word` | `16#0000` | Operação sem falhas |
| `ERR_CMD_SYNTAX` | `Word` | `16#8001` | Prompt `'?>'` recebido: Erro de sintaxe RS-232 |
| `ERR_EXEC_PARAM` | `Word` | `16#8002` | Prompt `'!>'` recebido: Erro de execução RS-232 |
| `ERR_INVALID_MODE` | `Word` | `16#8004` | `usiModeSelect > 3` (Modo fora da faixa `0..3`) |
| `ERR_OVERLOAD_VOLT` | `Word` | `16#8006` | Sobrecarga de tensão detectada nos modos `VDC` ou `VAC` |
| `ERR_COMM_TIMEOUT` | `Word` | `16#FF01` | Timeout de comunicação RS-232 |

---

## Seção 4: Lógica Ladder Interna do FB (Network por Network)

### Network 1: Habilitação do Bloco (`xEnable`) e Inicialização / Safe State
**Explicação Técnica:** Se `#xEnable = FALSE`, reinicia a máquina de estados (`#iState := 0`), força `#usiActiveMode := 255` (garantindo reconfiguração na próxima habilitação), zera `#wRawControlWord` e `#wRawStatusWord`, zera `#rActualValue` e desativa todas as saídas (`#xReady`, `#xRunning`, `#xContinuityOk`, `#xOpenCircuit`, `#xFault`, `#xCommError`).

```text
Network 1: Habilitação do Bloco (xEnable) e Safe State
|
|  #xEnable                                               #xInternalStart
|----]/[----+---------------------------------------------------(R)----|
|           |                                                  #xSendReq
|           +---------------------------------------------------(R)----|
|           |                    +----------+
|           +--------------------|   MOVE   |--------------------------|
|           |             16#0000|IN     OUT|-#wRawControlWord
|           |                    +----------+
|           |                    +----------+
|           +--------------------|   MOVE   |--------------------------|
|           |             16#0000|IN     OUT|-#wRawStatusWord
|           |                    +----------+
|           |                    +----------+
|           +--------------------|   MOVE   |--------------------------|
|           |                   0|IN     OUT|-#iState
|           |                    +----------+
|           |                    +----------+
|           +--------------------|   MOVE   |--------------------------|
|           |                 255|IN     OUT|-#usiActiveMode
|           |                    +----------+
|           |                    +----------+
|           +--------------------|   MOVE   |--------------------------|
|           |                 0.0|IN     OUT|-#rActualValue
|           |                    +----------+
|           |                                                   #xReady
|           +---------------------------------------------------(R)----|
|           |                                                 #xRunning
|           +---------------------------------------------------(R)----|
|           |                                            #xContinuityOk
|           +---------------------------------------------------(R)----|
|           |                                             #xOpenCircuit
|           +---------------------------------------------------(R)----|
|           |                                                   #xFault
|           +---------------------------------------------------(R)----|
|           |                                               #xCommError
|           +---------------------------------------------------(R)----|
```

---

### Network 2: Leitura de Dados / Desempacotamento da *Status Word* (Unpacking)
**Explicação Técnica:** Analisa os prompts enviados pelo Fluke 8808A (`'=>'`, `'?>'`, `'!>'`) em `#sRxBuffer` quando `#xRecvNdr` é acionado e desempacota `#wRawStatusWord` nas saídas de diagnóstico. Se `#wRawStatusWord.%X6` (`0L` / `1.0E+9`) ocorrer nos modos de **Tensão (`0=VDC` ou `1=VAC`)**, aciona `#xFault`; nos modos de **Resistência (`2`)** ou **Continuidade (`3`)**, `0L` indica apenas circuito aberto (`#xOpenCircuit = TRUE`), sem gerar falha bloqueante.

```text
Network 2: Desempacotamento da Status Word e Prompts RS-232 (=>, ?>, !>)
|
|  #xEnable  #xRecvNdr        +---------------+       +--------+  #wRawStatusWord.%X0
|----] [--------] [----+------|     FIND      |-------| GT_INT |----------(S)----|
|                      |      |#sRxBuffer  IN1| #iPos |IN1     |
|                      |      |'=>'        IN2|-Ok    |0    IN2|
|                      |      +---------------+       +--------+
|                      |
|                      |      +---------------+       +--------+  #wRawStatusWord.%X5
|                      +------|     FIND      |-------| GT_INT |----------(S)----|
|                      |      |#sRxBuffer  IN1| #iPos |IN1     |
|                      |      |'?>'        IN2|-CmdEr |0    IN2|
|                      |      +---------------+       +--------+
|                      |
|                      |      +---------------+       +--------+  #wRawStatusWord.%X4
|                      +------|     FIND      |-------| GT_INT |----------(S)----|
|                             |#sRxBuffer  IN1| #iPos |IN1     |
|                             |'!>'        IN2|-ExeEr |0    IN2|
|                             +---------------+       +--------+
|
|  #xEnable  #wRawStatusWord.%X6                                    #xOpenCircuit
|----] [-------------] [--------------------------------------------------( )----|
|
|  #xEnable  #wRawStatusWord.%X4                                         #xFault
|----] [-----+-------] [---------------------------------------+----------( )----|
|            |                                                 |
|            |#wRawStatusWord.%X5                              |
|            +-------] [---------------------------------------+
|            |                                                 |
|            | #xCommError                                     |
|            +-------] [---------------------------------------+
|            |                     +----------+                |
|            |#wRawStatusWord.%X6  | LE_USINT |                |
|            +-------] [-----------|          |----------------+
|                                  |#usiMode  |
|                                  |         1|
|                                  +----------+
|
|  #xEnable  #wRawStatusWord.%X0  #xFault  #xCommError                  #xReady
|----] [-------------] [------------]/[--------]/[------------------------( )----|
|
|  #xEnable  #wRawStatusWord.%X1  #xFault  #xCommError                 #xRunning
|----] [-------------] [------------]/[--------]/[------------------------( )----|
```

---

### Network 3: Conversão de Escala das Leituras (`STRG_VAL`) e Lógica de Continuidade
**Explicação Técnica:** Converte a string ASCII científica recebida em `#sRxBuffer` para `#rActualValue` (`REAL`) usando `STRG_VAL`. Se $\lvert\text{#rTempParsedVal}\rvert \ge 9.99 \times 10^8$, ativa `#wRawStatusWord.%X6` (`#xOpenCircuit`). Quando o modo ativo é **Continuidade (`#usiModeSelect = 3`)**, se `#xOpenCircuit = FALSE` e `#rActualValue <= 20.0 Ω` (`#CONT_THRESHOLD_OHM`), ativa a saída `#xContinuityOk` (`#wRawStatusWord.%X2`).

```text
Network 3: Conversão STRG_VAL (ASCII -> REAL), Detecção de 0L e Continuidade (<= 20 Ohms)
|
|  #xEnable  #xRecvNdr    +--------------------+       +---------------+
|----] [--------] [-------|     STRG_VAL       |-------|      ABS      |---------|
|                         |                    |       |   (Real)      |
|               #sRxBuffer|IN               OUT|-#rTemp|IN          OUT|-#rTempAbsVal
|                  16#0000|FORMAT              | Parsed+---------------+
|                        1|P                   |
|                         +--------------------+
|
|  #xEnable  #xRecvNdr    +--------------------+                  #wRawStatusWord.%X6
|----] [--------] [---+---|      GE_REAL       |--------------------------(S)----|
|                     |   |     #rTempAbsVal   |
|                     |   |#OVERLOAD_THRESHOLD |
|                     |   +--------------------+
|                     |
|                     |   +--------------------+       +---------------+
|                     +---|      LT_REAL       |---+---|     MOVE      |---------|
|                         |     #rTempAbsVal   |   |   |#rTempParsedVal|-#rActualValue
|                         |#OVERLOAD_THRESHOLD |   |   +---------------+
|                         +--------------------+   |              #wRawStatusWord.%X6
|                                                  +----------------------(R)----|
|
|  #xEnable     +----------+     #xOpenCircuit    +--------------------+  #xContinuityOk
|----] [--------| EQ_USINT |----------]/[---------|      LE_REAL       |-------( )----|
|  #usiModeSelec|IN1       |                      |       #rActualValue|
|     #MODE_CONT|IN2       |                      | #CONT_THRESHOLD_OHM|
|               +----------+                      +--------------------+
```

---

### Network 4: Lógica de Comandos, Intertravamentos, Seleção dos 4 Modos e Borda de Reset
**Explicação Técnica:**
1. **Validação dos 4 Modos (`0..3`):** Decodifica `#usiModeSelect` para carregar o comando serial exato em `#sModeCmdStr`:
   - `0` (`#MODE_VDC`) $\rightarrow$ `'VDC; AUTO$R$L'`
   - `1` (`#MODE_VAC`) $\rightarrow$ `'VAC; AUTO$R$L'`
   - `2` (`#MODE_OHMS`) $\rightarrow$ `'OHMS; WIRE2; AUTO$R$L'`
   - `3` (`#MODE_CONT`) $\rightarrow$ `'CONT$R$L'` *(sem `AUTO`, conforme restrição da Tabela 4-11 do manual)*.
2. **Detecção de Troca de Modo (*On-the-Fly*):** Se `#usiModeSelect <> #usiActiveMode`, aciona `#wRawControlWord.%X5` para enviar imediatamente o novo modo ao multímetro.
3. **Borda de Subida de Reset (`P_TRIG`):** Pulsa `#xResetPulse` por 1 scan para limpar falhas e comandar `'*CLS$R$L'`.

```text
Network 4: Decodificação dos 4 Modos (VDC, VAC, OHMS, CONT), Reset e Intertravamento
|
|  #xEnable     +----------+     +----------------------------+     +---------------+
|----] [----+---| EQ_USINT |-----|           S_MOVE           |-----|     S_MOVE    |---|
|           |   |#usiMode  |     |'VDC; AUTO$R$L'          OUT|-#sMd|'VDC'       OUT|-#stHmi.sUnit
|           |   |         0|     +----------------------------+     +---------------+
|           |   +----------+
|           |   +----------+     +----------------------------+     +---------------+
|           +---| EQ_USINT |-----|           S_MOVE           |-----|     S_MOVE    |---|
|           |   |#usiMode  |     |'VAC; AUTO$R$L'          OUT|-#sMd|'VAC'       OUT|-#stHmi.sUnit
|           |   |         1|     +----------------------------+     +---------------+
|           |   +----------+
|           |   +----------+     +----------------------------+     +---------------+
|           +---| EQ_USINT |-----|           S_MOVE           |-----|     S_MOVE    |---|
|           |   |#usiMode  |     |'OHMS; WIRE2; AUTO$R$L'  OUT|-#sMd|'OHMS'      OUT|-#stHmi.sUnit
|           |   |         2|     +----------------------------+     +---------------+
|           |   +----------+
|           |   +----------+     +----------------------------+     +---------------+
|           +---| EQ_USINT |-----|           S_MOVE           |-----|     S_MOVE    |---|
|               |#usiMode  |     |'CONT$R$L'               OUT|-#sMd|'CONT'      OUT|-#stHmi.sUnit
|               |         3|     +----------------------------+     +---------------+
|               +----------+
|
|  #xEnable  #xCmdResetFault       +------------+                    #xResetPulse
|----] [-----------] [-------------|   P_TRIG   |-------------------------( )----|
|                                  |#xResetEdge |
|                                  +------------+
|
|  #xEnable  #xCmdStart    #xFault  #xCommError                   #xInternalStart
|----] [---------] [---------]/[--------]/[-------------------------------( )----|
```

---

### Network 5: Empacotamento da *Control Word* (Packing) e Limitação de Setpoint por Modo
**Explicação Técnica:**
1. Empacota os bits de comando e o modo ativo em `#wRawControlWord` (`%X0` = Run, `%X1..%X4` = Modo ativo `VDC`/`VAC`/`OHMS`/`CONT`, `%X5` = Troca de Modo pendente, `%X7` = Reset).
2. Seleciona os limites de escala (`#rTempMinLimit` e `#rTempMaxLimit`) de acordo com o modo (`-1000..+1000 V` para `VDC`; `0..750 V` para `VAC`; `0..100 MΩ` para `OHMS`; `0..199.99 Ω` para `CONT`) e aplica a instrução `LIMIT` sobre `#rSetpointLimit`.

```text
Network 5: Empacotamento da Control Word e Limitação Dinâmica do Setpoint por Modo
|
|  #xEnable  #xInternalStart                                  #wRawControlWord.%X0
|----] [-----------] [----------------------------------------------------( )----|
|
|  #xEnable     +----------------+                            #wRawControlWord.%X5
|----] [--------|    NE_USINT    |----------------------------------------( )----|
|  #usiModeSelec|IN1             |
| #usiActiveMode|IN2             |
|               +----------------+
|
|  #xEnable     +----------+     +-------------------+     +-------------------+
|----] [----+---| EQ_USINT |-----|       MOVE        |-----|       MOVE        |---|
|           |   |#usiMode 0|     | -1000.0        OUT|-#Min| 1000.0         OUT|-#Max
|           |   +----------+     +-------------------+     +-------------------+
|           |   +----------+     +-------------------+     +-------------------+
|           +---| EQ_USINT |-----|       MOVE        |-----|       MOVE        |---|
|           |   |#usiMode 1|     | 0.0            OUT|-#Min| 750.0          OUT|-#Max
|           |   +----------+     +-------------------+     +-------------------+
|           |   +----------+     +-------------------+     +-------------------+
|           +---| EQ_USINT |-----|       MOVE        |-----|       MOVE        |---|
|           |   |#usiMode 2|     | 0.0            OUT|-#Min| 1.0E+8         OUT|-#Max
|           |   +----------+     +-------------------+     +-------------------+
|           |   +----------+     +-------------------+     +-------------------+
|           +---| EQ_USINT |-----|       MOVE        |-----|       MOVE        |---|
|               |#usiMode 3|     | 0.0            OUT|-#Min| 199.99         OUT|-#Max
|               +----------+     +-------------------+     +-------------------+
|
|  #xEnable        +--------------------+                     #stHmi.xAboveSetpoint
|----] [-----------|       LIMIT        |---[ GT_REAL ]---------------( )--------|
|    #rTempMinLimit|MN                  |   (#rActualValue > #rClampedSetpoint)
|   #rSetpointLimit|IN               OUT|-#rClampedSetpoint
|    #rTempMaxLimit|MX                  |
|                  +--------------------+
```

---

### Network 6: Máquina de Estados Acíclica RS-232 (`Send_P2P` / `Receive_P2P`) e Watchdog (`TON`)
**Explicação Técnica:** Gerencia o envio sequencial de comandos ASCII e a troca dinâmica entre `VDC`, `VAC`, `OHMS` e `CONT`:
- **Passo `0` $\rightarrow$ `10` (Init):** Ao ativar `#wRawControlWord.%X0`, envia `'REMS; FORMAT 1; *CLS$R$L'`. Quando confirmado com `'=>'`, avança para o Passo `20`.
- **Passo `20` (Configuração do Modo `VDC` / `VAC` / `OHMS` / `CONT`):** Copia `#sModeCmdStr` para `#sTxBuffer`, envia ao Fluke 8808A, grava `#usiActiveMode := #usiModeSelect` e avança para o Passo `40`.
- **Passo `40` $\rightarrow$ `50` (Ciclo de Leitura `'VAL1?$R$L'`):** Se `#wRawControlWord.%X5` (Troca de Modo) estiver ativo, desvia imediatamente para o Passo `20` para alterar a função do multímetro; caso contrário, envia `'VAL1?$R$L'` e aguarda a leitura no Passo `50`.
- **Watchdog (`#tonCommWatchdog`):** Se qualquer passo de recepção exceder `#tTimeoutComm`, ativa `#xCommError := TRUE` e força estado seguro.

```text
Network 6: Máquina de Estados RS-232 com Troca Dinâmica de Modo e Watchdog TON
|
|  #xEnable  #wRawControlWord.%X0   +--------+     +----------------------------+
|----] [-------------] [------------| EQ_INT |-----|           S_MOVE           |---|
|                             #iState|IN1     |     |'REMS; FORMAT 1; *CLS$R$L'  |
|                                   0|IN2     |     |                         OUT|-#sTxBuffer
|                                    +--------+     +----------------------------+
|                                                   |       MOVE 10 -> #iState   |
|                                                   +----------------------------+
|
|  #xEnable        +--------+       +----------------------------+     +---------------+
|----] [-----------| EQ_INT |-------|           S_MOVE           |-----|     MOVE      |---|
|            #iState|IN1     |       | #sModeCmdStr            IN |     |#usiModeSelect |
|                20|IN2     |       |                         OUT|-#sTx|            OUT|-#usiActiveMode
|                  +--------+       +----------------------------+     +---------------+
|
|  #xEnable        +--------+   #wRawControlWord.%X5          +--------------------+
|----] [-----------| EQ_INT |-----+------] [------------------| MOVE 20 -> #iState |---|
|            #iState|IN1     |     |                           +--------------------+
|                40|IN2     |     |
|                  +--------+     | #wRawControlWord.%X5      +--------------------+
|                                 +------]/[------------------| S_MOVE 'VAL1?$R$L' |---|
|                                                             | MOVE 50 -> #iState |
|                                                             +--------------------+
|
|  #xEnable        +---------------------------+       +---------------------------+
|----] [-----------|         Send_P2P          |-------|        Receive_P2P        |---|
|         #hwPortId|PORT                   DONE|-#xDone|PORT                    NDR|-#xRecvNdr
|        #sTxBuffer|BUFFER                ERROR|-#xSErr|BUFFER                ERROR|-#xRecvError
|                  +---------------------------+       +---------------------------+
|
|  #xEnable   #xWaitRxState  #xRecvNdr   +--------------------+            #xCommError
|----] [-----------] [----------]/[------|        TON         |----------------(S)----|
|                                        | (#tonCommWatchdog) |
|                            #tTimeoutCom|PT                  |
|                                        +--------------------+
```

---

## Seção 5: Guia de Instanciação no Programa Principal (`OB1` / `MainRoutine`)

Exemplo no **`OB1` (`Main`)** mostrando como operar o mesmo bloco `FB_FLUKE_8808A` nos 4 modos através do pino `usiModeSelect` (`0` = Tensão DC, `1` = Tensão AC, `2` = Resistência, `3` = Continuidade):

```text
Network 1 (OB1): Chamada do Multímetro Fluke 8808A com Seleção de Modo (0:VDC, 1:VAC, 2:OHMS, 3:CONT)
|
|  "Tag_SystemEnable"        +-----------------------------------+
|---------] [----------------|          "FB_FLUKE_8808A"         |------------------|
|                            |        "instFluke8808A_Bancada"   |
|                            |                             (DB10)|
|                        TRUE|xEnable                      xReady|-"DB_Teste".xReady
| "Local~CM_1241_RS232_1"    |hwPortId                   xRunning|-"DB_Teste".xRunning
|  "Tag_CmdStart"            |xCmdStart                    xFault|-"DB_Teste".xFault
|  "Tag_CmdReset"            |xCmdResetFault           xCommError|-"DB_Teste".xCommError
|  "Tag_ModoMultimetro"      |usiModeSelect         xContinuityOk|-"DB_Teste".xContinuidade_OK
|  "Tag_SetpointLimite"      |rSetpointLimit         xOpenCircuit|-"DB_Teste".xCircuitoAberto_0L
|                       T#2S |tTimeoutComm           rActualValue|-"DB_Teste".rValorMedido
|      "DB_HMI".Fluke8808A   |stHmi                    wFaultCode|-"DB_Teste".wFaultCode
|                            +-----------------------------------+
```

### Tabela Rápida de Uso do Pino `usiModeSelect` no `OB1`:
- **`"Tag_ModoMultimetro" := 0`** $\rightarrow$ Mede **Tensão DC** em Volts (`rActualValue` em `V`).
- **`"Tag_ModoMultimetro" := 1`** $\rightarrow$ Mede **Tensão AC** em Volts True-RMS (`rActualValue` em `V`).
- **`"Tag_ModoMultimetro" := 2`** $\rightarrow$ Mede **Resistência** em Ohms (`rActualValue` em `Ω`, `xOpenCircuit = TRUE` se ponteiras abertas).
- **`"Tag_ModoMultimetro" := 3`** $\rightarrow$ Mede **Continuidade** (`rActualValue` em `Ω` até `199.99 Ω` e aciona **`xContinuityOk = TRUE`** quando a resistência medida é $\le 20.0\ \Omega$).
