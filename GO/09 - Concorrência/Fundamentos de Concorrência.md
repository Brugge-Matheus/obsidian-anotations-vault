---
tags:
  - go
  - go/concorrencia
aliases:
  - Concorrência em Go
  - Concorrência vs Paralelismo
  - CSP
  - Data race
  - Race detector
ordem: 36
modulo: "09 - Concorrência"
---
# Fundamentos de Concorrência

> **Nota 36 da trilha de Go.** Abrimos o módulo de **concorrência**, o recurso pelo qual Go é mais conhecido. Antes de ver goroutines, channels e mutexes em detalhe, esta nota monta o **mapa mental**: a diferença entre **concorrência** e **paralelismo**, por que programas concorrentes existem, a filosofia de Go (**CSP**: "compartilhe memória comunicando"), o grande perigo do assunto (**condições de corrida** e **data races**), a ferramenta que os encontra (o **race detector**), a ideia de **happens-before** e um guia para escolher entre channels e locks. Tudo isso conversa diretamente com o capítulo 2 do Tanenbaum.
>
> **Pré-requisitos:** [[Funções — Fundamentos]], [[Closures e Funções Anônimas]], [[Como Go Funciona]].

---

## 🧭 De onde viemos

Até aqui, todo programa que escrevemos era **uma única sequência** de instruções: uma coisa de cada vez, na ordem do código. Em [[O que é Go]] vimos que Go nasceu no Google justamente para servidores que atendem milhares de clientes ao mesmo tempo, e em [[Como Go Funciona]] montamos uma tabela ligando conceitos de SO (processos, threads, escalonador, troca de mensagens) às suas versões em Go. Do lado do Tanenbaum, você já estudou [[O Modelo de Processos]], [[O Modelo Clássico de Thread]], [[Race Condition]], [[Regiões Críticas]], [[Semáforos]], [[Mutexes]], [[Monitores]] e [[Troca de Mensagens]]. Este módulo mostra como Go empacota essas ideias.

> 💡 **Concorrência:** a capacidade de um programa **lidar com várias tarefas ao mesmo tempo**, no sentido de que todas estão **em andamento** no mesmo período, mesmo que, num dado instante, só uma esteja de fato executando. É uma propriedade da **estrutura** do programa.

> 💡 **Paralelismo:** **executar** várias tarefas literalmente **no mesmo instante**, em núcleos de CPU diferentes. É uma propriedade da **execução**, e depende do hardware.

---

## 🤹 1. Concorrência não é paralelismo

A frase é de Rob Pike (um dos criadores de Go), numa palestra de 2012 com esse título: *"Concorrência é sobre **lidar** com muitas coisas ao mesmo tempo. Paralelismo é sobre **fazer** muitas coisas ao mesmo tempo."*

```
CONCORRÊNCIA com 1 núcleo (intercalação, o "pseudoparalelismo" do Tanenbaum):

  núcleo 0: [A][B][A][C][B][A][C][C][B]   ← todas em andamento, uma por vez
            ──────────────────────────► tempo

PARALELISMO com 3 núcleos:

  núcleo 0: [A][A][A][A]
  núcleo 1: [B][B][B][B]                   ← executando no mesmo instante
  núcleo 2: [C][C][C][C]
            ──────────────────────────► tempo
```

| | Concorrência | Paralelismo |
| --- | --- | --- |
| O que é | Estrutura: várias tarefas independentes em andamento | Execução: várias instruções no mesmo instante |
| Precisa de vários núcleos? | Não | Sim |
| Ganha com | Tarefas que **esperam** (rede, disco, usuário) | Tarefas que **calculam** (CPU) |
| Em Go | Goroutines | Goroutines + `GOMAXPROCS` > 1 |

Um programa concorrente **pode** rodar em paralelo se houver núcleos disponíveis, mas continua correto com um núcleo só. Em Go você escreve código concorrente (goroutines) e o runtime decide quanto paralelismo usar.

### 1.1 Experimento: esperar não precisa de núcleos

