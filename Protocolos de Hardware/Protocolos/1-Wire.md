---
tags:
  - hardware
  - hardware/protocolos
  - hardware/serial
  - hardware/sensores
source: "Vídeo 'Every Hardware Protocol Explained Simply in 10 Minutes!' + datasheet do DS18B20 e notas de aplicação do 1-Wire (Maxim/Analog Devices)"
---
# 1-Wire

📚 **Referência:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* (cap. 7 — One wire efficiency) | Pré-requisito: [[Fundamentos de Protocolos de Hardware]]

---

# 🪢 O que é o 1-Wire?

O nome diz tudo: **1-Wire** ("um fio") se comunica por **uma única linha de dados**. Foi desenvolvido pela **Dallas Semiconductor** (hoje parte da Maxim Integrated, que por sua vez pertence à Analog Devices).

> 💡 **1-Wire:** protocolo serial de **um único fio de dados** em que **vários dispositivos** compartilham a linha, cada um identificado por um **número único de 64 bits** gravado na fábrica. Pode **alimentar** os dispositivos pela própria linha de dados.

É o **mínimo absoluto de fiação**: sensores podem ser ligados em uma **corrente** (daisy-chain) com **um fio por onde passa tudo**.

```
                 +3,3 V
                    │
                 [4,7 kΩ]        ← resistor de pull-up (obrigatório)
                    │
 MESTRE ──── DQ ────┴──────┬───────────┬───────────┬── ... ─── (linha de dados)
 (ESP32)                   │           │           │
                        Sensor 1    Sensor 2    Sensor 3
                       (ID único)  (ID único)  (ID único)
 GND ──────────────────────┴───────────┴───────────┴────────── (terra em comum)
```

> ⚠️ **"Um fio" é uma simplificação.** O vídeo diz "um fio para dados e outro para energia, o mesmo fio" — na prática, **é preciso também o fio de terra (GND)** para fechar o circuito elétrico. Então há **2 condutores** (DQ + GND); o "1" se refere ao **fio de sinal**. Em montagens com alimentação separada, há um terceiro fio (VDD).

---

# 🆔 O ID único de 64 bits

Cada dispositivo 1-Wire vem com um **código ROM de 64 bits gravado a laser** na fábrica, que **nunca muda** e é **globalmente único**:

```
┌───────────────┬──────────────────────────────┬──────────────┐
│ Família (8 b) │      Nº de série (48 bits)   │  CRC (8 b)   │
│  ex.: 0x28    │  único no mundo              │  confere o ID│
└───────────────┴──────────────────────────────┴──────────────┘
      DS18B20 = 0x28         2⁴⁸ ≈ 281 trilhões de números
```

| Parte | Função |
|---|---|
| **Código de família** | Diz que tipo de chip é (0x28 = sensor de temperatura DS18B20; 0x01 = iButton) |
| **Número de série (48 bits)** | O que torna o chip **único** |
| **CRC (8 bits)** | Permite verificar se o ID chegou sem erro |

Isso resolve **sem endereços configuráveis** o problema de "quem é quem": como cada chip tem identidade própria, você pode ligar **dezenas na mesma linha** e o mestre os **descobre** e fala com **cada um individualmente**.

> 💡 **Código ROM:** a "identidade" gravada de fábrica. Pense no **chassi de um carro** ou no **IMEI de um celular**: ninguém escolhe, e não existem dois iguais.

## Como o mestre descobre os dispositivos?

Pelo comando **Search ROM** (0xF0): o mestre faz uma **busca binária bit a bit** pelos IDs presentes, e a própria linha (wired-AND) resolve as dúvidas. Assim ele **enumera** todos os dispositivos conectados, mesmo sem saber de antemão quais são.

Outros comandos de ROM: **Read ROM** (0x33, só para 1 dispositivo na linha), **Match ROM** (0x55: "fale **só** o dispositivo com este ID") e **Skip ROM** (0xCC: "todos ouçam" — usado quando há um único dispositivo).

---

# ⚡ Dados e energia no mesmo fio: alimentação parasita

> 💡 **Alimentação parasita (*parasitic power*):** o dispositivo possui um **capacitor interno** que se **carrega** enquanto a linha está em **nível alto** e usa essa energia armazenada enquanto a linha está em nível baixo. Assim, ele funciona sem um fio de alimentação (VDD) separado.

```
Linha DQ:  ‾‾‾‾‾\__/‾‾‾‾‾‾‾‾‾\______/‾‾‾‾‾‾‾‾
              ↓      ↑ carrega capacitor     ↓
           capacitor interno alimenta o chip enquanto a linha está baixa
```

