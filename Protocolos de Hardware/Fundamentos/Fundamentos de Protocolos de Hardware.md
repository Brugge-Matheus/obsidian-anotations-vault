---
tags:
  - hardware
  - hardware/protocolos
  - hardware/fundamentos
source: "Vídeo 'Every Hardware Protocol Explained Simply in 10 Minutes!' + especificações e datasheets dos protocolos"
---
# Fundamentos de Protocolos de Hardware

📚 **Referência:** Vídeo *Every Hardware Protocol Explained Simply in 10 Minutes!* (jul/2026), aprofundado com as especificações de cada protocolo | Complementa [[Barramentos]] e [[Dispositivos de IO]]

---

# 🧩 O que é um protocolo de hardware?

Dentro de qualquer aparelho eletrônico existem dezenas de chips: o processador, a memória, sensores, displays, controladores de rede... Para trabalharem juntos, eles precisam **trocar informação**. Só que um chip é só um pedaço de silício com pinos metálicos — ele não "entende" o que o outro está dizendo a menos que os dois **combinem as regras da conversa antes**.

Essas regras são o **protocolo**.

> 💡 **Protocolo de hardware:** conjunto de regras que define **(1)** quantos fios (pinos) são usados e para quê, **(2)** quais tensões elétricas representam 0 e 1, **(3)** como os dois lados se sincronizam no tempo, **(4)** como os bits são organizados em mensagens e **(5)** quem fala e quando. Se os dois chips seguem o mesmo protocolo, conseguem se entender; se não, o que um envia é só ruído para o outro.

Pense em uma ligação telefônica: os dois precisam falar o **mesmo idioma**, no **mesmo ritmo**, e saber **quem fala primeiro**. Cada protocolo deste MOC é uma resposta diferente para essas mesmas perguntas.

---

# 🎯 Por que existem tantos protocolos?

Porque **não existe protocolo perfeito** — cada um troca uma coisa por outra. As quatro "moedas de troca" são:

```
        Velocidade
            ▲
            │
   Simplicidade ◄────┼────► Quantidade de fios
            │
            ▼
      Robustez (contra ruído e falhas)
```

| Se você quer... | Normalmente paga com... |
|---|---|
| Mais **velocidade** | Mais fios, ou circuitos mais complexos |
| Menos **fios** | Menos velocidade e mais lógica de endereçamento |
| Mais **robustez** | Mais complexidade e mais hardware |
| Mais **simplicidade** | Menos recursos (sem endereço, sem correção de erro) |

Por isso um carro usa [[CAN]] (robustez), um cartão SD usa [[SPI]] (velocidade), uma placa com vários sensores usa [[I2C]] (poucos fios) e um termômetro simples pode usar [[1-Wire]] (mínimo de fios).

---

# 🔑 Vocabulário essencial

Antes de estudar cada protocolo, estes conceitos aparecem em **todos** eles. Vale entender bem agora.

## 1. Sinal digital e níveis lógicos

Um chip digital só "enxerga" dois estados em um fio: **tensão alta** (representa **1**) e **tensão baixa** (representa **0**). Já vimos em [[Bits e Bytes]] que tudo no computador é feito de bits — aqui está a parte **física**: um bit é uma voltagem em um fio durante um intervalo de tempo.

> 💡 **Nível lógico:** a faixa de tensão que o chip interpreta como 0 ou como 1. Os mais comuns em microcontroladores são **3,3 V** (ESP32, Raspberry Pi) e **5 V** (Arduino Uno clássico). Ligar um sinal de 5 V em um pino de 3,3 V pode **queimar o chip** — por isso existe o *level shifter* (conversor de nível).

```
Nível lógico 3,3 V:

3,3 V ┤ ‾‾‾‾‾‾‾‾‾‾   ← "1" (alto): o chip entende como 1
      │
~1,6 V┤ - - - - - -  ← zona incerta: o chip pode errar a leitura
      │
0 V   ┤ __________   ← "0" (baixo): o chip entende como 0
```

## 2. Clock (relógio) e sincronização

Para o receptor saber **quando** olhar o fio, os dois lados precisam concordar com o tempo. Existem duas estratégias:

> 💡 **Comunicação síncrona:** existe um fio extra, o **clock**, que dá o "tique" de cada bit. O receptor lê o dado **exatamente** nas bordas do clock. Exemplos: [[SPI]] e [[I2C]].

> 💡 **Comunicação assíncrona:** **não há** fio de clock. Os dois lados combinam a **velocidade antes** (ex.: *baud rate*) e cada um usa seu relógio interno. Exemplos: [[UART]], [[RS-232]], [[1-Wire]] (por tempo de pulso).

