---
tags:
  - hardware
  - hardware/protocolos
  - hardware/serial
  - hardware/automotivo
source: "Vídeo 'Every Hardware Protocol Explained Simply in 10 Minutes!' + especificação CAN da Bosch e norma ISO 11898"
---
# CAN

📚 **Referência:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* (cap. 5 — CAN for robust networks) | Pré-requisito: [[Fundamentos de Protocolos de Hardware]]

---

# 🚗 O que é o CAN?

**CAN** significa ***Controller Area Network*** — Rede de Área de Controladores. Foi desenvolvido pela **Bosch** a partir de **1983** (apresentado em 1986) para um ambiente específico: **carros**.

Pense em um carro moderno. Ele tem **dezenas de pequenos computadores** — as **ECUs** (*Electronic Control Units*):

> 💡 **ECU (Unidade de Controle Eletrônico):** pequeno computador responsável por uma função do veículo — motor, freios (ABS), airbags, transmissão, direção, painel...

Todos precisam conversar o tempo todo (o freio precisa avisar o motor, o airbag precisa saber da colisão...) e **não podem falhar**, porque uma falha pode causar um acidente. Antes do CAN, cada par de ECUs usava um **fio dedicado** — um feixe gigante, pesado, caro e propenso a defeitos. O CAN substitui tudo isso por **um barramento de dois fios** compartilhado.

```
Sem CAN (fios dedicados):               Com CAN (um barramento):

Motor ──── Freio                        Motor ─┐
Motor ──── Câmbio                       Freio ─┤
Freio ──── Painel                       Câmbio ┼─── CAN_H ═══════════
Airbag ─── Painel ...                   Airbag ┤    CAN_L ═══════════
(centenas de fios)                      Painel ┘
                                        (2 fios para todos)
```

> 💡 **CAN:** protocolo serial de **dois fios diferenciais**, **multi-mestre**, orientado a **mensagens**, com **arbitragem por prioridade**, **detecção de erro** robusta e **retransmissão automática**. Projetado para **tempo real** e **alta confiabilidade**.

---

# 🔌 A camada física: dois fios diferenciais

Os fios se chamam **CAN_H** (*High*) e **CAN_L** (*Low*) e formam um **par trançado**. A informação está na **diferença de tensão** entre eles — sinalização **diferencial** (veja [[Fundamentos de Protocolos de Hardware]]).

```
Recessivo (bit 1):    CAN_H ≈ 2,5 V    CAN_L ≈ 2,5 V   → diferença ≈ 0 V
Dominante (bit 0):    CAN_H ≈ 3,5 V    CAN_L ≈ 1,5 V   → diferença ≈ 2 V

3,5 V ┤        ┌────┐          CAN_H
2,5 V ┤────────┘    └──────── (repouso)
1,5 V ┤        └────┘          CAN_L
         repouso  dominante  repouso
```

**Por que diferencial?** Um carro é um ambiente **eletricamente terrível**: motor de partida, ignição, alternador geram muito ruído. Como o ruído afeta os dois fios do par **igualmente**, ele **se cancela** na subtração `CAN_H − CAN_L` (mesmo princípio dos [[Cabos Par Trançado (UTP e STP)]]).

## Estados dominante × recessivo

> 💡 **Bit dominante (0):** estado em que os transmissores **forçam** uma diferença de tensão. **Vence** sempre que alguém o coloca no barramento.
> **Bit recessivo (1):** estado de **repouso**, sem diferença. **Perde** para qualquer dominante.

É um **wired-AND**, igual ao do [[I2C]]: se um nó transmitir 0 e outro transmitir 1, o barramento fica em **0**. Esse detalhe é a base da arbitragem.

## Terminação de 120 Ω

Em cada **ponta** do barramento vai um **resistor de 120 Ω** entre CAN_H e CAN_L. Ele casa a impedância do cabo e evita **reflexões** do sinal (o "eco" que corrompe os bits).

```
 [120 Ω]──┬──────── CAN_H ═══════════════════════ ┬──[120 Ω]
          └──────── CAN_L ═══════════════════════ ┘
   NÓ A        NÓ B       NÓ C        NÓ D        NÓ E
```

Se você medir com um multímetro, o barramento desligado mostra **≈ 60 Ω** (dois resistores de 120 Ω em paralelo) — truque clássico para diagnosticar problemas.

---

# 🏷️ O grande diferencial: mensagens e o ID

Em quase todos os outros protocolos deste MOC, o endereço identifica **um dispositivo** ("fale com o 0x48"). No CAN é **diferente**:

> 💡 **Orientado a mensagens:** o **ID** da mensagem identifica **o conteúdo** ("rotação do motor", "pressão do freio"), **não o destinatário**. A mensagem é **transmitida para todos**; cada nó decide se ela lhe interessa.

Analogia: é como uma **rádio**. O ID é o **assunto** ("trânsito"): quem se interessa por trânsito ouve, os outros ignoram. E o **ID também é a prioridade** da mensagem — quanto **menor** o número, **maior** a prioridade.

