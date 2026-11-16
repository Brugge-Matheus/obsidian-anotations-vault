---
tags:
  - hardware
  - hardware/protocolos
  - hardware/serial
source: "Vídeo 'Every Hardware Protocol Explained Simply in 10 Minutes!' + especificação de fato do SPI (Motorola) e datasheets de microcontroladores"
---
# SPI

📚 **Referência:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* (cap. 3 — SPI high-speed interface) | Pré-requisito: [[Fundamentos de Protocolos de Hardware]]

---

# 🎼 O que é o SPI?

**SPI** significa ***Serial Peripheral Interface*** (Interface Periférica Serial). Foi criado pela Motorola na década de 1980 e é, na prática, um **padrão de fato**: não existe um documento oficial único, e cada fabricante descreve o seu no datasheet — mas todos seguem a mesma ideia.

> 💡 **SPI:** protocolo serial **síncrono**, **full-duplex**, com **um mestre** e um ou mais escravos. Usa **quatro fios** e um **clock** controlado pelo mestre, o que o torna **muito mais rápido** que o [[UART]] e o [[I2C]].

Analogia do vídeo: é como uma **orquestra com maestro**. O maestro (mestre) marca o tempo, e todos tocam **exatamente no mesmo compasso** — não há margem para "perder o ritmo".

---

# 🔌 Os quatro fios

```
         MESTRE                              ESCRAVO
     ┌─────────────┐                      ┌─────────────┐
     │       SCLK  ├─────────────────────►┤ SCLK        │
     │       MOSI  ├─────────────────────►┤ MOSI        │
     │       MISO  ├◄─────────────────────┤ MISO        │
     │        CS   ├─────────────────────►┤ CS          │
     └─────────────┘                      └─────────────┘
```

| Sinal | Nome por extenso | Direção | Função |
|---|---|---|---|
| **SCLK** (ou SCK, CLK) | *Serial Clock* | Mestre → Escravo | O **relógio**. Cada pulso desloca **1 bit** |
| **MOSI** | *Master Out, Slave In* | Mestre → Escravo | Dados **do mestre para o escravo** |
| **MISO** | *Master In, Slave Out* | Escravo → Mestre | Dados **do escravo para o mestre** |
| **CS** (ou SS) | *Chip Select* (*Slave Select*) | Mestre → Escravo | "Você aí: **fale comigo agora**" |

> ⚠️ **Atenção ao significado de MOSI:** é *Master **Out**, Slave **In*** — a saída do **mestre** é a entrada do **escravo**. (A transcrição automática do vídeo escreveu "slave in" de forma truncada, mas a sigla completa é essa.)

> 💡 **Nomenclatura moderna:** como a indústria está abandonando "master/slave", você também vai ver **COPI** (*Controller Out, Peripheral In* = MOSI), **CIPO** (*Controller In, Peripheral Out* = MISO) e **PICO/POCI** em alguns fabricantes. Mesma coisa, nomes novos.

## O fio CS (Chip Select)

O CS é o que permite ter **vários escravos**. Fica normalmente em **nível alto** (escravo ignorando o barramento) e o mestre o **puxa para baixo (0)** para "acordar" aquele escravo e iniciar a conversa. Por isso diz-se que o CS é **ativo em nível baixo** (e costuma ser escrito `/CS` ou `CS̅`).

```
Vários escravos: SCLK, MOSI e MISO são COMPARTILHADOS, mas cada escravo tem seu CS:

              ┌────────────────────────── SCLK ──┬───────────┬───────────┐
   MESTRE ────┼────────────────────────── MOSI ──┤           │           │
              ├────────────────────────── MISO ◄─┤           │           │
              │                                ESCRAVO 1  ESCRAVO 2  ESCRAVO 3
              ├─── CS1 ───────────────────────────►┘           │           │
              ├─── CS2 ─────────────────────────────────────────►           │
              └─── CS3 ───────────────────────────────────────────────────►
```

Escravos **não selecionados** deixam o MISO em **alta impedância** (como se "desconectassem" do fio) para não atrapalhar quem está falando.

---

# 🔁 Como o dado trafega: o registrador de deslocamento

O SPI é **full-duplex de verdade**: a cada pulso de clock, o mestre **envia 1 bit e recebe 1 bit ao mesmo tempo**. Isso é possível porque mestre e escravo funcionam como **dois registradores de deslocamento ligados em anel**:

