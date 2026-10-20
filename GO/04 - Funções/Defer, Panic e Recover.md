---
tags:
  - go
  - go/funcoes
aliases:
  - defer
  - panic
  - recover
  - Panics em Go
ordem: 20
modulo: "04 - Funções"
---
# Defer, Panic e Recover

> **Nota 20 da trilha de Go.** Fechamos o módulo de funções com três mecanismos ligados ao **fim da execução de uma função**. `defer` agenda uma chamada para rodar quando a função terminar (seja como for), garantindo limpeza de recursos. `panic` interrompe a execução normal quando algo **irrecuperável** acontece, desenrolando a stack. `recover` permite capturar um panic dentro de um `defer`. Veremos as regras exatas de cada um, os padrões idiomáticos, a fronteira entre **`panic` e `error`**, o que acontece com goroutines e como isso tudo é implementado.
>
> **Pré-requisitos:** [[Múltiplos Retornos e Erros]], [[Closures e Funções Anônimas]].

---

## 🧭 De onde viemos

Em [[Múltiplos Retornos e Erros]] vimos que falhas **esperadas** são `error`, e que resultados nomeados podem ser alterados "depois do `return`", dentro de um `defer`. Em [[Closures e Funções Anônimas]] vimos funções anônimas que capturam variáveis. Também já esbarramos em vários panics: índice fora dos limites, map nil, ponteiro nil, divisão por zero. Agora vamos entender o mecanismo completo.

---

## ⏳ 1. `defer`: "faça isto quando eu sair"

> 💡 **`defer`:** agenda uma chamada de função para ser executada **imediatamente antes** de a função atual retornar, não importa por qual caminho ela saia: `return` normal, `return` antecipado em caso de erro, ou panic.

O uso clássico é **liberar recursos** logo depois de adquiri-los:

```go
func lerArquivo(caminho string) ([]byte, error) {
	f, err := os.Open(caminho)
	if err != nil {
		return nil, err
	}
	defer f.Close()          // ✅ escrito AO LADO da abertura; roda ao sair, por qualquer caminho

	info, err := f.Stat()
	if err != nil {
		return nil, err      // f.Close() roda aqui
	}
	buf := make([]byte, info.Size())
	_, err = io.ReadFull(f, buf)
	return buf, err          // e aqui também
}
```

Sem `defer`, seria preciso chamar `f.Close()` antes de **cada** `return`, e um caminho esquecido vira vazamento de descritor de arquivo. O `defer` mantém a aquisição e a liberação **juntas** no código.

Pares típicos:

```go
f, _ := os.Open(p);      defer f.Close()
mu.Lock();               defer mu.Unlock()
resp, _ := http.Get(u);  defer resp.Body.Close()
tx, _ := db.Begin();     defer tx.Rollback()      // não faz nada se já houve Commit
ctx, cancel := context.WithTimeout(ctx, 5*time.Second); defer cancel()
inicio := time.Now();    defer func() { log.Println(time.Since(inicio)) }()
```

---

## 📏 2. As três regras do `defer`

### Regra 1 — Ordem LIFO (o último agendado roda primeiro)

Saída real:

```go
func ordem() {
	for i := range 3 {
		defer fmt.Print(i, " ")
	}
	fmt.Print("corpo ")
}
// corpo 2 1 0
```

Os defers formam uma **pilha**: o último empilhado é o primeiro a sair. Isso é natural para recursos dependentes: se você abriu A e depois B (que depende de A), deve fechar B antes de A.

### Regra 2 — Os argumentos são avaliados no momento do `defer`

A **chamada** é adiada, mas os **argumentos** são calculados na hora em que o `defer` é executado. Com uma closure, o que é capturado é a **variável** (saída real):

```go
func avaliacao() {
	x := 1
	defer fmt.Println("defer viu x =", x)                  // x avaliado AGORA: 1
	x = 2
	defer func() { fmt.Println("closure viu x =", x) }()   // captura a variável x
	x = 3
}
// closure viu x = 3
// defer viu x = 1
```

### Regra 3 — `defer` pode ler e alterar resultados nomeados

O `return` acontece em duas etapas: **(1)** atribui os valores aos resultados, **(2)** roda os defers, **(3)** sai. Um defer roda entre (1) e (3) e pode alterar resultados **nomeados** (saída real):

