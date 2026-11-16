---
tags:
  - hardware
  - hardware/protocolos
  - hardware/serial
source: "Vídeo 'Every Hardware Protocol Explained Simply in 10 Minutes!' + datasheets de UART (ESP32, 16550) e práticas comuns de uso"
---
# UART

📚 **Referência:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* (cap. 2 — UART serial communication) | Pré-requisito: [[Fundamentos de Protocolos de Hardware]]

---

# 📞 O que é o UART?

**UART** significa ***Universal Asynchronous Receiver/Transmitter*** — Transmissor/Receptor Assíncrono Universal. É um dos jeitos mais antigos e mais simples de dois dispositivos conversarem, e continua sendo **o primeiro protocolo que a maioria dos engenheiros aprende**.

> 💡 **UART:** circuito (dentro do chip) que converte bytes em uma sequência de bits enviada **um de cada vez por um fio** (e faz o inverso na recepção). Usa **dois fios de dados**, **não tem clock** e liga **exatamente dois** dispositivos.

Cada palavra do nome conta uma parte da história:

| Palavra | Significado |
|---|---|
| **Universal** | Funciona com várias velocidades e formatos de quadro, configuráveis |
| **Asynchronous** | **Não há fio de clock** compartilhado — os dois lados combinam a velocidade antes |
| **Receiver/Transmitter** | Cada lado tem uma parte que envia e outra que recebe |

---

# 🔌 A ligação física: dois fios (mais o terra)

```
   Dispositivo A                    Dispositivo B
  ┌──────────┐                    ┌──────────┐
  │       TX ├───────────────────►┤ RX       │
  │       RX ├◄───────────────────┤ TX       │
  │      GND ├────────────────────┤ GND      │
  └──────────┘                    └──────────┘
```

- **TX (Transmit):** pino por onde o dispositivo **envia**.
- **RX (Receive):** pino por onde o dispositivo **recebe**.
- **A ligação é cruzada:** o **TX de um vai no RX do outro**. É o erro mais comum de iniciantes ligar TX com TX — nada funciona.
- **GND em comum:** os dois dispositivos precisam compartilhar o **terra**, senão não têm uma referência de tensão em comum para saber o que é "1" e o que é "0".

Como há um fio para cada sentido, o UART é **full-duplex**: pode enviar e receber **ao mesmo tempo**. E como liga só dois dispositivos, é **ponto a ponto**.

---

# ⏱️ Assíncrono: a questão do *baud rate*

Sem fio de clock, como o receptor sabe **quando** olhar o fio? Os dois lados **combinam a velocidade antes** de começar.

> 💡 **Baud rate:** a velocidade da comunicação, em símbolos por segundo. No UART, cada símbolo é 1 bit, então baud rate = **bits por segundo**. Valores comuns: **9600** e **115200**.

Analogia do vídeo: é como duas pessoas combinando falar no **mesmo ritmo** antes de uma ligação. Se uma fala rápido demais e a outra escuta devagar demais, a conversa se perde.

**Quanto dura cada bit?** O tempo de um bit é o inverso do baud rate:

```
115200 baud → 1 bit dura 1 / 115200 ≈ 8,68 µs
9600   baud → 1 bit dura 1 / 9600   ≈ 104 µs
```

> ⚠️ **Os dois lados precisam usar exatamente o mesmo baud rate.** Se um estiver em 9600 e o outro em 115200, o receptor mede o tempo errado e lê **lixo** (caracteres estranhos no monitor serial são o sintoma clássico).

---

# 📦 O quadro (*frame*) UART: como um byte é enviado

Sem clock, o receptor precisa saber **onde começa** cada byte. Por isso o UART "embrulha" cada byte com bits extras:

```
Linha em repouso (idle) = nível ALTO (1)

   ┌─────────────────── 1 quadro (frame) ───────────────────┐
   │ START │ D0 D1 D2 D3 D4 D5 D6 D7 │ (PARIDADE) │ STOP │
   │  (0)  │   8 bits de dados       │ (opcional) │ (1)  │
```

| Parte | Função |
|---|---|
| **Idle (repouso)** | A linha fica em **1** (alta) quando ninguém fala |
| **Start bit** | Sempre **0**. A descida de 1 → 0 avisa "um byte vai começar" e o receptor **inicia seu cronômetro** |
| **Bits de dados** | Normalmente **8** (podem ser 5 a 9). Enviados com o **bit menos significativo primeiro (LSB first)** |
| **Bit de paridade** | **Opcional.** Detecta erro de 1 bit |
| **Stop bit(s)** | **1 ou 2** bits em **1**. Devolvem a linha ao repouso e marcam o fim do quadro |