Dez "requisições" que só esperam 100 ms (simulando a rede). Saída real, com 2 núcleos e depois com `GOMAXPROCS=1` (um único núcleo para código Go):

```go
func requisicao() { time.Sleep(100 * time.Millisecond) } // simula esperar a rede

inicio := time.Now()
for range 10 {
	requisicao()
}
fmt.Println("sequencial: ", time.Since(inicio).Round(time.Millisecond))

inicio = time.Now()
var wg sync.WaitGroup
for range 10 {
	wg.Add(1)
	go func() { defer wg.Done(); requisicao() }()    // "go" dispara uma goroutine
}
wg.Wait()                                            // espera todas terminarem
fmt.Println("concorrente:", time.Since(inicio).Round(time.Millisecond))
```

```
GOMAXPROCS: 2                 GOMAXPROCS: 1
sequencial:  1.002s           sequencial:  1.002s
concorrente: 100ms            concorrente: 100ms
```

Mesmo com **um núcleo**, a versão concorrente é 10× mais rápida: enquanto uma tarefa espera, outra avança. É exatamente o argumento de [[Modelando a Multiprogramação]]: manter a CPU ocupada enquanto processos esperam E/S.

### 1.2 Experimento: calcular precisa de núcleos

Agora quatro tarefas que só **calculam** (um laço de 300 milhões de iterações cada), na mesma máquina de 2 núcleos. Saída real:

```
GOMAXPROCS=2:  4 tarefas de CPU: 230ms
GOMAXPROCS=1:  4 tarefas de CPU: 490ms
```

Com trabalho de CPU, concorrência sozinha não ajuda: o ganho vem do **paralelismo**, e ele é limitado pelo número de núcleos (com 2 núcleos, no máximo ~2×).

---

## 🎯 2. Por que escrever programas concorrentes?

O Tanenbaum, em [[Utilização de Threads]], lista as razões clássicas, e elas valem igualmente para goroutines:

| Motivo | Exemplo | Tipo de tarefa |
| --- | --- | --- |
| **Esconder latência de E/S** | Buscar 50 URLs ao mesmo tempo em vez de uma por uma | E/S (*I/O-bound*) |
| **Atender muitos clientes** | Um servidor HTTP com uma goroutine por conexão | E/S |
| **Usar todos os núcleos** | Comprimir 100 imagens em paralelo | CPU (*CPU-bound*) |
| **Manter a interface responsiva** | Processar em segundo plano enquanto o programa responde | Misto |
| **Estruturar o problema** | Um pipeline: ler → transformar → gravar, cada etapa independente | Qualquer |

> 💡 **I/O-bound** (limitada por E/S): a tarefa passa a maior parte do tempo **esperando** dispositivos (rede, disco). **CPU-bound** (limitada por CPU): passa a maior parte do tempo **calculando**. Os mesmos termos aparecem em [[Introdução ao Escalonamento]].

O último motivo é o mais subestimado: concorrência é também uma ferramenta de **design**. Um servidor escrito como "uma goroutine por cliente, código sequencial dentro dela" é muito mais simples de ler do que um laço de eventos com callbacks (compare os três modelos em [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]]).

---

## 🧱 3. O modelo de Go: CSP

### 3.1 A ideia

Go se inspira em **CSP** (*Communicating Sequential Processes*), um modelo formal publicado por Tony Hoare em 1978: o programa é um conjunto de **processos sequenciais** independentes que **não compartilham variáveis** e se coordenam **trocando mensagens**.

> 💡 **CSP (Communicating Sequential Processes):** modelo de concorrência em que unidades independentes, cada uma executando código sequencial, interagem apenas enviando e recebendo mensagens por canais. A troca de mensagem é também o ponto de sincronização.

Em Go, os "processos sequenciais" são as **goroutines** e os canais são os **channels**. Daí o lema da linguagem:

> ✅ ***"Não comunique compartilhando memória; compartilhe memória comunicando."***

