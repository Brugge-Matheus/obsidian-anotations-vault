---
tags:
  - hardware
  - hardware/protocolos
  - hardware/serial
source: "Vídeo 'Every Hardware Protocol Explained Simply in 10 Minutes!' + especificação I2C-bus (NXP/Philips, UM10204) e datasheets"
---
# I2C

📚 **Referência:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* (cap. 4 — I2C bus addressing) | Pré-requisito: [[Fundamentos de Protocolos de Hardware]]

---

# 📞 O que é o I2C?

**I2C** (lê-se "I-quadrado-C" ou "I-dois-C") vem de ***Inter-Integrated Circuit*** — comunicação **entre circuitos integrados**. Foi criado pela Philips (hoje NXP) no começo dos anos 1980 para ligar chips dentro de TVs usando **poucos fios**.

> 💡 **I2C:** barramento serial **síncrono** com **apenas dois fios** — **SDA** (dados) e **SCL** (clock) — onde **vários dispositivos** compartilham os mesmos fios. Cada dispositivo tem um **endereço** único, e o mestre "chama" aquele com quem quer falar.

Ele resolve o grande problema do [[SPI]]: no SPI, cada novo dispositivo custa **um fio extra** (o CS). No I2C, você pode ligar **10 ou 20 sensores usando sempre os mesmos 2 fios**.

---

# 🔌 Os dois fios

| Fio | Nome | Função |
|---|---|---|
| **SDA** | *Serial Data* | Os dados, nos **dois sentidos** (por isso é half-duplex) |
| **SCL** | *Serial Clock* | O clock, gerado pelo **mestre** |

```
              VCC (3,3 V ou 5 V)
               │         │
            [4,7 kΩ]  [4,7 kΩ]      ← resistores de PULL-UP (obrigatórios!)
               │         │
 ──────────────┴─────────┼──────────── SDA
 ──────────────────────┴────────────── SCL
      │         │         │         │
   ┌──┴──┐   ┌──┴──┐   ┌──┴──┐   ┌──┴──┐
   │MESTRE│   │ Sensor│   │ Display│   │  RTC │
   │      │   │ 0x48  │   │  0x3C  │   │ 0x68 │
   └──────┘   └───────┘   └────────┘   └──────┘
   (todos ligados nos mesmos dois fios + GND em comum)
```

## Por que os resistores de pull-up são obrigatórios?

Vimos em [[Fundamentos de Protocolos de Hardware]] a diferença entre *push-pull* e *open-drain*. O I2C usa **open-drain** nos dois fios:

- Qualquer dispositivo só consegue **puxar a linha para baixo (0)** ou **soltá-la**.
- Quando **todos soltam**, o **resistor de pull-up** leva a linha para **alto (1)**.
- Se **um ou mais** puxam para baixo, a linha fica em **0** (wired-AND).

Isso é o que permite vários dispositivos no mesmo fio **sem curto-circuito** — e também permite que dois mestres tentem falar ao mesmo tempo sem se destruírem (veja *arbitragem* adiante).

> ⚠️ **Sem pull-up, o I2C não funciona** — a linha nunca sobe para 1. Muitos módulos prontos (ex.: GY-521, OLED) já trazem os resistores na placa. Valor típico: **4,7 kΩ** (2,2 kΩ a 10 kΩ dependendo da velocidade e do comprimento).

---

# 🏷️ Endereçamento: "um número de telefone por dispositivo"

> 💡 **Endereço I2C:** número de **7 bits** fixo (ou parcialmente configurável) que identifica cada dispositivo no barramento. Todos "ouvem" o endereço enviado, mas **só o dono responde**.

Analogia do vídeo: é um **grupo de chat** onde todos estão na mesma linha, mas cada pessoa só fala quando o **seu nome é chamado**.

## Quantos endereços existem?

7 bits dão 2⁷ = **128** combinações (0x00 a 0x7F). O vídeo fala em "127 dispositivos", mas na prática:

- Alguns endereços são **reservados** pela especificação (0x00–0x07 e 0x78–0x7F: chamada geral, modo de alta velocidade, endereços de 10 bits...).
- Sobram cerca de **112 endereços utilizáveis** (de **0x08 a 0x77**).

O limite real também é **elétrico**: cada dispositivo adiciona capacitância ao fio, e o barramento permite no máximo **400 pF** no total. Em projetos reais raramente se passa de algumas dezenas de dispositivos.

> 💡 **Endereço 10 bits:** extensão da especificação para quem precisa de mais endereços; é pouco usada em hobby.

## Conflito de endereços

