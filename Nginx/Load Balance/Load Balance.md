---
tags:
  - nginx
  - nginx/load-balance
---

# Load Balance

## 1. O que é Load Balancing?

**Load Balancing** (Balanceamento de Carga) é a técnica de distribuir o tráfego de rede ou requisições entre múltiplos servidores ou recursos para garantir que nenhum deles fique sobrecarregado. O objetivo principal é melhorar a **disponibilidade**, **desempenho** e **resiliência** de uma aplicação ou serviço.

---

## 2. Por que usar Load Balancing?

- **Alta Disponibilidade:** Se um servidor falhar, o balanceador redireciona o tráfego para os servidores restantes, evitando downtime.
- **Escalabilidade:** Permite adicionar mais servidores para lidar com o aumento de usuários sem degradar a performance.
- **Melhor Uso dos Recursos:** Evita que um servidor fique ocioso enquanto outro está sobrecarregado.
- **Redução de Latência:** Pode direcionar usuários para servidores mais próximos geograficamente ou menos carregados.

---

## 3. Como funciona o Load Balancing no Nginx?

O Nginx atua como um **balanceador de carga reverso**, recebendo as requisições dos clientes e distribuindo-as para um grupo de servidores backend (chamados de *upstreams*).

### Como os Workers compartilham o estado upstream

O estado do upstream (contagem de conexões ativas por servidor, flags de saúde, contadores de falha) é mantido em **memória compartilhada** (`ngx_slab_pool_t`) acessível por todos os Workers via `mmap`. Quando um Worker seleciona um backend para uma requisição, ele:

1. Adquire um lock na shared memory (operação atômica curta).
2. Lê o estado atual dos servidores (conexões ativas, peso, flags).
3. Aplica o algoritmo de balanceamento.
4. Incrementa o contador de conexões ativas do servidor escolhido.
5. Libera o lock.

Para Round Robin simples, o lock pode ser evitado com operações atômicas (`atomic_fetch_add`), tornando a seleção praticamente sem overhead.

---

## 4. Configuração Básica de Load Balancing no Nginx

### 4.1 Definindo o grupo de servidores (upstream)

```nginx
upstream backend {
    server 192.168.0.101;
    server 192.168.0.102;
    server 192.168.0.103;
}
```

Aqui, o Nginx sabe que o grupo chamado `backend` tem três servidores para distribuir as requisições.

### 4.2 Configurando o proxy para usar o grupo

```nginx
server {
    listen 80;
    server_name meusite.com;

    location / {
        proxy_pass http://backend;
    }
}
```

---

## 5. Métodos de Balanceamento de Carga no Nginx

O Nginx suporta vários métodos para decidir como distribuir as requisições:

| Método | Descrição |
| --- | --- |
| **Round Robin** | Distribui as requisições de forma sequencial e igualitária entre os servidores. (Padrão) |
| **Least Connections** | Envia a requisição para o servidor com o menor número de conexões ativas no momento. |
| **IP Hash** | Usa o IP do cliente para sempre direcionar para o mesmo servidor, útil para sessões persistentes. |

### Exemplo de configuração com Least Connections:

```nginx
upstream backend {
    least_conn;
    server 192.168.0.101;
    server 192.168.0.102;
}
```

---

## 6. Health Checks (Verificação de Saúde)

Para garantir que o Nginx não envie requisições para servidores que estão fora do ar, ele pode ser configurado para fazer **health checks** (verificações periódicas).

No Nginx Open Source, isso é limitado (passive health checks), mas no Nginx Plus (versão comercial) há suporte nativo para health checks automáticos.

Ver nota [[Servidores de Backup e Health Check]] para detalhes completos.

### Rastreamento de saúde na shared memory

Cada servidor no `upstream` tem uma entrada na shared memory com:
- `fails` — contador de falhas consecutivas.
- `accessed` — timestamp da última verificação.
- `checked` — timestamp do último teste.
- `down` — flag booleano de estado.

Todos os Workers leem e escrevem nessas entradas. Quando um Worker detecta uma falha (timeout, conexão recusada), ele incrementa `fails` atomicamente.

---

## 7. Resumo

- **Load Balancer:** Distribui requisições para múltiplos servidores.
- **Benefícios:** Alta disponibilidade, escalabilidade, melhor uso dos recursos.
- **Nginx:** Atua como balanceador reverso com métodos como Round Robin, Least Connections e IP Hash.
- **Configuração:** Usa blocos `upstream` para definir servidores e `proxy_pass` para encaminhar requisições.
- **Health Checks:** Importante para evitar enviar tráfego a servidores indisponíveis.

---

## Conexão com Sistemas Operacionais

O load balancing é um problema de coordenação entre processos sobre recursos compartilhados — território central de SO:

- **Shared memory entre Workers:** O estado do upstream (conexões ativas, flags de saúde) é uma região de memória acessada por múltiplos processos do SO simultaneamente via `mmap`. Ver [[Memória Virtual]] (shared memory, mmap) e [[Processos]].
- **Workers independentes sem coordenador:** Cada Worker é um processo do SO com seu próprio event loop. A coordenação acontece apenas via shared memory + mutexes atômicos — sem um processo coordenador central. Ver [[O Modelo de Processos]] e [[Processos]].
- **Contagem de conexões ativas:** Cada Worker incrementa/decrementa contadores na shared memory ao abrir/fechar conexões com backends — requer operações atômicas para evitar race conditions. Ver [[Memória]] e [[Processadores]] (operações atômicas).
- **Flags de saúde por servidor:** Estado de cada backend armazenado em shared memory, atualizado por qualquer Worker que detecte falha. Ver [[Estados de Processos]] (analogia com estados de processos) e [[Processos]].
- **DNS resolution para upstreams:** Nginx resolve nomes de hostname de backends via `getaddrinfo()` — system call bloqueante que o Nginx executa no startup (ou periodicamente com `resolver`). Ver [[System Calls]].

---

## Conexão com Go

- **[[HTTP (net-http)]]:** `httputil.ReverseProxy` com um `Director` customizado implementa load balancing em Go. O `Director` escolhe o backend antes de cada requisição, permitindo implementar round-robin, least-connections ou qualquer outro algoritmo.
- **[[sync.WaitGroup e sync.Mutex]]:** Um round-robin thread-safe em Go requer um mutex para proteger o contador compartilhado entre goroutines: `mu.Lock(); idx = (idx+1) % len(backends); mu.Unlock()`. O mesmo problema que o Nginx resolve com atomic ops na shared memory.
- **[[Goroutines]]:** Em Go, o load balancer pode ser implementado sem locks usando um `channel` como fila de backends — cada goroutine pega o próximo backend disponível do channel, e o channel garante serialização sem mutex explícito.
- **[[context.Context]]:** Propagando `context.Context` com deadline através do proxy, o load balancer pode cancelar automaticamente requisições a backends lentos e tentar outro — failover a nível de código.
