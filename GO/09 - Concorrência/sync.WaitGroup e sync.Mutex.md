---
tags:
  - go
  - go/concorrencia
aliases:
  - Pacote sync
  - sync.Mutex
  - sync.RWMutex
  - sync.WaitGroup
  - sync.Once
  - sync.Cond
  - sync.Map
  - sync.Pool
  - sync/atomic
  - Deadlock em Go
ordem: 40
modulo: "09 - Concorrência"
---
# sync.WaitGroup e sync.Mutex

> **Nota 40 da trilha de Go.** Channels resolvem a comunicação, mas muitas vezes o problema é outro: **esperar** um grupo de goroutines, ou **proteger** um dado que várias goroutines leem e alteram no mesmo lugar. Para isso existe o pacote **`sync`** (e o **`sync/atomic`**). Apesar do título, esta nota cobre o pacote inteiro: `WaitGroup` (inclusive o novo `wg.Go` do Go 1.25), `Mutex` e `RWMutex`, **deadlocks**, `Once` e suas variantes funcionais, `Cond`, `sync.Map`, `sync.Pool` e as operações **atômicas**, terminando com a implementação do mutex de Go (rotação, semáforos e o "modo de inanição") e com o paralelo direto com os capítulos de sincronização do Tanenbaum.
>
> **Pré-requisitos:** [[Fundamentos de Concorrência]], [[Goroutines]], [[Channels]].

---

## 🧭 De onde viemos

Em [[Fundamentos de Concorrência]] vimos o contador que perde incrementos e a conta bancária com saldo negativo, e dissemos que "mutex para proteger estado, channel para transferir posse". Em [[Goroutines]] vimos que o `main` não espera ninguém e usamos `WaitGroup` sem explicá-lo. No Tanenbaum, [[Regiões Críticas]], [[Exclusão mútua com espera ocupada]], [[Mutexes]] (com os futexes), [[Monitores]] (com variáveis de condição), [[Barreiras]] e [[Inversão de Prioridade e RCU]] apresentaram exatamente as primitivas desta nota. Em [[Maps]] e [[Structs]] prometemos voltar aqui para maps concorrentes e o alinhamento dos atômicos, e em [[Closures e Funções Anônimas]], para o `OnceValue`.

> 💡 **Primitiva de sincronização:** mecanismo de baixo nível que impõe uma **ordem** entre ações de fluxos de execução diferentes: "espere até X", "só um por vez aqui dentro", "faça isto uma única vez".

---

## 🧰 1. O pacote em uma tabela

| Tipo | Problema que resolve | Equivalente no SO / Tanenbaum |
| --- | --- | --- |
| `sync.WaitGroup` | Esperar N goroutines terminarem | `pthread_join` em lote; [[Barreiras]] |
| `sync.Mutex` | Só uma goroutine por vez na seção crítica | [[Mutexes]] |
| `sync.RWMutex` | Vários leitores **ou** um escritor | Problema dos leitores e escritores ([[Semáforos]]) |
| `sync.Once` (+ `OnceFunc`, `OnceValue`, `OnceValues`) | Inicializar uma única vez | `pthread_once` |
| `sync.Cond` | Esperar uma **condição** sobre dados protegidos | Variáveis de condição ([[Monitores]]) |
| `sync.Map` | Map concorrente para padrões específicos | — |
| `sync.Pool` | Reaproveitar objetos temporários | Caches de objetos do kernel (slab) |
| `sync/atomic` | Operações indivisíveis numa palavra de memória | Instruções TSL/`LOCK` ([[Exclusão mútua com espera ocupada]]) |

> ⚠️ **Nenhum** desses tipos pode ser **copiado** depois de usado: a cópia leva o estado interno (contador, flag de travado) e as duas cópias passam a viver vidas separadas. Passe sempre **ponteiros** e coloque-os em structs que também são usadas por ponteiro. O `go vet` detecta cópias (seção 2.3).

---

## ⏳ 2. `sync.WaitGroup`: esperar um grupo

### 2.1 O jeito clássico: `Add`, `Done`, `Wait`

Um `WaitGroup` é um **contador**: `Add(n)` soma, `Done()` subtrai 1 e `Wait()` bloqueia até o contador chegar a zero.

```go
var wg sync.WaitGroup
for _, url := range urls {
	wg.Add(1)                 // ✅ ANTES do go
	go func() {
		defer wg.Done()       // ✅ com defer: roda mesmo com return antecipado
		baixar(url)
	}()
}
wg.Wait()                     // espera todas
```

### 2.2 O jeito moderno: `wg.Go` (Go 1.25+)

O padrão acima é tão comum que o Go 1.25 o empacotou num método. Saída real:

