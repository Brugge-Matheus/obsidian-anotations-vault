---
tags:
  - nginx
  - nginx/cgi
---

# Processos e Forks no CGI/FastCGI

### Como o FastCGI reaproveitava processos?

Você chegou muito perto com a ideia de fork, mas a técnica é um pouco diferente. Vamos entender:

### O CGI clássico

O fluxo era:

- chega requisição;
- servidor faz `fork()` + `exec()` para criar um processo filho;
- esse processo carrega o interpretador do zero;
- executa o script;
- morre.

O problema não era só o fork em si. O custo maior era o **`exec()`**, que substituía o processo filho por um novo programa, carregando o interpretador inteiro na memória do zero a cada requisição.

---

### O que o FastCGI fez diferente?

O FastCGI **não usa fork por requisição**. A técnica usada foi diferente:

- os processos são criados **uma única vez** na inicialização;
- eles ficam rodando continuamente em um **loop de espera**;
- quando uma requisição chega, o servidor web envia os dados para um processo já vivo via **socket**;
- o processo processa, responde e volta ao loop de espera;
- o processo **nunca morre** entre requisições (a menos que seja reciclado por política do gerenciador).

Então a técnica não é exatamente fork por requisição, mas sim:

> **processos de longa duração que ficam em loop aguardando trabalho via socket**

Isso é chamado de **persistent process model**.

---

### E o fork aparece onde no FastCGI?

O fork aparece apenas no momento de **inicialização** ou quando o gerenciador precisa **escalar** o número de workers.

Por exemplo, se você configurou o PHP-FPM para ter entre 5 e 20 workers dinâmicos:

- na inicialização, ele faz fork para criar os 5 iniciais;
- se o tráfego aumentar, ele faz fork para criar mais;
- se o tráfego cair, ele mata os excedentes.

Mas durante o processamento normal, **não há fork por requisição**. O processo já existe e apenas recebe o trabalho.

---

### E a memória compartilhada?

Aqui está um detalhe importante que você tocou indiretamente.

No CGI clássico, cada processo carregava tudo do zero:

- interpretador;
- extensões;
- configurações;
- código da aplicação.

No FastCGI com processos persistentes, o processo já tem tudo isso carregado em memória desde o início. Então:

- o interpretador já está na memória;
- as extensões já estão carregadas;
- as configurações já foram lidas;
- em alguns casos, o próprio código da aplicação já está em cache de opcode.

Isso elimina o custo de inicialização a cada requisição.

---

### O FastCGI trabalha com paralelismo?

Sim, mas de uma forma específica. É importante entender o modelo.

### O PHP em si é single-threaded por padrão

Cada processo PHP executa **uma requisição por vez**, de forma sequencial.

Não há paralelismo dentro de um único processo PHP.

### O paralelismo no FastCGI vem dos múltiplos workers

O PHP-FPM resolve isso com **múltiplos processos paralelos**:

- Worker 1 → atende requisição A;
- Worker 2 → atende requisição B;
- Worker 3 → atende requisição C;
- todos ao mesmo tempo.

Então o paralelismo é alcançado por **múltiplos processos**, não por múltiplas threads dentro de um processo.

### Existe suporte a threads no PHP?

Sim, existe uma extensão chamada **pthreads** e mais recentemente o **parallel**, mas:

- não é o modelo padrão;
- tem limitações;
- não é amplamente usado em produção web;
- a maioria das aplicações PHP usa o modelo multi-processo do PHP-FPM.

### Resumo do paralelismo no FastCGI

| Nível | Como funciona |
| --- | --- |
| **Dentro de um processo PHP** | Single-threaded, sequencial |
| **Entre processos PHP-FPM** | Paralelo, múltiplos workers simultâneos |
| **Gerenciamento** | PHP-FPM controla quantos workers existem |

---

### Resumo

### Sobre o FastCGI e processos

- não usa fork por requisição;
- usa processos persistentes em loop de espera;
- fork acontece apenas na inicialização ou ao escalar workers;
- a memória do interpretador e extensões já está carregada.

### Sobre paralelismo no FastCGI

- PHP é single-threaded por processo;
- paralelismo vem de múltiplos workers simultâneos;
- PHP-FPM gerencia quantos workers existem.

---

## Conexão com Sistemas Operacionais

- **Custo detalhado do CGI: fork() + exec()** — o `fork()` duplica o PCB (process control block), a tabela de file descriptors e as page tables com Copy-on-Write (~100μs); o `exec()` carrega o interpretador do disco (~1ms); o interpretador inicializa (~10ms). Total: ~11ms de overhead por requisição vs ~0.1ms do FastCGI → [[Criação de Processos]], [[Término de Processos]]

- **Reaproveitamento de processo no FastCGI** — o worker FastCGI já existe, aguarda no estado bloqueado (waiting for I/O). Quando uma requisição chega, o kernel acorda o processo com `recvmsg()` e a aplicação faz o dispatch internamente — sem nenhum fork → [[System Calls]], [[Processos]]

- **Pool dinâmico do PHP-FPM** — com `pm = dynamic`, o PHP-FPM mantém `min_spare_servers` processos ociosos e faz fork de novos workers sob carga até atingir `max_children`. Cada fork segue o ciclo normal: `fork()` → novo PCB → processo filho aguarda no socket → [[Criação de Processos]], [[Hierarquia de Processos]]

- **Reciclagem de workers (pm.max_requests)** — após atender N requisições, o worker chama `exit()` e o processo mãe faz fork de um novo worker no lugar. Isso previne vazamentos de memória acumulados em PHP (objetos globais, connections abertas, etc.) → [[Término de Processos]], [[Criação de Processos]]

- **Goroutine vs fork: diferença de custo** — criar uma goroutine em Go custa ~300ns (pilha inicial de 2KB, alocada em heap) vs ~100μs de um fork do kernel. Isso é uma diferença de 300x, tornando o modelo de Go dramaticamente mais eficiente para alta concorrência → [[Goroutines]]

---

## Conexão com Go

- **Go: goroutine-per-request vs fork-per-request** — um servidor Go inicia uma vez e cria uma goroutine por requisição (~300ns cada), enquanto CGI faz fork por requisição (~100μs). Com Go, é prático ter dezenas de milhares de requisições simultâneas sem custo proibitivo de criação de processos → [[Goroutines]]