Dois sensores **iguais** (mesmo endereço fixo) **não podem** dividir o mesmo barramento — ambos responderiam juntos. Soluções:

- Usar o **pino de configuração de endereço** do chip (muitos têm pinos `ADDR`/`A0`/`A1` que mudam o endereço ligando-os a GND ou VCC);
- Usar um **multiplexador I2C** (como o TCA9548A), que cria "ramos" separados;
- Usar um **segundo barramento I2C** (o ESP32 tem dois).

---

# 📨 Como uma conversa I2C acontece

Tudo gira em torno de duas condições especiais, que são os **únicos momentos em que o SDA muda com o SCL em nível alto**:

> 💡 **START:** o mestre **puxa SDA para baixo enquanto SCL está alto**. Significa "vou falar; todos prestem atenção".
> **STOP:** o mestre **solta SDA para cima enquanto SCL está alto**. Significa "terminei; barramento livre".

Em qualquer outro momento, o SDA **só muda quando SCL está baixo** — assim, enquanto SCL está alto, o dado fica **estável** e pode ser lido.

```
       START                                                    STOP
SDA  ‾‾‾‾\___ A6 A5 A4 A3 A2 A1 A0 R/W ACK  D7 D6 ... D0 ACK  ___/‾‾‾‾
SCL  ‾‾‾‾‾‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾‾‾‾‾‾‾‾
```

## A estrutura de uma mensagem

```
┌───────┬───────────────┬─────┬─────┬───────────┬─────┬ ... ┬───────┐
│ START │ ENDEREÇO (7b) │ R/W │ ACK │ DADO (8b) │ ACK │     │ STOP  │
└───────┴───────────────┴─────┴─────┴───────────┴─────┴ ... ┴───────┘
```

1. **START** — o mestre toma o barramento.
2. **Endereço de 7 bits** — quem é o destinatário.
3. **Bit R/W** — **0** = o mestre vai **escrever**; **1** = o mestre quer **ler**.
4. **ACK** — o escravo endereçado **puxa SDA para baixo** no 9º pulso de clock para dizer "estou aqui".
5. **Bytes de dados** — cada byte de 8 bits é seguido de **um ACK** do receptor.
6. **STOP** — fim.

> 💡 **ACK / NACK:** no 9º pulso após cada byte, o **receptor** puxa SDA para baixo (**ACK = "recebi"**) ou deixa em alto (**NACK = "não recebi / pare de mandar"**). É o que o SPI **não tem**: o mestre fica sabendo se o dispositivo existe e respondeu.

## Exemplo completo: lendo a temperatura de um sensor no endereço `0x48`

```
Passo 1: START
Passo 2: envia 0x90   ← (endereço 0x48 << 1) | 0 (escrita)  → sensor responde ACK
Passo 3: envia 0x00   ← "quero acessar o registrador 0 (temperatura)" → ACK
Passo 4: START repetido (sem STOP no meio!)
Passo 5: envia 0x91   ← (endereço 0x48 << 1) | 1 (leitura)  → ACK
Passo 6: sensor envia o byte alto da temperatura → mestre responde ACK
Passo 7: sensor envia o byte baixo               → mestre responde NACK ("já chega")
Passo 8: STOP
```

O **START repetido** (*repeated start*) permite trocar de escrita para leitura **sem liberar o barramento**, evitando que outro mestre entre no meio da operação.

> 💡 **Registrador do dispositivo:** quase todo sensor I2C tem uma "tabela interna" de posições numeradas (registradores) com configurações e leituras. A comunicação típica é: *"escreva o número do registrador que quero; depois leia o conteúdo dele"*.

---

# 🐢 Velocidade

O I2C é **mais lento** que o SPI. Os modos da especificação são:

| Modo | Velocidade do clock |
|---|---|
| **Standard** | 100 kHz (≈ 100 kbit/s) |
| **Fast** | 400 kHz |
| **Fast-mode Plus** | 1 MHz |
| **High-speed** | 3,4 MHz |
| **Ultra-fast** | 5 MHz (só em um sentido) |

Por que é mais lento?

1. **Todos compartilham os mesmos dois fios** — mais dispositivos, mais capacitância e mais tráfego.
2. A linha só **sobe** pelo **resistor de pull-up** — um RC lento. Isso limita a frequência (diferente do push-pull do SPI, que "empurra" ativamente).
3. Cada byte tem **9 bits no barramento** (8 + ACK), e há endereço em cada mensagem.

O vídeo resume bem: *"a troca é velocidade — mas para a maioria dos projetos reais, a simplicidade de fiação vale mais."*

---

# 🧑‍🤝‍🧑 Recursos avançados

