---
tags:
  - nginx
  - nginx/performance
---

# Conexões, Workers e Keep-Alive

## Conexões, Workers e Keep-Alive no Nginx

Para entender performance no Nginx, você precisa entender como ele gerencia o trabalho internamente. Tudo começa nos **processos**, passa pelas **conexões** e é otimizado pelo **Keep-Alive**.

---

## 1. Worker Processes

### O que é um Worker Process?

Como vimos no início dos estudos, o Nginx opera com um modelo **Master + Workers**.

- o **Master** gerencia configuração e controla os Workers;
- os **Workers** fazem o trabalho real: aceitam conexões, leem arquivos, encaminham requisições.

Cada Worker é um processo independente do sistema operacional.

---

### Quantos Workers usar?

A recomendação padrão é:

> um Worker por núcleo (core) de CPU

A lógica é simples:

- cada Worker usa um núcleo;
- ter mais Workers que núcleos gera troca de contexto desnecessária;
- ter menos Workers que núcleos desperdiça capacidade de processamento.

---

### Como o Worker aceita conexões?

Cada Worker tem um **Event Loop** próprio.

Esse Event Loop:

- monitora eventos de rede;
- aceita novas conexões;
- lê dados recebidos;
- escreve respostas;
- gerencia timers e timeouts.

Tudo isso de forma **assíncrona e não-bloqueante**.

Ou seja, um único Worker pode gerenciar **milhares de conexões simultâneas** sem criar uma thread por conexão.

---

### Worker Connections

Cada Worker tem um limite de conexões simultâneas que pode gerenciar.

Isso é controlado pela diretiva `worker_connections`.

A capacidade total de conexões do servidor é:

```
total = worker_processes × worker_connections
```

Exemplo:

- 4 Workers
- 1024 conexões por Worker
- total = 4096 conexões simultâneas

---

### Exemplo de configuração

```
worker_processes auto;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}
```

---

### Explicando as diretivas

### `worker_processes auto;`

O Nginx detecta automaticamente o número de núcleos e cria um Worker por núcleo.

### `worker_connections 1024;`

Cada Worker pode gerenciar até 1024 conexões simultâneas.

### `use epoll;`

Define o mecanismo de notificação de eventos do sistema operacional.

O `epoll` é o mais eficiente no Linux moderno.

Outros mecanismos existem para outros sistemas:

- `kqueue` → FreeBSD / macOS
- `select` → mais antigo, menos eficiente
- `poll` → similar ao select

O `epoll` é o padrão recomendado para Linux.

### `multi_accept on;`

Permite que o Worker aceite múltiplas conexões de uma vez quando elas chegam simultaneamente.

Sem isso, o Worker aceita uma conexão por vez mesmo que várias estejam esperando.

---

## 2. O que é uma Conexão HTTP?

Antes de falar de Keep-Alive, é importante entender o que é uma conexão no contexto HTTP.

### Conexão TCP

O HTTP é construído sobre o protocolo TCP.

Para que um navegador possa enviar uma requisição HTTP, ele precisa primeiro estabelecer uma **conexão TCP** com o servidor.

Esse processo é chamado de **TCP Handshake** e envolve:

1. cliente envia `SYN`
2. servidor responde `SYN-ACK`
3. cliente confirma com `ACK`

Só depois disso a requisição HTTP pode ser enviada.

---

### O problema do HTTP/1.0

No HTTP/1.0, o comportamento padrão era:

- abre conexão TCP;
- envia requisição;
- recebe resposta;
- fecha conexão.

Para cada recurso de uma página (HTML, CSS, JS, imagem), esse ciclo se repetia.

Isso significa:

- múltiplos handshakes TCP;
- múltiplos ciclos de abertura e fechamento;
- latência acumulada;
- alto custo de recursos.

---

## 3. Keep-Alive

### O que é Keep-Alive?

**Keep-Alive** é um mecanismo que permite **reutilizar a mesma conexão TCP** para múltiplas requisições HTTP.

Em vez de abrir e fechar uma conexão para cada recurso, o cliente e o servidor mantêm a conexão aberta por um tempo.

Isso elimina o custo repetido do handshake TCP.

---

### Como funciona?

1. cliente abre conexão TCP com o servidor;
2. envia requisição HTTP;
3. servidor responde;
4. **a conexão permanece aberta**;
5. cliente envia outra requisição pela mesma conexão;
6. servidor responde novamente;
7. isso se repete até que o tempo limite expire ou um dos lados feche.

---

### Benefícios do Keep-Alive

### 1. Redução de latência

