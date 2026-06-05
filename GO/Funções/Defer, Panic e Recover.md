---
tags:
  - go
  - go/funções
---
# Defer, Panic e Recover

> Três mecanismos que trabalham em conjunto. `defer` adiada execução garantindo limpeza de recursos. `panic` interrompe o fluxo em situações irrecuperáveis. `recover` captura panics para tratamento controlado. Juntos formam o sistema de tratamento de erros excepcionais de Go.
> 

---

## 1. `defer` — Implementação e Garantias

### Como o Runtime Implementa Defer

```
Antes do Go 1.14:
- Cada defer cria uma struct "deferred call" na heap
- Armazenada em uma lista ligada no goroutine frame
- Overhead: ~40ns por defer (alocação + inserção na lista)

Go 1.14+: "Open-coded defer" (otimização)
- Para funções com ≤ 8 defers em caminhos não-loop:
  O compilador INLINA os defers diretamente no código
- Zero overhead na maioria dos casos
- Apenas defers em loops ou situações dinâmicas usam a lista

Verificar se um defer foi inlined:
go build -gcflags="-d=ssa/opt/debug=1" — mostra otimizações SSA
```

### Uso Básico

```go
func lerArquivo(caminho string) (string, error) {
	f, err := os.Open(caminho)
	if err != nil {
		return "", fmt.Errorf("lerArquivo: %w", err)
	}
	defer f.Close()   // GARANTIDO — executa quando lerArquivo retornar

	// f.Close() executa aqui mesmo se houver panic ou return antecipado
	dados, err := io.ReadAll(f)
	if err != nil {
		return "", fmt.Errorf("lerArquivo: ler: %w", err)
	}
	return string(dados), nil
}
```

### Múltiplos Defers — Ordem LIFO

```go
func demonstrarLIFO() {
	defer fmt.Println("3º — último registrado, primeiro a executar")
	defer fmt.Println("2º")
	defer fmt.Println("1º — primeiro registrado, último a executar")
	fmt.Println("corpo da função")
}
// corpo da função
// 3º — último registrado, primeiro a executar
// 2º
// 1º — primeiro registrado, último a executar
```

### Argumentos Avaliados no Momento do `defer`

```go
x := 10
defer fmt.Println("x =", x)   // x=10 é capturado AGORA
x = 99
// Quando executar: imprime "x = 10" (não 99)

// Para capturar o valor final, use uma closure
defer func() { fmt.Println("x =", x) }()   // captura &x
x = 99
// Quando executar: imprime "x = 99"
```

---

## 2. Padrões Essenciais de `defer`

### Unlock de Mutex

```go
var mu sync.Mutex

func atualizarDados(dados *Dados) {
	mu.Lock()
	defer mu.Unlock()   // sempre desbloqueia, mesmo em panic

	dados.Valor++
	dados.UltimaAtualização = time.Now()
}
```

### Fechar Conexões de Banco

```go
func executarQuery(db *sql.DB, query string) (*sql.Rows, error) {
	rows, err := db.Query(query)
	if err != nil {
		return nil, err
	}
	// ERRADO: o chamador deve fechar rows
	// Sempre documente quem é responsável por fechar!
	return rows, nil
}

// Padrão correto para transação
func executarTransacao(db *sql.DB, fn func(*sql.Tx) error) (err error) {
	tx, err := db.Begin()
	if err != nil {
		return fmt.Errorf("iniciar transação: %w", err)
	}

	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p)   // re-panic após rollback
		} else if err != nil {
			tx.Rollback()   // rollback se houve erro
		} else {
			err = tx.Commit()   // commit se tudo ok — modifica retorno nomeado!
		}
	}()

	return fn(tx)
}
```

### Medição de Tempo

```go
func medirTempo(nome string) func() {
	inicio := time.Now()
	return func() {
		slog.Info("duração", "operação", nome, "elapsed", time.Since(inicio))
	}
}

// Uso:
func processarPedido(id int) {
	defer medirTempo("processarPedido")()   // ← () chama medirTempo, defer recebe a closure
	// ... lógica
}
```

### `defer` com Retorno Nomeado — Modificar o Resultado

