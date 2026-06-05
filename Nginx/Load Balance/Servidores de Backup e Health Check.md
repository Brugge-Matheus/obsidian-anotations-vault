---
tags:
  - nginx
  - nginx/load-balance
---

# Servidores de Backup e Health Check

## 1. Resiliência e Monitoramento no Nginx

Quando trabalhamos com múltiplos servidores (Upstream), o Nginx precisa saber se eles estão "vivos" e o que fazer se um deles parar de responder. Para isso, usamos diretivas específicas dentro do bloco `upstream`.

---

## 2. Diretivas de Resiliência

### 2.1 Max Fails (`max_fails`)

É o "limite de paciência" do Nginx. Esta diretiva define o número de tentativas de comunicação mal sucedidas que devem ocorrer em um determinado período antes que o Nginx considere o servidor como **indisponível**.

- **Padrão:** 1 tentativa.
- **Teoria:** Se o Nginx tentar enviar uma requisição e o servidor der erro (ou timeout), ele conta 1 falha. Se chegar ao número definido em `max_fails`, ele marca o servidor como "caído".

### Como funciona internamente

O contador de falhas por servidor é armazenado na **shared memory** do upstream — um campo `fails` na entrada desse servidor. Quando um Worker detecta uma falha de conexão (errno `ECONNREFUSED`, `ETIMEDOUT`, etc.) ou recebe um erro HTTP configurado como fatal, ele:

1. Adquire o lock da shared memory.
2. Incrementa `peer->fails`.
3. Compara com `max_fails`.
4. Se `peer->fails >= max_fails`, seta `peer->down = 1` e registra o timestamp em `peer->checked`.

Todos os outros Workers leem o campo `down` antes de selecionar um backend — servidor marcado como `down` é pulado na seleção.

---

### 2.2 Fail Timeout (`fail_timeout`)

Esta diretiva tem dois papéis simultâneos e complementares:

1. **Janela de tempo:** Define o tempo durante o qual o número de `max_fails` deve ocorrer para o servidor ser considerado indisponível.
2. **Tempo de castigo:** Define por quanto tempo o Nginx deixará de enviar requisições para aquele servidor após ele ter sido marcado como indisponível.

- **Exemplo:** Se `max_fails=3` e `fail_timeout=30s`, o servidor precisa falhar 3 vezes em 30 segundos para ser removido. Após ser removido, o Nginx esperará 30 segundos antes de tentar enviar uma nova requisição para ele (para ver se ele "acordou").

### Retomada automática

Após o `fail_timeout` expirar, o Nginx não envia um probe ativo — em vez disso, **a próxima requisição real de um usuário** é usada como teste. Se essa requisição for bem-sucedida, `peer->fails` é zerado e o servidor volta ao pool normalmente. Se falhar, o `fail_timeout` se reinicia.

---

### 2.3 Servidores de Backup (`backup`)

O parâmetro `backup` marca um servidor como "reserva de emergência".

- **Funcionamento:** O Nginx **nunca** envia tráfego para o servidor de backup enquanto houver pelo menos um servidor principal online e saudável.
- **Cenário de uso:** Se todos os seus servidores principais atingirem o `max_fails`, o Nginx ativa o servidor de backup instantaneamente. É ideal para exibir uma página de erro amigável ou uma versão simplificada do site hospedada em outro lugar.

### Como o estado de backup é rastreado

Na shared memory do upstream, cada servidor tem um campo `backup` (flag booleano). O algoritmo de seleção verifica primeiro se há pelo menos um servidor não-backup ativo:

- Se sim: ignora todos os servidores com `backup = 1`.
- Se não (todos primários `down`): inclui os backups na seleção, aplicando o mesmo algoritmo de balanceamento entre eles.

Os contadores de falha por servidor são mantidos independentemente para primários e backups.

---

### 2.4 Servidores Desativados (`down`)

Marca um servidor como permanentemente indisponível.

- **Teoria:** É usado para manutenção manual. Em vez de apagar a linha do servidor da configuração, você apenas adiciona `down`. O Nginx ignora esse servidor completamente até que você remova a flag.

---

## 3. Health Checks (Verificação de Saúde)

Existem dois tipos de verificações de saúde no mundo dos balanceadores de carga:

### 3.1 Passive Health Checks (Nginx Open Source)

É o que explicamos acima com `max_fails` e `fail_timeout`.

- **Como funciona:** O Nginx só percebe que o servidor caiu **quando um usuário tenta acessá-lo e falha**. Ou seja, pelo menos um usuário (ou o número definido em `max_fails`) terá uma experiência ruim antes do Nginx agir.
- **Vantagem:** Não gera tráfego extra entre o Nginx e o Backend.

### Máquina de estados por servidor (Passive)

O ciclo de vida de um servidor no upstream pode ser modelado como uma máquina de estados:

```
        falha detectada              fail_timeout expira
UP  ─────────────────────────► DOWN ──────────────────────► PROBANDO
 ▲                                                              │
 └──────────────────── requisição bem-sucedida ─────────────────┘
```

