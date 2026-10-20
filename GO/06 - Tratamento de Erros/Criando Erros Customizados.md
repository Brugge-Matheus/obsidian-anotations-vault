---
tags:
  - go
  - go/erros
aliases:
  - Erros customizados
  - Tipos de erro
  - Custom errors
ordem: 28
modulo: "06 - Tratamento de Erros"
---
# Criando Erros Customizados

> **Nota 28 da trilha de Go.** `errors.New` e `fmt.Errorf` cobrem a maioria dos casos. Mas às vezes o chamador precisa de **mais do que uma mensagem**: o código HTTP, o campo inválido, a operação que falhou, se vale a pena tentar de novo. Para isso, criamos **tipos de erro próprios**: qualquer tipo com o método `Error() string`. Nesta nota veremos como projetar esses tipos (campos, receptor valor ou ponteiro), como fazê-los participar de cadeias de erros (`Unwrap`), como personalizar a comparação (`Is`), erros que agregam vários erros, erros que expõem **comportamento**, como capturar stack traces e como a biblioteca padrão faz.
>
> **Pré-requisitos:** [[O Padrão error em Go]], [[Métodos e Receptores]].

---

## 🧭 De onde viemos

Em [[O Padrão error em Go]] vimos as três formas de inspecionar erros: **sentinela** (`errors.Is`), **tipo** (`errors.As`) e **comportamento** (interfaces como `Timeout()`). Também vimos que `errors.New` devolve um `*errorString`. Agora vamos criar nossos próprios tipos, como a biblioteca padrão faz com `*fs.PathError`, `*strconv.NumError`, `*json.SyntaxError` e `*time.ParseError`.

---

## 🧱 1. O tipo de erro mais simples

Qualquer tipo com o método `Error() string` satisfaz a interface `error` ([[Interfaces]]):

```go
type ErroValidacao struct {
	Campo  string
	Motivo string
}

func (e ErroValidacao) Error() string {
	return e.Campo + ": " + e.Motivo
}

func validarIdade(idade int) error {          // ✅ o retorno é do tipo error (a interface)
	if idade < 0 {
		return ErroValidacao{Campo: "idade", Motivo: "não pode ser negativa"}
	}
	return nil                                 // ✅ nil literal (ver a armadilha em Interfaces)
}
```

Quem chama pode tratar como erro comum (`err != nil`, `err.Error()`) ou extrair os dados:

```go
var ve ErroValidacao
if errors.As(err, &ve) {
	fmt.Println("campo problemático:", ve.Campo)
}
```

---

## 🏗️ 2. Projetando os campos

Um bom tipo de erro carrega o que o **chamador** precisa para **decidir** e o que **quem depura** precisa para **entender**. O modelo clássico é o `fs.PathError`:

```go
// biblioteca padrão (io/fs)
type PathError struct {
	Op   string   // a operação: "open", "stat", "remove"
	Path string   // o caminho envolvido
	Err  error    // a causa (normalmente um syscall.Errno)
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
func (e *PathError) Unwrap() error { return e.Err }
```

Saída real ao abrir um arquivo inexistente: `open /nao/existe: no such file or directory`, com `Op = "open"`, `Path = "/nao/existe"`, `Err = syscall.ENOENT`.

Um erro de HTTP no mesmo estilo (saídas reais):

```go
type ErroHTTP struct {
	Status int      // para o handler decidir a resposta
	Msg    string
	Causa  error    // o erro original, para depuração e para errors.Is/As
}

func (e *ErroHTTP) Error() string {
	if e.Causa != nil {
		return fmt.Sprintf("%d %s: %v", e.Status, e.Msg, e.Causa)
	}
	return fmt.Sprintf("%d %s", e.Status, e.Msg)
}

func (e *ErroHTTP) Unwrap() error { return e.Causa }

err := handler("x")
fmt.Println(err)
// 404 falha no handler: buscar usuário "x": não encontrado

var he *ErroHTTP
fmt.Println(errors.As(err, &he), he.Status)   // true 404
fmt.Println(errors.Is(err, ErrNaoEncontrado)) // true  ← graças ao Unwrap
```

Diretrizes:

| Campo | Para quê |
| --- | --- |
| Operação / contexto (`Op`) | Saber o que estava sendo feito |
| Dado envolvido (`Path`, `ID`, `Campo`) | Saber com o quê |
| Código (`Status`, `Code`) | O chamador decide sem olhar a mensagem |
| Causa (`Err`) + `Unwrap()` | Preservar a cadeia para `errors.Is`/`As` |