```go
var wg sync.WaitGroup
resultados := make([]int, 5)
for i := range 5 {
	wg.Go(func() { // Add(1) + go + defer Done() num só lugar
		time.Sleep(time.Duration(5-i) * 10 * time.Millisecond)
		resultados[i] = i * i // cada goroutine escreve no SEU índice: sem corrida
	})
}
wg.Wait()
fmt.Println(resultados)
// [0 1 4 9 16]
```

Escrever em **índices diferentes** de um slice pré-alocado é seguro: são posições de memória distintas, e o `Wait` garante que todas as escritas acontecem antes da leitura (happens-before). A implementação real de `wg.Go` (Go 1.27, `sync/waitgroup.go`) é curta e tem um detalhe interessante:

```go
func (wg *WaitGroup) Go(f func()) {
	wg.Add(1)
	go func() {
		defer func() {
			if x := recover(); x != nil {
				// f entrou em panic: NÃO chama Done, para o Wait não liberar
				// o main (que poderia sair com sucesso antes do crash)
				panic(x)
			}
			wg.Done()
		}()
		f()
	}()
}
```

> ✅ Prefira `wg.Go` em código novo: elimina as duas regras que mais causam bugs (o `Add` antes do `go` e o `Done` com `defer`).

### 2.3 As regras, e o `go vet` cobrando

Saídas reais do `go vet` (Go 1.27) num código que comete os dois erros clássicos:

```go
func trabalhar(wg sync.WaitGroup) { defer wg.Done() }   // ❌ recebe uma CÓPIA

for i := range 3 {
	go func() {
		wg.Add(1)            // ❌ Add dentro da goroutine
		defer wg.Done()
		fmt.Println(i)
	}()
}
wg.Add(1)
go trabalhar(wg)             // ❌ copia o WaitGroup
wg.Wait()
```

```
main.go:8:19: trabalhar passes lock by value: sync.WaitGroup contains sync.noCopy
main.go:20:15: call of trabalhar copies lock value: sync.WaitGroup contains sync.noCopy
main.go:14:10: WaitGroup.Add called from inside new goroutine
```

| Regra | O que dá errado se violar |
| --- | --- |
| `Add` **antes** do `go` | O `Wait` pode ver o contador em 0 e retornar antes de a goroutine fazer o `Add` |
| `Done` com `defer` | Um `return` antecipado ou panic esquece o `Done` e o `Wait` trava para sempre |
| Nunca copiar | `Done` na cópia não afeta o original: `Wait` trava |
| Contador negativo | Panic: `sync: negative WaitGroup counter` |

O analisador `waitgroup` (que gerou a última linha) entrou no `go vet` no Go 1.25; o `copylocks` é antigo e reconhece qualquer tipo que contenha um campo `noCopy`.

> 💡 Precisa do **primeiro erro** e de cancelar as outras quando uma falha? Use `errgroup` (ver [[Padrões de Concorrência]]).

---

## 🔒 3. `sync.Mutex`: um de cada vez

### 3.1 O uso padrão

```go
type Contador struct {
	mu sync.Mutex           // ✅ o mutex fica junto dos dados que protege
	n  map[string]int       //    (convenção: campos protegidos logo abaixo do mutex)
}

func (c *Contador) Incrementar(chave string) {
	c.mu.Lock()
	defer c.mu.Unlock()     // ✅ libera em qualquer caminho de saída
	c.n[chave]++
}
```

Pontos importantes:

- O **zero value** é um mutex destravado, pronto para uso (sem construtor)
- Métodos que mexem no mutex precisam de **receptor ponteiro** (ver [[Métodos e Receptores]]); com receptor valor, cada chamada travaria uma **cópia**
- O mutex protege **dados**, não código: todo acesso aos dados protegidos, de qualquer método, precisa travar o **mesmo** mutex
- **Não é reentrante**: a mesma goroutine travar duas vezes = deadlock (seção 5.2)
- O mutex não tem "dono": uma goroutine pode destravar o que outra travou (diferente de um `pthread_mutex` do tipo *errorcheck*). Isso é permitido, mas raramente uma boa ideia

### 3.2 Seção crítica pequena (mas não pequena demais)

```go
// ❌ trava durante uma chamada de rede de 200 ms: todas as outras goroutines esperam
c.mu.Lock()
defer c.mu.Unlock()
dados := c.cache[k]
resultado := consultarAPI(dados)          // lento e não precisa do lock
c.cache[k] = resultado

// ✅ trava só para ler e para escrever
c.mu.Lock()
dados := c.cache[k]
c.mu.Unlock()

resultado := consultarAPI(dados)

c.mu.Lock()
c.cache[k] = resultado
c.mu.Unlock()
```

