# 🛠️ PLC Device FB Developer (`plc-device-fb-developer`)

Skill especializada para criação automatizada de **Function Blocks (FB)** e **Add-On Instructions (AOI)** em **Lógica Ladder (Norma IEC 61131-3)** a partir de manuais técnicos, datasheets, mapas de registradores (Modbus RTU/TCP, RS-232 ASCII) ou telegramas de rede (Profinet, EtherNet/IP, CANopen, Profibus) de dispositivos industriais.

---

## 📂 Estrutura do Projeto

```text
my-skill/
├── AGENTS.md                                      # Regras globais do Agente Engenheiro de Automação
├── template_de_especifica_o_do_fb.md              # Template para preencher os requisitos do seu FB
├── FB_FLUKE_8808A.md                              # Exemplo completo gerado (Multímetro Fluke 8808A via RS-232)
├── README.md                                      # Este guia de utilização
└── .agents/
    └── skills/
        └── plc-device-fb-developer/
            ├── SKILL.md                           # Instruções principais e workflow da skill
            ├── references/
            │   └── iec61131-3-ladder-patterns.md  # Padrões de Networks Ladder e equivalência entre IDEs
            └── resources/
                └── output-template.md             # Template da documentação técnica de saída
```

---

## 🚀 Como Usar a Skill (Passo a Passo)

### Passo 1: Adicione o Manual ou Mapa de Memória do Dispositivo
Coloque o manual em PDF (ex.: `FLUKE_8808A_MANUAL.pdf`, datasheet de um inversor WEG/Siemens, etc.) na pasta do projeto ou copie a tabela de registradores/parâmetros do equipamento.

### Passo 2: Preencha o Template de Especificação
Abra o arquivo [`template_de_especifica_o_do_fb.md`](./template_de_especifica_o_do_fb.md) e preencha os **4 blocos principais**:

1. **Identificação do Dispositivo:** Tipo de equipamento, fabricante/modelo e o nome desejado para o bloco (ex.: `FB_FLUKE_8808A`, `FB_WEG_CFW500`).
2. **Plataforma do CLP / IDE:** Marque com `[x]` a plataforma alvo:
   - **Siemens TIA Portal** (S7-1200 / S7-1500)
   - **Rockwell Studio 5000** (ControlLogix / CompactLogix — AOI)
   - **WEG / Codesys V3** (PLC300 / PLC410 / PLC500)
   - **Schneider EcoStruxure** (M241 / M251 / M340 / M580)
   - **Delta ISPSoft** / **Mitsubishi GX Works**
3. **Escopo de Controle do FB:** Marque os comandos de entrada (`VAR_INPUT`) e leituras/diagnósticos de saída (`VAR_OUTPUT`) que você precisa nos pinos do bloco.
4. **Mapeamento do Manual:** Indique o nome do arquivo PDF presente na pasta ou cole os registradores/comandos brutos.

### Passo 3: Peça ao Agente no Chat
Envie um comando simples no chat, como:

> *"Leia o template de especificação e utilize a skill para criar o FB."*

Ou solicite diretamente no prompt sem precisar editar o template:

> *"Use a skill `plc-device-fb-developer` para criar um FB no Siemens TIA Portal para controlar um inversor via Modbus RTU (Control Word = 40001, Speed Ref = 40002 [0..6000 = 0..60.0 Hz], Status Word = 40003, Actual Speed = 40004)."*

### Passo 4: Refine ou Adicione Modos de Operação
Você pode pedir ajustes iterativos em linguagem natural a qualquer momento. Exemplo:

> *"Eu gostaria de poder trabalhar com o multímetro nos modos Tensão DC, Tensão AC, Resistência e Continuidade."*

---

## 📦 O Que a Skill Entrega (5 Seções Padronizadas)

Toda geração de Function Block segue rigorosamente a arquitetura de **5 seções** definida em [`AGENTS.md`](./AGENTS.md):

1. **Seção 1 — Mapeamento Técnico e Fatores de Escala:**
   - Tabelas detalhando a *Control Word* (bit a bit), *Status Word* (bit a bit), endereços/comandos de leitura e escrita e as equações matemáticas de conversão (*Scaling* e *Descaling*).
2. **Seção 2 — Estrutura de Dados Auxiliar (`UDT` / `DUT`):**
   - Código estruturado pronto para criar a `UDT` de integração com IHM/SCADA (WinCC, FactoryTalk, Codesys TargetVisu) sem precisar mapear tags avulsas.
3. **Seção 3 — Tabela de Declaração de Variáveis do FB (Norma IEC 61131-3):**
   - Tabelas completas de `VAR_INPUT`, `VAR_OUTPUT`, `VAR_IN_OUT`, `VAR` (`STATIC`), `VAR_TEMP` e `CONSTANT` com notação húngara padronizada (`x`, `i`, `ui`, `r`, `w`, `s`, `st`).
4. **Seção 4 — Lógica Ladder Interna do FB (Network 1 a Network 6):**
   - Diagramas **Ladder em ASCII** claros e comentados:
     - **Network 1:** Habilitação (`xEnable`) e Inicialização / *Safe State*.
     - **Network 2:** Desempacotamento da *Status Word* (*Unpacking*).
     - **Network 3:** Conversão de escala das leituras (Bruto $\rightarrow$ Engenharia).
     - **Network 4:** Intertravamentos operacionais, seleção de modos e detector de borda (`R_TRIG` / `P_TRIG` / `ONS`) para Reset de Falha.
     - **Network 5:** Empacotamento da *Control Word* (*Packing*) e limitação (`LIMIT`) / escala de Setpoints (Engenharia $\rightarrow$ Bruto).
     - **Network 6:** Comunicação de rede (I/O Cíclico ou Máquina de Estados Acíclica Modbus / Serial PtP) + Watchdog Timer (`TON`).
5. **Seção 5 — Guia de Instanciação no Programa Principal (`OB1` / `MainRoutine`):**
   - Exemplo visual em Ladder mostrando como chamar 1 ou mais dispositivos no programa principal trocando apenas o endereço de hardware/rede e o DB de instância.

---

## 🛡️ Regras de Ouro de Engenharia Garantidas pela Skill

- **Zero Variáveis Globais dentro do FB:** Nenhuma memória global (`%M`, `%MW`, `GVL`) é utilizada dentro do bloco, garantindo multi-instanciação segura para dezenas de equipamentos idênticos.
- **Unidades de Engenharia na Interface:** Os pinos externos do FB operam sempre em grandezas físicas reais (`Hz`, `RPM`, `V`, `A`, `Ω`, `°C`, `%`, `Bar`). Toda conversão bruta acontece de forma transparente dentro do FB.
- **Segurança Fail-Safe:** Queda de `xEnable`, falha ativa (`xFault`) ou timeout de comunicação (`xCommError`) bloqueiam partidas e levam as saídas imediatamente para estado seguro.