> 💡 **Registrador de deslocamento (shift register):** conjunto de 8 (ou mais) flip-flops em que, a cada pulso de clock, **todos os bits andam uma posição**: o bit mais à esquerda **sai** e um bit novo **entra** pela direita.

```
   MESTRE (8 bits)                      ESCRAVO (8 bits)
 ┌─┬─┬─┬─┬─┬─┬─┬─┐                    ┌─┬─┬─┬─┬─┬─┬─┬─┐
 │7│6│5│4│3│2│1│0│ ──► MOSI ────────► │7│6│5│4│3│2│1│0│
 └─┴─┴─┴─┴─┴─┴─┴─┘                    └─┴─┴─┴─┴─┴─┴─┴─┘
   ▲                                              │
   └────────────────── MISO ◄─────────────────────┘

Após 8 pulsos de clock: o conteúdo dos dois registradores foi TROCADO.
```

Isso gera um detalhe importante: **toda leitura é também uma escrita**. Para **ler** um sensor, o mestre normalmente envia um byte "de enchimento" (dummy, como `0x00`) só para gerar os 8 pulsos de clock que empurram o dado do escravo para fora.

---

# ⚙️ Os quatro modos do SPI (CPOL e CPHA)

Este é o ponto que mais confunde iniciantes — e **a causa nº 1 de "o SPI não funciona"**. Como os dois lados usam o mesmo clock, eles precisam concordar em **dois detalhes**:

> 💡 **CPOL (*Clock Polarity*):** qual o nível do clock **quando está em repouso**. `CPOL = 0` → repouso em **baixo**; `CPOL = 1` → repouso em **alto**.
> **CPHA (*Clock Phase*):** em **qual borda** o dado é lido (amostrado). `CPHA = 0` → lê na **primeira** borda do clock; `CPHA = 1` → lê na **segunda** borda.

Combinando os dois, existem **4 modos**:

| Modo | CPOL | CPHA | Clock em repouso | Dado lido (amostrado) em |
|---|---|---|---|---|
| **0** | 0 | 0 | Baixo | Borda de **subida** |
| **1** | 0 | 1 | Baixo | Borda de **descida** |
| **2** | 1 | 0 | Alto | Borda de **descida** |
| **3** | 1 | 1 | Alto | Borda de **subida** |

```
Modo 0 (o mais comum) — repouso baixo, lê na subida:

CS    ‾‾‾‾\____________________________/‾‾‾‾
SCLK  ____/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\____
MOSI  ‾‾‾‾X D7 X D6 X D5 X D4 X D3 X D2 X D1 X D0 X‾‾
            ↑    ↑    ↑          (o dado é lido na SUBIDA de cada pulso;
         lê aqui                  muda durante a descida)
```

O **datasheet do escravo** diz qual modo ele exige (ex.: "SPI mode 0"). Se o mestre usar outro modo, os bits chegam **deslocados** ou **invertidos** e o resultado é lixo.

Outros dois parâmetros também vêm do datasheet:

- **Ordem dos bits:** quase sempre **MSB first** (bit mais significativo primeiro), ao contrário do [[UART]].
- **Frequência máxima do clock** que o escravo aguenta.

---

# 🚀 Velocidade

O SPI é **muito rápido** porque:

1. Tem **clock dedicado** — não perde tempo com start/stop bits.
2. Usa saídas **push-pull** (veja [[Fundamentos de Protocolos de Hardware]]) — sobe e desce rápido, sem depender de resistor de pull-up.
3. Não tem endereçamento nem confirmação (ACK) a cada byte.

**Ordem de grandeza:** dezenas de megabits por segundo. O ESP32 chega a **80 MHz** de clock em SPI; displays e memórias flash costumam trabalhar entre 10 e 80 MHz. Para comparar: o [[UART]] a 115200 baud é ~0,1 Mbit/s; o [[I2C]] em modo rápido é 0,4 Mbit/s.

> 💡 **Variantes:** *Dual SPI* e **Quad SPI (QSPI)** usam 2 ou 4 linhas de dados ao mesmo tempo para multiplicar a velocidade — é como a memória flash dos microcontroladores e SSDs são lidas.

---

# ⚖️ Pontos fortes e fracos

| ✅ Vantagens | ❌ Limitações |
|---|---|
| **Muito rápido** (dezenas de Mbit/s) | **Um fio CS a mais para cada escravo** |
| **Full-duplex** de verdade | **4 fios** no mínimo (vs. 2 do I2C) |
| Hardware simples; sem endereço, sem start/stop | **Sem ACK**: o mestre não sabe se o escravo recebeu ou sequer existe |
| Sem sobrecarga de protocolo | **Sem detecção de erro** embutida |
| Sem resistores de pull-up | Distâncias curtas (placa, poucos cm a ~1 m) |
| Clock controlado pelo mestre = sem divergência de relógio | Não há padrão único (variações de modo, ordem de bits) |