> 💡 **LSB first:** o bit **menos** significativo (o da direita, valor 1) é enviado primeiro. Veja [[Bits e Bytes]] para relembrar o que é o bit menos significativo.

## Exemplo: enviando a letra `A`

A letra `A` em ASCII é `0x41` = `0100 0001`. Como o UART manda o LSB primeiro, os bits saem na ordem **1, 0, 0, 0, 0, 0, 1, 0**:

```
Bit:    IDLE START D0   D1   D2   D3   D4   D5   D6   D7   STOP
Valor:   1     0    1    0    0    0    0    0    1    0     1
Linha: ‾‾‾‾ ____ ‾‾‾‾ ____ ____ ____ ____ ____ ‾‾‾‾ ____ ‾‾‾‾
              ↑
        a descida inicia o cronômetro do receptor
```

## A notação **8N1**

Você verá configurações como **115200 8N1**. Leia assim:

```
115200 8 N 1
  │    │ │ └── 1 stop bit
  │    │ └──── N = sem paridade (None)   [outras: E = Even/par, O = Odd/ímpar]
  │    └────── 8 bits de dados
  └─────────── baud rate
```

**8N1** é a configuração mais usada do mundo.

## Quanto "custa" cada byte? (overhead)

Em 8N1, cada byte útil ocupa **10 bits** na linha (1 start + 8 dados + 1 stop):

```
115200 baud ÷ 10 bits por byte = 11.520 bytes por segundo (≈ 11,25 KB/s)
9600 baud   ÷ 10 bits por byte =    960 bytes por segundo
```

Ou seja, **20% da banda** é "overhead" de enquadramento. Isso é o preço de não ter clock.

---

# 👁️ Como o receptor lê os bits (amostragem)

O receptor não olha o fio o tempo todo: ele **amostra** (mede) o nível **no meio de cada bit**, onde o sinal é mais estável.

```
              ┌─ amostra no meio do bit
              ▼
        START        D0
Linha: ‾‾‾‾‾|_________|‾‾‾‾‾‾‾‾‾|
             ↑          ↑
        detecta a    espera 1,5 bit e lê D0,
        descida      depois 1 bit para cada próximo
```

Para isso o UART usa **sobreamostragem (oversampling)**, tipicamente **16×**: o clock interno do receptor roda 16 vezes mais rápido que o baud rate, e ele conta 8 ciclos após a descida do start para cair no centro do bit.

> 💡 **Tolerância de relógio:** como cada lado tem seu próprio oscilador, os dois nunca são **perfeitamente** iguais. O erro **acumula** ao longo do quadro. Como o receptor lê o último bit (stop) depois de ~9,5 tempos de bit, a diferença total entre os relógios precisa ficar abaixo de ~5%. Na prática, projetistas buscam **menos de 2–3%** — por isso cristais (quartzo) são preferidos a osciladores RC internos em comunicações sérias.

---

# 🚦 Erros que o UART consegue detectar

O UART **não corrige** nada, mas o hardware sinaliza algumas falhas:

| Erro | O que significa |
|---|---|
| **Framing error** | O receptor esperava o stop bit (1) e achou 0 — baud rate errado ou ruído |
| **Parity error** | A paridade não bate — algum bit foi corrompido |
| **Overrun error** | Chegou um byte novo antes de o processador ler o anterior — o buffer encheu |
| **Break** | A linha ficou em 0 por mais que um quadro inteiro (usado como sinal especial) |

> 💡 **Buffer FIFO:** pequena fila dentro do UART que guarda os bytes recebidos até a CPU buscá-los, evitando overrun. É o mesmo princípio de "fila entre produtor e consumidor" que vimos em [[Semáforos]] (produtor-consumidor).

---

# 🚧 Controle de fluxo (opcional)

E se o receptor for mais lento que o transmissor? Existem dois mecanismos:

- **Por hardware (RTS/CTS):** dois fios extras. O receptor ergue **RTS** ("estou pronto") e o transmissor só envia quando enxerga **CTS** ativo.
- **Por software (XON/XOFF):** o receptor envia caracteres especiais pelo próprio canal de dados para mandar o outro **parar** (XOFF) ou **continuar** (XON).

Na maioria dos projetos de microcontrolador não se usa nenhum dos dois.

---

# ⚖️ Pontos fortes e fracos