```go
// O defer pode modificar o valor de retorno nomeado!
func dividirSeguro(a, b float64) (resultado float64, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panic na divisão: %v", r)
			resultado = 0
		}
	}()

	return a / b, nil   // panic se b == 0 (divisão por zero)
}

r, err := dividirSeguro(10, 0)
fmt.Println(r, err)   // 0 "panic na divisão: ..."
```

---

## 3. `panic` — Quando Usar

`panic` interrompe a execução imediatamente. Os defers ainda executam (em ordem LIFO), depois o programa encerra com stack trace — a menos que `recover` capture o panic.

### Quando é Apropriado

```go
// ✅ Bug de programação — estado impossível
func novoServidor(porta int) *Servidor {
	if porta < 1 || porta > 65535 {
		panic(fmt.Sprintf("porta inválida: %d — bug do programador", porta))
	}
	return &Servidor{porta: porta}
}

// ✅ Invariante violada — que nunca deveria acontecer
func (t *ArvoreB) rotacionar() {
	if t.raiz == nil {
		panic("BUG: tentativa de rotacionar árvore vazia")
	}
	// ...
}

// ✅ Inicialização — falha fatal no startup
func init() {
	var err error
	config, err = carregarConfig()
	if err != nil {
		// Sem config, o programa não pode funcionar
		panic(fmt.Sprintf("falha na inicialização: %v", err))
	}
}

// ✅ Panics gerados automaticamente pelo runtime:
var s []int
_ = s[10]              // panic: index out of range
var p *int
_ = *p                 // panic: nil pointer dereference
var i interface{} = 1
_ = i.(string)        // panic: interface conversion
ch := make(chan int)
close(ch)
ch <- 1               // panic: send on closed channel

// ❌ NUNCA use panic para erros de negócio esperados
func buscarUsuario(id int) *Usuario {
	u, ok := db[id]
	if !ok {
		panic("usuário não encontrado")   // ERRADO — use error!
	}
	return u
}
```

### Propagação pela Call Stack

```go
func a() { b() }
func b() { c() }
func c() { panic("algo deu muito errado") }

// a() chama b() chama c() que faz panic
// Execução:
// 1. c() para, executa seus defers, propaga panic para b()
// 2. b() executa seus defers, propaga para a()
// 3. a() executa seus defers, propaga para main()
// 4. main() executa seus defers, programa encerra com stack trace
```

---

## 4. `recover` — Capturar Panics

`recover` para a propagação do panic e retorna o valor passado para `panic()`. **Só funciona dentro de um `defer`**:

```go
// ✅ Correto: recover dentro de defer
defer func() {
	if r := recover(); r != nil {
		fmt.Println("capturado:", r)
	}
}()
panic("test")

// ❌ Errado: recover fora de defer — não funciona
if r := recover(); r != nil { }   // sempre retorna nil

// ❌ Errado: recover em defer aninhado — não funciona
defer func() {
	func() {
		recover()   // não captura o panic do goroutine pai
	}()
}()
```

### Converter Panic em Error — Padrão de Biblioteca

```go
// Padrão comum em bibliotecas que usam panic internamente
// mas expõem API baseada em error externamente
func executarSeguro(fn func()) (err error) {
	defer func() {
		if r := recover(); r != nil {
			// Determinar o tipo do panic
			switch v := r.(type) {
			case error:
				err = fmt.Errorf("panic: %w", v)
			case string:
				err = fmt.Errorf("panic: %s", v)
			default:
				err = fmt.Errorf("panic: %v (tipo: %T)", v, v)
			}
			// Incluir stack trace para debug
			buf := make([]byte, 4096)
			n := runtime.Stack(buf, false)
			slog.Error("panic capturado", "stack", string(buf[:n]))
		}
	}()
	fn()
	return nil
}

// Uso
err := executarSeguro(func() {
	var m map[string]int
	m["chave"] = 1   // panic: assignment to entry in nil map
})
fmt.Println(err)   // "panic: assignment to entry in nil map"
```

### Recover em Handler HTTP — Proteger o Servidor