```
Compartilhando memória (estilo threads + locks):

  goroutine A ──┐                  ┌── goroutine B
                ├──► [ variável ] ◄┤      todos mexem no mesmo lugar;
  goroutine C ──┘     + mutex      └──    a disciplina de travar é sua

Comunicando (estilo CSP):

  goroutine A ──[ch]──► goroutine B ──[ch]──► goroutine C
                  ↑ o dado MUDA DE DONO a cada envio; só um dono por vez
```

A palavra-chave é **posse** (*ownership*): ao enviar um valor por um channel, a goroutine que envia "entrega" o dado e não deve mais mexer nele. Se só uma goroutine é dona de um dado em cada momento, não há disputa.

### 3.2 Go não é dogmático

Go oferece **os dois estilos**. O pacote `sync` tem mutexes, e a própria biblioteca padrão os usa muito. O Tanenbaum apresenta ambos como soluções para o mesmo problema: memória compartilhada com exclusão mútua ([[Mutexes]], [[Monitores]]) e [[Troca de Mensagens]].

### 3.3 A caixa de ferramentas do módulo

| Ferramenta | Para quê | Equivalente no SO | Nota |
| --- | --- | --- | --- |
| Goroutine (`go f()`) | Uma tarefa concorrente, barata | Thread (de usuário) | [[Goroutines]] |
| Channel (`chan T`) | Enviar valores entre goroutines e sincronizar | Troca de mensagens, pipe | [[Channels]] |
| `select` | Esperar vários channels ao mesmo tempo | `select`/`poll`/`epoll` | [[Select]] |
| `sync.WaitGroup` | Esperar um grupo de goroutines | `pthread_join` | [[sync.WaitGroup e sync.Mutex]] |
| `sync.Mutex` / `RWMutex` | Exclusão mútua sobre dados compartilhados | Mutex, futex | [[sync.WaitGroup e sync.Mutex]] |
| `sync.Once`, `sync.Cond` | Inicialização única, variáveis de condição | `pthread_once`, `pthread_cond` | [[sync.WaitGroup e sync.Mutex]] |
| `sync/atomic` | Operações indivisíveis em uma palavra | Instruções `LOCK`/TSL | [[sync.WaitGroup e sync.Mutex]] |
| `context.Context` | Cancelamento e prazos em árvore | Sinais para grupos de processos | [[context.Context]] |
| `errgroup`, pipelines, pools | Composições prontas | Thread pools, pipes do shell | [[Padrões de Concorrência]] |

---

## 💥 4. O perigo: condições de corrida e data races

### 4.1 Dois nomes, dois conceitos

> 💡 **Condição de corrida (race condition):** situação em que o **resultado** do programa depende da **ordem** (do *timing*) em que tarefas concorrentes executam. É o conceito geral de [[Race Condition]].

> 💡 **Data race (corrida de dados):** caso específico e mais grave: **duas goroutines acessam a mesma posição de memória ao mesmo tempo, pelo menos um acesso é escrita, e não há sincronização** ordenando os acessos. Em Go, um programa com data race tem comportamento **não definido** pela especificação do modelo de memória.

Todo data race causa uma condição de corrida, mas **eliminar data races não elimina todas as condições de corrida**, como veremos em 4.3.

### 4.2 O contador que perde incrementos

Quatro goroutines incrementam o mesmo contador 2,5 milhões de vezes cada. Saída real (2 núcleos, o programa repete a experiência 5 vezes):

```go
contador := 0
var wg sync.WaitGroup
for range 4 {
	wg.Add(1)
	go func() {
		defer wg.Done()
		for range 2_500_000 {
			contador++        // ❌ data race: escrita concorrente sem sincronização
		}
	}()
}
wg.Wait()
fmt.Println("esperado 10000000, obtido", contador)
```

```
esperado 10000000, obtido 6635897
esperado 10000000, obtido 5257154
esperado 10000000, obtido 5091704
esperado 10000000, obtido 5878180
esperado 10000000, obtido 6632428
```

