# Especificação para Criação de Function Block (FB)

> **Instrução de uso:** Preencha os campos abaixo com os dados do seu projeto e envie junto com o trecho do manual ou tabela de registradores do dispositivo para que o agente gere o Function Block completo em Ladder.

---

## 1. Identificação do Dispositivo
- **Tipo de Equipamento:** `MULTIMETRO`
- **Fabricante e Modelo:** `FLUKE 8808A`
- **Nome desejado para o FB:** `FB_FLUKE_8808A`

---

## 2. Plataforma do CLP / IDE
*(Marque com um `X` ou preencha a plataforma utilizada)*

- **Fabricante / Família do CLP:**
  - [x] Siemens (S7-1200 / S7-1500) — **TIA Portal**
  - [ ] Rockwell Automation (ControlLogix / CompactLogix) — **Studio 5000 (AOI)**
  - [ ] WEG / Codesys (PLC300 / PLC410 / PLC500) — **Codesys V3**
  - [ ] Schneider Electric (M241 / M251 / M340 / M580) — **EcoStruxure Machine/Control Expert**
  - [ ] Delta Electronics (DVP / AS Series) — **ISPSoft**
  - [ ] Outro: `[Especificar fabricante, modelo e software]`

- **Meio Físico e Protocolo de Comunicação:**
  - [ ] **I/O Cíclico (Mapeamento direto no Hardware):** Profinet / EtherNet/IP / Profibus-DP / CANopen
  - [x] **Comunicação Acíclica / Serial (Máquina de Estados):** RS232
  - **Bloco de comunicação nativo preferido (opcional):** `[Ex: MB_MASTER, DPRD_DAT, MSG, MODRW]`

---

## 3. Escopo de Controle do FB (O que o bloco deve fazer)
*(Marque as funcionalidades que devem estar presentes nos pinos do seu Function Block)*

### 3.1 Comandos de Entrada (`VAR_INPUT`)
- [x] Habilitação geral do bloco (`xEnable`)
- [x] Comando Ligar / Partir (`xCmdStart` / `xCmdRun`)
- [ ] Comando Sentido de Giro — Avanço/Recuo (`xCmdForward` / `xCmdReverse`)
- [ ] Comando de JOG / Impulso (`xCmdJog`)
- [x] Comando de Reset de Falhas (`xCmdResetFault`)
- [ ] Seleção de Parada Rápida / Parada por Rampa (`xCmdQuickStop`)
- [x] Setpoint Principal (Ex: Velocidade/Temperatura): `Unidade: [Ex: 0.0 a 60.0 Hz]`
- [ ] Setpoint Secundário (Ex: Rampa Acel/Desacel ou Limite de Torque): `Unidade: [Especificar]`
- [ ] Outros comandos específicos: `[Descrever]`

### 3.2 Diagnósticos e Leituras de Saída (`VAR_OUTPUT`)
- [x] Equipamento Pronto para Partir (`xReady`)
- [x] Equipamento Operando / Em Movimento (`xRunning`)
- [x] Equipamento em Falha/Alarme (`xFault`)
- [x] Erro / Timeout de Comunicação (`xCommError`)
- [ ] Velocidade / Setpoint Atingido (`xAtReference`)
- [x] Leitura da Variável Principal (Ex: Frequência/Velocidade Atual): `Unidade: [Ex: Hz ou RPM]`
- [ ] Leitura de Corrente do Motor (`rActualCurrent_A`)
- [ ] Leitura de Torque / Potência / Tensão CC (`rActualTorque_Pct`)
- [x] Código da Falha Atual (`wFaultCode`)
- [ ] Outras leituras específicas: `[Descrever]`

---

## 4. Mapeamento do Manual do Dispositivo (Dados Brutos)
*(Cole abaixo o trecho do manual, tabela de registradores ou preencha os campos que você já conhece)*

- presente em FLUKE_8808A_MANUAL.pdf