Elimina o handshake TCP repetido para cada recurso.

### 2. Redução de uso de CPU

Menos conexões sendo abertas e fechadas.

### 3. Melhor throughput

Mais dados podem ser transferidos em menos tempo.

### 4. Melhor experiência do usuário

Páginas carregam mais rápido.

---

### Keep-Alive no HTTP/1.1

No HTTP/1.1, o Keep-Alive é o comportamento **padrão**.

Ou seja, a conexão é mantida aberta por padrão, a menos que um dos lados envie:

```
Connection: close
```

---

### Keep-Alive no HTTP/2

No HTTP/2, o conceito evoluiu ainda mais com **multiplexação**.

Em vez de apenas reutilizar a conexão sequencialmente, o HTTP/2 permite enviar múltiplas requisições **ao mesmo tempo** pela mesma conexão.

Isso é chamado de **multiplexing** e elimina o problema de **Head-of-Line Blocking** do HTTP/1.1.

---

## 4. Keep-Alive no Nginx

O Nginx tem dois contextos de Keep-Alive:

### A. Keep-Alive com o cliente (downstream)

Entre o navegador e o Nginx.

### B. Keep-Alive com o backend (upstream)

Entre o Nginx e a aplicação (Node.js, PHP-FPM, etc.).

Esses dois são configurados de formas diferentes e têm impactos diferentes.

---

### A. Keep-Alive com o cliente

```
http {
    keepalive_timeout 65;
    keepalive_requests 100;
}
```

---

### `keepalive_timeout 65;`

Define por quantos segundos o Nginx mantém a conexão aberta aguardando novas requisições do cliente.

- se o cliente não enviar nada em 65 segundos, a conexão é fechada;
- valor muito alto → conexões abertas desnecessariamente, consumindo recursos;
- valor muito baixo → perde o benefício do Keep-Alive.

---

### `keepalive_requests 100;`

Define quantas requisições podem ser feitas pela mesma conexão antes de ela ser fechada.

Isso evita que uma única conexão fique aberta indefinidamente.

---

### B. Keep-Alive com o backend (upstream)

Esse é um ponto muito importante e frequentemente esquecido.

Quando o Nginx encaminha requisições para um backend, ele também pode manter conexões persistentes com esse backend.

Sem isso, o Nginx abre e fecha uma conexão TCP com o backend a cada requisição.

