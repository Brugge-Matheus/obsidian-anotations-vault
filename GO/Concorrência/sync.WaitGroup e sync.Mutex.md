---
tags:
  - go
  - go/concorrência
---
# sync.WaitGroup e sync.Mutex

> Goroutines e channels resolvem a maior parte dos problemas de concorrência em Go. Mas às vezes você precisa de algo mais direto: esperar que um grupo de goroutines termine (`WaitGroup`), ou proteger um dado que múltiplas goroutines precisam ler e modificar ao mesmo tempo (`Mutex`). Esse é o papel do pacote `sync`.
> 

---

## 1. O Problema que Cada Primitiva Resolve

Antes de ver o código, é importante entender **qual problema cada uma resolve**:

```go
WaitGroup → "Preciso esperar N goroutines terminarem antes de continuar"

Mutex     → "Múltiplas goroutines acessam o mesmo dado — preciso garantir
              que apenas uma modifique por vez"

RWMutex   → "Igual ao Mutex, mas leituras podem acontecer em paralelo —
              apenas escritas precisam de exclusividade"

Once      → "Preciso que uma inicialização rode exatamente uma vez,
              mesmo que várias goroutines tentem ao mesmo tempo"
```

---

## 2. `sync.WaitGroup` — Esperar Goroutines Terminarem

### O Problema

Quando você dispara múltiplas goroutines, `main()` não sabe quando elas terminam. Se não fizer nada, o programa encerra antes de todas concluírem:

```go
func main() {
	for i := range 5 {
		go func(n int) {
			time.Sleep(100 * time.Millisecond)
			fmt.Printf("worker %d terminou\n", n)
		}(i)
	}
	// main retorna aqui — todas as goroutines são mortas!
	// Nada é impresso.
}
```

### A Solução: WaitGroup

`WaitGroup` é um contador que você incrementa antes de cada goroutine e decrementa quando ela termina. `Wait()` bloqueia até o contador chegar a zero:

```go
func main() {
	var wg sync.WaitGroup

	for i := range 5 {
		wg.Add(1)   // "vai ter mais uma goroutine para esperar"
		go func(n int) {
			defer wg.Done()   // "terminei" — decrementa o contador
			time.Sleep(100 * time.Millisecond)
			fmt.Printf("worker %d terminou\n", n)
		}(i)
	}

	wg.Wait()   // bloqueia aqui até o contador chegar a zero
	fmt.Println("todas as goroutines terminaram")
}
```

### Como WaitGroup Funciona Internamente

```
WaitGroup tem um contador atômico interno:

wg.Add(3)  →  contador = 3
wg.Done()  →  contador = 2   (equivale a Add(-1))
wg.Done()  →  contador = 1
wg.Done()  →  contador = 0   →  Wait() desbloqueia
```

O contador usa operações atômicas de CPU — não precisa de lock. `Wait()` usa um semáforo do runtime para dormir eficientemente em vez de ficar em loop.

### Regras Críticas do WaitGroup

**Regra 1: `Add()` ANTES do `go`**

```go
// ✅ Correto — Add() antes de disparar a goroutine
wg.Add(1)
go trabalhar()

// ❌ Errado — race condition!
go func() {
	wg.Add(1)    // pode executar DEPOIS de Wait() já ter retornado
	defer wg.Done()
	trabalhar()
}()
```

Por que isso importa? Se você chama `wg.Add(1)` dentro da goroutine, pode acontecer assim:

```
main:       wg.Wait()  ← vê contador = 0, retorna
goroutine:  wg.Add(1)  ← tarde demais, programa já encerrou
```

**Regra 2: Use `defer` para o `Done()`**

```go
// ✅ defer garante que Done() seja chamado mesmo se a goroutine entrar em panic
go func() {
	defer wg.Done()   // sempre executa — mesmo com panic, return antecipado, etc.
	trabalhar()
}()

// ❌ Sem defer — se trabalhar() der panic, Done() nunca é chamado
// e Wait() bloqueia para sempre
go func() {
	trabalhar()
	wg.Done()   // não executa em caso de panic
}()
```