```
SÍNCRONA (com clock):                ASSÍNCRONA (sem clock):
CLK  ┐_┌‾┐_┌‾┐_┌‾┐_┌‾                  (cada lado conta o tempo sozinho,
DADO ──X──X──X──X──                    com a velocidade combinada antes)
     ↑ lê o dado a cada borda          DADO ──‾‾──__──‾‾‾‾──__──
```

| | Síncrona | Assíncrona |
|---|---|---|
| Fios | +1 (o clock) | Menos fios |
| Velocidade | Maior e mais estável | Limitada pela precisão dos relógios |
| Risco | Mais fios para rotear | Se as velocidades não baterem, os dados corrompem |

## 3. Serial × Paralelo

> 💡 **Comunicação paralela:** vários bits trafegam **ao mesmo tempo**, cada um em um fio (ex.: 8 fios = 1 byte por vez).
> **Comunicação serial:** os bits trafegam **um depois do outro** pelo mesmo fio.

Parece que paralelo seria sempre mais rápido, mas em alta velocidade é muito difícil garantir que **todos os bits cheguem exatamente juntos** (pequenas diferenças no comprimento dos fios já bagunçam). Por isso a tendência moderna é **serial rápido** — a mesma lição que vimos no PCIe em [[Barramentos]]. **Todos os protocolos deste MOC são seriais.**

## 4. Direção da comunicação: simplex, half-duplex, full-duplex

Já estudado em [[Tipos de Transmissão]] (Redes), aqui aplicado a fios:

| Modo | Significado | Exemplo neste MOC |
|---|---|---|
| **Simplex** | Só um sentido | Um sensor que só envia dados |
| **Half-duplex** | Os dois sentidos, mas **um de cada vez** | [[I2C]], [[1-Wire]], [[CAN]] |
| **Full-duplex** | Os dois sentidos **ao mesmo tempo** | [[UART]], [[SPI]], [[RS-232]] |

## 5. Mestre/escravo (controlador/periférico)

> 💡 **Mestre (master) / Escravo (slave):** o mestre **inicia** a conversa e, em protocolos síncronos, **controla o clock**; o escravo só **responde**. Hoje a indústria vem trocando os nomes: **controlador (controller)** e **periférico (peripheral)** ou **alvo (target)**. Você verá os dois nomes nos datasheets — são a mesma coisa.

Em um sistema com um ESP32 e vários sensores, o ESP32 é o mestre e cada sensor é um escravo. Há também protocolos **multi-mestre**, onde qualquer nó pode iniciar a conversa — o [[CAN]] é o exemplo mais forte.

## 6. Topologia: ponto a ponto × barramento

> 💡 **Ponto a ponto (point-to-point):** exatamente **dois** dispositivos, ligados diretamente. Ex.: [[UART]], [[RS-232]].
> **Barramento (bus):** **vários** dispositivos compartilhando os mesmos fios. Ex.: [[I2C]], [[CAN]], [[1-Wire]]. Exige um jeito de **escolher quem fala** (endereço, ID, chip select).

É a mesma ideia de [[Topologia de Rede]], só que dentro de uma placa — e a mesma de "barramento compartilhado × ponto a ponto" que vimos em [[Barramentos]].

```
Ponto a ponto:       Barramento (vários nós nos mesmos fios):
A ←──────→ B         A ──┬────┬────┬──
                         B    C    D
```

## 7. Como escolher com quem falar (seleção de dispositivo)

Quando há vários dispositivos no mesmo conjunto de fios, é preciso identificar o destinatário. Cada protocolo resolve de um jeito:

| Estratégia | Como funciona | Protocolo |
|---|---|---|
| **Chip Select** | Um fio dedicado por dispositivo | [[SPI]] |
| **Endereço** | Cada dispositivo tem um número; o mestre o "chama" | [[I2C]] |
| **ID de 64 bits** | Número único gravado de fábrica | [[1-Wire]] |
| **ID de mensagem** | O ID identifica o **conteúdo** e a prioridade, não o nó | [[CAN]] |
| **Nenhuma** | Só existem dois dispositivos | [[UART]], [[RS-232]] |

## 8. Sinalização: *single-ended* × diferencial

> 💡 **Single-ended (referenciado ao terra):** um fio carrega o sinal e a tensão é medida **em relação ao GND**. Simples, mas vulnerável a ruído. Usado por [[UART]], [[SPI]], [[I2C]], [[1-Wire]] e [[RS-232]].
> **Diferencial:** o sinal é a **diferença de tensão entre dois fios** (um par). O ruído que entra nos dois fios ao mesmo tempo se **cancela** na subtração. Usado por [[CAN]] (CAN_H e CAN_L), RS-485 e Ethernet.

