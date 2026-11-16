---
tags:
  - hardware
  - hardware/protocolos
  - hardware/comparativo
source: "Vídeo 'Every Hardware Protocol Explained Simply in 10 Minutes!' + síntese das notas deste MOC"
---
# Comparativo e Escolha do Protocolo

📚 **Referência:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* | Síntese de [[UART]], [[SPI]], [[I2C]], [[CAN]], [[RS-232]] e [[1-Wire]]

---

# 🧭 A grande tabela

| | [[UART]] | [[SPI]] | [[I2C]] | [[CAN]] | [[RS-232]] | [[1-Wire]] |
|---|---|---|---|---|---|---|
| **Fios de sinal** | 2 (TX, RX) | 4 (SCLK, MOSI, MISO, CS) | 2 (SDA, SCL) | 2 (CAN_H, CAN_L) | 2+ (TXD, RXD + controle) | 1 (DQ) |
| **Clock** | ❌ Assíncrono | ✅ Síncrono | ✅ Síncrono | ❌ (ressincroniza por bit) | ❌ Assíncrono | ❌ (tempo de pulso) |
| **Direção** | Full-duplex | Full-duplex | Half-duplex | Half-duplex | Full-duplex | Half-duplex |
| **Nº de dispositivos** | 2 | 1 mestre + N (1 CS cada) | até ~112 (7 bits) | dezenas de nós | 2 | dezenas |
| **Como escolhe o destino** | — | Chip Select | Endereço de 7 bits | ID da mensagem | — | ID de 64 bits |
| **Mestre** | Nenhum (iguais) | Único | Um ou vários | **Multi-mestre** | Nenhum (DTE/DCE) | Único |
| **Velocidade típica** | 9,6 k – 115,2 kbit/s (até alguns Mbit/s) | **Dezenas de Mbit/s** | 100 k – 400 kbit/s (até 3,4 M) | até 1 Mbit/s (FD: 8 M) | até ~115 kbit/s | ~16 kbit/s |
| **Distância** | Curta (placa/cabo curto) | Muito curta | Curta (placa) | **Até centenas de metros** | ~15 m | Dezenas de metros (limitado) |
| **Detecção de erro** | Paridade (opcional) | ❌ Nenhuma | ACK | **CRC + ACK + mais** | Paridade (opcional) | CRC do ID |
| **Sinal** | Single-ended | Single-ended | Single-ended | **Diferencial** | Single-ended (±12 V) | Single-ended |
| **Tensão lógica** | 3,3 V / 5 V | 3,3 V / 5 V | 3,3 V / 5 V | ~2 V diferencial | **±3 a ±15 V** | 3 V – 5,5 V |
| **Complexidade** | ⭐ Mínima | ⭐⭐ Baixa | ⭐⭐ Média | ⭐⭐⭐⭐ Alta | ⭐⭐ Baixa (+ conversor) | ⭐⭐ Média (tempo crítico) |
| **Ano / origem** | Décadas atrás (genérico) | Motorola, anos 80 | Philips, início dos 80 | Bosch, 1983 | EIA, **1960** | Dallas Semiconductor |

---

# 🪄 O que cada um faz de melhor — em uma frase

| Protocolo | Especialidade | Analogia do vídeo |
|---|---|---|
| [[UART]] | **Simplicidade**: dois dispositivos, sem complicação | Duas pessoas combinando falar no mesmo ritmo |
| [[SPI]] | **Velocidade** com poucos dispositivos | Orquestra seguindo o maestro |
| [[I2C]] | **Muitos sensores** com 2 fios | Grupo de chat onde só responde quem é chamado |
| [[CAN]] | **Confiabilidade** em ambiente crítico | Sistema de prioridade que resolve conflitos sozinho |
| [[RS-232]] | **Compatibilidade** com equipamento legado | "Se funciona e não quebra, ninguém troca" |
| [[1-Wire]] | **Fiação mínima** para sensores simples | Um fio faz tudo |

---

# 🌳 Como escolher: árvore de decisão

```
                    Qual é a sua necessidade?
                              │
     ┌────────────┬───────────┼────────────┬──────────────┐
     ▼            ▼           ▼            ▼              ▼
  Falar com   Muita       Vários      Ambiente        Sensor de
  UM só       velocidade  sensores    ruidoso /       temperatura
  dispositivo (display,   lentos      crítico         com o mínimo
     │        SD, flash)  com poucos  (carro,         de fios
     │            │       fios         indústria)         │
     │            │           │            │              │
     ▼            ▼           ▼            ▼              ▼
   UART          SPI         I2C          CAN          1-Wire
     │
     └─ equipamento legado ou cabo longo? → RS-232 (ou RS-485)
```

## Perguntas rápidas para decidir

| Se a resposta é "sim"... | ...pense em |
|---|---|
| "Preciso só de 2 dispositivos e quero que funcione hoje." | **[[UART]]** |
| "O chip é um display, cartão SD ou memória flash — tem que ser rápido." | **[[SPI]]** |
| "Tenho 5 sensores e poucos pinos livres." | **[[I2C]]** |
| "Falha de comunicação pode causar acidente ou prejuízo grave." | **[[CAN]]** |
| "O equipamento é de fábrica, CNC ou instrumento antigo." | **[[RS-232]]** |
| "Quero vários termômetros em um único cabo." | **[[1-Wire]]** (DS18B20) |