**Regra 3: NUNCA copie um WaitGroup**

```go
// ❌ Cópia do WaitGroup — comportamento indefinido
wg2 := wg
go func(w sync.WaitGroup) { }(wg)   // passa por valor = cópia

// ✅ Sempre use ponteiro
func worker(wg *sync.WaitGroup) {
	defer wg.Done()
	trabalhar()
}
go worker(&wg)
```

### Padrão Completo com WaitGroup

```go
func processarLote(urls []string) []Resultado {
	resultados := make([]Resultado, len(urls))
	var wg sync.WaitGroup

	for i, url := range urls {
		wg.Add(1)
		go func(idx int, u string) {
			defer wg.Done()
			resultados[idx] = buscar(u)   // cada goroutine escreve em índice diferente
		}(i, url)                          // safe: índices não se sobrepõem
	}

	wg.Wait()
	return resultados
}
```

---

## 3. O Que É um Mutex e Por Que Você Precisa Dele

### O Problema — Race Condition

Imagine um contador incrementado por múltiplas goroutines. Parece simples, mas `contador++` **não é uma operação atômica**. Na verdade são três operações separadas:

```go
1. LOAD  → lê o valor atual da memória para um registrador
2. ADD   → soma 1 ao registrador
3. STORE → escreve o resultado de volta na memória
```

Com duas goroutines rodando ao mesmo tempo:

```go
Tempo   Goroutine A          Goroutine B
  1     LOAD  → reg=0
  2                          LOAD  → reg=0
  3     ADD   → reg=1
  4                          ADD   → reg=1
  5     STORE → contador=1
  6                          STORE → contador=1   ← sobrescreveu!

Resultado: contador=1 em vez de 2. Um incremento foi perdido.
```

Isso é uma **race condition**: o resultado depende do timing de execução das goroutines, que é imprevisível. Com 1000 goroutines fazendo isso, você pode obter qualquer valor entre 1 e 1000.

```go
// ❌ Race condition — NUNCA faça isso com múltiplas goroutines
contador := 0
var wg sync.WaitGroup

for range 1000 {
	wg.Add(1)
	go func() {
		defer wg.Done()
		contador++   // não é thread-safe!
	}()
}
wg.Wait()
fmt.Println(contador)   // pode ser 847, 923, 1000... imprevisível
```

Detectar com o race detector:

```bash
go run -race main.go
# WARNING: DATA RACE
# Write at 0x... by goroutine 7:
# Previous write at 0x... by goroutine 6:
```

### A Solução — Mutex

`Mutex` (Mutual Exclusion) é um cadeado: apenas quem tem o cadeado pode entrar na seção crítica. Todas as outras goroutines esperam na fila até o cadeado ser liberado:

```go
var (
	mu       sync.Mutex
	contador int
)

go func() {
	mu.Lock()      // adquire o cadeado — bloqueia se outra goroutine tem
	contador++     // seção crítica — apenas uma goroutine aqui por vez
	mu.Unlock()    // libera o cadeado — próxima goroutine pode entrar
}()
```

Com mutex, as três operações (LOAD, ADD, STORE) de uma goroutine completam **antes** de qualquer outra goroutine poder começar as dela. Race condition eliminada.

---

## 4. `sync.Mutex` — Uso Correto

### Padrão Básico

```go
type ContadorSeguro struct {
	mu    sync.Mutex   // ← mutex junto com os dados que protege
	valor int
}

func (c *ContadorSeguro) Incrementar() {
	c.mu.Lock()
	defer c.mu.Unlock()   // ← sempre use defer para garantir o Unlock
	c.valor++
}

func (c *ContadorSeguro) Valor() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.valor
}

// Uso
contador := &ContadorSeguro{}
var wg sync.WaitGroup

for range 1000 {
	wg.Add(1)
	go func() {
		defer wg.Done()
		contador.Incrementar()
	}()
}

wg.Wait()
fmt.Println(contador.Valor())   // sempre 1000 — correto!
```

