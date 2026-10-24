# Template de Saída — `PLC_Device_FB_Developer`

Utilize este template estruturado ao responder a solicitações de criação de Function Blocks para dispositivos de CLP.

---

## 1. Resumo do Dispositivo e Protocolo

- **Dispositivo:** `<Fabricante e Modelo>`
- **Protocolo de Comunicação:** `<Modbus RTU / Modbus TCP / Profinet / EtherNet/IP / CANopen>`
- **Plataforma / IDE Alvo:** `<Siemens TIA Portal / Rockwell Studio 5000 / Codesys / WEG / etc.>`

### 1.1 Mapeamento de Registradores / Telegrama

| Tipo | Endereço / Offset | Nome no Manual | Tipo de Dado | Escala Bruta | Escala Engenharia | Unidade |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Escrita | `<Ex: 40001 / PZD1>` | Control Word | `WORD` | Bits `0..15` | Booleano por bit | - |
| Escrita | `<Ex: 40002 / PZD2>` | Speed Reference | `INT` / `UINT` | `0..6000` | `0.0..60.0` | `Hz` |
| Leitura | `<Ex: 40003 / PZD1>` | Status Word | `WORD` | Bits `0..15` | Booleano por bit | - |
| Leitura | `<Ex: 40004 / PZD2>` | Actual Speed | `INT` / `UINT` | `0..6000` | `0.0..60.0` | `Hz` |
| Leitura | `<Ex: 40005 / PZD3>` | Motor Current | `INT` / `UINT` | `0..1000` | `0.0..100.0` | `A` |
| Leitura | `<Ex: 40006 / PZD4>` | Fault Code | `WORD` | `0..65535` | Código Hex/Dec | - |

### 1.2 Detalhamento Bit a Bit (Control Word & Status Word)

| Bit | Control Word (Escrita CLP -> Dispositivo) | Status Word (Leitura Dispositivo -> CLP) |
| :--- | :--- | :--- |
| `.0` | `Run / Stop` (`1` = Run, `0` = Stop) | `Ready` (`1` = Pronto para partir) |
| `.1` | `Direction` (`0` = Avanço, `1` = Reverso) | `Running` (`1` = Em operação) |
| `.2` | `Jog` (`1` = Comando Jog ativo) | `Reverse` (`1` = Sentido reverso ativo) |
| `.3` | `Quick Stop` (`0` = Parada rápida, se aplicável) | `Fault` (`1` = Dispositivo em falha) |
| `.7` | `Fault Reset` (`0 -> 1` = Borda de subida) | `Warning / Alarm` (`1` = Alarme ativo) |

---

## 2. Definição de UDT / DUT (`UDT_<Dispositivo>`)

```iecst
TYPE UDT_<Dispositivo>_HMI :
STRUCT
    // Comandos vindos da IHM/SCADA
    xCmdStart           : BOOL;   // Comando de partida
    xCmdStop            : BOOL;   // Comando de parada
    xCmdReverse         : BOOL;   // Seleção de sentido reverso
    xCmdResetFault      : BOOL;   // Pulso para reset de falha
    rSetpointSpeed_Hz   : REAL;   // Referência de velocidade (Hz)

    // Status e Leituras para a IHM/SCADA
    xReady              : BOOL;   // Dispositivo pronto
    xRunning            : BOOL;   // Dispositivo operando
    xFault              : BOOL;   // Falha ativa no dispositivo
    xCommError          : BOOL;   // Falha de comunicação
    rActualSpeed_Hz     : REAL;   // Velocidade atual (Hz)
    rActualCurrent_A    : REAL;   // Corrente atual (A)
    wFaultCode          : WORD;   // Código de falha atual
END_STRUCT
END_TYPE
```

---

## 3. Declaração de Interface do FB (`FB_<Dispositivo>`)