```
upstream meu_backend {
    server 127.0.0.1:3000;
    keepalive 32;
}

server {
    location / {
        proxy_pass http://meu_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

---

### `keepalive 32;`

Mantém até 32 conexões persistentes com o backend no pool de cada Worker.

Isso reduz drasticamente o overhead de conexão entre Nginx e aplicação.

### `proxy_http_version 1.1;`

O Keep-Alive com upstream requer HTTP/1.1.

Por padrão, o Nginx usa HTTP/1.0 para comunicação com backends.

### `proxy_set_header Connection "";`

Remove o header `Connection: close` que o HTTP/1.0 envia por padrão.

Sem isso, o backend fecharia a conexão após cada resposta.

---

## 5. Relação entre Workers, Conexões e Keep-Alive

Esses três conceitos se conectam diretamente.

### Cenário sem Keep-Alive

- cada requisição abre uma nova conexão TCP;
- o Worker precisa gerenciar o ciclo completo de abertura e fechamento;
- mais overhead por requisição;
- mais uso de recursos.

### Cenário com Keep-Alive

- uma conexão TCP serve múltiplas requisições;
- o Worker mantém a conexão no Event Loop;
- menos overhead;
- mais eficiência.

---

### Impacto no número de conexões simultâneas

Com Keep-Alive, uma conexão pode ficar aberta mesmo sem estar transferindo dados ativamente.

Isso significa que o número de conexões abertas pode ser maior que o número de requisições ativas.

Por isso, o valor de `worker_connections` precisa ser pensado levando em conta conexões Keep-Alive ociosas.

---

## 6. Outros parâmetros importantes de conexão

### `send_timeout`

Tempo máximo para enviar dados ao cliente.

Se o cliente parar de receber dados por esse tempo, a conexão é fechada.

```
send_timeout 30s;
```

---

### `client_header_timeout`

Tempo máximo para receber os headers da requisição do cliente.

```
client_header_timeout 15s;
```

---

### `client_body_timeout`

Tempo máximo para receber o corpo da requisição.

Útil para uploads ou requisições POST grandes.

```
client_body_timeout 15s;
```

---

### `reset_timedout_connection on;`

Quando uma conexão expira por timeout, o Nginx envia um pacote TCP RST para liberar os recursos mais rapidamente.

```
reset_timedout_connection on;
```

---

## 7. Exemplo prático completo

```
worker_processes auto;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    # Keep-Alive com o cliente
    keepalive_timeout 65;
    keepalive_requests 100;

    # Timeouts de conexão
    client_header_timeout 15s;
    client_body_timeout 15s;
    send_timeout 30s;
    reset_timedout_connection on;

    upstream meu_backend {
        server 127.0.0.1:3000;
        keepalive 32;
    }

    server {
        listen 80;
        server_name meusite.com;

        location / {
            proxy_pass http://meu_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

---

## 8. Resumo

### Workers

- processos independentes que fazem o trabalho real;
- um por núcleo de CPU é o ideal;
- cada Worker tem um Event Loop assíncrono;
- cada Worker gerencia milhares de conexões simultâneas.

### Worker Connections

- limite de conexões por Worker;
- total do servidor = `worker_processes × worker_connections`.

### Keep-Alive com cliente

- reutiliza conexão TCP entre navegador e Nginx;
- controlado por `keepalive_timeout` e `keepalive_requests`.

### Keep-Alive com backend

- reutiliza conexão TCP entre Nginx e aplicação;
- controlado por `keepalive` no bloco `upstream`;
- requer `proxy_http_version 1.1` e `proxy_set_header Connection ""`.

### Timeouts

- evitam que conexões travadas consumam recursos indefinidamente.

---

### Frase para memorizar

**Workers definem quantas conexões o Nginx pode gerenciar. Keep-Alive define quantas requisições cada conexão pode carregar. Timeouts definem por quanto tempo o Nginx espera antes de desistir.**

---

## Conexão com Sistemas Operacionais

- **worker_processes auto: um worker por core** — cada worker process é fixado a um núcleo de CPU (com `worker_cpu_affinity auto`). Ter mais workers que cores gera context switches desnecessários: o scheduler do kernel precisa salvar e restaurar o estado do processo (registradores, stack pointer, TLB flush) a cada troca → [[Processadores]]

- **worker_connections e file descriptor limit** — cada conexão ativa consome um file descriptor no processo worker. O limite total de fds por processo é configurado via `ulimit -n` (ou `worker_rlimit_nofile` no Nginx). Se o sistema tiver muitas conexões simultâneas e o limit de fds for baixo, novas conexões serão recusadas → [[Processos]]

- **Event loop: epoll_wait()** — cada worker chama `epoll_wait()` que bloqueia até que algum fd (socket) esteja pronto para I/O. O kernel mantém uma lista de fds monitorados e notifica o worker apenas quando há dados disponíveis. Isso elimina polling e permite gerenciar milhares de fds com custo O(1) → [[Dispositivos de IO]]

- **Non-blocking sockets: O_NONBLOCK** — todos os sockets gerenciados pelo Nginx têm a flag `O_NONBLOCK` ativada via `fcntl()`. Com isso, `read()` e `write()` retornam imediatamente com `EAGAIN` se não há dados prontos, em vez de bloquear o worker inteiro → [[System Calls]]

- **Keep-Alive: economia de handshake TCP e TLS** — sem Keep-Alive, cada requisição exige TCP 3-way handshake (~1.5 RTT) e, com HTTPS, TLS handshake (~2 RTT adicionais). Com Keep-Alive, esses custos são pagos uma vez por conexão e amortizados por dezenas de requisições → [[System Calls]]

- **keepalive_timeout 65: fechamento pelo Nginx** — após 65s de inatividade, o Nginx chama `close()` no fd da conexão. O kernel envia um pacote TCP FIN ao cliente e remove o socket da lista do epoll. Analogia com ciclo de vida de recursos que precisam ser liberados → [[Término de Processos]]

- **SO_REUSEPORT: distribuição de conexões pelo kernel** — com `SO_REUSEPORT`, cada worker abre seu próprio socket no mesmo endereço/porta. O kernel usa um BPF program para distribuir novas conexões entre os sockets dos workers, eliminando contenção no `accept()` → [[System Calls]], [[Processadores]]

---

## Conexão com Go

- **Goroutines vs event loop: mesmo modelo subjacente** — o runtime do Go usa um netpoller baseado em `epoll` (Linux) internamente. As goroutines ficam suspensas quando aguardam I/O de rede; o runtime as reacorda quando o epoll notifica. Do ponto de vista do kernel, o comportamento é equivalente ao event loop do Nginx, mas a API para o programador é muito mais simples (goroutine por conexão vs callbacks) → [[Goroutines]], [[Implementando Threads em User Space]]