Até metade dos incrementos sumiu, e o resultado muda a cada execução. (Com poucas iterações o erro pode nem aparecer: cada goroutine termina antes de a outra começar. Um bug de concorrência que "só aparece às vezes" é a regra, não a exceção.)

Por quê? Olhando o assembly gerado (`go build -gcflags=-S`), o incremento vira **uma única instrução**:

```
INCQ (CX)        ; incrementa o inteiro de 8 bytes no endereço CX
```

Uma instrução só, mas **não atômica**: dentro do núcleo ela é executada como *ler da memória → somar 1 → escrever na memória*. Dois núcleos podem intercalar essas etapas:

```
Tempo   Núcleo 0 (goroutine A)      Núcleo 1 (goroutine B)     contador
  1     lê 41                                                   41
  2                                 lê 41                       41
  3     soma → 42                                               41
  4                                 soma → 42                   41
  5     escreve 42                                              42
  6                                 escreve 42                  42  ← um incremento sumiu
```

É o mesmo cenário do *spooler* de impressão do Tanenbaum em [[Race Condition]]. Para tornar a instrução atômica, o x86 precisa do prefixo **`LOCK`** (`LOCK XADDQ`), que é o que `sync/atomic` gera (ver [[sync.WaitGroup e sync.Mutex]]).

### 4.3 Condição de corrida sem data race: "verificar e agir"

Aqui **cada** operação da conta é protegida por um mutex, então **não há data race**, mas o resultado ainda está errado. Saída real (e o `-race` **não** reclama):

```go
type Conta struct {
	mu    sync.Mutex
	saldo int
}

func (c *Conta) Saldo() int    { c.mu.Lock(); defer c.mu.Unlock(); return c.saldo }
func (c *Conta) Debitar(v int) { c.mu.Lock(); defer c.mu.Unlock(); c.saldo -= v }

// ❌ cada operação é protegida, mas o PAR "verificar e agir" não é
func (c *Conta) Sacar(v int) bool {
	if c.Saldo() >= v {
		time.Sleep(time.Millisecond) // a "janela" entre verificar e agir
		c.Debitar(v)
		return true
	}
	return false
}

// ✅ verificar e agir dentro da MESMA seção crítica
func (c *Conta) SacarSeguro(v int) bool {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.saldo >= v {
		c.saldo -= v
		return true
	}
	return false
}
```

Duas goroutines sacando 100 de uma conta com saldo 100 (saída real, igual com e sem `-race`):

```
Sacar       saldo final = -100
SacarSeguro saldo final = 0
```

As duas viram `saldo >= 100` antes de qualquer uma debitar. Esse padrão (*check-then-act*, "verificar e depois agir") é a forma mais comum de condição de corrida em código que "já usa mutex". A lição: a **região crítica** ([[Regiões Críticas]]) é a operação **lógica** inteira, não cada acesso isolado.

### 4.4 Quando o runtime pega você: maps

Maps comuns não são seguros para escrita concorrente, e o runtime faz uma verificação barata que **aborta o programa**. Saída real:

```go
m := map[int]int{}
for i := range 2 {
	go func() {
		for j := range 100000 {
			m[j] = i          // duas goroutines escrevendo no mesmo map
		}
	}()
}
```

```
fatal error: concurrent map writes

goroutine 6 [running]:
internal/runtime/maps.fatal({0x48180c?, 0x0?})
main.main.func1()
	/tmp/gt/ex/n36/mapw/main.go:13 +0x6c
```

Repare: é um `fatal error`, não um panic, então **não dá para usar `recover`** (vimos em [[Defer, Panic e Recover]]). A verificação é "melhor esforço": nem toda corrida em map é pega. A correção é um mutex ou `sync.Map` (ver [[Maps]] e [[sync.WaitGroup e sync.Mutex]]).

---

## 🕵️ 5. O race detector

Go traz um detector de data races embutido no compilador. Basta a flag `-race`:

```bash
go run -race .
go test -race ./...
go build -race -o app .
```

