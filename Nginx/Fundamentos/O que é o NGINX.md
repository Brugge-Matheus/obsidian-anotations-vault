---
tags:
  - nginx
  - nginx/fundamentos
---

# O que é o NGINX

## 1. História do Nginx

O **Nginx** (pronuncia-se "Engine-X") é um software de código aberto que atua simultaneamente como servidor web, proxy reverso, balanceador de carga e cache de HTTP. Criado por Igor Sysoev em 2004, ele foi projetado especificamente para resolver o "problema C10k" (lidar com 10 mil conexões simultâneas em um único servidor).

O **C10K problem** foi o grande desafio de engenharia da época: como um único servidor consegue gerenciar 10.000 conexões abertas ao mesmo tempo sem esgotar memória ou CPU? A resposta de Sysoev foi abandonar o modelo thread-per-connection e adotar um event loop assíncrono não bloqueante.

Diferente de servidores tradicionais que perdem performance conforme o tráfego aumenta, o Nginx mantém um consumo de memória baixo e previsível, sendo hoje a escolha principal para aplicações de alta escala.

---

## 2. Arquitetura e Funcionamento Interno

Para entender o Nginx, é preciso entender como ele gerencia o trabalho. Ele não cria um novo processo para cada usuário que acessa o site; em vez disso, ele usa uma **Arquitetura Orientada a Eventos (Event-Driven)**.

### 2.1 O Modelo Master-Worker

O Nginx opera com um processo principal e vários processos auxiliares:

- **Master Process:** É o "gerente". Ele lê e valida as configurações, gerencia as portas de rede e controla os processos filhos. Ele não processa requisições de usuários diretamente.
- **Worker Processes:** São os "operários". Eles realizam o trabalho real, como ler arquivos do disco ou encaminhar requisições. Geralmente, o número de Workers é igual ao número de núcleos (cores) da CPU do servidor para maximizar a eficiência.

O Master **faz fork** dos Workers na inicialização — é o modelo clássico de processo pai/filho do Unix. Cada Worker é um processo independente do SO; se um Worker travar (raro), o Master pode reiniciá-lo sem derrubar os demais. Isso reflete diretamente o modelo de processos estudado em SO.

### 2.2 Processamento Assíncrono e Não Bloqueante

Esta é a "mágica" do Nginx. Enquanto servidores como o Apache (em seu modo padrão) dedicam uma thread inteira para cada conexão (o que consome muita RAM), o Nginx utiliza um **Event Loop**:

- Um único Worker pode gerenciar milhares de conexões simultaneamente.
- Quando uma requisição chega, o Worker inicia o processamento. Se ele precisar esperar por uma resposta do disco ou de um banco de dados, ele não fica parado; ele coloca essa tarefa em "espera" e parte para atender a próxima requisição.
- Assim que o recurso solicitado fica pronto, um "evento" avisa o Worker, que volta e finaliza aquela tarefa específica.

### 2.3 O problema do modelo Apache (thread-per-connection)

No modelo tradicional do Apache, cada conexão TCP recebe uma thread do SO dedicada:

- Cada thread ocupa **1 a 8 MB de stack** por padrão.
- 10.000 conexões simultâneas = até **8 GB de RAM** usados apenas para stacks de threads.
- O kernel passa a maior parte do tempo fazendo **context switches** entre threads, não executando trabalho real.
- Com a maioria das threads bloqueadas em I/O, a CPU fica ociosa mesmo com alta carga.

### 2.4 Como o event loop resolve isso

O Nginx usa `epoll` no Linux e `kqueue` no BSD/macOS para monitorar milhares de descritores de arquivo (sockets) simultaneamente com uma única chamada de sistema:

- `epoll_wait()` bloqueia até que pelo menos um fd esteja pronto (leitura, escrita ou erro).
- Retorna **apenas os fds que têm dados prontos** — sem varrer a lista inteira.
- O Worker processa cada fd pronto, sem nunca bloquear esperando por dados.
- Uma conexão ociosa não consome CPU — apenas um entry no kernel's event table.

Isso significa: **zero CPU gasto em conexões que estão aguardando** (cliente lento, banco de dados pensando, disco buscando arquivo).

### 2.5 Workers e CPUs

- O número recomendado de Workers é igual ao número de cores do servidor (`worker_processes auto;`).
- Cada Worker "pina" em um core (com `worker_cpu_affinity`), eliminando context switching entre Workers.
- Cada Worker mantém seu próprio event loop, sem compartilhar estado com os demais (lock-free por design).
- A memória compartilhada entre Workers (para rate limiting, upstream state, etc.) é acessada via `mmap`.