### `VAR_INPUT`
| Nome da Variável | Tipo de Dado | Valor Inicial | Descrição |
| :--- | :--- | :--- | :--- |
| `xEnable` | `BOOL` | `FALSE` | Habilita o processamento do bloco |
| `udiHwId_Or_NodeAddr` | `UINT` | `1` | Endereço escravo na rede ou Hardware ID |
| `xCmdStart` | `BOOL` | `FALSE` | Comando para ligar o equipamento |
| `xCmdStop` | `BOOL` | `FALSE` | Comando de parada (prioritário) |
| `xCmdReverse` | `BOOL` | `FALSE` | `FALSE` = Horário/Avanço, `TRUE` = Anti-horário/Reverso |
| `xCmdResetFault` | `BOOL` | `FALSE` | Reset de falha (acionado por borda de subida) |
| `rSetpointSpeed_Hz` | `REAL` | `0.0` | Setpoint de velocidade em Hz |
| `tTimeoutComm` | `TIME` | `T#2S` | Tempo máximo de espera de comunicação (Watchdog) |

### `VAR_OUTPUT`
| Nome da Variável | Tipo de Dado | Valor Inicial | Descrição |
| :--- | :--- | :--- | :--- |
| `xReady` | `BOOL` | `FALSE` | Dispositivo pronto para operar |
| `xRunning` | `BOOL` | `FALSE` | Dispositivo em operação |
| `xFault` | `BOOL` | `FALSE` | Dispositivo em estado de falha |
| `xCommError` | `BOOL` | `FALSE` | Erro de timeout ou falha de comunicação na rede |
| `rActualSpeed_Hz` | `REAL` | `0.0` | Velocidade atual convertida (Hz) |
| `rActualCurrent_A` | `REAL` | `0.0` | Corrente atual do motor convertida (A) |
| `wFaultCode` | `WORD` | `16#0000` | Código numérico da falha ativa |
| `sDiagnosticMsg` | `STRING[40]` | `'OK'` | Mensagem de diagnóstico amigável |

### `VAR_IN_OUT`
| Nome da Variável | Tipo de Dado | Descrição |
| :--- | :--- | :--- |
| `stHmiData` | `UDT_<Dispositivo>_HMI` | Estrutura de interface direta com IHM/SCADA |

### `VAR` (`STATIC` — Memória Interna da Instância)
| Nome da Variável | Tipo de Dado | Valor Inicial | Descrição |
| :--- | :--- | :--- | :--- |
| `iState` | `INT` | `0` | Passo atual da máquina de estados de comunicação |
| `wRawControlWord` | `WORD` | `16#0000` | Palavra de controle bruta enviada ao dispositivo |
| `wRawStatusWord` | `WORD` | `16#0000` | Palavra de status bruta recebida do dispositivo |
| `iRawSetpointSpeed` | `INT` | `0` | Setpoint de velocidade escalonado para registrador |
| `iRawActualSpeed` | `INT` | `0` | Velocidade bruta lida do registrador |
| `iRawActualCurrent` | `INT` | `0` | Corrente bruta lida do registrador |
| `awTxBuffer` | `ARRAY[0..3] OF WORD` | `[4(16#0000)]` | Buffer local de transmissão |
| `awRxBuffer` | `ARRAY[0..3] OF WORD` | `[4(16#0000)]` | Buffer local de recepção |
| `rtResetFault` | `R_TRIG` | - | Detector de borda de subida para Reset de Falha |
| `tonCommWatchdog` | `TON` | - | Timer IEC de detecção de timeout de comunicação |
| `xInternalRunCmd` | `BOOL` | `FALSE` | Comando interno validado após intertravamentos |

### `VAR_TEMP`
| Nome da Variável | Tipo de Dado | Descrição |
| :--- | :--- | :--- |
| `rTempLimitedSpeed` | `REAL` | Valor de velocidade limitado dentro da faixa segura |
| `rTempScaled` | `REAL` | Variável temporária de cálculo de escala |
| `iRetVal` | `INT` | Código de retorno de instruções de comunicação |

---

## 4. Lógica Interna do FB (Network por Network)

*(Inserir diagramas Ladder em ASCII de Network 1 a Network 6 com explicação das instruções)*

---

## 5. Exemplo de Instanciação no Programa Principal (`OB1` / `MainRoutine`)

*(Inserir chamada de múltiplas instâncias em Ladder mostrando o isolamento por DB de instância)*