> ⚠️ Mas lembre do "verificar e agir" de [[Fundamentos de Concorrência]]: se a lógica é "se não está no cache, busque e guarde", dividir a seção crítica permite que duas goroutines busquem a mesma chave ao mesmo tempo. Às vezes isso é aceitável; às vezes é preciso verificar de novo depois de retomar o lock (ou usar `singleflight`, ver [[Padrões de Concorrência]]).

### 3.3 Não exponha o mutex

```go
type Cache struct {
	sync.Mutex            // ❌ embutido: Lock/Unlock viram métodos PÚBLICOS de Cache
	dados map[string]string
}

type Cache struct {
	mu    sync.Mutex      // ✅ campo não exportado: só o pacote controla a trava
	dados map[string]string
}
```

Embutir é prático em tipos internos, mas num tipo exportado entrega a trava a qualquer usuário (ver [[Composição vs Herança]]).

### 3.4 `TryLock`

Desde o Go 1.18 existe `TryLock()`, que tenta travar sem esperar. Saída real (e o que acontece ao destravar um mutex já destravado):

```go
var mu sync.Mutex
fmt.Println("1ª TryLock:", mu.TryLock())   // 1ª TryLock: true
fmt.Println("2ª TryLock:", mu.TryLock())   // 2ª TryLock: false
mu.Unlock()
mu.Unlock()                                // ❌
```

```
fatal error: sync: unlock of unlocked mutex
```

> ⚠️ A documentação avisa: usos corretos de `TryLock` são **raros**, e frequentemente indicam um problema de design. Repare também que destravar duas vezes é um `fatal error` (não um panic recuperável).

---

## 📚 4. `sync.RWMutex`: leitores em paralelo

Quando um dado é **lido muito mais do que escrito** (configuração, tabela de rotas, cache), um `RWMutex` deixa **vários leitores** entrarem juntos e só exige exclusividade para o escritor:

```go
type Rotas struct {
	mu    sync.RWMutex
	tabela map[string]Handler
}

func (r *Rotas) Buscar(p string) (Handler, bool) {
	r.mu.RLock()                 // trava de LEITURA: compartilhada
	defer r.mu.RUnlock()
	h, ok := r.tabela[p]
	return h, ok
}

func (r *Rotas) Registrar(p string, h Handler) {
	r.mu.Lock()                  // trava de ESCRITA: exclusiva
	defer r.mu.Unlock()
	r.tabela[p] = h
}
```

```
              Mutex                          RWMutex
leitor 1  ████                          leitor 1  ████
leitor 2      ████                      leitor 2  ████     ← juntos
leitor 3          ████                  leitor 3  ████
escritor              ███               escritor      ███  ← sozinho
```

É a solução do **problema dos leitores e escritores** ([[Semáforos]]), com uma escolha importante: o `RWMutex` de Go **prioriza o escritor**. Quando um `Lock` está esperando, **novos** `RLock` também esperam, para que um fluxo contínuo de leitores não deixe o escritor faminto.

### 4.1 Consequência: `RLock` recursivo pode travar

Por causa dessa prioridade, uma goroutine que já tem um `RLock` e pede **outro** pode travar para sempre, se um escritor chegar no meio. Saída real:

```go
var mu sync.RWMutex

func leitura(nivel int) {
	mu.RLock()
	defer mu.RUnlock()
	if nivel == 0 {
		time.Sleep(50 * time.Millisecond) // nesse meio tempo um escritor chega
		leitura(1)                         // ❌ RLock recursivo
		return
	}
	fmt.Println("leitura aninhada ok")
}

// em main: uma goroutine faz mu.Lock() depois de 10 ms; o main chama leitura(0)
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.RWMutex.RLock]:
main.leitura(0x1)
	/tmp/gt/ex/n40/rwdead/main.go:12 +0x4b
main.leitura(...)
	/tmp/gt/ex/n40/rwdead/main.go:16 +0x90

goroutine 19 [sync.RWMutex.Lock]:
main.main.func1()
	/tmp/gt/ex/n40/rwdead/main.go:25 +0x25
```

O escritor espera o leitor sair; o leitor (aninhado) espera o escritor passar. Um ciclo.

### 4.2 Quando vale a pena

| Use `RWMutex` quando | Fique com `Mutex` quando |
| --- | --- |
| Leituras dominam com folga (muito mais que escritas) | Leituras e escritas equilibradas |
| A seção de leitura não é trivial | A seção crítica é de poucos nanossegundos |
| Há muitas goroutines lendo **ao mesmo tempo** | Pouca disputa |

O `RWMutex` tem mais estado para atualizar, então com seções minúsculas ele pode ser **mais lento** que um `Mutex`. Meça antes (ver [[Benchmarks]]).

---

## 💀 5. Deadlock

