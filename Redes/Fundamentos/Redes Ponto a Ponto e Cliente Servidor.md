---
tags:
  - redes
  - redes/fundamentos
---

# Redes Ponto a Ponto e Cliente Servidor

É fundamental entender que esses dois conceitos definem a **arquitetura lógica** da rede, ou seja, como os dispositivos se organizam para compartilhar recursos e serviços.

Aqui está o detalhamento técnico de cada uma:

---

### 1. Redes Cliente-Servidor (Client-Server)

É o modelo mais utilizado na internet e em redes corporativas. Existe uma distinção clara de papéis entre os dispositivos.

- **O Servidor:** É uma máquina (ou software) potente que fornece recursos, dados ou serviços. Ele fica "ouvindo" a rede, esperando por solicitações.
- **O Cliente:** É o dispositivo (computador, celular) que solicita o serviço ao servidor.
- **Características:**
    - **Centralização:** Os dados e a segurança são gerenciados em um ponto central (o servidor).
    - **Escalabilidade:** É fácil adicionar muitos clientes, pois o servidor é projetado para lidar com múltiplas conexões.
    - **Dependência:** Se o servidor cair, todos os clientes perdem o acesso ao serviço (ponto único de falha).
- **Exemplos:**
    - Navegação Web (Seu navegador é o cliente, o site é o servidor).
    - E-mail (Outlook/Gmail é o cliente, os servidores da Microsoft/Google processam as mensagens).
    - Banco de Dados de uma empresa.

---

### 2. Redes Ponto a Ponto (Peer-to-Peer ou P2P)

Neste modelo, não existe um servidor central fixo. Todos os dispositivos na rede têm capacidades e responsabilidades iguais.

- **Funcionamento:** Cada computador (chamado de "nó" ou "peer") pode atuar como **cliente e servidor ao mesmo tempo**. Ele pode solicitar um arquivo de outro computador e, simultaneamente, fornecer um arquivo para um terceiro.
- **Características:**
    - **Descentralização:** Não há um administrador central. Cada usuário controla seus próprios recursos.
    - **Resiliência:** Se um computador sair da rede, o restante continua funcionando normalmente. Não há um "ponto único de falha".
    - **Dificuldade de Gerenciamento:** É difícil fazer backup de tudo ou aplicar políticas de segurança uniformes, pois os dados estão espalhados.
- **Exemplos:**
    - **BitTorrent:** Compartilhamento de arquivos onde você baixa pedaços de vários usuários.
    - **Blockchain/Bitcoin:** Uma rede financeira onde todos os nós validam as transações sem um banco central.
    - **Rede Doméstica Simples:** Quando você compartilha uma impressora ou uma pasta diretamente entre dois notebooks em casa.

---

### Tabela Comparativa

| Característica | Cliente-Servidor | Ponto a Ponto (P2P) |
| --- | --- | --- |
| **Hierarquia** | Existe (Servidor é superior) | Não existe (Todos são iguais) |
| **Custo** | Alto (Servidores são caros) | Baixo (Usa hardware comum) |
| **Segurança** | Centralizada e mais robusta | Descentralizada e difícil de controlar |
| **Desempenho** | Depende da capacidade do servidor | Depende do número de usuários (mais usuários = mais fontes) |
| **Configuração** | Complexa (exige administrador) | Simples (plug and play) |

---

### 3. Quando usar cada uma? (Dica de Projeto)

- **Use Cliente-Servidor quando:** A segurança for crítica, os dados precisarem ser centralizados para backup e houver muitos usuários (ex: uma empresa com 50 funcionários).
- **Use P2P quando:** O objetivo for compartilhar arquivos de forma rápida entre poucos dispositivos ou quando você quiser criar um sistema que ninguém consiga "derrubar" desligando um único servidor (ex: redes de arquivos ou criptomoedas).

---

## 4. Aprofundamento: O que acontece no nível do SO e das syscalls

**Cliente-Servidor no nível do kernel:**

O cliente realiza uma **abertura ativa** (`active open`):
1. `socket()` — cria um file descriptor de socket.
2. `connect(ip, port)` — envia um pacote SYN TCP ao servidor. O kernel gerencia o handshake TCP de 3 vias internamente.

O servidor realiza uma **abertura passiva** (`passive open`):
1. `socket()` — cria o socket.
2. `bind(porta)` — associa o socket a uma porta (ex: 80 para HTTP).
3. `listen(backlog)` — o kernel começa a aceitar conexões entrantes. O `backlog` é o tamanho da **fila de conexões pendentes no kernel** — conexões que completaram o handshake TCP mas ainda não foram aceitas pelo processo servidor. Se a fila estiver cheia, novas conexões são descartadas (RST ou simplesmente ignoradas).
4. `accept()` — o servidor bloqueia até que uma conexão chegue. O kernel retorna um novo file descriptor representando aquela conexão específica.

Cada conexão aceita pelo servidor é gerenciada como um [[Processos]] ou thread separado (ou via I/O não-bloqueante com `epoll`/`select`).

**P2P no nível do kernel:**

Cada nó faz ao mesmo tempo:
- `bind()` + `listen()` + `accept()` — age como servidor para os peers que se conectam a ele.
- `connect()` — age como cliente ao se conectar a outros peers.

O mesmo processo gerencia múltiplos sockets simultaneamente — geralmente usando `epoll` (Linux) para monitorar todos os file descriptors com uma única thread.

**Fila de conexões (`listen backlog`):** quando o servidor chama `listen(backlog)`, o kernel mantém duas filas internas:
- **SYN queue (incomplete):** conexões cujo SYN chegou mas o handshake ainda não completou.
- **Accept queue (complete):** conexões com handshake completo aguardando `accept()`.

O `backlog` controla o tamanho da accept queue. Um servidor sobrecarregado que não chama `accept()` rápido o suficiente deixa a fila encher → novos clientes recebem RST ou timeout.

---

## Conexão com Sistemas Operacionais

- **Cliente — `connect()`:** syscall que inicia o handshake TCP → [[System Calls]].
- **Servidor — `bind()` + `listen()` + `accept()`:** três syscalls que colocam o servidor em modo de espera → [[System Calls]].
- **`listen(backlog)`:** o backlog é a fila de conexões pendentes gerenciada pelo kernel → [[System Calls]], [[Processos]] (cada conexão aceita geralmente vira um processo ou thread).
- **Múltiplos peers simultâneos (P2P):** requer multiplexação de I/O com `select()`/`epoll()` → [[System Calls]].

---

## Conexão com Go

- `http.ListenAndServe("addr", handler)` implementa o modelo servidor (passive open internamente): `socket → bind → listen → accept` em loop → `[[HTTP (net-http)]]`.
- `http.Get(url)` e `http.NewRequest()` implementam o modelo cliente (active open: `connect()` interno) → `[[HTTP (net-http)]]`.
- Em Go, cada conexão aceita pelo servidor HTTP é servida em uma goroutine separada — o runtime mapeia goroutines em threads do SO → `[[Goroutines]]`.
