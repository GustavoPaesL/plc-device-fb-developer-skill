---
name: plc-device-fb-developer
description: >-
  Use esta skill quando o usuário pedir para criar, projetar ou estruturar um
  Function Block (FB) ou Add-On Instruction (AOI) para CLP (IEC 61131-3 /
  Lógica Ladder) a partir de manuais, datasheets, mapas de registradores
  (Modbus RTU/TCP) ou telegramas de rede (Profinet, EtherNet/IP, CANopen,
  Profibus) de dispositivos industriais (inversores de frequência, servo drives,
  medidores de energia, controladores de temperatura, válvulas inteligentes, etc.).
---

# PLC_Device_FB_Developer — Desenvolvedor de Function Block de Dispositivo para CLP

## 1. Identificação e Objetivo

- **Nome Técnico da Skill:** `PLC_Device_FB_Developer` (`plc-device-fb-developer`)
- **Problema que Resolve:** Interpreta o manual técnico, datasheet ou mapa de registradores de um dispositivo industrial (inversor de frequência, servo drive, medidor de energia, controlador de temperatura, válvula inteligente, etc.) e projeta um **Function Block (FB) encapsulado, padronizado e reutilizável em Lógica Ladder (IEC 61131-3)** para controlar e monitorar esse dispositivo especificamente.
- **Benefício Principal:** Permite instanciar vários dispositivos iguais no projeto apenas arrastando o FB criado e alterando o endereço de rede/hardware e o bloco de dados de instância (*Instance Data Block* / *Tag de Instância*), sem duplicar código ou causar conflitos de memória.

---

## 2. Entradas Necessárias (Inputs)

Antes de gerar o Function Block, colete ou extraia das informações fornecidas pelo usuário:

1. **Documentação do Dispositivo:**
   - Trechos do manual, tabela de parâmetros, mapa de registradores (Modbus RTU/TCP) ou estrutura do telegrama de comunicação (Profinet, EtherNet/IP, CANopen, Profibus).
2. **Plataforma do CLP / IDE:**
   - Ex.: Siemens TIA Portal (S7-1200/S7-1500), Rockwell Studio 5000 (AOI), WEG TPW04 / Codesys, Schneider EcoStruxure Machine Expert, Delta ISPSoft, Mitsubishi GX Works.
   - *Nota:* Caso o usuário não especifique a IDE, utilize o padrão genérico **IEC 61131-3 (Codesys / TIA Portal)** e informe claramente as convenções adotadas.
3. **Escopo de Controle do FB:**
   - Funções desejadas no bloco (ex.: controle básico de Liga/Desliga + Referência de Velocidade, ou controle completo com JOG, Sentido de Giro, Reset de Falha, leitura de corrente/torque/potência e diagnóstico detalhado).

---

## 3. Fluxo de Trabalho (Workflow)

Execute os 4 passos abaixo em sequência rigorosa:

### Passo 1: Extração e Mapeamento do Dispositivo
1. **Palavra de Controle (*Control Word*):** Identifique o endereço/registrador e detalhe a função de cada bit (ex.: `Bit 0` = Run/Stop, `Bit 1` = Sentido de Giro, `Bit 7` = Reset de Falha).
2. **Palavra de Status (*Status Word*):** Identifique o endereço/registrador e detalhe a função de cada bit (ex.: `Bit 0` = Pronto/Ready, `Bit 1` = Operando/Running, `Bit 3` = Em Falha/Fault).
3. **Variáveis Analógicas/Numéricas (*Setpoints* e *Feedbacks*):**
   - Mapeie os registradores de escrita (*Setpoints*) e leitura (*Feedbacks*).
   - Especifique os **Fatores de Escala (*Scaling*)**, limites operacionais, unidades de engenharia (ex.: `0 a 6000` bruto equivale a `0.0 a 60.0 Hz`) e tipos de dados (`INT`, `UINT`, `DINT`, `REAL`, `WORD`).

### Passo 2: Definição da Interface do Function Block (Pinagem / Assinatura)
Projete a tabela de variáveis locais do FB dividida rigorosamente segundo a norma **IEC 61131-3** (utilizando prefixos de notação húngara: `x` = BOOL, `i` = INT, `ui` = UINT, `di` = DINT, `udi` = UDINT, `r` = REAL, `w` = WORD, `dw` = DWORD, `t` = TIME, `s` = STRING, `st` = STRUCT/UDT, `aw` = ARRAY OF WORD):