> 💡 **Deadlock (impasse):** um conjunto de goroutines em que cada uma espera por algo que só outra do conjunto pode liberar. Nenhuma avança, para sempre.

As quatro **condições de Coffman** precisam valer **todas ao mesmo tempo**:

| Condição | No contexto de mutexes |
| --- | --- |
| Exclusão mútua | Um mutex só tem um dono por vez |
| Posse e espera | A goroutine segura um mutex enquanto pede outro |
| Sem preempção | Ninguém tira um mutex de quem o segura |
| Espera circular | A espera B, B espera A |

Para **prevenir**, basta quebrar uma delas. A mais prática é a **espera circular**: defina uma **ordem global** de aquisição e respeite-a sempre.

### 5.1 Espera circular (AB-BA)

Saída real (trecho):

```go
var muA, muB sync.Mutex
wg.Go(func() {
	muA.Lock()
	time.Sleep(10 * time.Millisecond)
	muB.Lock() // espera B, que está com a outra
})
wg.Go(func() {
	muB.Lock()
	time.Sleep(10 * time.Millisecond)
	muA.Lock() // espera A, que está com a primeira
})
wg.Wait()
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
...
goroutine 7 [sync.Mutex.Lock]:
internal/sync.runtime_SemacquireMutex(0x0?, 0x0?, 0x0?)
	/tmp/gt/go127/src/runtime/sema.go:95 +0x25
internal/sync.(*Mutex).lockSlow(0x32d47d53e0e8)
	/tmp/gt/go127/src/internal/sync/mutex.go:149 +0x15a
...
main.main.func1()
	/tmp/gt/ex/n40/dlab/main.go:15 +0x65
```

O motivo de bloqueio entre colchetes (`[sync.Mutex.Lock]`, `[sync.WaitGroup.Wait]`) é a primeira pista ao investigar um travamento: num servidor real, você obtém esse dump com `SIGQUIT` (`kill -QUIT <pid>`) ou pelo endpoint `/debug/pprof/goroutine?debug=2` (ver [[Profiling e Diagnóstico]]).

```go
// ✅ correção: todas as goroutines travam na MESMA ordem (A antes de B)
muA.Lock(); defer muA.Unlock()
muB.Lock(); defer muB.Unlock()
```

### 5.2 Auto-travamento: mutex não é reentrante

O caso mais comum na prática: um método que segura o lock chama **outro método do mesmo tipo** que também trava. Saída real:

```go
func (c *Cache) Get(k string) string {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.dados[k]
}

func (c *Cache) GetOuPadrao(k string) string {
	c.mu.Lock()
	defer c.mu.Unlock()
	if v := c.Get(k); v != "" { // ❌ Get tenta travar o mesmo mutex
		return v
	}
	return "padrão"
}
```

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.Mutex.Lock]:
...
main.(*Cache).Get(0xff992678e48?, {0x47f000, 0x1})
	/tmp/gt/ex/n40/selflock/main.go:11 +0x5c
main.(*Cache).GetOuPadrao(0x10?, {0x47f000?, 0xff992678ea8?})
	/tmp/gt/ex/n40/selflock/main.go:19 +0x93
```

A correção idiomática é separar métodos **públicos que travam** de métodos **internos que assumem o lock já travado**:

```go
func (c *Cache) Get(k string) string {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.getLocked(k)
}

// getLocked exige que c.mu esteja travado.
func (c *Cache) getLocked(k string) string { return c.dados[k] }
```

> ⚠️ A detecção "all goroutines are asleep" só funciona se **todas** as goroutines travarem. Num servidor (com goroutines de rede, timers etc.), o deadlock é **silencioso**: as requisições simplesmente param de responder.

---

## 1️⃣ 6. `sync.Once` e as variantes funcionais

### 6.1 `Once.Do`

`once.Do(f)` executa `f` **uma única vez**, não importa quantas goroutines chamem nem quantas vezes. As que chegarem durante a execução **esperam** ela terminar.

```go
type Cliente struct {
	once sync.Once
	conn *Conexao
}

func (c *Cliente) conexao() *Conexao {
	c.once.Do(func() {
		c.conn = conectar()          // só a primeira chamada executa
	})
	return c.conn                    // todas veem a conexão pronta (happens-before)
}
```

### 6.2 `OnceFunc`, `OnceValue`, `OnceValues` (Go 1.21+)

Variantes que devolvem uma **função** já protegida, muito mais convenientes para valores calculados preguiçosamente. Saída real:

```go
config := sync.OnceValue(func() map[string]string {
	fmt.Println("(carregando configuração...)")
	return map[string]string{"porta": "8080"}
})
fmt.Println(config()["porta"], config()["porta"])
// (carregando configuração...)
// 8080 8080