### Por Que `defer mu.Unlock()` é Essencial

```go
// ❌ Sem defer — se a função retornar cedo ou entrar em panic, o lock fica preso!
func (c *Cache) Get(k string) string {
	c.mu.Lock()
	v := c.dados[k]
	if v == "" {
		c.mu.Unlock()   // precisa lembrar de desbloquear em cada caminho de retorno
		return "padrão"
	}
	c.mu.Unlock()   // e aqui também
	return v
}

// ✅ Com defer — limpo, seguro, impossível de esquecer
func (c *Cache) Get(k string) string {
	c.mu.Lock()
	defer c.mu.Unlock()   // sempre executa, não importa como a função retorna
	if v := c.dados[k]; v != "" {
		return v
	}
	return "padrão"
}
```

### Manter a Seção Crítica Pequena

O mutex bloqueia **todas as outras goroutines** enquanto está travado. Quanto menor a seção crítica, menos tempo as goroutines ficam esperando:

```go
// ❌ Seção crítica grande — bloqueia outras goroutines desnecessariamente
func (c *Cache) ProcessarEAtualizar(k string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	dados := c.dados[k]
	resultado := processamentoPesado(dados)   // isso não precisa de lock!
	c.dados[k] = resultado
}

// ✅ Seção crítica mínima — lock apenas quando necessário
func (c *Cache) ProcessarEAtualizar(k string) {
	// Leitura protegida
	c.mu.Lock()
	dados := c.dados[k]
	c.mu.Unlock()

	// Processamento sem lock — pode demorar, não atrapalha outras goroutines
	resultado := processamentoPesado(dados)

	// Escrita protegida
	c.mu.Lock()
	c.dados[k] = resultado
	c.mu.Unlock()
}
```

---

## 5. Deadlock — O Perigo do Mutex

Um deadlock acontece quando duas goroutines ficam esperando uma pela outra para sempre. Nenhuma pode continuar:

### Deadlock 1 — Self-lock

```go
// ❌ Goroutine tenta adquirir o próprio lock — trava para sempre
mu.Lock()
// ...
mu.Lock()   // já está travado por esta mesma goroutine — deadlock!
```

`sync.Mutex` não é reentrante — a mesma goroutine não pode adquirir um lock que ela mesma já tem.

### Deadlock 2 — Circular

```go
// ❌ A espera B, B espera A — deadlock circular
var muA, muB sync.Mutex

// Goroutine 1: adquire A, depois tenta B
go func() {
	muA.Lock()
	defer muA.Unlock()
	time.Sleep(1 * time.Millisecond)   // dá tempo para goroutine 2 travar B
	muB.Lock()   // espera muB — goroutine 2 tem
	defer muB.Unlock()
}()

// Goroutine 2: adquire B, depois tenta A
go func() {
	muB.Lock()
	defer muB.Unlock()
	time.Sleep(1 * time.Millisecond)
	muA.Lock()   // espera muA — goroutine 1 tem
	defer muA.Unlock()
}()

// Resultado: deadlock. Nenhuma avança.
// O runtime Go detecta e encerra: "all goroutines are asleep - deadlock!"
```

**Solução: sempre adquira múltiplos locks na mesma ordem:**

```go
// ✅ Ambas as goroutines adquirem na ordem: muA → muB
// Nunca haverá deadlock circular

go func() {
	muA.Lock(); defer muA.Unlock()
	muB.Lock(); defer muB.Unlock()
	// ...
}()

go func() {
	muA.Lock(); defer muA.Unlock()   // mesma ordem
	muB.Lock(); defer muB.Unlock()
	// ...
}()
```

---

## 6. `sync.RWMutex` — Leituras em Paralelo

Às vezes o dado é lido muito mais do que escrito — um cache, uma tabela de configuração, um registro de handlers. `sync.Mutex` bloqueia **tanto leituras quanto escritas** mutuamente. Isso é mais restritivo do que precisa ser.