Como a linha **fica em alto por padrão** (via pull-up), o capacitor se recarrega **quase o tempo todo**. Tem um limite: operações que consomem mais corrente (ex.: a medição de temperatura do DS18B20) exigem um **pull-up "forte"** (um transistor que liga a linha direto ao VDD) ou então a **alimentação normal** por um terceiro fio.

---

# 🕹️ Como os bits trafegam: por tempo de pulso

O 1-Wire é **assíncrono** e **não tem clock**. A informação é codificada pela **duração do pulso baixo**:

> 💡 **Linha em alto por padrão:** quando ninguém fala, o resistor de pull-up mantém a linha em **1**. Para enviar dados, o mestre (ou o escravo) **puxa a linha para 0 por um tempo muito preciso**. Um pulso **curto** significa uma coisa; um **longo** significa outra.

Cada bit ocupa uma **janela de tempo** (*time slot*) de pelo menos **60 µs**:

```
Escrever bit 1:   ‾‾\_‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾   pulso baixo CURTO (1–15 µs), depois solta
Escrever bit 0:   ‾‾\____________/‾‾‾‾‾‾‾   pulso baixo LONGO (≥ 60 µs)
Ler bit:          ‾‾\_/‾‾?‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾  mestre dá pulso curto, solta e amostra ~15 µs depois:
                                            se o escravo mantiver em 0 → bit 0; se soltar → bit 1
```

## O pulso de reset e presença

Toda conversa começa com um **reset**, que também serve para **detectar se há alguém na linha**:

```
Mestre:   ‾‾‾\______________________/‾‾‾‾‾‾‾‾‾‾   puxa para 0 por ≥ 480 µs, depois solta
Escravo:                             \_______/‾‾   responde puxando para 0 por 60–240 µs
                                      "presença"    (= "estou aqui!")
```

Se **nenhum** escravo responde, o mestre sabe que **não há nada conectado**.

## Sequência típica de uma conversa

1. **Reset + presença**
2. **Comando de ROM** (quem vai falar: Search / Match / Skip ROM)
3. **Comando de função** (o que fazer: "converter temperatura", "ler dados")
4. **Troca de dados**, bit a bit

---

# 🌡️ O dispositivo mais famoso: DS18B20

O vídeo chama o **DS18B20** de *"o dispositivo 1-Wire mais famoso já feito"* — um sensor de temperatura **barato, preciso** e com a vantagem de encadear **dezenas** na mesma linha.

| Característica | Valor |
|---|---|
| Faixa de medição | −55 °C a +125 °C |
| Precisão | ±0,5 °C (de −10 °C a +85 °C) |
| Resolução | configurável de **9 a 12 bits** (de 0,5 °C até **0,0625 °C**) |
| Tensão | 3,0 V a 5,5 V |
| Tempo de conversão (12 bits) | até **750 ms** |
| Formato | encapsulamento **TO-92** ou **sonda à prova d'água** |

## Como uma leitura funciona (um sensor na linha)

```
1. Reset  →  presença
2. Skip ROM        (0xCC)   "todos ouçam" (só há um)
3. Convert T       (0x44)   "meça a temperatura"
4. Espera até 750 ms
5. Reset  →  presença
6. Skip ROM        (0xCC)
7. Read Scratchpad (0xBE)   "me dê o resultado"
8. Lê 9 bytes → os 2 primeiros são a temperatura (16 bits)
```

Para converter o valor bruto (12 bits, complemento de dois — veja [[Bits e Bytes]]): **temperatura = valor × 0,0625 °C**. Exemplo: bruto `0x0191` = 401 → **401 × 0,0625 = 25,0625 °C**.

Com **vários sensores**, em vez de Skip ROM o mestre usa **Match ROM** + o ID do sensor desejado.

---

# 🧰 Onde o 1-Wire aparece

- **Sensores de temperatura** (DS18B20): estufas, aquários, geladeiras, **monitoramento HVAC**, sondas de processos industriais
- **Chaves iButton**: botões metálicos de aço que guardam um ID; usados em **controle de acesso** e **identificação**
- **Medidores de carga de bateria** (*fuel gauges*) dentro de notebooks
- **Memórias EEPROM** pequenas de identificação (ex.: ID de cartuchos de impressora e de acessórios)
- **Autenticação** de acessórios

---

# ⚖️ Pontos fortes e fracos