ler := sync.OnceValues(func() ([]byte, error) {
	fmt.Println("(lendo arquivo...)")
	return nil, fmt.Errorf("arquivo não encontrado")
})
_, err1 := ler()
_, err2 := ler()
fmt.Println(err1, "|", err2)
// (lendo arquivo...)
// arquivo não encontrado | arquivo não encontrado
```

Repare que o **erro também é memorizado**: `OnceValues` não tenta de novo. Se você precisa de "tente até dar certo", `Once` não é a ferramenta.

### 6.3 A diferença com panics

Saída real: `f` entra em panic na primeira chamada.

```
Once.Do 1ª: panic(falhou)
Once.Do 2ª: retornou normalmente      ← Once considera "já feito"; a 2ª chamada não faz nada
OnceFunc 1ª: panic(falhou)
OnceFunc 2ª: panic(falhou)            ← OnceFunc repete o MESMO panic em toda chamada
```

O comportamento do `Once.Do` é perigoso: depois de um panic recuperado, o código segue achando que a inicialização aconteceu (com `c.conn == nil`, por exemplo). `OnceFunc`/`OnceValue` são mais seguros: o erro continua visível.

---

## 🔔 7. `sync.Cond`: esperar uma condição

Uma **variável de condição** permite que goroutines durmam até que **uma condição sobre dados protegidos por um mutex** fique verdadeira. É a peça que faltava aos mutexes nos [[Monitores]] do Tanenbaum. Saída real, com uma fila bloqueante:

```go
type Fila struct {
	mu    sync.Mutex
	vazia *sync.Cond // "a fila deixou de estar vazia"
	itens []string
}

func NovaFila() *Fila {
	f := &Fila{}
	f.vazia = sync.NewCond(&f.mu)   // a Cond é ligada ao mutex
	return f
}

func (f *Fila) Colocar(s string) {
	f.mu.Lock()
	f.itens = append(f.itens, s)
	f.mu.Unlock()
	f.vazia.Signal() // acorda UM consumidor esperando
}

func (f *Fila) Tirar() string {
	f.mu.Lock()
	defer f.mu.Unlock()
	for len(f.itens) == 0 { // SEMPRE em laço: a condição pode ter mudado
		f.vazia.Wait() // solta o mutex, dorme, retoma o mutex ao acordar
	}
	s := f.itens[0]
	f.itens = f.itens[1:]
	return s
}

// dois consumidores esperando; depois Colocar("A") e Colocar("B"):
// consumidor 0 pegou A
// consumidor 1 pegou B
```

| Método | Faz |
| --- | --- |
| `Wait()` | **Atomicamente** destrava o mutex e dorme; ao acordar, trava de novo antes de retornar |
| `Signal()` | Acorda **uma** goroutine esperando |
| `Broadcast()` | Acorda **todas** |

> ⚠️ O `for` em volta do `Wait` não é opcional: entre ser acordada e retomar o mutex, outra goroutine pode ter consumido o item (a semântica é a de **Mesa**, não a de Hoare, na discussão "o problema do signal" de [[Monitores]]).

> 💡 Na prática, `sync.Cond` é **raro** em Go: um channel resolve a maioria desses casos com menos código, e `Cond` não combina com `select` nem com cancelamento por `context`. Ele aparece em código de baixo nível (o próprio `net/http` e o `io.Pipe` usam).

---

## 🗺️ 8. `sync.Map`

Um map seguro para uso concorrente, **sem** precisar de mutex externo:

```go
var sessoes sync.Map                          // zero value pronto; chaves e valores são any

sessoes.Store("abc", &Sessao{Usuario: 7})
v, ok := sessoes.Load("abc")                  // v é any: precisa de type assertion
s := v.(*Sessao)
atual, carregou := sessoes.LoadOrStore("xyz", novaSessao())
sessoes.Delete("abc")
sessoes.Range(func(k, v any) bool { ...; return true })
sessoes.Clear()                               // Go 1.23+
// também: LoadAndDelete, Swap, CompareAndSwap, CompareAndDelete
```

| Use `sync.Map` quando | Use `map` + `Mutex`/`RWMutex` quando |
| --- | --- |
| Cada chave é escrita uma vez e lida muitas (caches que só crescem) | Caso geral |
| Goroutines diferentes mexem em conjuntos **disjuntos** de chaves | Você quer tipos concretos (sem `any`) |
| Muita disputa entre muitos núcleos | Precisa de operações compostas (ler várias chaves de forma consistente) |

Desde o **Go 1.24**, o `sync.Map` é implementado sobre um **Hash-Trie concorrente** (`internal/sync.HashTrieMap`), que escala melhor com escritas do que a implementação anterior (dois maps, um só de leitura). A desvantagem continua: sem generics na API, cada acesso exige `any` e type assertion.

---

## ♻️ 9. `sync.Pool`: reaproveitar objetos

Um `Pool` guarda objetos **temporários** já alocados para reaproveitamento, reduzindo o trabalho do coletor de lixo em caminhos quentes. Saída real (contando alocações com `testing.AllocsPerRun`):

```go
var buffers = sync.Pool{
	New: func() any { return new(bytes.Buffer) },   // chamado quando o pool está vazio
}