Saída real do contador da seção 4.2:

```
==================
WARNING: DATA RACE
Read at 0x00c000116028 by goroutine 10:
  main.main.func1()
      /tmp/gt/ex/n36/race/main.go:17 +0x99

Previous write at 0x00c000116028 by goroutine 8:
  main.main.func1()
      /tmp/gt/ex/n36/race/main.go:17 +0xab

Goroutine 10 (running) created at:
  main.main()
      /tmp/gt/ex/n36/race/main.go:14 +0x139

Goroutine 8 (running) created at:
  main.main()
      /tmp/gt/ex/n36/race/main.go:14 +0x139
==================
```

O relatório diz **qual endereço**, **quais goroutines**, **em que linhas** e **onde cada goroutine foi criada**. Ao final, o programa sai com código 66 se encontrou alguma corrida.

Como funciona, em uma frase: o compilador instrumenta **cada leitura e escrita de memória** com uma chamada à biblioteca **ThreadSanitizer** (do LLVM), que mantém para cada palavra de memória um histórico de quem acessou e usa **relógios vetoriais** para saber se dois acessos foram ordenados por alguma sincronização. Se não foram, é corrida.

| Característica | Detalhe |
| --- | --- |
| Custo | Memória 5–10× maior, execução 2–20× mais lenta (documentação oficial) |
| Falsos positivos | Não: tudo que ele reporta é corrida real |
| Falsos negativos | Sim: só encontra corridas que **acontecem** durante a execução observada |
| Onde usar | Testes e CI sempre; um canário em produção às vezes; nunca o binário principal de produção |
| Plataformas | As principais de 64 bits (Linux, macOS, Windows, FreeBSD em amd64/arm64, entre outras) |

> ✅ **Regra prática:** `go test -race ./...` no CI de todo projeto com concorrência. Veremos isso em [[Testes com go test]].

> ⚠️ Como o detector só vê corridas que ocorrem, testes que **exercitam** os caminhos concorrentes (várias goroutines de verdade) são essenciais. Um teste que roda tudo numa goroutine só nunca vai acusar nada.

---

## 🔗 6. Happens-before: a ordem que você pode garantir

Por que não basta "esperar um pouco" para outra goroutine ver um valor? Porque **compilador e CPU reordenam** operações de memória quando isso não muda o resultado de **uma** goroutine isolada. Exemplo clássico, errado:

```go
var dado string
var pronto bool

go func() {
	dado = "olá"
	pronto = true        // ❌ sem sincronização
}()

for !pronto {}           // ❌ pode nunca terminar (o compilador pode ler 'pronto' uma vez só)
fmt.Println(dado)        // ❌ e, se terminar, pode imprimir "" (as escritas podem ser vistas fora de ordem)
```

> 💡 **Happens-before ("acontece antes"):** relação de ordem entre eventos de goroutines diferentes. Se a escrita W **acontece antes** da leitura R, então R garantidamente vê o valor escrito por W (ou um posterior). A relação só é criada por **operações de sincronização**.

As operações que criam happens-before em Go:

| Sincronização | Garantia |
| --- | --- |
| `go f()` | Tudo antes do `go` acontece antes do início de `f` |
| Envio num channel | Acontece antes de o recebimento correspondente **completar** |
| `close(ch)` | Acontece antes de um recebimento que retorna por causa do fechamento |
| `mu.Unlock()` | Acontece antes do próximo `mu.Lock()` retornar |
| `wg.Done()` | Acontece antes de `wg.Wait()` retornar |
| `once.Do(f)` | O retorno de `f` acontece antes de qualquer `once.Do` retornar |
| Operações `sync/atomic` | Se uma leitura atômica observa uma escrita atômica, a escrita acontece antes |

> ✅ **A garantia DRF-SC:** o modelo de memória de Go (revisado em 2022) promete que um programa **sem data races** se comporta como se as goroutines fossem intercaladas numa **única ordem sequencial** (consistência sequencial). Ou seja: elimine as corridas com sincronização correta e você pode raciocinar sobre o código "como se fosse simples". Os detalhes formais ficam para [[Modelo de Memória]].

