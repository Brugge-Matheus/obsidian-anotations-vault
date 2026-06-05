---
tags:
  - redes
  - redes/osi
---

# Camada de Transporte - 4

A **Camada 4 (Transporte)** é o "Gerente de Logística" ou o "Correio" do Modelo OSI. Esta é uma das camadas mais críticas, pois é aqui que a rede decide **como** o dado será entregue: com garantia total de que chegou ou com foco em velocidade máxima.

---

### 📦 Camada 4: Transporte (Transport Layer)

A Camada de Transporte é a primeira camada que se preocupa com a comunicação **ponta a ponta** (do computador de origem ao computador de destino). Ela pega os dados grandes das camadas superiores e os quebra em pedaços menores chamados **Segmentos**.

### 1. O que ela faz?

Ela realiza quatro funções vitais para que a internet não seja um caos:

- **Segmentação e Reagrupamento:** Divide arquivos grandes em pequenos pedaços para caber na rede e os remonta na ordem correta no destino.
- **Controle de Fluxo:** Se o servidor é muito rápido e o seu celular é lento, a Camada 4 avisa o servidor para "ir mais devagar" para não travar o seu aparelho.
- **Controle de Erros:** Verifica se algum pedaço do dado se perdeu ou corrompeu no caminho e pede para reenviar.
- **Endereçamento de Serviço (Portas):** É aqui que entram as **Portas (80, 443, 22)**. Ela decide se o dado vai para o Navegador, para o Spotify ou para o E-mail.

### 2. Exemplo da Vida Real: A Transportadora de Mudanças

Imagine que você vai mudar de casa:

- **Camada 7, 6 e 5:** Você decide o que levar, embala tudo em caixas e marca o horário da mudança.
- **Camada 4 (Transporte):** É a **empresa de mudança**.
    1. **Segmentação:** Eles percebem que o guarda-roupa não cabe inteiro no caminhão, então o desmontam em peças numeradas (1, 2, 3...).
    2. **Controle de Erros:** Ao chegar na casa nova, eles conferem a lista. Se a peça número 2 sumiu, eles ligam para o depósito e pedem outra.
    3. **Reagrupamento:** Eles montam o guarda-roupa exatamente como era antes.

### 3. Os dois "Chefes" da Camada 4

Existem dois protocolos principais aqui, e eles trabalham de formas opostas:

- **TCP (Transmission Control Protocol):** É o "Sedex com AR". Ele é **confiável**. Ele só envia o próximo pedaço quando o destino confirma que recebeu o anterior. Se algo sumir, ele reenvia. (Usado em: Sites, E-mails, Arquivos).
- **UDP (User Datagram Protocol):** É o "Panfleto de rua". Ele é **rápido**. Ele joga os dados na rede e não quer saber se chegaram. Se um pedaço sumir, paciência. (Usado em: Jogos online, Chamadas de vídeo, Streaming ao vivo).

### 4. Resumo (Callout):

> 💡 **Ponto Chave:** A Camada 4 é a "camada da entrega". Ela garante que os dados cheguem ao destino correto (usando portas) e na ordem certa (usando segmentação).
> 

---

### 🛠️ Exemplo Prático no Navegador:

Quando você baixa um arquivo:

1. O arquivo é quebrado em milhares de **Segmentos** (Camada 4).
2. O protocolo **TCP** numera cada um deles.
3. Se o segmento 500 chegar com erro, seu PC avisa o servidor: *"Ei, o 500 veio ruim, manda de novo!"*.
4. Quando todos chegam, a Camada 4 junta tudo e entrega o arquivo perfeito para a Camada 5.

---

### 🔗 Conexões com SO e Go

- **TCP/UDP = tipos de socket:** `SOCK_STREAM` (TCP, orientado a conexão) vs `SOCK_DGRAM` (UDP, sem conexão) — o tipo é definido na chamada `socket()` → [[System Calls]]
- **Números de porta (16 bits):** um processo se vincula a uma porta com a syscall `bind()`; o kernel mantém a tabela (IP:porta) → socket fd → processo → [[System Calls]], [[Processos]]
- **Portas < 1024 exigem root** (ou a capability `CAP_NET_BIND_SERVICE`); proteção implementada no kernel → [[Proteção]]
- **Máquina de estados do TCP** (CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT → TIME_WAIT → CLOSED): o kernel rastreia o estado de cada conexão separadamente — analogia com [[Estados de Processos]]
- **Buffer de recepção do TCP:** ring buffer no kernel; `recv()` copia kernel → espaço de usuário → [[Memória]], [[System Calls]]
- **Three-way handshake** (SYN / SYN-ACK / ACK): cada passo é uma transição na máquina de estados do kernel → [[System Calls]]
- **Go:** `net.Dial("tcp", ...)` abre um `SOCK_STREAM`; `net.Listen("tcp", ...)` + loop de `Accept()` cria um servidor; cada conexão é tratada em uma goroutine → [[HTTP (net-http)]], [[Goroutines]]