- **UP:** Servidor saudável, recebe tráfego normalmente.
- **DOWN:** Marcado como indisponível após `max_fails` falhas dentro de `fail_timeout`.
- **PROBANDO:** Após `fail_timeout` expirar, a próxima requisição real testa o servidor. Se ok → UP. Se falha → DOWN novamente.

---

### 3.2 Active Health Checks (Nginx Plus / Comercial)

O Nginx envia requisições de teste especiais (ex: `GET /health`) para os servidores em intervalos regulares, **independente de haver usuários acessando**.

- **Como funciona:** Se o servidor parar de responder ao teste, o Nginx o remove do rodízio **antes** que um usuário real tente acessá-lo.
- **Vantagem:** Proativo. O usuário nunca chega a "sentir" a falha do servidor.

### Como funciona internamente (Active Health Check)

Em Nginx Plus, existe um **processo de health check** (ou timer no event loop de cada Worker) que:

1. Em intervalos configurados (`interval=5s`), faz `connect()` + `send(GET /health HTTP/1.1)` a cada backend.
2. Aguarda resposta no event loop (não bloqueante).
3. Avalia o status HTTP e o body (se `match` configurado).
4. Atualiza o estado do servidor na shared memory.

É um uso do mesmo event loop do Worker, com timers adicionados via `ngx_add_timer()` — timers são entradas na árvore rubro-negra de eventos temporizados do Nginx.

---

## 4. Exemplo de Configuração

```nginx
upstream meu_cluster {
    # Servidor principal 1: tolera 3 falhas em 30s
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;

    # Servidor principal 2: mais potente (peso 2)
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s weight=2;

    # Servidor em manutenção manual
    server 10.0.0.3:8080 down;

    # Servidor de emergência: só assume se o 1 e o 2 caírem
    server 10.0.0.4:8080 backup;
}
```

---

## 5. Resumo para Revisão

- **`max_fails`**: Quantas vezes pode errar antes de ser removido.
- **`fail_timeout`**: Tempo de observação das falhas e tempo de espera para tentar de novo.
- **`backup`**: O "estepe" do carro; só entra em campo na emergência.
- **`down`**: Manutenção programada; o Nginx finge que ele não existe.
- **Passive vs Active**: O passivo reage ao erro do usuário; o ativo testa o servidor preventivamente.

---

## Conexão com Sistemas Operacionais

Resiliência e health checks são fundamentalmente sobre gerenciamento de estado entre processos e detecção de falhas:

- **Estado de saúde em shared memory:** Flags `down`, contadores `fails`, timestamps `checked` — todos campos em shared memory acessados por múltiplos Workers. Requer sincronização com operações atômicas. Ver [[Memória]] e [[Processos]].
- **Servidor de backup e failover:** Quando todos os primários estão `down`, o Nginx ativa backups — transição de estado gerenciada pela shared memory, sem nenhum processo coordenador central. Ver [[Processos]].
- **Passive health check como máquina de estados:** UP → DOWN → PROBANDO → UP. A transição é disparada por erros de syscall (`ECONNREFUSED`, `ETIMEDOUT`) nos Workers. Ver [[Estados de Processos]] (analogia direta: estados de backend como estados de processo).
- **Active health check com timers no event loop:** Timer adicionado à árvore de eventos do Worker que dispara `connect()` + HTTP request periodicamente — I/O assíncrono não bloqueante. Ver [[Dispositivos de IO]] (event loop, timers, I/O assíncrono).
- **Detecção de falha via errno:** O Worker detecta backend indisponível quando `connect()` retorna `ECONNREFUSED` (porta fechada) ou quando `read()` retorna 0 (conexão encerrada) ou a conexão expira. Ver [[System Calls]] (códigos de erro, errno).

---

## Conexão com Go

Em Go, os mesmos padrões de resiliência podem ser implementados com primitivas da linguagem:

- **[[sync.WaitGroup e sync.Mutex]]:** Um health checker em Go mantém um `map[string]bool` de backends saudáveis protegido por `sync.RWMutex` — múltiplas goroutines leem (selecionar backend) enquanto o checker escreve (atualizar estado). Análogo à shared memory com locks do Nginx.
- **[[Goroutines]]:** O active health checker em Go é uma goroutine que roda em loop com `time.Ticker`: a cada intervalo, faz requisições HTTP para cada backend e atualiza o mapa de estado. Enquanto o Nginx usa timers no event loop, Go usa goroutines com `time.Sleep`/`time.Ticker`.
- **[[context.Context]]:** Cada probe de health check usa `context.WithTimeout` para garantir que o teste não fique pendurado indefinidamente — se o backend não responder em N segundos, o context é cancelado e o backend é marcado como down.
- **[[HTTP (net-http)]]:** `http.Get("http://backend/health")` é a forma Go de fazer o probe ativo. O `http.Client` tem `Timeout` configurável — equivalente ao `fail_timeout` do Nginx para o probe.
- **[[Channels]]:** Um channel pode ser usado para notificar o seletor de backends quando o estado muda: o health checker envia no channel, o seletor recebe e atualiza sua visão dos backends disponíveis — comunicação assíncrona entre goroutines sem mutex.