func formatarComPool(nome string) string {
	b := buffers.Get().(*bytes.Buffer)
	b.Reset()               // ✅ sempre limpe: o objeto pode vir sujo
	defer buffers.Put(b)
	b.WriteString("Olá, ")
	b.WriteString(nome)
	return b.String()
}
// alocações por chamada: sem pool = 2, com pool = 1
```

(A alocação que sobra é a string do resultado.)

| Regra | Por quê |
| --- | --- |
| O pool pode ser **esvaziado a qualquer coleta de lixo** | Não é um cache: nunca guarde algo que você não pode perder (conexões, por exemplo) |
| `Get` pode devolver um objeto "sujo" | Sempre resete antes de usar |
| Não guarde objetos de tamanho muito variável | Um buffer que cresceu para 50 MB volta ao pool e fica retido |
| Não use depois do `Put` | Outra goroutine pode estar com ele |

Internamente, cada P tem sua lista local (sem trava no caminho comum), e desde o Go 1.13 há um **cache vítima**: na coleta, os objetos não são jogados fora de uma vez, mas movidos para uma geração "vítima" que só é descartada na coleta seguinte. É parecido com os caches de objetos (*slab*) que o kernel usa para estruturas frequentes. Veremos o GC em [[Garbage Collector]].

---

## ⚛️ 10. `sync/atomic`: uma instrução, sem trava

### 10.1 Os tipos atômicos

Para um **contador, uma flag ou um ponteiro**, uma operação atômica é mais barata que um mutex: é uma única instrução de hardware. Desde o Go 1.19, o jeito recomendado são os **tipos** atômicos:

```go
var reqs atomic.Int64            // também: Int32, Uint32, Uint64, Uintptr, Bool
reqs.Add(1)
total := reqs.Load()
reqs.Store(0)
antigo := reqs.Swap(10)
trocou := reqs.CompareAndSwap(10, 11)   // "se vale 10, passe a valer 11"
reqs.And(0xFF); reqs.Or(0x100)          // Go 1.23+

var pronto atomic.Bool
pronto.Store(true)
```

As **funções** antigas (`atomic.AddInt64(&x, 1)`) continuam existindo, mas os tipos têm duas vantagens: é impossível acessar o valor **sem** passar por uma operação atômica, e o **alinhamento** fica garantido. Esse é o problema prometido em [[Structs]]: em plataformas de 32 bits, uma operação atômica de 64 bits exige que o campo esteja alinhado em 8 bytes, o que o compilador não garante para um `int64` comum dentro de uma struct, mas garante para `atomic.Int64`. O **Go 1.27** inclui no `go fix` o modernizador `atomictypes`, que converte o estilo antigo para os tipos.

### 10.2 O que o hardware faz

Saída real do `go tool objdump` (amd64) para um incremento atômico e um comum:

```
TEXT main.incAtomico(SB)
  LOCK XADDQ AX, 0(CX)          ← contador.Add(1)
TEXT main.incComum(SB)
  INCQ main.comum(SB)           ← comum++
```

O prefixo **`LOCK`** faz o núcleo obter a linha de cache em modo **exclusivo** (protocolo de coerência de cache MESI) e completar a leitura-modificação-escrita sem que nenhum outro núcleo a observe pela metade. É a versão moderna da instrução **TSL** do Tanenbaum ([[Exclusão mútua com espera ocupada]]): `CompareAndSwap` vira `LOCK CMPXCHG`, a base de todo algoritmo *lock-free*. Em ARM64, são as instruções LSE (`LDADDAL`, `CASAL`) ou laços `LDAXR`/`STLXR`.

### 10.3 `atomic.Pointer`: trocar uma estrutura inteira sem travar leitores

```go
type Config struct {
	Timeout time.Duration
	Rotas   map[string]string
}

var cfg atomic.Pointer[Config]

// leitores (muitos, o tempo todo): nunca travam
func timeoutAtual() time.Duration { return cfg.Load().Timeout }