`RWMutex` tem dois modos:

- **RLock/RUnlock** — leitura: múltiplas goroutines podem ler ao mesmo tempo
- **Lock/Unlock** — escrita: exclusivo, bloqueia todo mundo

```
Com sync.Mutex:          Com sync.RWMutex:
Leitor A  |████|         Leitor A  |████|
Leitor B       |████|    Leitor B  |████|   ← leem juntos!
Leitor C            |████|Leitor C  |████|   ← muito mais rápido
Escritor                 Escritor        |██|  ← escrita exclusiva
```

### Implementando um Cache com RWMutex

```go
type Cache struct {
	mu   sync.RWMutex
	dados map[string]string
}

// Get — leitura: múltiplas goroutines podem chamar simultaneamente
func (c *Cache) Get(k string) (string, bool) {
	c.mu.RLock()          // lock de leitura — não bloqueia outros leitores
	defer c.mu.RUnlock()
	v, ok := c.dados[k]
	return v, ok
}

// Set — escrita: exclusivo, bloqueia leitores e outros escritores
func (c *Cache) Set(k, v string) {
	c.mu.Lock()           // lock de escrita — bloqueia todo mundo
	defer c.mu.Unlock()
	c.dados[k] = v
}

// Delete — escrita: exclusivo
func (c *Cache) Delete(k string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	delete(c.dados, k)
}
```

### Quando RWMutex Vale a Pena

RWMutex é mais complexo internamente do que Mutex. Só use quando:

```
✅ Use RWMutex quando:
   - Leituras são muito mais frequentes que escritas (ratio > 10:1)
   - Seção crítica é longa o suficiente para o overhead valer
   - Múltiplas goroutines concorrem para leitura ao mesmo tempo

❌ Prefira Mutex quando:
   - Mix equilibrado de leituras e escritas
   - Seção crítica é muito curta (nanosegundos)
   - Código mais simples é preferível
```

### Armadilha: Promoção de RLock para Lock

```go
// ❌ NUNCA faça isso — deadlock garantido!
func (c *Cache) GetOuSet(k, defaultVal string) string {
	c.mu.RLock()
	v, ok := c.dados[k]
	if ok {
		c.mu.RUnlock()
		return v
	}
	c.mu.RUnlock()

	c.mu.Lock()              // problema: entre RUnlock e Lock, outro pode ter escrito
	defer c.mu.Unlock()      // e se houver outros esperando RLock, pode deadlock
	c.dados[k] = defaultVal
	return defaultVal
}

// ✅ Correto: use Lock direto quando há chance de escrita
func (c *Cache) GetOuSet(k, defaultVal string) string {
	// Tentativa rápida com RLock
	c.mu.RLock()
	v, ok := c.dados[k]
	c.mu.RUnlock()
	if ok {
		return v
	}

	// Precisa escrever: adquire Lock completo e verifica de novo
	c.mu.Lock()
	defer c.mu.Unlock()
	if v, ok := c.dados[k]; ok {
		return v   // outra goroutine inseriu enquanto esperávamos
	}
	c.dados[k] = defaultVal
	return defaultVal
}
```

---

## 7. `sync.Once` — Inicialização Exatamente Uma Vez

Às vezes você quer inicializar algo preguiçosamente (lazy) — só quando for precisar — mas precisa garantir que a inicialização roda **apenas uma vez**, mesmo que várias goroutines tentem ao mesmo tempo:

```go
type ServicoEmail struct {
	once   sync.Once
	client *smtp.Client
}

func (s *ServicoEmail) cliente() *smtp.Client {
	s.once.Do(func() {
		// Essa função roda UMA ÚNICA VEZ, não importa quantas goroutines chamem
		var err error
		s.client, err = smtp.Dial("smtp.gmail.com:587")
		if err != nil {
			panic("falha ao conectar ao SMTP: " + err.Error())
		}
	})
	return s.client
}

// Mesmo que 100 goroutines chamem ao mesmo tempo:
// - A primeira executa a inicialização
// - As outras 99 esperam ela terminar
// - Depois todas pegam o mesmo client já inicializado
```