---

## 🧭 7. Channel ou mutex? Um guia de decisão

A recomendação da wiki oficial de Go é pragmática: **use o que deixar o código mais simples**. Na prática:

| Situação | Prefira | Por quê |
| --- | --- | --- |
| Passar um dado de uma goroutine para outra (transferir a posse) | Channel | O envio é a sincronização |
| Distribuir trabalho entre workers | Channel | A fila já é o channel |
| Avisar que algo terminou ou foi cancelado | Channel fechado / `context` | `close` acorda todos |
| Proteger um cache, um contador, um map compartilhado | Mutex | O estado fica parado; só o acesso precisa de ordem |
| Um contador ou flag simples, muito acessado | `sync/atomic` | Uma instrução de hardware |
| Esperar N goroutines | `sync.WaitGroup` / `errgroup` | Feito para isso |
| Limitar quantas coisas rodam ao mesmo tempo | Channel com buffer (semáforo) ou `errgroup.SetLimit` | Ver [[Padrões de Concorrência]] |

> ⚠️ Um sinal de alerta: se você está usando channels para **proteger** uma variável (um channel de capacidade 1 como "trava"), provavelmente um `sync.Mutex` seria mais claro. Se está usando mutex + variável de condição para **passar mensagens**, provavelmente um channel seria mais claro.

---

## 🔬 8. Por dentro: como Go mapeia esses conceitos no SO

Tudo isto será aprofundado nas próximas notas e em [[Scheduler do Go]], mas vale ter o esqueleto agora:

```
            seu código                     runtime de Go                 kernel
  ┌──────────────────────────┐   ┌──────────────────────────────┐   ┌────────────┐
  │ go f()   go g()   go h() │──►│ escalonador G-M-P            │──►│ threads do │
  │ (milhares de goroutines) │   │ N goroutines sobre M threads │   │ SO (M)     │──► núcleos
  │ ch <- v   mu.Lock()      │   │ filas, stacks que crescem,   │   │ futex,     │
  └──────────────────────────┘   │ netpoller (epoll)            │   │ epoll      │
                                 └──────────────────────────────┘   └────────────┘
```

| Conceito do Tanenbaum (cap. 2) | Em Go |
| --- | --- |
| [[O Modelo Clássico de Thread]] | Goroutine: stack própria, compartilha o espaço de endereçamento |
| [[Implementações Híbridas]] (N:M) | Escalonador G-M-P: N goroutines sobre M threads do kernel |
| [[Race Condition]], [[Regiões Críticas]] | Data races; detector `-race` |
| [[Exclusão mútua com espera ocupada]] | O mutex de Go gira (*spin*) um pouco antes de dormir |
| [[Dormir e Despertar]] | `gopark`/`goready`: estacionar e acordar goroutines |
| [[Semáforos]] | Channel com buffer como semáforo; semáforos internos do runtime |
| [[Mutexes]] (e futexes) | `sync.Mutex`; o runtime usa futex para adormecer threads ociosas |
| [[Monitores]] (variáveis de condição) | `sync.Cond`; struct com mutex embutido |
| [[Troca de Mensagens]] (com rendezvous) | Channels (sem buffer = rendezvous) |
| [[Barreiras]] | `sync.WaitGroup` |
| [[Inversão de Prioridade e RCU]] | `atomic.Pointer` para trocar estruturas sem travar leitores (estilo RCU) |

### 8.1 Detecção de deadlock global

Se **todas** as goroutines ficam bloqueadas, o runtime percebe que ninguém poderá acordá-las e aborta. Saída real:

```go
ch := make(chan int)
ch <- 1               // ninguém vai receber
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/gt/ex/n36/dl/main.go:7 +0x36
exit status 2
```