## O problema dos fios do CS — por que "vira um pesadelo"

O vídeo destaca o custo: **cada escravo adicional exige 1 fio CS próprio**.

```
Dispositivos   Fios do SPI
     1       →  SCLK + MOSI + MISO + 1 CS  = 4 fios
     2       →  3 + 2 CS                   = 5 fios
     5       →  3 + 5 CS                   = 8 fios
    10       →  3 + 10 CS                  = 13 fios
```

Com muitos escravos, isso consome **pinos** do microcontrolador e **trilhas** na placa. É exatamente o problema que o [[I2C]] resolve com endereçamento.

> 💡 **Daisy-chain (corrente):** alguns chips permitem ligar o MISO de um escravo ao MOSI do próximo, formando um anel de registradores gigante com **um único CS**. Economiza fios, mas todos os dados passam por todos os chips (comum em LEDs e registradores de deslocamento).

---

# 🧰 Onde o SPI aparece

- **Cartões SD e microSD** (modo SPI)
- **Displays** TFT, OLED e e-paper — precisam mover muitos pixels rápido
- **Memória flash** externa (a que guarda o firmware do ESP32 conversa por SPI/QSPI)
- **Conversores analógico-digital (ADC/DAC)** de alta velocidade
- **Módulos de rádio** (LoRa, NRF24L01) e **controladores Ethernet** (W5500)
- Alguns **sensores** que movem muitos dados (acelerômetros em alta taxa)

---

# 🧪 No ESP32 e no Arduino

- O ESP32 tem controladores SPI de uso geral conhecidos como **VSPI (SPI3)** e **HSPI (SPI2)**; os outros (SPI0/1) são ocupados pela flash interna.
- Pinos padrão do VSPI nas placas DevKit comuns: **MOSI = GPIO23, MISO = GPIO19, SCLK = GPIO18, CS = GPIO5** (podem variar de placa para placa — confira o *pinout*).
- No Arduino: `#include <SPI.h>` e `SPI.begin();`. Para escolher o modo e a frequência: `SPI.beginTransaction(SPISettings(8000000, MSBFIRST, SPI_MODE0));`.
- ⚠️ Lembre do **nível lógico 3,3 V** do ESP32 ao ligar módulos de 5 V.

---

# ✅ Resumo do Conceito

- **SPI** = *Serial Peripheral Interface*: síncrono, full-duplex, **um mestre** e vários escravos, **4 fios**: SCLK, MOSI, MISO e CS
- **MOSI** = Master Out, Slave In; **MISO** = Master In, Slave Out; **CS** (ativo em baixo) escolhe o escravo
- Mestre e escravo são **registradores de deslocamento em anel**: a cada pulso 1 bit sai e 1 bit entra — **toda leitura é também uma escrita**
- Os **modos 0–3** (CPOL e CPHA) definem o nível do clock em repouso e em que borda o dado é lido; **mestre e escravo precisam usar o mesmo modo**
- É **muito rápido** (dezenas de Mbit/s), pois tem clock próprio, saídas push-pull e quase nenhum overhead
- Desvantagens: **1 fio CS por escravo**, sem ACK, sem detecção de erro, distância curta
- Melhor escolha quando se precisa de **velocidade bruta** e há **poucos dispositivos**: SD, displays, flash

---

# 🔗 Conexões com outras anotações

- [[Fundamentos de Protocolos de Hardware]] — síncrono × assíncrono, push-pull, mestre/escravo
- [[UART]] — o oposto em simplicidade: sem clock, 2 fios, mais lento
- [[I2C]] — resolve o problema dos fios CS com endereçamento (ao custo de velocidade)
- [[Comparativo e Escolha do Protocolo]] — quando escolher SPI
- [[Barramentos]] — o SPI é um "barramento" pequeno; o mesmo raciocínio de serial rápido × paralelo
- [[Memória]] e [[Armazenamento não Volátil]] — flash e SD, dispositivos de armazenamento que usam SPI
- [[Dispositivos de IO]] — controladores, DMA para mover grandes blocos pelo SPI sem ocupar a CPU
- [[Bits e Bytes]] — MSB (bit mais significativo) e LSB
