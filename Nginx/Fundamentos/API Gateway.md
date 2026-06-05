---
tags:
  - nginx
  - nginx/fundamentos
---

# API Gateway

## 1. O que é um API Gateway?

Em uma arquitetura moderna, você pode ter dezenas de microserviços (um para Usuários, outro para Pagamentos, outro para Estoque). O cliente (Mobile ou Web) não deve falar com cada um deles individualmente.

O **API Gateway** é o ponto de entrada único que aceita todas as chamadas de API, agrega os serviços necessários e devolve o resultado. O Nginx é uma das ferramentas mais populares do mundo para cumprir esse papel.

Em termos mais precisos: um API Gateway é um **proxy reverso com lógica ativa**. Enquanto um proxy reverso simples apenas encaminha requisições (passivo), o API Gateway executa lógica de negócio na borda: autenticação, rate limiting, transformação de dados, roteamento condicional.

---

## 2. Funções Principais de um API Gateway no Nginx

Diferente de um simples proxy, o API Gateway executa tarefas lógicas complexas:

### 2.1 Roteamento Inteligente (Routing)

O Nginx analisa a URL e decide para qual microserviço enviar a requisição.

- `api.meusite.com/usuarios` → Vai para o Microserviço A.
- `api.meusite.com/pagamentos` → Vai para o Microserviço B.

O matching de `location` no Nginx é compilado na inicialização em estruturas de dados otimizadas:
- **Matches exatos** (`location = /health`): armazenados em hash table, lookup O(1).
- **Matches de prefixo** (`location /api/`): árvore de prefixos.
- **Matches por regex** (`location ~ ^/api/v[0-9]+/`): lista encadeada percorrida em ordem de definição.

O Nginx prioriza match exato > prefixo mais longo sem regex > regex.

### 2.2 Autenticação e Autorização Centralizada

Em vez de cada microserviço ter que validar se o usuário está logado, o **Nginx faz isso na porta de entrada**.

- Ele verifica tokens (como JWT) ou chaves de API.
- Se o token for inválido, o Nginx bloqueia a requisição ali mesmo, economizando processamento dos seus servidores internos.
- Com `auth_request`, o Nginx faz uma **subrequisição** para um microserviço de autenticação antes de encaminhar a requisição original. Se a subrequisição retornar 2xx, a requisição é liberada; qualquer outro código, e o Nginx retorna 401/403 diretamente ao cliente.

### 2.3 Limitação de Taxa (Rate Limiting)

Protege sua infraestrutura de abusos ou ataques de força bruta.

- **Exemplo:** Você pode configurar o Nginx para permitir apenas 10 requisições por segundo para cada endereço IP. Se o usuário exceder isso, ele recebe um erro `429 Too Many Requests`.

O rate limiting usa um **algoritmo de token bucket** (ou leaky bucket, dependendo da diretiva):

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
        }
    }
}
```

- `limit_req_zone` define uma zona de memória compartilhada (`10m` = 10 megabytes) rastreada entre todos os Workers via `mmap`.
- Cada endereço IP tem seu contador de tokens armazenado nessa zona — Workers atualizam atomicamente.
- `burst=20` permite picos de até 20 requisições acima da taxa, `nodelay` processa o burst sem atrasar.

### 2.4 Transformação de Requisições

O Gateway pode modificar a requisição antes de entregá-la ao serviço.

- Adicionar headers de segurança.
- Converter formatos (ex: o cliente envia JSON, mas o serviço legado interno só aceita XML).

### 2.5 Agregação de Respostas (Offloading)

O Nginx pode "juntar" informações. Se uma página precisa de dados de 3 serviços diferentes, o Gateway pode fazer as 3 chamadas internamente e entregar um único pacote de dados ao cliente, reduzindo a latência na rede do usuário.

---

## 3. Estrutura Teórica de Configuração

No Nginx, usamos o bloco `upstream` para definir os grupos de serviços e o bloco `location` para o roteamento:

```nginx
# Definição dos grupos de microserviços
upstream servico_usuarios {
    server 10.0.0.10:8080;
}