```go
func dobrarResultado() (n int) {
	defer func() { n *= 2 }()
	return 21                // n = 21 → defer: n = 42 → retorna 42
}
fmt.Println(dobrarResultado())    // 42
```

O uso real disso é **anotar erros** ou **capturar o erro do `Close`**:

```go
func salvar(caminho string, dados []byte) (err error) {
	f, err := os.Create(caminho)
	if err != nil {
		return err
	}
	defer func() {
		if cerr := f.Close(); cerr != nil && err == nil {
			err = cerr       // ✅ num arquivo escrito, o Close pode falhar (dados não gravados!)
		}
	}()
	_, err = f.Write(dados)
	return err
}
```

> ⚠️ Em arquivos **abertos para escrita**, ignorar o erro de `Close()` pode esconder perda de dados (o sistema pode só gravar de verdade no `close`). Para leitura, `defer f.Close()` simples é aceitável. Desde o Go 1.20 há também `errors.Join(err, f.Close())` para juntar os dois.

---

## 🔁 3. Cuidado: `defer` dentro de laços

```go
// ❌ os arquivos só fecham quando a FUNÇÃO termina, não a cada volta
for _, caminho := range milArquivos {
	f, _ := os.Open(caminho)
	defer f.Close()          // acumula 1000 defers e 1000 arquivos abertos!
	processar(f)
}

// ✅ extraia o corpo para uma função: o defer roda ao fim de cada chamada
for _, caminho := range milArquivos {
	if err := processarArquivo(caminho); err != nil {
		return err
	}
}

func processarArquivo(caminho string) error {
	f, err := os.Open(caminho)
	if err != nil {
		return err
	}
	defer f.Close()
	return processar(f)
}
```

`defer` é por **função**, não por bloco. Com mil arquivos, a versão errada pode estourar o limite de descritores abertos do processo (`ulimit -n`).

---

## 💥 4. `panic`: quando continuar não faz sentido

> 💡 **Panic:** interrupção da execução normal de uma goroutine. A função atual para, seus defers rodam, e o panic **sobe** para a função que a chamou, que também para e roda seus defers, e assim por diante (**desenrolar a stack**). Se chegar ao topo da goroutine sem ser recuperado, o **programa inteiro** termina com uma mensagem e um stack trace (código de saída 2).

### 4.1 Panics do runtime

O próprio runtime causa panic em erros de programação. Mensagens reais:

| Causa | Mensagem |
| --- | --- |
| Índice fora dos limites | `runtime error: index out of range [3] with length 0` |
| Escrever em map nil | `assignment to entry in nil map` |
| Desreferenciar nil | `runtime error: invalid memory address or nil pointer dereference` |
| Divisão inteira por zero | `runtime error: integer divide by zero` |
| Type assertion inválida | `interface conversion: interface {} is string, not int` |
| Fechar channel fechado | `close of closed channel` |
| Comparar tipos não comparáveis em interface | `runtime error: comparing uncomparable type []int` |
| Deslocamento negativo | `runtime error: negative shift amount` |

Esses panics (os prefixados com `runtime error:`) implementam a interface **`runtime.Error`**.

### 4.2 Chamando `panic` você mesmo

`panic` aceita **qualquer valor** (`panic(v any)`), mas o idiomático é uma string ou um `error`:

```go
func MustCompile(expr string) *regexp.Regexp {
	re, err := regexp.Compile(expr)
	if err != nil {
		panic(`regexp: Compile(` + expr + `): ` + err.Error())
	}
	return re
}

var emailRe = regexp.MustCompile(`^[^@]+@[^@]+$`)   // usado em variáveis globais
```

### 4.3 Quando usar `panic` (e quando não)

| ✅ Use `panic` para... | ❌ Não use `panic` para... |
| --- | --- |
| **Erros de programação**: invariantes quebradas, estados "impossíveis" | Arquivo inexistente, rede fora, timeout |
| Funções `MustX` usadas na inicialização com valores constantes (`regexp.MustCompile`, `template.Must`) | Entrada inválida do usuário |
| Configuração obrigatória ausente na inicialização (o programa não tem como funcionar) | Qualquer coisa que o chamador possa querer tratar |
| Dentro de um pacote, como atalho interno, desde que **convertido em error** antes de sair do pacote (ex.: parsers recursivos) | Controle de fluxo "como exceções" entre pacotes |