// escritor (raro): monta uma Config NOVA e publica o ponteiro de uma vez
func recarregar() {
	nova := &Config{Timeout: 5 * time.Second, Rotas: lerRotas()}
	cfg.Store(nova)            // os leitores veem a antiga OU a nova, nunca metade
}
```

É a ideia do **RCU** de [[Inversão de Prioridade e RCU]]: leitores nunca bloqueiam, o escritor publica uma cópia nova e a antiga é liberada quando ninguém mais a usa (no kernel, depois de um "período de graça"; em Go, pelo coletor de lixo). A regra é **nunca modificar** a `Config` depois de publicada.

### 10.4 Quanto custa (medido)

Go 1.27, 2 núcleos:

```
mutex, 1 goroutine                         20.0 ns/op
atomic.Add, 1 goroutine                     6.8 ns/op
mutex, todas as CPUs disputando            23.6 ns/op
atomic.Add, todas as CPUs disputando       21.2 ns/op
```

Sem disputa, o atômico é ~3× mais barato. **Com** disputa, os dois ficam caros: o gargalo passa a ser a **linha de cache** pulando de um núcleo para o outro, e nenhuma primitiva escapa disso. Para contadores muito quentes em máquinas com muitos núcleos, a solução é **não compartilhar** (um contador por goroutine/shard, somados no fim).

> ⚠️ Atômicos protegem **uma** palavra. Se a invariante envolve **dois** campos (`saldo` e `historico`), use um mutex: duas operações atômicas seguidas não formam uma operação atômica.

---

## 🔬 11. Por dentro: como o `sync.Mutex` funciona

Desde o Go 1.24 a implementação vive em `internal/sync/mutex.go`. A struct inteira tem 8 bytes:

```go
type Mutex struct {
	state int32   // bits: travado | acordado | modo de inanição | nº de esperando (resto)
	sema  uint32  // semáforo do runtime onde as goroutines dormem
}
```

```
 state (32 bits)
┌──────────────────────────────────┬───┬───┬───┐
│  quantas goroutines esperando    │ S │ W │ L │   L = mutexLocked   (travado)
│  (state >> 3)                    │   │   │   │   W = mutexWoken    (alguém foi acordado)
└──────────────────────────────────┴───┴───┴───┘   S = mutexStarving (modo de inanição)
```

### 11.1 `Lock`, passo a passo

```
1. CAMINHO RÁPIDO: CompareAndSwap(state, 0, travado)
   → deu certo? pronto. É só uma instrução LOCK CMPXCHG (o caso sem disputa)

2. CAMINHO LENTO (lockSlow):
   a) se está travado, em modo normal, há outros núcleos ativos e poucas tentativas:
      GIRA (spin) um pouco — até 4 rodadas de 30 instruções PAUSE —
      esperando o dono soltar em breve (sem dormir)
   b) não deu? incrementa o contador de esperando e
      DORME no semáforo do runtime (runtime_SemacquireMutex → gopark)
   c) ao acordar, tenta de novo; se esperou mais de 1 ms,
      coloca o mutex em MODO DE INANIÇÃO