upstream servico_pagamentos {
    server 10.0.0.11:9090;
}

server {
    listen 80;
    server_name api.meusite.com;

    # Roteamento para Usuários
    location /v1/usuarios {
        proxy_pass http://servico_usuarios;
        # Aqui entraria a lógica de autenticação
    }

    # Roteamento para Pagamentos
    location /v1/pagamentos {
        proxy_pass http://servico_pagamentos;
        # Aqui entraria o Rate Limiting
    }
}
```

---

## 4. Proxy Reverso vs. API Gateway

| Característica | Proxy Reverso Simples | API Gateway (Nginx) |
| --- | --- | --- |
| **Foco** | Encaminhamento de tráfego | Gerenciamento de APIs |
| **Complexidade** | Baixa (1 para 1) | Alta (1 para N serviços) |
| **Segurança** | SSL e Ocultação de IP | JWT, OAuth, Rate Limit |
| **Lógica** | Passiva | Ativa (modifica requisições) |

---

## 5. Por que usar Nginx como Gateway?

- **Performance Extrema:** Ele processa milhares de regras de roteamento com latência quase zero.
- **Consolidação:** Você não precisa de uma ferramenta extra (como Kong ou Tyk) se suas necessidades de Gateway forem atendidas pelo Nginx (que você provavelmente já usa).
- **Observabilidade:** Centraliza os logs de acesso de toda a sua arquitetura em um único lugar.

---

## Conexão com Sistemas Operacionais

O API Gateway no Nginx faz uso pesado de mecanismos do SO para entregar sua funcionalidade:

- **Rate limiting com shared memory:** A zona `limit_req_zone` é uma região de memória compartilhada entre todos os Workers via `mmap`. Workers distintos (processos separados do SO) precisam de sincronização para atualizar contadores — o Nginx usa mutexes em shared memory. Ver [[Memória Virtual]] (shared memory, mmap) e [[Processos]].
- **Location matching compilado:** As tabelas de hash para match exato e a árvore de prefixos são alocadas na inicialização e consultadas para cada requisição com overhead mínimo. Ver [[Processadores]] (hash tables, pattern matching eficiente).
- **auth_request como subrequisição:** O Nginx abre uma conexão TCP adicional ao serviço de auth — mais um par de sockets gerenciados pelo Worker. Ver [[System Calls]] (`connect`, `read`, `write`).
- **Workers sem coordenação central:** Cada Worker aplica rate limiting consultando a shared memory — não há um processo coordenador central. O design é intencional: evitar lock contention. Ver [[Processos]] e [[O Modelo de Processos]].

---

## Conexão com Go

- **[[HTTP (net-http)]]:** Em Go, um API Gateway é implementado como uma cadeia de middlewares sobre `net/http`. Cada middleware é uma função que envolve o próximo handler — roteamento, autenticação, rate limiting, logging, tudo como `http.Handler`.
- **[[Interfaces]]:** `http.Handler` é a interface que conecta toda a cadeia. Qualquer struct que implemente `ServeHTTP(ResponseWriter, *Request)` pode ser inserido na cadeia — é o padrão de composição do Go.
- **[[context.Context]]:** Em um API Gateway Go, o `context.Context` propaga o deadline da requisição e dados de autenticação ao longo de toda a cadeia de handlers e chamadas a microserviços downstream — equivalente ao que o Nginx faz configurativamente.
- **[[sync.WaitGroup e sync.Mutex]]:** Rate limiting em Go requer um mutex para proteger contadores compartilhados entre goroutines — o mesmo problema que o Nginx resolve com shared memory + mutex entre Workers.
- **[[Goroutines]]:** Cada requisição recebida corre em sua própria goroutine, permitindo que o servidor processe milhares de requisições simultaneamente sem o custo de threads do SO.
