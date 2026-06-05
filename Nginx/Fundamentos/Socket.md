---
tags:
  - nginx
  - nginx/fundamentos
---

# Socket

## 1. O que é um Socket? (Conceito Geral)

Imagine que um processo (ex: Nginx) quer enviar dados para outro processo (ex: PHP-FPM). Eles não podem simplesmente "jogar" dados na memória um do outro por segurança. Eles precisam de um **ponto de extremidade (endpoint)** de comunicação.

**Definição:** Um Socket é uma interface (uma "porta" lógica) que permite que dois programas troquem fluxos de dados, seja no mesmo computador ou através de uma rede.

### O socket como file descriptor

No kernel Linux, um socket **é um file descriptor** — apenas um inteiro na tabela de fds do processo que aponta para uma estrutura interna do kernel (`struct socket` + `struct sock`). Isso é o princípio "tudo é um arquivo" do Unix levado às comunicações de rede.

Quando você faz `socket()`, o kernel:
1. Aloca uma `struct socket` no kernel heap.
2. Aloca uma `struct sock` com o estado do protocolo (TCP, UDP, Unix).
3. Cria uma entrada na tabela de file descriptors do processo.
4. Retorna o inteiro (fd) — ex: `fd = 5`.

A partir daí, você usa `read(fd, ...)` e `write(fd, ...)` exatamente como faria com um arquivo. O kernel sabe que fd 5 é um socket e roteia as chamadas para o subsistema de rede.

---

## 2. Unix Domain Sockets (IPC - Inter-Process Communication)

No Linux/Unix, existe um mantra: *"Tudo é um arquivo"*. O Unix Socket leva isso ao pé da letra.

- **O que é:** É um arquivo especial que reside no sistema de arquivos (ex: `/var/run/php-fpm.sock`).
- **Como funciona:** O Nginx escreve nesse "arquivo" e o PHP-FPM lê dele. O Kernel do Linux gerencia essa troca de dados diretamente na memória RAM.
- **Vantagens:**
    - **Velocidade:** É extremamente rápido porque não há o "overhead" (custo extra) de processar protocolos de rede (IP, TCP, Checksums).
    - **Segurança:** Você pode usar permissões de arquivo padrão do Linux (chmod/chown) para decidir quem pode ler ou escrever naquele socket.
- **Limitação:** Só funciona se os dois processos estiverem no **mesmo servidor físico ou virtual**.

### Como funciona internamente o Unix Domain Socket

Quando Nginx e PHP-FPM comunicam via UDS (`/run/php-fpm.sock`):

1. PHP-FPM faz `bind("/run/php-fpm.sock")` — cria o arquivo especial no filesystem.
2. PHP-FPM faz `listen()` — marca o socket como pronto para aceitar conexões.
3. Nginx faz `connect("/run/php-fpm.sock")` — o kernel encontra o socket pelo path no filesystem.
4. A transferência de dados é uma **cópia direta entre buffers no kernel** — sem passar pela pilha IP/TCP, sem calcular checksums, sem fragmentar em pacotes.

Resultado: latência muito menor que TCP loopback para comunicação local.

---

## 3. Network Sockets (Internet Sockets / TCP-IP)

Estes são os sockets que usamos para comunicação via rede. Eles são definidos por uma combinação de **Endereço IP + Porta**.

- **O que é:** Um ponto de conexão que utiliza a pilha de protocolos de rede (TCP/IP).
- **Como funciona:** O Nginx envia os dados para `127.0.0.1:9000`. O sistema operacional empacota isso em pacotes TCP, coloca um cabeçalho IP e "envia" para a interface de rede (mesmo que seja a local/loopback).
- **Vantagens:**
    - **Escalabilidade:** Permite que o Nginx esteja no "Servidor A" e o PHP no "Servidor B".
    - **Flexibilidade:** É o padrão universal da internet.
- **Desvantagem:** É ligeiramente mais lento que o Unix Socket devido ao processamento extra da pilha de rede (encapsulamento de pacotes).

### Ciclo de vida de um socket TCP (syscalls)

**Servidor (Nginx):**
```
socket()  → cria o fd
bind()    → associa ao IP:porta (ex: 0.0.0.0:80)
listen()  → marca como servidor, define backlog
accept()  → bloqueia até nova conexão, retorna novo fd para cada cliente
```

**Cliente (browser):**
```
socket()   → cria o fd
connect()  → inicia o TCP three-way handshake (SYN → SYN-ACK → ACK)
```

Após `accept()`, o Nginx tem **dois fds ativos** por conexão:
- O fd original do `listen()` — continua aceitando novas conexões.
- O novo fd retornado pelo `accept()` — usado para ler/escrever dados desta conexão específica.

---

## 4. Outros Tipos de Sockets (Para Conhecimento)

Embora no Nginx você lide 99% do tempo com os dois acima, vale conhecer estes termos:

- **Stream Sockets (SOCK_STREAM):** Utilizam o protocolo **TCP**. Garantem que os dados cheguem na ordem correta e sem erros. É o que o Nginx usa para HTTP e FastCGI.
- **Datagram Sockets (SOCK_DGRAM):** Utilizam o protocolo **UDP**. São mais rápidos, mas não garantem a entrega nem a ordem. Usados em streaming de vídeo ou DNS.
- **Raw Sockets:** Permitem enviar pacotes sem passar pelas camadas de transporte padrão (usado por ferramentas de rede como o `ping`).

---

## 5. Comparativo: Unix vs. TCP

| Característica | Unix Domain Socket | TCP/IP Socket |
| --- | --- | --- |
| **Identificador** | Caminho de arquivo (`/tmp/php.sock`) | IP e Porta (`127.0.0.1:9000`) |
| **Localização** | Mesma máquina apenas | Mesma máquina ou máquinas diferentes |
| **Performance** | Alta (Direto na memória/Kernel) | Média (Passa pela pilha de rede) |
| **Segurança** | Permissões de arquivo (Linux) | Firewall e regras de rede |
| **Uso no Nginx** | `fastcgi_pass unix:/path/to.sock;` | `fastcgi_pass 127.0.0.1:9000;` |

---

## 6. SO_REUSEPORT: múltiplos Workers no mesmo socket

Por padrão, apenas um processo pode chamar `accept()` em um socket. O Nginx usa a opção de socket `SO_REUSEPORT` para que **todos os Workers possam `listen()` e `accept()` na mesma porta**:

```
socket() + setsockopt(SO_REUSEPORT) → bind(:80) → listen()
```

O kernel distribui as novas conexões entre os Workers que estão bloqueados em `accept()` — fazendo load balancing no nível do SO, sem coordenação entre os Workers. Cada Worker mantém sua própria fila de conexões pendentes, reduzindo lock contention.

---

## 7. Analogia para facilitar a memorização

- **Unix Socket** é como um **Tubo Acústico** dentro de uma casa: Você fala de um quarto para o outro através de um cano físico. É rápido e privado, mas você não consegue falar com o vizinho.
- **TCP Socket** é como uma **Ligação Telefônica**: Você precisa discar um número (IP) e um ramal (Porta). Você pode falar com alguém no mesmo prédio ou do outro lado do mundo, mas a ligação passa pela central telefônica (Pilha de Rede).

---

## 8. Como isso aparece no seu estudo de Nginx?

Quando você configura o `fastcgi_pass`, você está escolhendo qual "tomada" o Nginx vai usar para plugar no PHP-FPM:

1. Se você quer **performance máxima** e tudo está no mesmo servidor: Use **Unix Socket**.
2. Se você planeja **separar os servidores** no futuro ou usa Docker (em alguns cenários): Use **TCP Socket**.

---

## Conexão com Sistemas Operacionais

Sockets são uma das abstrações mais fundamentais do SO — conectam diretamente ao kernel:

- **Socket como file descriptor:** Um socket é apenas um inteiro na fd table do processo — o mesmo mecanismo de "tudo é arquivo". Ver [[Arquivos]] e [[Processos]] (tabela de fds por processo).
- **socket() syscall:** Aloca `struct socket` + `struct sock` no kernel, retorna fd. Ver [[System Calls]].
- **Ciclo de vida TCP:** `socket() → bind() → listen() → accept()` no servidor; `socket() → connect()` no cliente — todas system calls. Ver [[System Calls]].
- **Unix Domain Socket:** O arquivo `.sock` no filesystem é criado via `bind()` — o kernel cria um inode especial. Comunicação sem pilha de rede, apenas cópia de buffer no kernel. Ver [[Arquivos]] e [[System Calls]].
- **TCP vs Unix socket:** TCP passa pela pilha de rede completa (IP routing, checksums, loopback). Unix socket vai direto para o buffer do kernel. Ver [[Dispositivos de IO]] e [[System Calls]].
- **SO_REUSEPORT:** Permite múltiplos Workers (`accept()` em paralelo) no mesmo porto — o kernel faz load balancing das conexões. Ver [[System Calls]] e [[Processos]].

---

## Conexão com Go

- **[[HTTP (net-http)]]:** `net.Listen("tcp", ":8080")` e `net.Listen("unix", "/tmp/nginx.sock")` criam sockets TCP e Unix respectivamente. `net/http` usa `net.Listen` internamente para seu servidor HTTP.
- **[[Interfaces]]:** `net.Conn` é a interface que abstrai qualquer socket (TCP, Unix, TLS) — qualquer tipo que implementa `Read`, `Write`, `Close` e métodos de deadline é uma `net.Conn`. Isso permite escrever código que funciona com TCP e Unix sockets sem mudanças.
- **[[Goroutines]]:** O servidor `net/http` do Go chama `Accept()` em loop e dispara uma goroutine para cada nova conexão — o equivalente Go do loop `accept()` no worker do Nginx.