> ✅ **Regra de ouro:** uma biblioteca **não deve deixar panics escaparem** pela sua API pública por causa de dados de entrada. Falhas esperadas viram `error`. Panic é para bugs.

---

## 🛟 5. `recover`: capturando um panic

> 💡 **`recover()`:** função embutida que, chamada **diretamente dentro de uma função adiada** (`defer`), interrompe o desenrolar de um panic e devolve o valor que foi passado para `panic`. Fora dessa situação, devolve `nil` e não faz nada.

O padrão: **converter panic em error** (saídas reais):

```go
func seguro(f func()) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("recuperado: %v", r)
		}
	}()
	f()
	return nil
}

fmt.Println(seguro(func() { panic("algo deu errado") }))
// recuperado: algo deu errado
fmt.Println(seguro(func() { var m map[string]int; m["a"] = 1 }))
// recuperado: assignment to entry in nil map
fmt.Println(seguro(func() { s := []int{}; _ = s[3] }))
// recuperado: runtime error: index out of range [3] with length 0
fmt.Println(seguro(func() {}))
// <nil>
```

Regras importantes:

```go
fmt.Println(recover())      // <nil>: fora de um defer, não faz nada

defer recover()             // ⚠️ funciona, mas é obscuro; prefira a forma explícita

defer func() {
	helper()                // ❌ se helper() chamar recover(), NÃO funciona:
}()                         //    precisa ser chamado DIRETAMENTE pela função adiada
```

### 5.1 Inspecionando o que foi recuperado (saída real)

```go
defer func() {
	r := recover()
	e, ok := r.(error)                    // o valor pode ser um error...
	var re runtime.Error
	fmt.Println(ok, errors.As(e, &re), re)
}()
var p *int
_ = *p
// true true runtime error: invalid memory address or nil pointer dereference
```

### 5.2 `panic(nil)` (Go 1.21+)

Antigamente, `panic(nil)` fazia `recover()` devolver `nil`, o que era indistinguível de "não houve panic". Desde o Go 1.21, `panic(nil)` vira um `*runtime.PanicNilError` (saída real):

```go
panic(nil)
// recover() devolve: *runtime.PanicNilError  "panic called with nil argument"
```

### 5.3 Onde `recover` é usado de verdade

1. **Servidores:** um panic num handler HTTP não deve derrubar o servidor inteiro. O `net/http` já recupera panics de cada requisição (e registra no log); frameworks oferecem middlewares de "recovery"
2. **Fronteira de pacotes** que usam panic internamente (o `encoding/json` faz isso no seu decodificador recursivo e devolve um `error`)
3. **Workers de longa duração:** recuperar, registrar e continuar com a próxima tarefa

```go
func comRecuperacao(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				slog.Error("panic no handler", "erro", rec, "stack", string(debug.Stack()))
				http.Error(w, "erro interno", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

> ⚠️ Recuperar não "conserta" o problema: depois de um panic, o estado do programa pode estar inconsistente (um lock que ficou travado, uma estrutura pela metade). Recupere **na fronteira** (requisição, tarefa), registre com o stack trace, e descarte o trabalho em andamento.

---

## 🧵 6. Panics e goroutines

> ⚠️ **Um panic não recuperado em QUALQUER goroutine derruba o programa inteiro**, e um `recover` numa goroutine **não** captura panics de outra goroutine.

Saída real: o `main` tem um `defer`, mas uma goroutine filha entra em panic:

```go
func main() {
	defer fmt.Println("defer do main")    // NÃO roda!
	go func() {
		panic("panic numa goroutine")
	}()
	time.Sleep(100 * time.Millisecond)
}
```

```
panic: panic numa goroutine

goroutine 6 [running]:
main.main.func1()
	/tmp/h/dp/g/main.go:16 +0x25