```

| Modo | Comportamento | Objetivo |
| --- | --- | --- |
| **Normal** | Quem acaba de chegar (já rodando na CPU) compete com quem foi acordado, e geralmente vence | Vazão: a mesma goroutine pode travar várias vezes seguidas sem troca de contexto |
| **Inanição** (*starvation*, desde o Go 1.9) | O `Unlock` **entrega** o mutex diretamente ao primeiro da fila; recém-chegados vão para o fim da fila e não giram | Justiça: limita a latência de cauda |

O mutex volta ao modo normal quando quem recebe a posse é o último da fila ou esperou menos de 1 ms.

### 11.2 Paralelo com o Tanenbaum

| Tanenbaum | `sync.Mutex` |
| --- | --- |
| [[Exclusão mútua com espera ocupada]] (TSL, giro) | Caminho rápido com CAS; giro curto antes de dormir |
| [[Dormir e Despertar]] | Dormir no semáforo do runtime em vez de girar indefinidamente |
| Futex: tenta em espaço de usuário, só entra no kernel se houver disputa ([[Mutexes]]) | Mesma filosofia, mas dorme a **goroutine** (`gopark`), sem syscall. A **thread** só dorme no kernel (via futex) se ficar sem nenhuma goroutine para rodar |
| [[Inversão de Prioridade e RCU]] (inanição, herança de prioridade) | Goroutines não têm prioridade, então não há inversão clássica; o modo de inanição combate a espera ilimitada |
| `pthread_join` e [[Barreiras]] | `WaitGroup`: um contador atômico de 64 bits (contador nos 32 bits altos, nº de esperando nos baixos) + um semáforo |

---

## ⚠️ 12. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Copiar `Mutex`/`WaitGroup`/`Once` (receptor valor, passar por valor) | Trava "não funciona"; `Wait` eterno | Ponteiros; rode `go vet` |
| `wg.Add` dentro da goroutine | `Wait` retorna cedo | `Add` antes do `go`, ou `wg.Go` |
| Esquecer `Unlock` num caminho de erro | Deadlock no próximo `Lock` | `defer mu.Unlock()` |
| Método que trava chamando outro que trava | Auto-deadlock | Métodos `xxxLocked` internos |
| Travar dois mutexes em ordens diferentes | Deadlock AB-BA | Ordem global de aquisição |
| `RLock` recursivo | Deadlock quando um escritor chega | Não aninhe; passe os dados já lidos |
| Lock segurado durante E/S | Vazão desaba | Seção crítica mínima |
| Dividir "verificar e agir" em duas seções | Condição de corrida | Verifique de novo ao retomar o lock |
| Mutex exportado (embutido em tipo público) | Usuários travam seu tipo | Campo `mu` não exportado |
| `Once.Do` com panic | Inicialização "feita" pela metade | `OnceValue`/`OnceValues` |
| `Cond.Wait` com `if` | Item fantasma, panic de índice | `for` em volta do `Wait` |
| `sync.Pool` como cache | Objetos somem no GC | Pool só para temporários |
| Dois atômicos achando que formam um | Estado inconsistente | Mutex para invariantes com vários campos |
| `atomic` em campo `int64` comum numa struct (32 bits) | Panic de alinhamento em ARM/386 | `atomic.Int64` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Não copiar | Nenhum tipo de `sync` pode ser copiado após o uso; `go vet` detecta |
| `WaitGroup` | Contador: `Add` antes do `go`, `defer Done`, `Wait`; ou `wg.Go` (1.25+) |
| `Mutex` | Zero value pronto; ao lado dos dados; `defer Unlock`; não reentrante |
| `RWMutex` | Vários leitores ou um escritor; prioriza escritor; `RLock` recursivo trava |
| Deadlock | Coffman; quebre a espera circular com ordem de aquisição |
| `Once` | Uma execução; `OnceFunc`/`OnceValue`/`OnceValues` (1.21+) são mais seguros com panics |
| `Cond` | Variável de condição; `Wait` sempre em laço; raro em Go |
| `sync.Map` | Para chaves estáveis ou disjuntas; HashTrieMap desde 1.24 |
| `sync.Pool` | Reaproveita temporários; pode esvaziar no GC |
| `sync/atomic` | `LOCK XADD`/`CMPXCHG`; tipos atômicos; `atomic.Pointer` estilo RCU |
| Implementação | CAS rápido; giro curto; semáforo do runtime; modo de inanição após 1 ms |

---

## 🔗 Conexões

**Dentro de Go**
- Data races, happens-before e channel vs mutex → [[Fundamentos de Concorrência]]
- Quem é esperado e protegido → [[Goroutines]]
- A alternativa por mensagens → [[Channels]], [[Select]]
- Cancelamento e errgroup → [[context.Context]], [[Padrões de Concorrência]]
- Receptores ponteiro e mutex → [[Métodos e Receptores]]
- Mutex embutido e exportação → [[Composição vs Herança]]
- Maps não são seguros para concorrência → [[Maps]]
- Alinhamento de campos → [[Structs]]
- `defer mu.Unlock()` e fatal errors → [[Defer, Panic e Recover]]
- Closures e `OnceValue` → [[Closures e Funções Anônimas]]
- Medir antes de trocar `Mutex` por `RWMutex` → [[Benchmarks]]
- Dumps de goroutines e profile de mutex → [[Profiling e Diagnóstico]]
- Formalizando happens-before dos atômicos → [[Modelo de Memória]]
- O GC por trás do `sync.Pool` → [[Garbage Collector]]

**Sistemas Operacionais**
- [[Race Condition]] e [[Regiões Críticas]] — o problema
- [[Exclusão mútua com espera ocupada]] — TSL e giro, ↔ CAS e `LOCK`
- [[Dormir e Despertar]] — por que dormir em vez de girar
- [[Semáforos]] — produtor-consumidor e leitores-escritores
- [[Mutexes]] — mutex, futex e variáveis de condição em pthreads
- [[Monitores]] — mutex + variáveis de condição; semântica de Mesa vs Hoare
- [[Barreiras]] — esperar um grupo
- [[Inversão de Prioridade e RCU]] — inanição; RCU ↔ `atomic.Pointer`
- [[Threads POSIX]] — `pthread_mutex_*`, `pthread_cond_*`, `pthread_once`
- [[Processadores]] — caches, coerência e o custo de compartilhar uma linha

**Algoritmos e Estruturas de Dados**
- [[Hashmap]] e [[Tries]] — a base do `HashTrieMap` usado pelo `sync.Map`
- [[Grafos]] — deadlock é um ciclo no grafo de espera

---

⬅️ **Anterior:** [[Select]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[context.Context]] ➡️