```
Single-ended: ruído soma ao sinal                Diferencial: ruído se cancela
   sinal  ─────/\/\───────                       fio +  ─────/\/\───── (sinal + ruído)
   ruído  ~~~~~~~~~~~~~~~~ → sinal corrompido    fio −  ─────\/\/───── (−sinal + ruído)
                                                 receptor faz (+) − (−) → ruído some
```

É exatamente o motivo de os cabos de rede usarem **pares trançados** (veja [[Cabos Par Trançado (UTP e STP)]]).

## 9. Saída *push-pull* × *open-drain*

Como um chip coloca uma tensão no fio:

> 💡 **Push-pull:** o chip tem dois "interruptores": um liga o fio ao **positivo** (força 1) e outro liga ao **terra** (força 0). Ele **controla ativamente** os dois estados. É rápido, mas **dois chips não podem dividir o mesmo fio** — se um força 1 e outro força 0, dá curto-circuito.
> **Open-drain (dreno aberto):** o chip só consegue **puxar o fio para baixo** (0) ou **soltá-lo**. Quem leva o fio para cima (1) é um **resistor de pull-up** externo. Vários chips podem dividir o mesmo fio sem curto: **basta um puxar para baixo para a linha ficar em 0**.

```
Open-drain (usado em I2C e 1-Wire):

            VCC
             │
          [pull-up]          ← sem ninguém puxando: linha = 1
             │
   ──────────┴──────────  linha compartilhada
        │        │
      [Chip A] [Chip B]   ← cada um só consegue ligar ao GND (0)
```

Essa técnica é chamada de **wired-AND**: a linha só é 1 se **todos** soltarem. É a base de [[I2C]], [[1-Wire]] e (de forma parecida, com os estados dominante/recessivo) do [[CAN]].

## 10. Estrutura de uma mensagem

Mandar bits soltos não basta — o receptor precisa saber **onde a mensagem começa, onde termina e se chegou certa**. Quase todo protocolo tem estes elementos:

| Elemento | Função |
|---|---|
| **Bit de início (start)** | Avisa "vou começar a transmitir" |
| **Dados (payload)** | A informação em si |
| **Endereço / ID** | Para quem (ou o que) é a mensagem |
| **Detecção de erro** | Paridade, *checksum* ou CRC |
| **Confirmação (ACK)** | O receptor avisa "recebi" |
| **Bit de fim (stop)** | Avisa "terminei" |

> 💡 **Paridade:** 1 bit extra que faz o total de bits "1" ficar par (ou ímpar). Detecta erro de **1 bit**, mas não corrige.
> **Checksum:** soma dos bytes enviada junto; o receptor refaz a soma e compara.
> **CRC (Cyclic Redundancy Check):** cálculo matemático mais robusto sobre os dados; detecta a grande maioria dos erros, inclusive rajadas de bits.
> **ACK (acknowledge):** bit/mensagem de confirmação enviada pelo receptor.

Quanto mais um protocolo se preocupa com isso (CRC, ACK, retransmissão), **mais confiável e mais complexo** ele é. O [[UART]] não tem nada disso (só paridade opcional); o [[CAN]] tem tudo.

## 11. Taxa de transmissão: baud × bps

> 💡 **bps (bits por segundo):** quantos bits passam por segundo.
> **Baud rate:** quantos **símbolos** (mudanças de sinal) por segundo. Nos protocolos digitais simples deste MOC, **1 símbolo = 1 bit**, então baud e bps coincidem — e a palavra "baud" é usada principalmente em [[UART]] e [[RS-232]].

⚠️ A **velocidade bruta** (taxa na linha) é maior que a **velocidade útil** (dados reais), porque parte dos bits é "overhead" (start, stop, endereço, CRC...).

---

# 🖥️ Como o sistema operacional enxerga esses protocolos

Esses protocolos ficam na camada de **hardware**, mas o SO também precisa lidar com eles — pela mesma lógica que vimos em [[Dispositivos de IO]]:

- Cada protocolo tem um **controlador** de hardware (ex.: o controlador UART, o controlador I2C) dentro do microcontrolador ou do chipset.
- O SO fala com esse controlador por meio de um **driver**.
- No Linux, o resultado aparece como **arquivos especiais** em `/dev` (veja [[Arquivos]]): `/dev/ttyUSB0` ou `/dev/ttyS0` (serial/UART/RS-232), `/dev/i2c-1` (I2C), `/dev/spidev0.0` (SPI) e interfaces `can0` (CAN, via SocketCAN). Ou seja: **abrir, ler e escrever** — as mesmas syscalls de [[Syscalls para Gerenciamento de Arquivos]].
- A decisão entre **espera ocupada (polling)**, **interrupção** e **DMA** (vista em [[Dispositivos de IO]]) também se aplica aqui: um UART pode avisar a CPU por interrupção quando chega um byte, e um SPI rápido pode usar DMA para mover blocos sem ocupar a CPU.

> 💡 **Em microcontroladores sem SO (bare-metal)**, como um Arduino simples, o próprio programa conversa direto com os controladores — por isso bibliotecas como `Serial`, `Wire` e `SPI` existem.

---

# 🧭 A família de protocolos deste MOC

```
                       PROTOCOLOS DE HARDWARE
                                │
        ┌──────────────┬────────┴───────┬───────────────┐
        │              │                │               │
   Ponto a ponto    Barramento     Barramento       Barramento
   (2 dispositivos) com endereço   robusto          de 1 fio
        │              │                │               │
    UART / RS-232   SPI / I2C           CAN           1-Wire
```

| Protocolo | Nota | Em uma frase |
|---|---|---|
| [[UART]] | Dois fios, assíncrono | O jeito mais simples de dois dispositivos conversarem |
| [[SPI]] | Quatro fios, síncrono | O mais rápido, mas gasta fios |
| [[I2C]] | Dois fios, endereçado | Muitos sensores com poucos fios |
| [[CAN]] | Dois fios diferenciais | Criado para o carro: não pode falhar |
| [[RS-232]] | Padrão elétrico de 1960 | Ainda vivo em máquinas industriais |
| [[1-Wire]] | Um fio de dados | O mínimo absoluto de fiação |

Para comparar todos lado a lado, veja [[Comparativo e Escolha do Protocolo]].

---

# ✅ Resumo do Conceito

- Um **protocolo de hardware** é o conjunto de regras (fios, tensões, tempo, formato das mensagens, quem fala quando) que permite a dois chips se entenderem
- Não existe protocolo perfeito: cada um troca **velocidade, fios, simplicidade e robustez** de um jeito
- **Síncrono** = tem fio de clock; **assíncrono** = os dois lados combinam a velocidade antes
- **Serial** = bits um após o outro; todos os protocolos deste MOC são seriais
- **Full-duplex** = os dois sentidos ao mesmo tempo; **half-duplex** = um de cada vez
- **Mestre/escravo** (ou controlador/periférico): o mestre inicia e, nos síncronos, dita o clock
- Seleção de dispositivo: **chip select** (SPI), **endereço** (I2C), **ID de 64 bits** (1-Wire), **ID de mensagem** (CAN)
- **Push-pull** controla os dois estados; **open-drain + pull-up** permite vários chips no mesmo fio (wired-AND)
- **Diferencial** cancela ruído; **single-ended** é mais simples e mais vulnerável
- Detecção de erro vai de **nada** (UART) a **CRC + ACK + retransmissão** (CAN)
- No SO, esses protocolos aparecem como **drivers** e **arquivos em `/dev`**

---

# 🔗 Conexões com outras anotações

- [[Barramentos]] — barramentos internos do PC (PCIe, USB, SATA) são parentes mais rápidos desses protocolos; mesma lição de serial × paralelo e compartilhado × ponto a ponto
- [[Dispositivos de IO]] — controladores, drivers, polling, interrupções e DMA: como o SO conversa com o hardware
- [[Bits e Bytes]] — o que um bit é na teoria; aqui, o que ele é como tensão em um fio
- [[Processadores]] — o microcontrolador contém os controladores UART/SPI/I2C
- [[Camada Física - 1]] — o equivalente em redes: tensões, cabos e sinais
- [[Camada de Enlace - 2]] — endereçamento e detecção de erro entre dispositivos (compare com ACK/CRC)
- [[Tipos de Transmissão]] — simplex, half-duplex, full-duplex
- [[Topologia de Rede]] — ponto a ponto e barramento
- [[Cabos Par Trançado (UTP e STP)]] — o mesmo princípio do sinal diferencial usado no CAN
- [[Arquivos]] — dispositivos expostos como arquivos especiais em `/dev`
- [[Exclusão mútua com espera ocupada]] — a analogia com *polling*; arbitragem de barramento é um problema de exclusão mútua em hardware