created by main.main in goroutine 1
exit status 2
```

Repare: nem o defer do `main` rodou. Por isso, toda goroutine que executa código "de terceiros" ou de longa duração deve ter seu próprio `recover`:

```go
func goSeguro(f func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				slog.Error("goroutine em panic", "erro", r)
			}
		}()
		f()
	}()
}
```

Veremos goroutines em [[Goroutines]].

---

## 🚪 7. Saídas que pulam o `defer`

| Como o programa termina | Defers rodam? |
| --- | --- |
| `return` (inclusive antecipado) | ✅ Sim |
| Panic (recuperado ou não) | ✅ Sim, na goroutine que entrou em panic |
| `runtime.Goexit()` (encerra só a goroutine) | ✅ Sim |
| `os.Exit(n)` | ❌ **Não** |
| `log.Fatal(...)` (chama `os.Exit(1)`) | ❌ **Não** |
| Panic em **outra** goroutine | ❌ Não (o processo morre na hora) |
| Sinal externo (`kill -9`, Ctrl+C sem tratamento) | ❌ Não |

Saída real: com `os.Exit(3)`, o defer não roda e o código de saída é 3:

```go
defer fmt.Println("este defer NÃO roda")
os.Exit(3)
```

> ✅ Um padrão comum: deixar `main` minúsculo e colocar a lógica numa função `run() error`, para que todos os defers de `run` rodem antes do `os.Exit`:
> ```go
> func main() {
> 	if err := run(); err != nil {
> 		fmt.Fprintln(os.Stderr, err)
> 		os.Exit(1)
> 	}
> }
> ```

Para encerrar de forma limpa em sinais como `SIGTERM` (o que o Docker envia ao parar um container, ver [[Processo raiz (PID 1)]]), use `signal.NotifyContext` (ver [[context.Context]]).

---

## 🔬 8. Por dentro: como `defer` e `panic` funcionam

### 8.1 Defers "open-coded" (Go 1.14+)

Nas primeiras versões de Go, cada `defer` alocava um registro numa lista ligada da goroutine, e custava ~35 ns. Desde o **Go 1.14**, na maioria dos casos o compilador faz **open-coding**: insere as chamadas adiadas **diretamente no código de saída** da função, controladas por um byte de bits que marca quais `defer` foram executados:

```
func f() {                         código gerado (simplificado):
	defer a()                      deferBits |= 1
	if cond {
		defer b()                  deferBits |= 2
	}
	...                            // em cada saída:
}                                  if deferBits&2 != 0 { b() }
                                   if deferBits&1 != 0 { a() }
```

O custo caiu para praticamente o de uma chamada direta (~1–2 ns). As condições: até 8 defers na função, e nenhum `defer` dentro de laço. Defers em laços continuam usando o mecanismo antigo (registros encadeados na goroutine), por isso são um pouco mais caros, além do problema de acúmulo que vimos.

### 8.2 O desenrolar da stack num panic

```
main() → processar() → validar() → panic("x")

1. runtime.gopanic cria um registro de panic na goroutine
2. Executa os defers de validar()       (do mais recente ao mais antigo)
3. Nenhum chamou recover? Executa os defers de processar()
4. ...depois os de main()
5. Ninguém recuperou: imprime "panic: x", o stack trace de todas
   as goroutines (configurável com GOTRACEBACK) e sai com código 2

Se algum defer chamar recover():
   → o panic é marcado como recuperado
   → a função que contém ESSE defer termina normalmente
     (devolvendo os valores atuais dos resultados nomeados)
   → a execução continua no chamador dela, como se fosse um return