---

# ⚔️ Arbitragem: quem fala primeiro?

Como vários nós podem decidir transmitir **ao mesmo tempo** (é multi-mestre), o CAN precisa de uma regra para que **ninguém perca dados**. É a **arbitragem** — o recurso que o vídeo destaca como o que diferencia o CAN de tudo mais.

> 💡 **Arbitragem bit a bit não destrutiva:** enquanto transmite o ID, cada nó **também lê** o barramento. Se enviou **1** (recessivo) mas leu **0** (dominante), sabe que **outro nó com ID menor** está transmitindo — então **para** e espera. O vencedor **não percebe a disputa e nem perde um bit**.

## Exemplo passo a passo

Três nós começam a transmitir juntos (IDs de 11 bits):

```
Nó A: ID 0x100 = 001 0000 0000
Nó B: ID 0x0F0 = 000 1111 0000
Nó C: ID 0x200 = 010 0000 0000

Bit 10:  A=0  B=0  C=0   → barramento = 0   (todos continuam)
Bit  9:  A=0  B=0  C=1   → barramento = 0   (C enviou 1, leu 0 → C PERDE e sai)
Bit  8:  A=1  B=0        → barramento = 0   (A enviou 1, leu 0 → A PERDE e sai)
Bit  7:  B segue sozinho → B transmite o restante do quadro

Vencedor: Nó B (ID 0x0F0, o MENOR)
A e C tentam de novo assim que o barramento ficar livre.
```

**Sem colisão destrutiva, sem perda de dados, sem atraso para o vencedor.** Compare com o Ethernet antigo (CSMA/CD), em que uma colisão destruía os dois quadros — aqui a mensagem de maior prioridade passa **intacta**.

Isso garante algo crucial em tempo real: a mensagem **mais importante** (freio) **sempre** passa na frente da menos importante (rádio).

---

# 📦 O quadro CAN

Um quadro de dados do CAN clássico (formato base, ID de 11 bits):

```
┌─────┬───────────┬─────┬─────┬────┬──────┬────────────┬───────┬─────┬─────┬─────┐
│ SOF │ ID (11 b) │ RTR │ IDE │ r0 │ DLC  │ DADOS      │ CRC   │ ACK │ EOF │ IFS │
│ 1 b │  prioridade│ 1 b │ 1 b │1 b │ 4 b  │ 0–8 bytes  │ 15 b  │ 2 b │ 7 b │ 3 b │
└─────┴───────────┴─────┴─────┴────┴──────┴────────────┴───────┴─────┴─────┴─────┘
```

| Campo | Função |
|---|---|
| **SOF** (*Start of Frame*) | Um bit dominante que marca o início e sincroniza os nós |
| **ID** | Identifica o conteúdo **e** define a prioridade (arbitragem) |
| **RTR** | Distingue quadro de **dados** de pedido remoto |
| **IDE** | Indica ID padrão (11 bits) ou **estendido (29 bits)** |
| **DLC** (*Data Length Code*) | Quantos bytes de dados (0 a 8) |
| **Dados** | A informação — **até 8 bytes** no CAN clássico |
| **CRC** | Código de 15 bits para **detectar erros** |
| **ACK** | Os receptores que leram corretamente **puxam o bit para dominante** |
| **EOF** | 7 bits recessivos marcando o fim |
| **IFS** | Espaço entre quadros |

> 💡 **Bit stuffing:** como o CAN **não tem clock**, os nós se ressincronizam pelas **transições** do sinal. Se houvesse uma sequência longa de bits iguais, ficariam sem referência. Por isso o transmissor **insere automaticamente** um bit oposto após **5 bits iguais seguidos**, e o receptor o remove.

## CAN FD

O **CAN FD** (*Flexible Data-rate*) é a evolução: **até 64 bytes** por quadro e velocidade mais alta na fase de dados (**de 2 a 8 Mbit/s**), mantendo a arbitragem do CAN clássico.

---

# 🛡️ Detecção de erros e confiabilidade

O CAN usa **vários níveis** de verificação — por isso é considerado um dos protocolos **mais robustos já criados**:

| Mecanismo | O que verifica |
|---|---|
| **Monitoramento de bit** | O transmissor lê o que enviou e compara |
| **Bit stuffing** | Mais de 5 bits iguais seguidos = erro |
| **CRC de 15 bits** | Integridade do conteúdo |
| **Verificação de formato** | Campos fixos no lugar certo |
| **ACK** | Pelo menos um receptor leu o quadro |

Ao detectar erro, o nó transmite um **quadro de erro** que avisa **todos** os outros para **descartarem** a mensagem — e o transmissor **tenta de novo automaticamente**.

> 💡 **Confinamento de falhas:** cada nó mantém **contadores de erro**. Um nó com defeito que gera muitos erros passa de **error-active** → **error-passive** (só envia sinais de erro "fracos") → **bus-off** (se **desconecta** do barramento). Assim **um nó quebrado não derruba a rede inteira**.