## Clock stretching (alongamento de clock)

> 💡 **Clock stretching:** um escravo lento pode **segurar o SCL em baixo** para dizer "espere, ainda estou processando". O mestre só continua quando o SCL subir de novo. É a forma do escravo controlar o ritmo.

## Multi-mestre e arbitragem

O I2C permite **mais de um mestre**. Se dois começam a falar juntos, a **arbitragem** decide pelo próprio fio: cada mestre confere se o nível que **colocou** no SDA é o que **leu**. Quem tentou enviar `1` mas leu `0` (porque o outro puxou para baixo) **perdeu** e se retira, sem corromper a mensagem do vencedor. Esse mesmo truque é aprimorado no [[CAN]].

---

# ⚖️ Pontos fortes e fracos

| ✅ Vantagens | ❌ Limitações |
|---|---|
| **Só 2 fios**, não importa quantos dispositivos | **Mais lento** que SPI |
| **Endereçamento embutido** | Máximo de ~112 endereços (7 bits) |
| **ACK** confirma recebimento | Conflito se dois chips têm o mesmo endereço |
| Suporta **multi-mestre** com arbitragem | **Precisa de resistores de pull-up** |
| Muito usado: milhares de chips compatíveis | **Alcance curto** (placa; capacitância ≤ 400 pF) |
| Hot-plug amigável a projetos de hobby | Half-duplex; sem detecção de erro por CRC |

---

# 🧰 Onde o I2C aparece

- **Sensores:** temperatura, umidade, pressão (BME280), luz, acelerômetro/giroscópio (MPU6050)
- **Displays OLED** pequenos (SSD1306)
- **RTC** — módulos de relógio de tempo real (DS3231)
- **Expansores de GPIO** (PCF8574, MCP23017), **ADCs** e **DACs** lentos
- **EEPROMs** pequenas e controladores de energia

O vídeo o descreve como *"provavelmente o protocolo mais popular para conectar vários sensores de baixa velocidade a um único microcontrolador"*.

---

# 🧪 No ESP32 e no Arduino

- O ESP32 tem **2 controladores I2C**. Pinos padrão nas placas DevKit comuns: **SDA = GPIO21** e **SCL = GPIO22** (qualquer GPIO pode ser remapeado).
- No Arduino: `#include <Wire.h>` e `Wire.begin();`.
- **Scanner I2C:** programa clássico que percorre os endereços 0x08–0x77 e imprime os que respondem com ACK — ótimo para **descobrir o endereço** de um módulo novo.
- Se o scanner não encontra nada: cheque **pull-ups**, **trocar SDA↔SCL**, **GND comum** e **tensão** (3,3 V × 5 V).

---

# ✅ Resumo do Conceito

- **I2C** (*Inter-Integrated Circuit*): síncrono, **2 fios** — **SDA** (dados) e **SCL** (clock) — com **vários dispositivos** no mesmo barramento
- Usa **open-drain + resistores de pull-up** (wired-AND): qualquer um pode puxar a linha para 0; é isso que permite compartilhar sem curto
- Cada dispositivo tem um **endereço de 7 bits** (≈ 112 utilizáveis, de 0x08 a 0x77); só o endereçado responde
- Mensagem: **START → endereço + R/W → ACK → bytes (cada um com ACK) → STOP**; o **ACK** confirma que o dispositivo existe e recebeu
- **START repetido** troca de escrita para leitura sem soltar o barramento
- Velocidades de **100 kHz**, **400 kHz**, **1 MHz** e acima — **mais lento que o SPI**, pois o fio é compartilhado e sobe por pull-up
- Suporta **clock stretching** e **multi-mestre** com arbitragem
- Melhor escolha para **muitos sensores lentos** com **poucos fios**

---

# 🔗 Conexões com outras anotações

- [[Fundamentos de Protocolos de Hardware]] — open-drain, pull-up, endereço e ACK explicados na base
- [[SPI]] — o concorrente rápido; I2C resolve o problema dos fios CS
- [[UART]] — ponto a ponto sem endereço; I2C é barramento com endereço
- [[CAN]] — leva a arbitragem por bits a outro nível (prioridade)
- [[Comparativo e Escolha do Protocolo]] — quando escolher I2C
- [[Dispositivos de IO]] — controlador I2C, driver, interrupções
- [[Endereço MAC]] — também é um "endereço único" para identificar um dispositivo, só que em redes
- [[Bits e Bytes]] — endereço de 7 bits = 2⁷ = 128 combinações
- [[Regiões Críticas]] — arbitragem do I2C é um problema de exclusão mútua resolvido em hardware