---

## 3. Principais Funções Teóricas

O Nginx é versátil e pode assumir diferentes papéis em uma infraestrutura:

### 3.1 Servidor de Conteúdo Estático

Ele é extremamente veloz para entregar arquivos que não mudam (HTML, CSS, JS, Imagens). Ele lê esses arquivos diretamente do disco e os envia ao navegador com o mínimo de processamento possível.

### 3.2 Proxy Reverso

Neste cenário, o Nginx fica "na frente" da sua aplicação (Node.js, Python, PHP). O cliente fala com o Nginx, e o Nginx fala com a aplicação. Isso traz benefícios como:

- **Segurança:** Esconde a identidade e a estrutura interna dos seus servidores.
- **Terminação SSL/TLS:** O Nginx lida com a criptografia, aliviando o peso da aplicação.

### 3.3 Balanceamento de Carga (Load Balancer)

Se você tem três servidores rodando a mesma aplicação, o Nginx distribui as requisições entre eles. Se um servidor cair, o Nginx percebe e para de enviar tráfego para ele, garantindo que o site continue no ar.

### 3.4 Cache de Conteúdo

O Nginx pode armazenar cópias de respostas de servidores lentos. Se um segundo usuário pedir a mesma informação, o Nginx entrega a cópia guardada instantaneamente, sem precisar consultar a aplicação novamente.

---

## 4. Por que usar o Nginx?

- **Alta Performance:** Capaz de lidar com volumes massivos de conexões com pouquíssima memória RAM.
- **Escalabilidade:** Facilita o crescimento de uma aplicação através do balanceamento de carga.
- **Flexibilidade:** Uma configuração simples pode transformar o servidor de um simples hospedeiro de site em um gateway de segurança complexo.
- **Estabilidade:** É conhecido por ser extremamente robusto, raramente precisando de reinicializações por falhas de software.

---

## Conexão com Sistemas Operacionais

O Nginx é um estudo vivo de como aproveitar as primitivas do SO de forma eficiente:

- **Modelo Master-Worker (fork):** O Master cria Workers via `fork()` — cada Worker herda os file descriptors abertos, incluindo o socket de escuta. Ver [[O Modelo de Processos]] e [[Criação de Processos]].
- **Processos independentes:** Cada Worker é um processo do SO com seu próprio espaço de endereçamento, tabela de fds e estado. Ver [[Processos]] e [[Estados de Processos]].
- **epoll / kqueue:** Mecanismo de notificação de eventos do kernel para I/O não bloqueante — o núcleo do event loop do Nginx. Ver [[Dispositivos de IO]] (multiplexação de I/O, epoll).
- **System calls no event loop:** `epoll_create`, `epoll_ctl`, `epoll_wait`, `read`, `write`, `accept` — todas chamadas de sistema invocadas continuamente pelo Worker. Ver [[System Calls]].
- **Thread-per-connection vs event loop:** O modelo Apache usa uma thread do kernel por conexão, com todo o custo de stack e context switch. Ver [[Implementando Threads em Kernel Space]] e [[Threads POSIX]].
- **Worker count = CPU cores:** Cada Worker idealmente roda em um core sem migrar. Ver [[Processadores]].
- **Memória compartilhada entre Workers:** Estado de rate limiting e upstream acessado via `mmap` entre processos. Ver [[Memória]] e [[Memória Virtual]].

---

## Conexão com Go

O Go resolve o mesmo problema (C10K) de uma forma diferente do Nginx, mas com raízes na mesma ideia:

- **[[Goroutines]]:** Go usa uma goroutine por conexão (pilha inicial de ~2KB, expansível), não uma thread do SO. O runtime do Go implementa seu próprio scheduler M:N (M goroutines em N threads), análogo ao event loop do Nginx mas com API síncrona para o programador.
- **netpoller do Go:** Internamente, o runtime Go usa `epoll` (Linux) / `kqueue` (BSD) para gerenciar I/O de rede — exatamente o mesmo mecanismo do Nginx. A diferença é que o Go esconde isso atrás de goroutines, enquanto o Nginx expõe explicitamente o event loop na configuração.
- **[[HTTP (net-http)]]:** `net/http` em Go cria uma goroutine por conexão aceita — a abstração de alto nível sobre o epoll subjacente. Para entender o que acontece "embaixo", o modelo Nginx é a referência direta.
- Nginx e Go coexistem naturalmente: Nginx como proxy reverso na frente de um servidor Go, tratando SSL, rate limiting e static files, enquanto o servidor Go processa a lógica de negócio.