- **`VAR_INPUT` (Entradas de Comando e Configuração):**
  - `xEnable` (`BOOL`): Habilita o funcionamento do bloco.
  - `udiHwId_Or_NodeAddr` (`UINT` / `HW_IO`): Endereço do nó na rede ou ID de hardware.
  - Comandos operacionais em linguagem humana: `xCmdStart`, `xCmdStop`, `xCmdReverse`, `xCmdJog`, `xCmdResetFault` (`BOOL`).
  - Setpoints em unidades de engenharia: `rSetpointSpeed_Hz` (`REAL`), `rSetpointTorque_Pct` (`REAL`), `tTimeoutComm` (`TIME`).
- **`VAR_OUTPUT` (Saídas de Status e Diagnóstico):**
  - Status padrão do bloco: `xReady`, `xRunning`, `xFault`, `xCommError` (`BOOL`).
  - Leituras convertidas em unidades de engenharia: `rActualSpeed_Hz` (`REAL`), `rActualCurrent_A` (`REAL`).
  - Diagnóstico detalhado: `wFaultCode` (`WORD`), `sDiagnosticMsg` (`STRING`).
- **`VAR_IN_OUT` (Passagem por Referência — se aplicável):**
  - Estruturas de barramento compartilhado, UDTs de comunicação/IHM ou ponteiros de controle de mestre de rede.
- **`VAR` / `STATIC` (Memória Interna da Instância):**
  - Máquina de estados (`iState` : `INT`), timers IEC (`tonWatchdog`, `tonCommTimeout` : `TON`), detectores de borda (`rtResetFault`, `rtStart` : `R_TRIG`), buffers brutos de envio/recebimento (`awTxBuffer`, `awRxBuffer`), palavras de controle e status brutas (`wRawControlWord`, `wRawStatusWord` : `WORD`).
- **`VAR_TEMP` (Variáveis Temporárias):**
  - Variáveis auxiliares de cálculo de ciclo único (ex.: `rTempScaled`, `iTempClamped`, resultados intermediários de conversão).

### Passo 3: Criação de UDT / DUT (Tipo de Dado do Usuário)
- Projete uma estrutura de dados (`UDT_<NomeDoDispositivo>` ou `DUT_<NomeDoDispositivo>`) para organizar o mapeamento de I/O ou para integração direta com faceplates de **IHM / SCADA**, evitando a ligação manual de dezenas de tags individuais.

### Passo 4: Desenvolvimento da Lógica Ladder Interna do FB (Dividida em Networks)
Estruture o código Ladder interno do FB na seguinte sequência padronizada de *Networks* (Rungs). Consulte [iec61131-3-ladder-patterns.md](./references/iec61131-3-ladder-patterns.md) para detalhes de implementação por fabricante:

1. **Network 1 — Inicialização e Validação (`xEnable`):**
   - Verifica se `xEnable` está ativo. Caso `xEnable = FALSE`, força estados seguros (zera `wRawControlWord`, reseta saídas operacionais `xRunning`/`xReady` e reinicia `iState := 0`).
2. **Network 2 — Desempacotamento da Status Word e Feedbacks (*Unpacking & Descaling*):**
   - Extrai os bits individuais de `wRawStatusWord` para as saídas booleanas (`xReady`, `xRunning`, `xFault`).
   - Converte os valores brutos recebidos (ex.: `INT` `0..6000`) para `REAL` em unidade de engenharia (`0.0..60.0 Hz`) utilizando blocos nativos (`NORM_X`/`SCALE_X`, `CPT` ou `INT_TO_REAL` + `DIV`/`MUL`).
3. **Network 3 — Intertravamentos e Lógica de Comando (*Interlocks*):**
   - Garante que comandos conflitantes (ex.: `xCmdStart` e `xCmdStop`, ou avanço/reverso indevidos) sejam intertravados com prioridade para parada (`xCmdStop`).
   - Aplica detecção de borda de subida (`R_TRIG` / `ONS`) para comandos pulsados como `xCmdResetFault`.
   - Bloqueia o acionamento (`xCmdStart`) caso o dispositivo esteja em falha (`xFault = TRUE`) ou em erro de comunicação (`xCommError = TRUE`).