**Atenção:** se a função passada para `Do()` entrar em `panic`, o `Once` considera a inicialização "feita" — não tentará novamente em chamadas futuras.

---

## 8. `sync/atomic` — Quando Mutex É Pesado Demais

Para operações simples em tipos primitivos (incrementar um contador, trocar um ponteiro), `sync/atomic` faz operações atômicas diretamente em CPU — sem overhead de lock/unlock:

```go
import "sync/atomic"

// Contador atômico — mais rápido que Mutex para esta operação específica
var contador atomic.Int64

go func() { contador.Add(1) }()   // atômico — thread-safe por design de CPU
go func() { contador.Add(1) }()

// Ler
v := contador.Load()

// Trocar e ler o valor anterior
anterior := contador.Swap(0)   // zera e retorna o valor anterior

// Compare-and-swap — base de algoritmos lock-free
ok := contador.CompareAndSwap(esperado, novo)
// "se o valor atual é 'esperado', troca para 'novo' e retorna true"
// "se não é 'esperado', não faz nada e retorna false"

// Para ponteiros — trocar configuração atomicamente (sem lock!)
var cfg atomic.Pointer[Config]

// Goroutine 1: atualiza a configuração
novaCfg := &Config{Timeout: 60}
cfg.Store(novaCfg)

// Goroutine 2: lê a configuração atual — sempre vê uma versão consistente
atual := cfg.Load()
```

**Quando usar `atomic` vs `Mutex`:**

```
Use atomic quando:
→ Operação simples: incrementar, decrementar, trocar um valor ou ponteiro
→ Precisa de máxima performance (hot path)
→ O tipo é int64, uint64, bool, ou ponteiro

Use Mutex quando:
→ Proteger estruturas de dados complexas (maps, slices, structs)
→ Múltiplas operações precisam ser atômicas juntas
→ A lógica é mais clara com Lock/Unlock explícitos
```

---

## 9. Padrão Completo — Combinando Tudo

Um exemplo real que combina `WaitGroup`, `Mutex` e `atomic`:

```go
type Agregador struct {
	mu       sync.Mutex
	resultados map[string]int   // protegido por mu

	totalProcessados atomic.Int64   // contador simples — atomic é suficiente
	erros           atomic.Int64
}

func (a *Agregador) ProcessarLote(itens []Item) error {
	var wg sync.WaitGroup

	for _, item := range itens {
		wg.Add(1)
		go func(it Item) {
			defer wg.Done()

			resultado, err := processar(it)
			if err != nil {
				a.erros.Add(1)   // atômico — sem mutex
				return
			}

			// Múltiplas operações no map precisam de mutex
			a.mu.Lock()
			a.resultados[it.Categoria] += resultado
			a.mu.Unlock()

			a.totalProcessados.Add(1)   // atômico — sem mutex
		}(item)
	}

	wg.Wait()

	slog.Info("lote processado",
		"total", a.totalProcessados.Load(),
		"erros", a.erros.Load(),
	)
	return nil
}
```

---

## 10. Resumo — Guia de Decisão

```
Problema                                    →  Solução
────────────────────────────────────────────────────────────────
Esperar N goroutines terminarem             →  sync.WaitGroup
Proteger dado com mix de leitura/escrita    →  sync.Mutex
Proteger dado com muito mais leitura        →  sync.RWMutex
Inicialização preguiçosa thread-safe        →  sync.Once
Incrementar/decrementar contador simples    →  sync/atomic
Trocar um ponteiro atomicamente             →  sync/atomic
Comunicar dados entre goroutines            →  channel (preferido)
Cancelamento e timeout                      →  context.Context

Regras gerais:
→ Always defer mu.Unlock() — nunca esqueça o Unlock
→ Adquira múltiplos locks sempre na mesma ordem — evita deadlock
→ Mantenha seções críticas pequenas — menos tempo bloqueando outros
→ Nunca copie Mutex, RWMutex ou WaitGroup — use ponteiros
→ Use -race em todos os testes — go test -race ./...
→ Prefira channels para comunicação; sync para estado compartilhado
```

