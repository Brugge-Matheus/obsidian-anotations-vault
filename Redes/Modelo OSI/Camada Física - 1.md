---
tags:
  - redes
  - redes/osi
---

# Camada Física - 1

A **Camada 1 (Física)** é o "Chão de Fábrica" ou a "Estrada" do Modelo OSI. Se todas as outras camadas lidam com dados lógicos e endereços, a Camada 1 é a única que lida com o **mundo real e tangível**: eletricidade, luz e ondas de rádio.

---

## 1. Camada 1: Física (Physical Layer)

A Camada Física é responsável pela transmissão e recepção de **bits brutos** (0s e 1s) através de um meio de transmissão físico. Ela não entende o que é um IP, um MAC ou um e-mail; ela só entende **sinais**.

### 1.1. O que ela faz?

Ela define as especificações técnicas para que o dado "saia" do computador e entre no cabo:

- **Codificação de Sinais:** Como transformar um "1" em um pulso elétrico de 5 volts ou um "0" em 0 volts.
- **Meios de Transmissão:** Define se o dado vai por **Cabo de Cobre** (eletricidade), **Fibra Óptica** (luz) ou **Wi-Fi** (ondas de rádio).
- **Topologia Física:** Como os cabos estão conectados (Estrela, Barramento, etc.).
- **Especificações Mecânicas:** O formato dos conectores (RJ-45, USB), a pinagem dos cabos e a voltagem.

### 1.2. Exemplo da Vida Real: O Código Morse ou o Telefone de Latinha

Imagine que você está enviando uma mensagem em Código Morse usando uma lanterna:

- **Camadas 7 a 2:** Você pensou na mensagem, traduziu para o código e organizou a ordem das letras.
- **Camada 1 (Física):** É o **feixe de luz** saindo da lanterna e viajando pelo ar.
    1. Se a bateria da lanterna acabar, a Camada 1 falhou.
    2. Se alguém colocar uma parede na frente, a Camada 1 foi interrompida.
    3. A lanterna não sabe o que está enviando, ela apenas pisca (0 e 1).

### 1.3. O Protagonista e o Equipamento

- **Equipamentos Principais:** **Hubs**, **Repetidores**, **Cabos** (UTP, Fibra), **Conectores** (RJ-45) e **Modems** (na parte da modulação).
- **Unidade de Dado (PDU):** **Bit**.
- **Função:** Transformar o Quadro (da Camada 2) em uma sequência de sinais físicos.

### 1.4. Problemas Comuns da Camada 1 (Dica de Suporte Técnico)

80% dos problemas de rede começam aqui:

- Cabo de rede mal crimpado ou quebrado.
- Interferência eletromagnética (cabo de rede passando junto com cabo de energia).
- Conector sujo ou oxidado.
- Distância excessiva (o sinal "enfraquece" após 100 metros no cabo de cobre).

---

## 2. Resumo da "Viagem do Dado"

Imagine que você enviou um "Oi" no WhatsApp:

1. **Camada 7:** O App gera o dado "Oi".
2. **Camada 4:** O TCP quebra em pedaços e põe a porta.
3. **Camada 3:** O IP coloca o endereço de destino.
4. **Camada 2:** O Ethernet coloca o MAC do seu roteador.
5. **Camada 1:** Tudo isso vira **eletricidade** e corre pelo fio.

---

## 3. Resumo Final

> **Ponto Chave:** A Camada 1 é o "hardware puro". Ela define como os bits são transmitidos fisicamente. Se o LED da sua placa de rede está piscando, a Camada 1 está tentando trabalhar!

---

## Conexão com Sistemas Operacionais

A Camada Física é onde bits se tornam sinais, e isso tem implicações profundas no SO:

**Codificação de sinais e representação binária:** Um bit "1" pode ser +5V, "0" pode ser 0V — isso é a representação binária no nível mais baixo possível, o nível elétrico. Esquemas de codificação mais sofisticados como **NRZ (Non-Return-to-Zero)** e **Manchester Encoding** embutem o sinal de clock nos dados para que o receptor saiba quando cada bit começa e termina — sem um clock separado, o receptor usaria amostragem assíncrona. Tudo isso é representação de dados em nível físico → [[Bits e Bytes]].

**Sincronismo de clock:** O receptor precisa saber exatamente quando amostrar o sinal para ler cada bit. Isso é o problema de **comunicação síncrona vs assíncrona**: no clock síncrono, transmissor e receptor compartilham o mesmo sinal de clock; no assíncrono (como RS-232), eles negociam uma taxa (baud rate) e usam bits de start/stop para sincronização. O processador tem um clock interno que sincroniza a execução de instruções — o mesmo princípio em escala diferente → [[Processadores]].

**DMA e interrupções de NIC:** Quando um frame chega na placa de rede, a NIC não interrompe a CPU para cada bit recebido. Ela usa **DMA (Direct Memory Access)** para escrever o frame recebido diretamente em uma região de memória do kernel (ring buffer), sem envolver a CPU durante a recepção. Somente quando o frame está completo na memória é que a NIC dispara uma **interrupção** (IRQ) para notificar o kernel. O kernel então executa o interrupt handler (`net_rx_action` no Linux) que processa o frame. Esse modelo — periférico escreve diretamente na memória e interrompe a CPU apenas quando termina — é central para o estudo de dispositivos de IO e DMA → [[Memória]], [[Dispositivos de IO]], [[Processadores]].