```

Para gerar o stack trace e encontrar os defers de cada frame, o runtime usa as tabelas que o compilador grava no binário (a `.gopclntab` que vimos em [[Como Go Funciona]]).

> 💡 `GOTRACEBACK=all` mostra todas as goroutines no crash; `GOTRACEBACK=crash` ainda gera um *core dump* (útil com depuradores como Delve).

### 8.3 Panics que vêm do hardware

Alguns panics começam como **exceções de hardware**:

- **Ponteiro nil:** a MMU gera falha de página no endereço 0 → kernel envia `SIGSEGV` → o tratador de sinais do runtime converte em panic (vimos em [[Ponteiros]])
- **Divisão por zero:** em amd64, o compilador insere uma checagem explícita e chama `runtime.panicdivide` (vimos em [[Operações Aritméticas]])

Já um `fatal error` (como `concurrent map writes`, `stack overflow` ou deadlock de todas as goroutines) **não** é um panic: o runtime aborta na hora, sem rodar defers e sem possibilidade de `recover`.

### 8.4 Paralelo com o sistema operacional

| Go | SO / C |
| --- | --- |
| `defer` | `atexit()` em C, mas por **função**, não por processo |
| `panic` desenrolando a stack | Exceções de C++; em C, `setjmp`/`longjmp` |
| `recover` num defer | Tratador de sinal que "salva" o processo e retoma a execução |
| Panic não recuperado | Processo morto por um sinal (`SIGSEGV`, `SIGABRT`) |
| Código de saída 2 | Status recebido pelo pai via `waitpid` (ver [[Término de Processos]]) |

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `defer` em laço | Recursos acumulados até a função terminar | Extraia o corpo para uma função |
| Esperar que o argumento do defer seja avaliado no fim | Valor "antigo" | Use closure se quer o valor final |
| Ignorar erro de `Close()` em escrita | Perda de dados silenciosa | Capture o erro com resultado nomeado |
| `recover()` fora de função adiada (ou indireto) | Não recupera nada | Chame direto dentro do `defer func() { ... }()` |
| Goroutine sem `recover` | Um panic derruba o processo | `recover` no topo de goroutines de longa duração |
| `os.Exit`/`log.Fatal` com defers pendentes | Limpeza não acontece | Padrão `run() error` |
| Usar `panic` para erros esperados | Programa cai por motivos normais | Devolva `error` |
| Recuperar e continuar com estado corrompido | Bugs estranhos depois | Recupere na fronteira e descarte o trabalho |
| `defer mu.Unlock()` em função longa | Lock segurado tempo demais | Seção crítica numa função pequena |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `defer` | Agenda uma chamada para o fim da função; roda em `return` e em panic |
| Ordem | LIFO |
| Argumentos | Avaliados na hora do `defer`; closures veem o valor final |
| Resultados nomeados | Podem ser alterados num defer (anotar erro, capturar erro de `Close`) |
| Em laços | Defers se acumulam: extraia uma função |
| `panic` | Interrompe e desenrola a stack rodando defers; sem recover, mata o processo (código 2) |
| Quando usar panic | Bugs, invariantes, `MustX` na inicialização; nunca para falhas esperadas |
| `recover` | Só funciona chamado diretamente num defer; converte panic em erro |
| Goroutines | Panic não recuperado em qualquer uma derruba tudo |
| Pulam defers | `os.Exit`, `log.Fatal`, crash por outra goroutine, `fatal error` |
| Implementação | Defers open-coded (1.14+); desenrolar via tabelas do binário |

---

## 🔗 Conexões

**Dentro de Go**
- Resultados nomeados e erros → [[Múltiplos Retornos e Erros]]
- Closures capturadas por defers → [[Closures e Funções Anônimas]]
- Erros vs panics no desenho de APIs → [[O Padrão error em Go]]
- `defer mu.Unlock()` → [[sync.WaitGroup e sync.Mutex]]
- `defer cancel()` e sinais → [[context.Context]]
- Middleware de recuperação → [[HTTP (net-http)]]
- `defer f.Close()` → [[Leitura e Escrita de Arquivos]]
- Panics em goroutines → [[Goroutines]]

**Sistemas Operacionais**
- [[Término de Processos]] — códigos de saída, término normal vs término por erro fatal
- [[Syscalls Diversas]] — sinais (`SIGSEGV`, `SIGTERM`, `kill`) e seus tratadores
- [[Arquivos]] e [[Syscalls para Gerenciamento de Arquivos]] — descritores de arquivo são recursos limitados por processo; `close` libera e pode reportar erro de escrita
- [[Pilha (Stack)]] — a lista de defers é uma pilha LIFO

**Docker**
- [[Processo raiz (PID 1)]] — o container recebe `SIGTERM` ao parar; um programa Go precisa tratar o sinal para rodar sua limpeza (defers não rodam se o processo for morto à força)

---

⬅️ **Anterior:** [[Closures e Funções Anônimas]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Métodos e Receptores]] ➡️