> ⚠️ Essa detecção só funciona quando **todas** estão presas. Se uma goroutine qualquer ainda roda (um servidor HTTP escutando, um `time.Ticker`), um deadlock parcial passa despercebido. Para esses casos existe o profile de **goroutines vazadas** (Go 1.27), que veremos em [[Goroutines]].

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Achar que concorrência acelera cálculo com 1 núcleo | Nenhum ganho em tarefas de CPU | Ganho de CPU vem de paralelismo (vários núcleos) |
| `x++` compartilhado sem sincronização | Incrementos perdidos | `sync/atomic` ou mutex |
| Proteger cada acesso, mas não a operação lógica | Saldo negativo, duplicatas | A seção crítica é o "verificar e agir" inteiro |
| Escrever num map de várias goroutines | `fatal error: concurrent map writes` | Mutex ou `sync.Map` |
| "Esperar um pouco" (`time.Sleep`) para sincronizar | Funciona na sua máquina, falha em produção | Use channel, WaitGroup ou mutex |
| Laço de espera ocupada numa flag comum | Laço infinito ou leitura de dados velhos | Sincronização de verdade (channel, atomic) |
| Não rodar `-race` | Corridas silenciosas em produção | `go test -race ./...` no CI |
| Confiar que o `-race` achou tudo | Corridas em caminhos não testados | Testes que exercitam a concorrência |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Concorrência | Estrutura: várias tarefas em andamento; funciona com 1 núcleo |
| Paralelismo | Execução simultânea em vários núcleos; acelera trabalho de CPU |
| I/O-bound vs CPU-bound | Esperar ganha com concorrência; calcular ganha com paralelismo |
| CSP | Goroutines sequenciais que se comunicam por channels |
| Lema | "Não comunique compartilhando memória; compartilhe memória comunicando" |
| Race condition | Resultado depende da ordem de execução |
| Data race | Acesso concorrente não sincronizado com escrita: comportamento indefinido |
| Check-then-act | Corrida sem data race; proteja a operação lógica inteira |
| `-race` | ThreadSanitizer; sem falsos positivos; só vê o que executa; use no CI |
| Happens-before | Só sincronização ordena memória entre goroutines; sem corridas = consistência sequencial |
| Channel vs mutex | Channel para transferir posse e coordenar; mutex para proteger estado |

---

## 🔗 Conexões

**Dentro de Go**
- A unidade de concorrência → [[Goroutines]]
- Comunicação e sincronização por mensagens → [[Channels]], [[Select]]
- Memória compartilhada com travas e atômicos → [[sync.WaitGroup e sync.Mutex]]
- Cancelamento e prazos → [[context.Context]]
- Composições prontas → [[Padrões de Concorrência]]
- O escalonador G-M-P em profundidade → [[Scheduler do Go]]
- O modelo de memória formal → [[Modelo de Memória]]
- `go test -race` → [[Testes com go test]]
- Maps não são seguros para concorrência → [[Maps]]
- `fatal error` não pode ser recuperado → [[Defer, Panic e Recover]]
- A tabela SO ↔ Go original → [[Como Go Funciona]]

**Sistemas Operacionais**
- [[O Modelo de Processos]] e [[Modelando a Multiprogramação]] — pseudoparalelismo e por que intercalar tarefas aumenta o uso da CPU
- [[Utilização de Threads]] — os motivos para ter várias linhas de execução
- [[Race Condition]] e [[Regiões Críticas]] — as definições que esta nota aplica
- [[Exclusão mútua com espera ocupada]], [[Dormir e Despertar]], [[Semáforos]], [[Mutexes]], [[Monitores]], [[Troca de Mensagens]], [[Barreiras]] — as primitivas que reaparecem em Go
- [[Implementações Híbridas]] — o modelo N:M usado pelas goroutines
- [[Servidores Single Threaded, Multi Threaded e Orientado a eventos]] — por que "uma goroutine por cliente" é simples e eficiente
- [[Processadores]] — múltiplos núcleos, a fonte do paralelismo

---

⬅️ **Anterior:** [[JSON (marshal e unmarshal)]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Goroutines]] ➡️