---

## Conexão com Sistemas Operacionais

### Race Condition — Raiz no Hardware ([[Processadores]])

A race condition de `contador++` não é uma peculiaridade de linguagem — ela é uma consequência direta de como a arquitetura x86 funciona. [[Processadores]] descreve que processadores modernos são superescalares e executam instruções fora de ordem (out-of-order execution). Mesmo ignorando isso, a instrução `contador++` compila para três microoperações distintas que podem ser intercaladas entre cores:

```
Código Go:    contador++

Assembly x86 (sem atomicidade):
  MOVQ  (contador), AX   ; LOAD: lê da memória para registrador
  ADDQ  $1, AX           ; ADD:  soma 1 no registrador
  MOVQ  AX, (contador)   ; STORE: escreve de volta na memória

Com dois goroutines (dois cores) executando ao mesmo tempo:
  Core 1:  LOAD(0) → ADD → ...
  Core 2:  LOAD(0) → ADD → ...       ← ambos leram 0
  Core 1:  ... → STORE(1)
  Core 2:  ... → STORE(1)            ← ambos escrevem 1
  Resultado: 1 em vez de 2. Um incremento perdido.
```

Esse problema existe em qualquer linguagem ou sistema: C, Java, Rust (unsafe), threads POSIX — sempre que dois fluxos de execução concorrente compartilham memória sem coordenação.

### sync.Mutex — Equivalente de pthread_mutex ([[Threads POSIX]])

[[Threads POSIX]] define `pthread_mutex_lock()` e `pthread_mutex_unlock()` como o mecanismo POSIX padrão de exclusão mútua. `sync.Mutex` é a implementação Go do mesmo conceito:

```
POSIX (C):                     Go:
pthread_mutex_t mu;            var mu sync.Mutex
pthread_mutex_lock(&mu);       mu.Lock()
  /* seção crítica */            // seção crítica
pthread_mutex_unlock(&mu);     mu.Unlock()
```

A diferença principal está na ergonomia: Go tem `defer mu.Unlock()` para garantir o desbloqueio mesmo em caso de panic, enquanto em C o programador deve gerenciar todos os caminhos de retorno manualmente — fonte clássica de deadlocks em código C com múltiplos `return`.

Internamente, `sync.Mutex` em Go é implementado com operações atômicas de CPU para o caminho sem contention (fast path) e syscalls do SO (`futex` no Linux, equivalente no macOS) para suspender goroutines quando o lock está contido — exatamente como `pthread_mutex` funciona internamente.

### Operações Atômicas — O LOCK Prefix do x86 ([[Processadores]])

[[Processadores]] menciona que processadores x86 têm o prefixo `LOCK` que transforma uma instrução read-modify-write em uma operação atômica indivisível. O cache coherence protocol (MESI) garante que apenas um core pode modificar uma linha de cache por vez.

`sync/atomic` expõe essas instruções diretamente:

```
Go:  contador.Add(1)

Assembly x86 gerado:
  LOCK XADDQ $1, (contador)
  ↑
  Prefixo LOCK: garante que a operação é atômica
  Nenhum outro core pode interromper entre a leitura e a escrita
  O barramento de memória é "travado" durante a instrução
```

Isso é mais eficiente que um mutex porque não envolve nem syscall nem troca de contexto — é uma instrução de hardware que leva poucos ciclos de clock. A operação `CompareAndSwap` (`CAS`) compila para `LOCK CMPXCHG` e é a base de todos os algoritmos lock-free.

A diferença de performance é significativa:
- Mutex Lock/Unlock: ~25 ns (envolve operações atômicas + potencial syscall)
- atomic.Add: ~2-3 ns (instrução LOCK direta)

### Deadlock — Conceito Universal de SO