---

# 📏 Velocidade × comprimento

Como o sinal precisa **atravessar o barramento e voltar** dentro de **um bit** (para a arbitragem funcionar), **quanto mais rápido, mais curto** o cabo:

| Velocidade | Comprimento máximo (aprox.) |
|---|---|
| **1 Mbit/s** | ~ 40 m |
| **500 kbit/s** | ~ 100 m |
| **250 kbit/s** | ~ 250 m |
| **125 kbit/s** | ~ 500 m |

Por isso o CAN **não é o mais rápido** — é um compromisso consciente em favor da **confiabilidade**.

---

# ⚖️ Pontos fortes e fracos

| ✅ Vantagens | ❌ Limitações |
|---|---|
| **Arbitragem por prioridade**: sem colisão, sem perda | **Mais complexo** que UART e I2C |
| **Muito robusto contra ruído** (diferencial + CRC + ACK) | Quadro **pequeno** (8 bytes no clássico) |
| **Multi-mestre**: qualquer nó fala | **Velocidade moderada** (1 Mbit/s clássico) |
| **Tolera falhas**: nó defeituoso se isola sozinho | Exige **transceptor** extra e **terminação** |
| Poucos fios para muitos nós | **Sem segurança** embutida (sem autenticação nem criptografia) |
| **40+ anos** de uso comprovado | Latência e prioridade dependem de bom projeto dos IDs |

---

# 🧰 Onde o CAN aparece

Em qualquer lugar onde **falha de comunicação não é aceitável**:

- **Carros** (porta de diagnóstico **OBD-II**), **caminhões** e **ônibus** (variante J1939)
- **Robôs industriais** e **linhas de automação**
- **Máquinas agrícolas** (ISOBUS) e equipamentos de construção
- **Dispositivos médicos**, elevadores, trens e embarcações
- **Drones e robôs** (via CAN ou o derivado **CANopen**)

---

# 🧪 No ESP32 e no Arduino

- O ESP32 possui um controlador CAN interno, chamado **TWAI** (*Two-Wire Automotive Interface*), compatível com **CAN 2.0**.
- ⚠️ O ESP32 **não liga direto** no barramento: precisa de um **transceptor CAN** externo (ex.: **SN65HVD230**, que trabalha em 3,3 V, ou o MCP2551, de 5 V) que converte os níveis lógicos nos níveis diferenciais CAN_H/CAN_L.
- Em placas sem controlador CAN (ex.: Arduino Uno), usa-se o módulo **MCP2515** (controlador CAN **via [[SPI]]**) + transceptor.
- Em um **PC Linux**, a interface CAN aparece como um dispositivo de rede (`can0`) — a pilha **SocketCAN** reaproveita o conceito de [[Socket]].

---

# ✅ Resumo do Conceito

- **CAN** (*Controller Area Network*): criado pela **Bosch** em **1983** para **carros**; **dois fios diferenciais** (CAN_H e CAN_L), **multi-mestre**, orientado a **mensagens**
- Sinalização **diferencial** cancela ruído; **terminação de 120 Ω** nas pontas evita reflexões
- Estados **dominante (0)** e **recessivo (1)**: o dominante sempre vence (wired-AND)
- O **ID** identifica o **conteúdo** da mensagem e define a **prioridade** (menor ID = maior prioridade)
- **Arbitragem bit a bit não destrutiva**: o nó que perde se retira sem corromper o vencedor — **sem colisão, sem perda de dados**
- **CRC de 15 bits**, **ACK**, **bit stuffing**, quadros de erro e **retransmissão automática**; **nós defeituosos se isolam** (bus-off)
- Quadro de **até 8 bytes** (64 no **CAN FD**); **1 Mbit/s** a cerca de 40 m, mais lento em cabos mais longos
- Escolha ideal quando **confiabilidade importa mais que velocidade bruta**

---

# 🔗 Conexões com outras anotações

- [[Fundamentos de Protocolos de Hardware]] — diferencial, wired-AND, CRC, ACK, multi-mestre
- [[I2C]] — arbitragem simples por leitura do fio; o CAN a eleva com prioridade
- [[SPI]] — controladores CAN externos como o MCP2515 conversam por SPI
- [[RS-232]] — o outro padrão "robusto" de décadas atrás, mas ponto a ponto
- [[Comparativo e Escolha do Protocolo]] — quando escolher CAN
- [[Cabos Par Trançado (UTP e STP)]] — o mesmo princípio de par trançado e cancelamento de ruído
- [[Topologia de Rede]] — CAN é uma topologia de **barramento**
- [[Escalonamento em Sistemas em Tempo Real]] — prioridades e prazos: a mesma filosofia da arbitragem do CAN em hardware
- [[Socket]] — SocketCAN expõe o CAN como interface de rede no Linux
- [[Camada de Enlace - 2]] — o CAN define quadros, erros e acesso ao meio (equivalente à camada 2)