---

# 🧮 Resumo por "moeda de troca"

Lembrando do que vimos em [[Fundamentos de Protocolos de Hardware]] — cada protocolo paga por um benefício:

| Benefício | Quem ganha | O que paga |
|---|---|---|
| **Velocidade** | SPI | Mais fios (CS por escravo), sem ACK, alcance curto |
| **Poucos fios** | I2C e 1-Wire | Menos velocidade, pull-ups, temporização cuidadosa |
| **Simplicidade** | UART | Só 2 dispositivos, sem correção de erro |
| **Robustez** | CAN e RS-232 | Complexidade (CAN) ou velocidade e ponto a ponto (RS-232) |

---

# 🧰 Combinações comuns em um projeto real

Raramente se usa **um só protocolo**. Um mesmo microcontrolador costuma falar **vários** ao mesmo tempo, cada um para o que faz melhor:

```
                        ┌───────────────────────────────────┐
                        │            ESP32                  │
                        └──┬─────────┬─────────┬─────────┬──┘
                       UART│      SPI│      I2C│   1-Wire│
                           │         │         │         │
                  ┌────────┴──┐ ┌────┴────┐ ┌──┴─────┐ ┌─┴──────────┐
                  │ GPS /     │ │ Cartão  │ │ OLED + │ │ DS18B20    │
                  │ Monitor   │ │ SD /    │ │ BME280 │ │ (vários na │
                  │ serial    │ │ Display │ │ + RTC  │ │ mesma linha│
                  └───────────┘ └─────────┘ └────────┘ └────────────┘
```

---

# 🧪 Na prática: ESP32 e Arduino

| Protocolo | Suporte no ESP32 | Biblioteca Arduino | Pinos padrão (DevKit comum) |
|---|---|---|---|
| UART | 3 controladores | `Serial`, `Serial1`, `Serial2` | UART0: TX=GPIO1, RX=GPIO3 |
| SPI | 2 controladores livres (VSPI, HSPI) | `SPI.h` | VSPI: SCLK=18, MISO=19, MOSI=23, CS=5 |
| I2C | 2 controladores | `Wire.h` | SDA=21, SCL=22 |
| CAN | Controlador **TWAI** (precisa de transceptor) | `ESP32-TWAI-CAN` ou ESP-IDF | configurável |
| RS-232 | Via UART + **MAX3232** | `Serial2` + conversor | configurável |
| 1-Wire | Qualquer GPIO (software) | `OneWire` + `DallasTemperature` | qualquer GPIO |

> ⚠️ Os pinos padrão podem variar entre fabricantes de placas — confira o *pinout* da sua.
> ⚠️ O ESP32 opera em **3,3 V**. Para módulos de 5 V use **conversor de nível**; para RS-232 de verdade (±12 V) use **MAX3232**.

## E o DHT11?

O **DHT11** usa **um fio de dados**, mas com um protocolo **próprio** (não é 1-Wire). Em termos de classificação neste MOC, é um **protocolo proprietário de fio único**. Detalhes em [[1-Wire]] (seção "1-Wire não é o mesmo que o protocolo do DHT11").

---

# ✅ Resumo do Conceito

- Os seis protocolos respondem às mesmas perguntas (fios, tensão, clock, mensagem, quem fala) com **respostas diferentes**
- **UART** = simples, 2 dispositivos; **SPI** = rápido, 4 fios + 1 CS por escravo; **I2C** = 2 fios, endereçado, mais lento; **CAN** = robusto, prioridade, para carros e indústria; **RS-232** = tensões altas, legado; **1-Wire** = um fio, ID de 64 bits, sensores simples
- **Mais velocidade → mais fios** (SPI); **menos fios → menos velocidade** (I2C, 1-Wire); **mais robustez → mais complexidade** (CAN)
- Um projeto real **combina vários**, usando cada um onde ele é melhor
- Regra prática: **UART** para debug e dois dispositivos; **SPI** para displays/SD; **I2C** para sensores; **CAN** para ambientes críticos; **RS-232** para falar com equipamento antigo; **1-Wire** para DS18B20
- Todos são **seriais**: bits um após o outro — o mesmo princípio dos [[Barramentos]] modernos (PCIe, USB, SATA)

---

# 🔗 Conexões com outras anotações

- [[Fundamentos de Protocolos de Hardware]] — o vocabulário usado na tabela
- [[Barramentos]] — a versão "de computador" desta discussão: PCIe, DDR, USB, SATA
- [[Dispositivos de IO]] — drivers, interrupções e DMA para operar esses controladores
- [[Processadores]] — o microcontrolador integra vários controladores desses
- [[Camada Física - 1]] e [[Camada de Enlace - 2]] — as mesmas questões (sinal, quadro, endereço, erro) em redes
- [[Protocolos e Portas de Comunicação]] — protocolos de **rede**; comparar com os de **hardware** ajuda a ver o que muda de nível
- [[Hardware de Proteção]] — outro exemplo de hardware dando suporte ao SO