| ✅ Vantagens | ❌ Limitações |
|---|---|
| **Extremamente simples**: dois fios, define o baud rate e pronto | **Ponto a ponto**: só dois dispositivos |
| Sem clock, sem endereçamento, sem hardware extra | **Sem correção de erro** embutida (só paridade opcional) |
| Full-duplex | **Mais lento** que SPI e I2C em modo rápido |
| Existe em praticamente todo microcontrolador | Os dois lados **precisam** do mesmo baud rate |
| Padrão universal para debug | Alcance curto sem um padrão elétrico melhor (veja [[RS-232]]) |

Como o vídeo resume: *"quando você só precisa que dois dispositivos conversem e a simplicidade importa mais que a velocidade, o UART ainda é a primeira escolha."*

---

# 🧰 Onde o UART aparece

- **Placas Arduino e ESP32** — o `Serial` do monitor serial é um UART ligado a um conversor USB
- **Módulos GPS** — enviam frases de texto (NMEA) por UART
- **Adaptadores Bluetooth** (ex.: HC-05) e módulos Wi-Fi/GSM
- **Consoles de debug** — roteadores, Raspberry Pi e placas embarcadas expõem um terminal por UART
- **Comunicação entre microcontroladores**

## 🔗 UART × RS-232: qual a diferença?

É uma confusão muito comum:

> 💡 **UART** é o **circuito e o formato do quadro** (start, dados, stop). Trabalha com tensões **de microcontrolador** (0 V e 3,3 V ou 5 V).
> **[[RS-232]]** é um **padrão elétrico** (e de conectores) que transporta esse **mesmo quadro**, só que com tensões **muito maiores** (±3 a ±15 V) para aguentar cabos mais longos e mais ruído.

Em um computador antigo, o UART gerava o quadro e um chip conversor (tipo MAX232) elevava as tensões para o padrão RS-232.

---

# 🧪 No ESP32 e no Arduino

- O **ESP32 tem 3 controladores UART** (UART0, UART1, UART2). O **UART0** costuma ser o usado pelo monitor serial da Arduino IDE e pelos logs de inicialização (pinos GPIO1 = TX e GPIO3 = RX nas placas DevKit comuns).
- No código: `Serial.begin(115200);` define o baud rate, e `Serial.println("oi");` envia pelo UART.
- Quando o monitor serial mostra **caracteres estranhos**, quase sempre o baud rate do monitor (canto da janela) é **diferente** do `Serial.begin(...)` do código.
- ⚠️ O ESP32 trabalha em **3,3 V**. Ligá-lo diretamente a um UART de 5 V (como um Arduino Uno) pede um conversor de nível; ligar a um **[[RS-232]] de verdade (±12 V) queima o pino**.

---

# ✅ Resumo do Conceito

- **UART** = *Universal Asynchronous Receiver/Transmitter*: dois fios (TX e RX), **sem clock**, **ponto a ponto**, **full-duplex**
- Os lados combinam o **baud rate** (9600, 115200...) antes; se não baterem, o dado vira lixo
- Cada byte é enviado em um **quadro**: start (0) + 5–9 bits de dados (LSB primeiro) + paridade opcional + 1–2 stop (1)
- **8N1** = 8 dados, sem paridade, 1 stop → 10 bits por byte → a 115200 baud, ≈ 11,5 KB/s
- O receptor amostra no **meio do bit**; a diferença de relógio entre os lados precisa ser pequena (idealmente < 2–3%)
- Detecta **framing**, **paridade** e **overrun**, mas **não corrige** erros
- Vantagem: **simplicidade**; limitações: só 2 dispositivos, sem correção de erro, velocidade moderada
- **UART ≠ RS-232**: UART é o formato do quadro; RS-232 é o padrão elétrico (tensões maiores) que o transporta

---

# 🔗 Conexões com outras anotações

- [[Fundamentos de Protocolos de Hardware]] — conceitos de síncrono/assíncrono, baud, paridade e full-duplex usados aqui
- [[RS-232]] — o mesmo quadro UART com níveis elétricos de longa distância
- [[SPI]] e [[I2C]] — as alternativas síncronas, mais rápidas, para vários dispositivos
- [[Comparativo e Escolha do Protocolo]] — quando escolher UART
- [[Dispositivos de IO]] — o UART é um controlador de I/O: o SO usa driver, interrupção (byte chegou) ou DMA
- [[Semáforos]] — o buffer FIFO do UART é um caso de produtor-consumidor
- [[Bits e Bytes]] — bit menos significativo e representação de bytes
- [[Arquivos]] — no Linux, o UART aparece como arquivo especial `/dev/ttyS0` ou `/dev/ttyUSB0`