```go
import "runtime/debug"

func middlewareRecovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				// Log completo para debug interno
				slog.Error("panic em handler HTTP",
					"err", err,
					"path", r.URL.Path,
					"method", r.Method,
					"stack", string(debug.Stack()),
				)

				// Resposta segura para o cliente (sem expor detalhes internos)
				http.Error(w, "Erro interno do servidor", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

---

## 5. `panic` e `recover` com Goroutines

**Recover só captura panics da mesma goroutine:**

```go
// ❌ Recover de uma goroutine NÃO captura panic de outra
defer func() {
	recover()   // não captura o panic da goroutine abaixo
}()

go func() {
	panic("panic em goroutine separada")   // derruba o programa inteiro!
}()

// ✅ Cada goroutine deve ter seu próprio recover
go func() {
	defer func() {
		if r := recover(); r != nil {
			slog.Error("panic na goroutine worker", "err", r)
		}
	}()
	trabalhoPerigoso()
}()
```

---

## 6. Boas Práticas — Quando Usar Cada Um

```go
             ┌─────────────────────────────────────────────┐
             │ Situação                  │ Use              │
             ├─────────────────────────────────────────────┤
             │ Arquivo não encontrado    │ return error      │
             │ Entrada do usuário ruim   │ return error      │
             │ Timeout de rede           │ return error      │
             │ Bug de programação        │ panic             │
             │ Estado irrecuperável      │ panic             │
             │ Índice fora do range      │ runtime panic     │
             │ Nil pointer               │ runtime panic     │
             │ Boundary de serviço HTTP  │ recover + error   │
             │ Entrada de goroutine      │ recover + log     │
             │ Biblioteca interna        │ recover → error   │
             └─────────────────────────────────────────────┘

Regra geral:
- Se o caller pode fazer algo útil com o problema → return error
- Se é um bug que não deveria acontecer → panic
- Se é uma boundary de sistema (HTTP, goroutine) → recover
```

---

## 7. Resumo de Comportamentos

```go
defer:
- Executa quando a função retorna (normal, return, panic)
- Ordem: LIFO (último registrado, primeiro executado)
- Argumentos: avaliados imediatamente (não na execução)
- Closures: capturam variáveis por referência
- Pode modificar retornos nomeados
- Go 1.14+: inlined em casos simples (zero overhead)

panic:
- Para execução imediatamente
- Executa todos os defers da goroutine (LIFO)
- Propaga pela call stack
- Se não capturado: crash com stack trace
- NÃO é capturado por goroutines diferentes

recover:
- Captura o panic em andamento
- Retorna o valor passado para panic()
- Só funciona DENTRO de um defer
- Só captura panics da MESMA goroutine
- Após recover, a execução continua normalmente após o defer
```

---

## 8. Conexão com Sistemas Operacionais

### `panic` — Exceções de Hardware Viram Panics

```go
// Nil pointer dereference → exceção de hardware → panic em Go:
// 1. CPU tenta acessar endereço 0x0 (ou próximo)
// 2. MMU detecta: página não mapeada → Page Fault (exceção #14 x86)
// 3. Kernel recebe interrupção, identifica o processo culpado
// 4. Kernel envia SIGSEGV para o processo
// 5. Go runtime tem signal handler para SIGSEGV → converte em runtime panic
// 6. panic propaga pela call stack, executando defers (stack unwinding)
//
// Em C: o mesmo acesso → crash direto (SIGSEGV não capturado)
// Em Go: o runtime "intercepta" o sinal e gerencia graciosamente

var p *int
_ = *p   // nil pointer → Page Fault → SIGSEGV → runtime panic
         //                            ↑ [[Processadores]] trap/exceção
         //                                       ↑ [[Processos]] sinais POSIX