| ✅ Vantagens | ❌ Limitações |
|---|---|
| **Fiação mínima** (1 fio de sinal + GND) | **Muito lento** (~16 kbit/s no modo padrão) |
| **ID único** de fábrica: não precisa configurar endereços | Temporização **crítica** (µs) — difícil por software em sistemas multitarefa |
| Dezenas de dispositivos em **uma linha** | **Só half-duplex** |
| **Alimentação parasita** possível | Depende de **pull-up** correto e comprimento de cabo limitado |
| **Barato** e robusto para sensores simples | Poucos tipos de chips (basicamente Maxim/Analog Devices) |

Como o vídeo resume: *"você nunca o usaria em algo que exija transferência rápida; mas quando tudo que você precisa é uma leitura simples de sensor com o mínimo de fio possível, nada supera o 1-Wire."*

---

# ⚠️ 1-Wire não é o mesmo que o protocolo do DHT11

Quem usa **ESP32/Arduino** costuma confundir. O sensor **DHT11** (temperatura e umidade) **também usa um único fio de dados**, mas o protocolo dele é **proprietário e diferente**:

| | **1-Wire (DS18B20)** | **DHT11** |
|---|---|---|
| Fio de dados | 1 | 1 |
| ID/endereço | **ID de 64 bits**, vários na mesma linha | **Sem ID**: um sensor por pino |
| Início | Pulso de reset ≥ 480 µs | Mestre puxa para 0 por **≥ 18 ms** |
| Dados | Comandos e registradores | **40 bits** fixos: umidade (16) + temperatura (16) + checksum (8) |
| Bit 0/1 | Duração do pulso baixo | Duração do pulso **alto** (~26–28 µs = 0; ~70 µs = 1) |
| Biblioteca | `OneWire` + `DallasTemperature` | `DHT` |

São "primos" no conceito (um fio, tempo de pulso), mas **não compatíveis**.

---

# 🧪 No ESP32 e no Arduino

- O 1-Wire pode usar **qualquer GPIO** do ESP32 — **não há controlador dedicado**: a biblioteca **OneWire** gera os pulsos por software (bit-banging), e por isso a temporização é **crítica**.
- Bibliotecas comuns: `OneWire.h` + `DallasTemperature.h`.
- Ligações: **VDD** (3,3 V) — **DQ** (um GPIO, com **resistor de 4,7 kΩ** entre DQ e VDD) — **GND**.
- Sem o resistor de pull-up, o sensor retorna `-127 °C` ou "dispositivo não encontrado" — erro clássico.

---

# ✅ Resumo do Conceito

- **1-Wire**: **um fio de dados** (mais GND), criado pela **Dallas Semiconductor**; assíncrono, **half-duplex**, **muito lento** (~16 kbit/s)
- Cada dispositivo tem um **ID de 64 bits** único (8 de família + 48 de série + 8 de CRC) gravado na fábrica; o mestre **descobre** e **seleciona** cada um (Search/Match/Skip ROM)
- A linha fica em **alto** por pull-up; os bits são codificados pela **duração do pulso baixo** (curto = 1, ≥ 60 µs = 0)
- Toda conversa começa com **reset (≥ 480 µs)** e **pulso de presença** dos escravos
- **Alimentação parasita**: o chip se alimenta pelo próprio fio de dados via capacitor interno
- Dispositivo mais famoso: **DS18B20** (sensor de temperatura, −55 a +125 °C, 9–12 bits); também iButton e medidores de bateria
- **Não é compatível** com o protocolo de fio único do **DHT11**
- Ideal para **leituras simples** de sensores com o **mínimo de fiação**

---

# 🔗 Conexões com outras anotações

- [[Fundamentos de Protocolos de Hardware]] — open-drain, pull-up, ID único, CRC
- [[I2C]] — também usa open-drain e pull-up; endereça por número configurável, enquanto o 1-Wire usa ID de fábrica
- [[UART]] — outro protocolo assíncrono; 1-Wire usa **duração de pulso**, UART usa **baud rate**
- [[CAN]] — também baseado em wired-AND, só que diferencial e muito mais robusto
- [[Comparativo e Escolha do Protocolo]] — quando escolher 1-Wire
- [[Bits e Bytes]] — o valor de 16 bits do DS18B20 e sua conversão
- [[Dispositivos de IO]] — leitura por *polling* e a importância do tempo (temporização crítica por software)
- [[Escalonamento em Sistemas em Tempo Real]] — por que temporizações de microssegundos sofrem em sistemas multitarefa