O deadlock circular (Goroutine A espera B, B espera A) é uma instância do problema clássico de deadlock de SO descrito no contexto de processos e threads: quatro condições de Coffman devem ser satisfeitas simultaneamente para que um deadlock ocorra:

```
1. Exclusão mútua:   o recurso (lock) só pode ser detido por um thread por vez
2. Hold-and-wait:    um thread detém um lock enquanto espera por outro
3. Sem preempção:    um lock não pode ser tomado à força — o thread deve liberá-lo
4. Espera circular:  A espera por B, B espera por A (ou A→B→C→A)

Solução canônica de SO: impor ordenação total nos recursos
→ Todos os threads adquirem locks na mesma ordem
→ Impossibilita a condição 4 (espera circular)
```

O runtime Go detecta o caso especial em que *todas* as goroutines estão bloqueadas (deadlock global) e encerra o programa com `all goroutines are asleep - deadlock!`. Entretanto, deadlocks parciais (onde algumas goroutines rodam mas outras ficam presas) não são detectados automaticamente.

### sync.WaitGroup vs pthread_join ([[Threads POSIX]])

[[Threads POSIX]] define `pthread_join(tid, &retval)` para esperar que uma thread específica termine. `WaitGroup` generaliza isso para N threads sem precisar rastrear IDs individuais:

```
POSIX — espera thread específica:
  pthread_t t1, t2, t3;
  pthread_create(&t1, NULL, worker, arg1);
  pthread_create(&t2, NULL, worker, arg2);
  pthread_create(&t3, NULL, worker, arg3);
  pthread_join(t1, NULL);   // espera t1
  pthread_join(t2, NULL);   // espera t2
  pthread_join(t3, NULL);   // espera t3

Go — espera N goroutines sem rastrear IDs:
  var wg sync.WaitGroup
  for i := range 3 {
      wg.Add(1)
      go func(n int) { defer wg.Done(); trabalhar(n) }(i)
  }
  wg.Wait()   // espera todas — sem precisar de IDs
```

A diferença de design é significativa: `pthread_join` exige que você tenha o `pthread_t` da thread criada. Com WaitGroup, você pode criar goroutines em loops, funções auxiliares, ou lugares onde não tem acesso ao "handle" da goroutine — o contador interno é suficiente.

Internamente, `wg.Wait()` não faz busy-waiting (ficar em loop verificando o contador) — ele usa um semáforo do runtime para dormir eficientemente até o contador chegar a zero, similar a `pthread_cond_wait` em POSIX.

### sync.Once — Inicialização de Processo ([[Processos]])

`sync.Once` resolve o problema de inicialização única em ambiente concorrente. Em [[Processos]], o processo tem uma fase de inicialização clara: o SO carrega o executável, inicializa a BSS (variáveis globais zeradas), executa construtores de bibliotecas e só então chama `main()`. Tudo isso acontece em um único fluxo sequencial — sem concorrência.

Em Go, com múltiplas goroutines que podem querer usar um recurso compartilhado (conexão de banco, cliente HTTP), a inicialização lazy (na primeira vez que é necessária) precisa ser thread-safe:

```
Sem sync.Once — race condition na inicialização:
  G1: if client == nil { client = criarCliente() }  ← lê nil
  G2: if client == nil { client = criarCliente() }  ← lê nil (antes de G1 terminar)
  G1: client = novoCliente1
  G2: client = novoCliente2  ← sobrescreveu! dois clientes criados, recurso vazado

Com sync.Once — correto:
  once.Do(func() { client = criarCliente() })
  → a função é chamada exatamente uma vez, não importa quantas goroutines
  → as outras goroutines bloqueiam até a inicialização terminar
  → depois todas usam o mesmo client já inicializado
```

`sync.Once` é implementado internamente com uma flag atômica e um mutex: a flag é verificada com uma operação atômica (fast path, custo mínimo para chamadas subsequentes), e o mutex protege o caso em que duas goroutines chegam simultaneamente antes da primeira execução (slow path).