> ⚠️ Não coloque **dados sensíveis** (senhas, tokens, dados pessoais) em erros: eles acabam em logs.

---

## 👉 3. Receptor valor ou ponteiro?

Os dois funcionam, mas mudam **como** o erro é comparado e extraído:

| | Receptor ponteiro `func (e *E) Error()` | Receptor valor `func (e E) Error()` |
| --- | --- | --- |
| Devolver | `return &E{...}` | `return E{...}` |
| Extrair | `var e *E; errors.As(err, &e)` | `var e E; errors.As(err, &e)` |
| `==` / `errors.Is` sem método `Is` | Compara **ponteiros** (identidade) | Compara **campos** (se forem comparáveis) |
| Uso típico | A maioria dos erros com dados (`*PathError`, `*NumError`) | Erros pequenos e comparáveis |

> ✅ **Recomendação:** use receptor **ponteiro** como padrão (é o que a biblioteca padrão faz para erros com campos) e seja consistente: se `Error()` tem receptor ponteiro, devolva sempre `&E{}`.

> ⚠️ **Go 1.27:** o `go vet` passou a avisar quando você embrulha com `%w` um **ponteiro** `*E` em que `E` implementa `error` com receptor **valor**. Nesse caso, quem procura por `E` com `errors.As` não encontra (o que está na cadeia é um `*E`), e comparações com `errors.Is` falham. A correção é embrulhar o valor, não o ponteiro.

---

## 🔗 4. Participando da cadeia: `Unwrap`

Se o seu erro **envolve** outro, implemente `Unwrap() error`. Isso permite que `errors.Is` e `errors.As` "atravessem" o seu erro e encontrem a causa:

```
err (*ErroHTTP)
  └── Unwrap() → *fmt.wrapError ("buscar usuário \"x\": ...")
                   └── Unwrap() → ErrNaoEncontrado
```

Veremos a travessia da cadeia em detalhes em [[Wrapping e Unwrapping de Erros]].

> 💡 Nem todo erro deve expor a causa. Se a causa é um detalhe de implementação que você não quer que vire parte da API (por exemplo, um erro de um driver de banco específico), **não** implemente `Unwrap`, e use `%v` em vez de `%w` ao formatar a mensagem.

---

## 🟰 5. Personalizando a comparação: o método `Is`

Por padrão, `errors.Is(err, alvo)` compara com `==`. Um tipo pode definir sua própria noção de igualdade com um método `Is(error) bool`. Exemplo: um erro com código, em que o alvo "vazio" casa com qualquer código (saídas reais):

```go
type ErroComCodigo struct{ Codigo string }

func (e *ErroComCodigo) Error() string { return "código " + e.Codigo }

func (e *ErroComCodigo) Is(alvo error) bool {
	t, ok := alvo.(*ErroComCodigo)
	return ok && (t.Codigo == "" || t.Codigo == e.Codigo)
}

c := fmt.Errorf("x: %w", &ErroComCodigo{"E42"})
fmt.Println(errors.Is(c, &ErroComCodigo{"E42"}))   // true:  mesmo código
fmt.Println(errors.Is(c, &ErroComCodigo{}))        // true:  "qualquer ErroComCodigo"
fmt.Println(errors.Is(c, &ErroComCodigo{"E1"}))    // false: outro código
```

A biblioteca padrão usa isso: `syscall.Errno` implementa `Is`, de modo que `syscall.ENOENT` "é" `fs.ErrNotExist` e `syscall.EACCES` "é" `fs.ErrPermission`. É por isso que `errors.Is(err, os.ErrNotExist)` funciona para erros vindos do kernel.

> 💡 Existe também o método opcional `As(any) bool`, que permite a um erro "se apresentar" como outro tipo para `errors.As`. É raro precisar.

---

## 📦 6. Vários erros de uma vez

Validações costumam encontrar **vários** problemas. Desde o **Go 1.20**, a biblioteca padrão tem suporte a erros múltiplos.

### 6.1 `errors.Join`

Saída real:

```go
func validar(nome, email string) error {
	var errs []error
	if nome == "" {
		errs = append(errs, ErroValidacao{"nome", "obrigatório"})
	}
	if !strings.Contains(email, "@") {
		errs = append(errs, ErroValidacao{"email", "inválido"})
	}
	return errors.Join(errs...)        // nil se a lista estiver vazia
}

err := validar("", "x")
fmt.Println(err)
// nome: obrigatório
// email: inválido           ← mensagens separadas por quebra de linha

var ve ErroValidacao
fmt.Println(errors.As(err, &ve), ve.Campo)   // true nome   ← encontra o primeiro
fmt.Println(validar("ana", "a@b") == nil)    // true
```

### 6.2 O seu próprio tipo de erro múltiplo: `Unwrap() []error`

Um tipo pode expor vários erros internos implementando `Unwrap() []error` (em vez de `Unwrap() error`). `errors.Is` e `errors.As` percorrem todos:

```go
type ErrosValidacao struct {
	Erros []ErroValidacao
}

func (e *ErrosValidacao) Error() string {
	partes := make([]string, len(e.Erros))
	for i, ve := range e.Erros {
		partes[i] = ve.Error()
	}
	return "validação falhou: " + strings.Join(partes, "; ")
}

func (e *ErrosValidacao) Unwrap() []error {
	out := make([]error, len(e.Erros))
	for i, ve := range e.Erros {
		out[i] = ve
	}
	return out
}
```

Isso é útil para APIs que precisam devolver **todos** os campos inválidos de uma vez (o cliente mostra os erros ao lado de cada campo do formulário).

---

## 🎭 7. Erros que expõem comportamento

Em vez de obrigar o chamador a conhecer seu tipo, exponha **métodos** que respondem perguntas úteis (saída real):

```go
type ErroTemporario struct{ Op string }

func (e *ErroTemporario) Error() string   { return e.Op + ": temporário" }
func (e *ErroTemporario) Temporary() bool { return true }

tmp := fmt.Errorf("enviar: %w", &ErroTemporario{"dial"})

var t interface{ Temporary() bool }        // o chamador só conhece o COMPORTAMENTO
fmt.Println(errors.As(tmp, &t) && t.Temporary())   // true
```

O exemplo mais importante da biblioteca padrão é `net.Error`:

```go
type Error interface {
	error
	Timeout() bool     // foi timeout?
}
```

Um cliente HTTP pode decidir tentar de novo em caso de timeout sem saber se o erro veio do DNS, do TCP ou do TLS.

---

## 🧭 8. Stack traces

Erros de Go **não** carregam stack trace automaticamente (diferente de exceções). Isso é intencional: o contexto acrescentado com `fmt.Errorf("...: %w")` costuma ser mais útil que uma pilha de 40 frames. Quando realmente precisar, dá para capturar a pilha no momento da criação:

```go
type ErroComPilha struct {
	Err   error
	Pilha []uintptr
}

func ComPilha(err error) error {
	pcs := make([]uintptr, 32)
	n := runtime.Callers(2, pcs)          // endereços das funções na pilha (pula esta e Callers)
	return &ErroComPilha{Err: err, Pilha: pcs[:n]}
}

func (e *ErroComPilha) Error() string { return e.Err.Error() }
func (e *ErroComPilha) Unwrap() error { return e.Err }

func (e *ErroComPilha) FormatarPilha() string {
	var b strings.Builder
	frames := runtime.CallersFrames(e.Pilha)
	for {
		f, mais := frames.Next()
		fmt.Fprintf(&b, "%s\n\t%s:%d\n", f.Function, f.File, f.Line)
		if !mais {
			break
		}
	}
	return b.String()
}
```

> 💡 `runtime.Callers` lê a pilha da goroutine e `runtime.CallersFrames` traduz endereços em nome de função, arquivo e linha usando a tabela `.gopclntab` do binário ([[Como Go Funciona]]). Capturar a pilha custa alguns microssegundos: faça isso só em erros raros, não em caminhos quentes.

---

## 📚 9. Inspiração: tipos de erro da biblioteca padrão

Saídas reais de alguns deles:

| Tipo | Campos principais | Exemplo de mensagem |
| --- | --- | --- |
| `*fs.PathError` | `Op`, `Path`, `Err` | `open /nao/existe: no such file or directory` |
| `*strconv.NumError` | `Func`, `Num`, `Err` | `strconv.Atoi: parsing "12a": invalid syntax` |
| `*time.ParseError` | `Layout`, `Value`, `Message` | `parsing time "2024-13-01": month out of range` |
| `*json.SyntaxError` | `Offset` | posição do erro no JSON |
| `*json.UnmarshalTypeError` | `Value`, `Type`, `Field` | valor JSON incompatível com o campo |
| `*net.OpError` | `Op`, `Net`, `Addr`, `Err` | `dial tcp 127.0.0.1:9: connect: connection refused` |
| `*url.Error` | `Op`, `URL`, `Err` | `Get "http://...": ...` |
| `syscall.Errno` | (é um número) | `no such file or directory` |
| `*exec.ExitError` | `ProcessState` | o código de saída de um processo filho |

Repare no padrão: **operação + objeto + causa**, quase sempre com `Unwrap`.

---

## 🧭 10. Quando criar um tipo de erro?

```
O chamador precisa reagir de forma diferente a esta falha?
│
├── Não → errors.New / fmt.Errorf com contexto (erro opaco)
│
└── Sim → Ele precisa só saber QUE aconteceu?
          │
          ├── Sim → sentinela: var ErrX = errors.New("...")
          │
          └── Não, precisa de DADOS (código, campo, posição) ou de
              COMPORTAMENTO (temporário? timeout?)
              │
              └── tipo de erro (struct + Error() + Unwrap() se houver causa)
```

> ✅ Comece com erros opacos. Promova para sentinela ou tipo **quando um chamador real** precisar distinguir.

---

## ⚠️ 11. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Função com retorno `*MeuErro` em vez de `error` | Ponteiro nil tipado vira `err != nil` | Retorne `error` e `nil` literal |
| Misturar `E{}` e `&E{}` | `errors.As` não encontra | Escolha uma forma (geralmente ponteiro) |
| Esquecer `Unwrap` num erro que envolve outro | `errors.Is` não acha a causa | Implemente `Unwrap() error` |
| Expor causa que é detalhe interno | Chamadores dependem do seu driver de banco | Não implemente `Unwrap`; use `%v` |
| `Error()` com receptor ponteiro chamado em nil | panic se acessar campos | Garanta que o erro não seja nil |
| Dados sensíveis no erro | Vazam para logs | Nunca inclua segredos |
| Stack trace em todo erro | Custo e logs enormes | Só em erros raros e inesperados |
| Tipo de erro para tudo | API inchada | Comece opaco |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Tipo de erro | Qualquer tipo com `Error() string` |
| Campos | Operação, objeto, código, causa |
| Receptor | Ponteiro como padrão; seja consistente |
| `Unwrap() error` | Deixa `errors.Is`/`As` alcançar a causa |
| `Is(error) bool` | Igualdade personalizada (ex.: `Errno` ≈ `ErrNotExist`) |
| Múltiplos | `errors.Join` ou `Unwrap() []error` (Go 1.20+) |
| Comportamento | Métodos como `Timeout()`, `Temporary()` (`net.Error`) |
| Stack trace | Opcional, com `runtime.Callers` |
| Go 1.27 | `go vet` avisa sobre `%w` com ponteiro para erro de receptor valor |

---

## 🔗 Conexões

**Dentro de Go**
- Filosofia e estratégias → [[O Padrão error em Go]]
- Cadeias de erros, `errors.Is`/`As`/`Join` → [[Wrapping e Unwrapping de Erros]]
- Métodos e receptores → [[Métodos e Receptores]]
- A armadilha do nil em interfaces → [[Interfaces]]
- Extração de tipos de erro → [[Type Assertion e Type Switch]]
- `net.OpError`, `url.Error` na prática → [[Sockets e o Pacote net]], [[HTTP (net-http)]]
- `.gopclntab` e stack traces → [[Como Go Funciona]]

**Sistemas Operacionais**
- [[System Calls]] — `errno` e os códigos `ENOENT`, `EACCES`, `ECONNREFUSED` que `syscall.Errno` representa
- [[Syscalls para Gerenciamento de Processos]] — `*exec.ExitError` carrega o status de término do processo filho (obtido via `waitpid`)

**Redes**
- [[Camada de Transporte - 4]] — `connection refused` (RST do TCP) e timeouts, os erros mais comuns de `net.OpError`

---

⬅️ **Anterior:** [[O Padrão error em Go]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Wrapping e Unwrapping de Erros]] ➡️