4. **Network 4 — Empacotamento da Control Word e Setpoints (*Packing & Scaling*):**
   - Valida e limita os Setpoints de entrada com bloco `LIMIT` entre os valores mínimo e máximo permitidos pelo manual.
   - Converte o Setpoint em unidade de engenharia (`REAL`) para o inteiro bruto exigido pelo registrador (`REAL_TO_INT` / `ROUND`).
   - Monta os bits individuais em `wRawControlWord` para envio ao dispositivo.
5. **Network 5 — Máquina de Estados de Comunicação / Atualização de I/O:**
   - **Se I/O Cíclico (Profinet / EtherNet/IP / Profibus):** Transfere os dados brutos de/para a periferia ou instruções de consistência (`DPRD_DAT`/`DPWR_DAT`, `CPS`, mapeamento de I/O).
   - **Se Acíclico / Serial (Modbus RTU / Modbus TCP):** Implementa um sequenciador passo a passo (`iState = 0, 10, 20, 30, 99...`) alternando entre leitura de status/feedbacks e escrita de comandos/setpoints, aguardando `DONE` ou `ERROR` antes de avançar de passo.
6. **Network 6 — Watchdog, Timeout e Tratamento de Falhas (*Safe State*):**
   - Monitora perda de comunicação ou travamento de requisição através de timer `TON` (`tonCommTimeout`).
   - Em caso de timeout ou erro de protocolo, aciona `xCommError := TRUE`, força as saídas de controle para **Estado Seguro (*Safe State*)** e registra o código em `wFaultCode` e a descrição em `sDiagnosticMsg`.

---

## 4. Padrão de Arquitetura e Regras de Ouro (Design Pattern)

1. **Zero Endereçamento Global Dentro do FB:**
   - **NUNCA** utilize memórias globais (ex.: `%M10.0`, `%MW20`, `DB1.DBX0.0`, tags de escopo Controller) dentro da lógica interna do FB. Toda memória deve residir em `VAR` (`STATIC`) ou `VAR_TEMP` para garantir multi-instanciação livre de conflitos.
2. **Abstração Total para o Usuário do Bloco:**
   - Quem instancia o FB no `OB1` / `MainRoutine` não deve precisar conhecer máscaras de bits da *Control Word*, códigos de função Modbus ou fatores de multiplicação. A interface externa opera exclusivamente com booleanos semânticos (`xCmdStart = TRUE`) e unidades físicas reais (`rSetpointSpeed_Hz = 45.5`).
3. **Segurança por Padrão (*Fail-Safe*):**
   - `xCmdStop`, `xFault`, `xCommError` ou queda de `xEnable` devem imediatamente zerar ou levar a palavra de controle ao estado de parada segura.
4. **Representação Visual Clara em ASCII Ladder:**
   - Como o Ladder é uma linguagem gráfica, desenhe todas as Networks em **ASCII esquemático claro e alinhado** (`|--[ ]--[TON]--( )--|`), acompanhado da explicação dos blocos nativos da plataforma escolhida.

---

## 5. Formato de Saída (Outputs)

Sempre entregue a resposta final seguindo exatamente a estrutura de 5 seções abaixo (veja o template detalhado em [output-template.md](./resources/output-template.md)):

1. **Resumo do Dispositivo e Protocolo:** Tabela consolidada com registradores/telegramas, palavras de controle/status (bit a bit) e fatores de escala extraídos da documentação.
2. **Definição de UDT / DUT:** Estrutura de dados padronizada para integração com IHM/SCADA e rede.
3. **Declaração de Interface do FB (`FB_<NomeDoDispositivo>`):** Tabelas completas de `VAR_INPUT`, `VAR_OUTPUT`, `VAR_IN_OUT`, `VAR` (`STATIC`) e `VAR_TEMP` com nome, tipo, valor inicial e comentário.
4. **Lógica Interna do FB (Network 1 a Network 6):** Diagramas Ladder em ASCII para cada Network + explicação técnica das instruções da IDE alvo.
5. **Exemplo de Instanciação (Chamada no `OB1` / `Main`):** Demonstração visual em Ladder de como instanciar múltiplos dispositivos no programa principal trocando apenas o DB de instância e o endereço de rede/hardware.