```

Cada tipo de panic por runtime corresponde a uma exceção de CPU:
- `nil pointer dereference` → Page Fault → SIGSEGV → [[Gerenciamento de Memória]]
- `index out of range` → verificação de bounds pelo compilador (não é exceção de CPU)
- `integer divide by zero` → Divide Error (#DE, exceção #0 x86) → SIGFPE → [[Processadores]]
- `send on closed channel` → detecção pelo runtime Go, não pelo hardware

### `recover` — Análogo ao Signal Handler + setjmp/longjmp

```go
// POSIX: signal(SIGSEGV, handler) registra função para tratar sinal
// Go: defer + recover registra função para capturar panic
//
// C com setjmp/longjmp (mecanismo similar de non-local exit):
// sigjmp_buf env;
// if (sigsetjmp(env, 1) == 0) {
//     codigo_perigoso();
// } else {
//     fprintf(stderr, "recuperado\n");  // análogo ao recover()
// }
//
// Go:
defer func() {
    if r := recover(); r != nil {
        log.Println("recuperado:", r)   // signal handler em Go
    }
}()
codigoPerigoso()
//
// setjmp salva o contexto de CPU (registradores, SP, PC) para
// longjmp restaurar depois — recover() faz algo similar internamente
// via estruturas do runtime que rastreiam o ponto de panic
```

**Conexão**: [[Processos]] → tratamento de sinais, signal handlers; setjmp/longjmp como mecanismo de non-local goto no nível de processo

### `defer` — Stack de Cleanup Análogo a `atexit()`

```go
// atexit() em C registra funções para executar quando o processo termina
// defer em Go registra para executar quando a FUNÇÃO retorna
//
//   C: atexit(cleanup)    → escopo: processo inteiro (LIFO)
//   Go: defer cleanup()   → escopo: função atual (LIFO)
//
// Ambos usam semântica LIFO, espelhando a própria pilha de chamadas:
//
// defer 1  ← registrado primeiro (fundo da pilha de defers)
// defer 2
// defer 3  ← registrado último (topo)
// --- função retorna ---
// executa: defer 3, defer 2, defer 1   (LIFO = ordem de desempilhamento)
//
// A pilha de defers espelha o call stack gerenciado via SP/FP:
// cada defer pertence a um frame — quando o frame é destruído (ret),
// seus defers executam, liberando recursos do frame (RAII em Go)
```

**Conexão**: [[Processos]] → call stack, stack pointer (SP), frames de chamada; [[Processadores]] → registradores SP, FP e instrução `ret`

### Stack Unwinding Durante Panic

```
panic em c() → unwind frame por frame, executando defers:

  SP → [frame main]
       [frame a()]
       [frame b()]
       [frame c()]  ← panic aqui
                       ↓ executa defers de c(), propaga
                    [frame b()]  ← executa defers de b(), propaga
                    [frame a()]  ← executa defers de a(), propaga
                    [frame main] ← executa defers de main(), crash final
```

```go
// C++ usa exceções com stack unwinding similar (RAII + destructors)
// Java usa try/catch/finally com unwinding na JVM
// Kernel Linux usa unwinding em exception handlers de arquitetura
//
// runtime.Callers() / runtime.Stack() expõe o call stack — análogo a:
// backtrace() / backtrace_symbols() do POSIX glibc
buf := make([]byte, 4096)
n := runtime.Stack(buf, false)   // false = só esta goroutine
// true = TODAS as goroutines (como listar /proc/$pid/task/)
```

**Conexão**: [[Processos]] → estrutura da pilha (stack frames, base pointer, return address), core dumps; [[Threads POSIX]] → backtrace por thread

### Por Que Panic Derruba o Programa Inteiro

```go
// Panic não capturado em goroutine → crash de TODAS as goroutines
//
// Analogia POSIX:
// SIGSEGV não tratado em qualquer thread de um processo
// → entregue a uma thread → processo inteiro termina (todas as threads)
//
// Razão: threads compartilham espaço de endereçamento
// → estado de memória pode estar corrompido
// → não faz sentido continuar com outros threads sobre heap corrompida
//
// Go adota a mesma semântica:
// goroutines compartilham heap do processo (via G-P-M scheduler)
// → panic não capturado = estado irrecuperável = crash total
// → [[Goroutines]] para o modelo G-P-M e compartilhamento de memória
```

**Conexão**: [[Threads POSIX]] → threads compartilham espaço de endereçamento; [[Processos]] → sinais e terminação de processo; [[Goroutines]] → modelo G-P-M