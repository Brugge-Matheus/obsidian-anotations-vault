---
tags:
  - go
  - go/erros
---
# Wrapping e Unwrapping de Erros

> Error wrapping preserva a causa raiz de um erro enquanto adiciona contexto. Go 1.13 introduziu o wrapping formal com `%w`, e Go 1.20 adicionou `errors.Join`. Juntos permitem construir chains de erros que são inspecionáveis em qualquer camada.
> 

---

## 1. O Problema que Wrapping Resolve

```go
// Sem wrapping — informação perdida
func buscarUsuario(id int) (*Usuario, error) {
	dados, err := db.Query("SELECT * FROM users WHERE id = ?", id)
	if err != nil {
		return nil, errors.New("falha na busca")   // ❌ causa raiz perdida
	}
	// ...
}

// Com wrapping — causa raiz preservada
func buscarUsuario(id int) (*Usuario, error) {
	dados, err := db.Query("SELECT * FROM users WHERE id = ?", id)
	if err != nil {
		return nil, fmt.Errorf("buscarUsuario(%d): %w", id, err)   // ✅
	}
	// ...
}
```

---

## 2. Wrapping com `fmt.Errorf` e `%w`

`%w` cria um erro que **embrulha** o original — o erro interno fica acessível via `errors.Unwrap`, `errors.Is` e `errors.As`:

```go
var ErrConexao = errors.New("falha de conexão")

// Wrapping simples
wrapped := fmt.Errorf("conectarDB: %w", ErrConexao)

// Wrapping múltiplo — chain
func conectar(host string) error {
	return fmt.Errorf("conectar(%q): %w", host, ErrConexao)
}

func iniciarApp() error {
	if err := conectar("db:5432"); err != nil {
		return fmt.Errorf("iniciarApp: %w", err)
	}
	return nil
}

err := iniciarApp()
fmt.Println(err)
// iniciarApp: conectar("db:5432"): falha de conexão

// A chain completa:
// iniciarApp: conectar("db:5432"): falha de conexão
//                   ↑ nível 1         ↑ nível 2        ↑ erro raiz
```

---

## 3. `errors.Unwrap` — Desembrulhar Uma Camada

```go
wrapped := fmt.Errorf("camada2: %w",
	fmt.Errorf("camada1: %w",
		errors.New("raiz"),
	),
)

fmt.Println(wrapped)                                // "camada2: camada1: raiz"
fmt.Println(errors.Unwrap(wrapped))                 // "camada1: raiz"
fmt.Println(errors.Unwrap(errors.Unwrap(wrapped)))  // "raiz"
fmt.Println(errors.Unwrap(errors.Unwrap(errors.Unwrap(wrapped))))  // nil
```

---

## 4. `errors.Is` — Verificar em Toda a Chain

`errors.Is` percorre recursivamente a chain de wrapping via `Unwrap()`:

```go
var (
	ErrDB          = errors.New("erro de banco")
	ErrNaoEncontrado = errors.New("não encontrado")
)

err := fmt.Errorf("buscar usuário: %w",
	fmt.Errorf("query falhou: %w", ErrDB),
)

// Verifica em TODA a chain
errors.Is(err, ErrDB)          // true — encontrou 2 camadas abaixo
errors.Is(err, ErrNaoEncontrado)  // false — não está na chain

// errors.Is também chama Is() customizado se definido
type ErrHTTP struct{ StatusCode int }
func (e *ErrHTTP) Error() string { return fmt.Sprintf("HTTP %d", e.StatusCode) }
func (e *ErrHTTP) Is(target error) bool {
	t, ok := target.(*ErrHTTP)
	return ok && t.StatusCode == e.StatusCode
}

err404 := &ErrHTTP{404}
wrapped := fmt.Errorf("handler: %w", err404)
errors.Is(wrapped, &ErrHTTP{404})   // true — usa método Is() customizado
errors.Is(wrapped, &ErrHTTP{500})   // false
```

---

## 5. `errors.As` — Extrair Tipo Específico da Chain

`errors.As` percorre a chain até encontrar um erro do tipo pedido:

```go
type ErrDB struct {
	Codigo   int
	Mensagem string
	Query    string
}

func (e *ErrDB) Error() string {
	return fmt.Sprintf("DB error %d (%s): %s", e.Codigo, e.Query, e.Mensagem)
}

// Múltiplas camadas de wrapping
err := fmt.Errorf("processarPedido: %w",
	fmt.Errorf("buscarEstoque: %w",
		&ErrDB{Codigo: 1045, Mensagem: "acesso negado", Query: "SELECT ..."},
	),
)

// errors.As percorre a chain e extrai o erro concreto
var dbErr *ErrDB
if errors.As(err, &dbErr) {
	fmt.Println("Código:", dbErr.Codigo)   // 1045
	fmt.Println("Query:", dbErr.Query)     // SELECT ...
	// Podemos agora verificar especificamente
	if dbErr.Codigo == 1045 {
		log.Fatal("sem permissão de acesso ao banco")
	}
}
```

---

## 6. `errors.Join` (Go 1.20+) — Múltiplos Erros

`errors.Join` combina múltiplos erros em um, preservando cada um para inspeção individual:

```go
err1 := errors.New("nome obrigatório")
err2 := errors.New("email inválido")
err3 := errors.New("idade negativa")

// Combinar — a mensagem exibe cada erro em uma linha
combined := errors.Join(err1, err2, err3)
fmt.Println(combined)
// nome obrigatório
// email inválido
// idade negativa

// Cada erro ainda é acessível via errors.Is
errors.Is(combined, err1)   // true
errors.Is(combined, err2)   // true
errors.Is(combined, err3)   // true

// Implementação interna: errors.Join retorna um erro que tem Unwrap() []error
// Isso permite que errors.Is e errors.As percorram a lista
```

### Validação com `errors.Join`

```go
func validarCadastro(nome, email string, idade int) error {
	var erros []error

	if nome == "" {
		erros = append(erros, errors.New("nome: obrigatório"))
	} else if len(nome) < 2 {
		erros = append(erros, fmt.Errorf("nome: mínimo 2 caracteres, recebido %d", len(nome)))
	}

	if !strings.Contains(email, "@") {
		erros = append(erros, errors.New("email: formato inválido"))
	}

	if idade < 0 || idade > 150 {
		erros = append(erros, fmt.Errorf("idade: valor inválido %d", idade))
	}

	return errors.Join(erros...)   // nil se erros estiver vazio
}

err := validarCadastro("A", "semArroba", -5)
if err != nil {
	fmt.Println(err)
	// nome: mínimo 2 caracteres, recebido 1
	// email: formato inválido
	// idade: valor inválido -5
}
```

---

## 7. `%w` vs `%v` — Quando Usar Cada Um

```go
// %w — wrapping: errors.Is/As percorrem a chain
return fmt.Errorf("operação: %w", err)

// %v — apenas mensagem: chain QUEBRADA, errors.Is/As não percorrem
return fmt.Errorf("operação: %v", err)

// Quando usar %v intencionalmente:
// - Quando você quer "apagar" o tipo de erro e criar uma fronteira de abstração
// - Ex: erros internos de banco não devem vazar para a camada de HTTP
func buscarUsuario(id int) (*Usuario, error) {
	u, err := repo.Find(id)
	if err != nil {
		// Usamos %v para não expor detalhes internos do banco para quem chama
		return nil, fmt.Errorf("usuário %d não disponível: %v", id, err)
	}
	return u, nil
}
```

---

## 8. Implementar `Unwrap` Manualmente

Para erros customizados que precisam de wrapping:

```go
type ErrOperacao struct {
	Op      string
	Wrapped error
}

func (e *ErrOperacao) Error() string {
	return fmt.Sprintf("%s: %v", e.Op, e.Wrapped)
}

// Implementar Unwrap() para que errors.Is/As percorram
func (e *ErrOperacao) Unwrap() error {
	return e.Wrapped
}

// Para Unwrap() []error (múltiplos erros embrulhados — Go 1.20+)
type ErrMultiplo struct {
	Erros []error
}

func (e *ErrMultiplo) Error() string {
	msgs := make([]string, len(e.Erros))
	for i, err := range e.Erros {
		msgs[i] = err.Error()
	}
	return strings.Join(msgs, "; ")
}

func (e *ErrMultiplo) Unwrap() []error {
	return e.Erros   // permite errors.Is/As percorrer todos
}
```

---

## 9. Stack Traces — Capturar Onde o Erro Ocorreu

Go não inclui stack traces em erros por padrão (por performance). Para debug, você pode capturar manualmente ou usar bibliotecas:

```go
import "runtime"

type ErrComStack struct {
	Mensagem string
	Stack    string
}

func (e *ErrComStack) Error() string {
	return e.Mensagem
}

func NovoErroComStack(msg string) error {
	buf := make([]byte, 4096)
	n := runtime.Stack(buf, false)
	return &ErrComStack{
		Mensagem: msg,
		Stack:    string(buf[:n]),
	}
}

// Para produção, prefira bibliotecas como:
// github.com/pkg/errors — errors.WithStack(err), errors.Cause(err)
// go.uber.org/zap — logging estruturado que captura stacks automaticamente
```

---

## 10. Resumo

| Função/Operação | Comportamento |
| --- | --- |
| `fmt.Errorf("...: %w", err)` | Cria erro que embrulha `err` |
| `fmt.Errorf("...: %v", err)` | Apenas mensagem — chain quebrada |
| `errors.Unwrap(err)` | Retorna o erro embrulhado imediato |
| `errors.Is(err, target)` | Percorre toda a chain com `==` ou `Is()` |
| `errors.As(err, &target)` | Percorre toda a chain e extrai tipo |
| `errors.Join(e1, e2, ...)` | Combina múltiplos erros (Go 1.20+) |
| `Unwrap() error` | Interface para chain simples |
| `Unwrap() []error` | Interface para múltiplos erros (Go 1.20+) |

```
Chain de wrapping:
processarPedido: %w
    ↓ Unwrap()
buscarEstoque: %w
    ↓ Unwrap()
&ErrDB{1045, "acesso negado"}

errors.Is(err, ErrAlvo) percorre: nível 1 → nível 2 → nível 3 → nil
errors.As(err, &dbErr)  percorre: nível 1 → nível 2 → encontrou! → preenche dbErr
```

---

## 11. Como Funciona Internamente

### A Chain de Erros — Lista Encadeada no Heap

```
Cada fmt.Errorf("...: %w", err) cria um nó no heap:

  type wrapError struct {   // tipo interno do pacote fmt
      msg string
      err error             // ponteiro para o próximo nó
  }
  func (e *wrapError) Unwrap() error { return e.err }

Memória após:
  err := fmt.Errorf("c: %w", fmt.Errorf("b: %w", errors.New("a")))

  HEAP:
  [wrapError "c: b: a"] ──Unwrap()──► [wrapError "b: a"] ──Unwrap()──► [errorString "a"] ──► nil
        nó 3                                 nó 2                            nó 1 (raiz)

  Cada fmt.Errorf com %w:  1 alocação de heap (wrapError)
  errors.New:              1 alocação de heap (errorString)
  Total aqui:              3 alocações + 3 ponteiros de interface
```

### `errors.Is` — Traversal de Lista Encadeada

```go
// Implementação simplificada de errors.Is:
func Is(err, target error) bool {
    for {
        // 1. Comparação direta (ponteiro ou método Is())
        if isComparable(err) && err == target {
            return true
        }
        // 2. Método Is() customizado
        if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
            return true
        }
        // 3. Avançar para o próximo nó
        err = Unwrap(err)        // pointer chase: err = err.err
        if err == nil {
            return false
        }
    }
}

// Complexidade: O(n) onde n = profundidade da chain
// Cada iteração: uma indireção de ponteiro (potencial cache miss)
// Chains longas (>5 níveis) têm custo perceptível em hot paths
```

### `errors.As` — Type Assertion em Cada Nó

```go
// errors.As percorre a chain fazendo type assertion em cada nó:
func As(err error, target any) bool {
    // target deve ser *T onde T implementa error, ou *interface
    for err != nil {
        // Tenta type assertion: err.(*ErrDB)?
        if reflectlite.TypeOf(err).AssignableTo(targetType) {
            targetVal.Elem().Set(reflectlite.ValueOf(err))
            return true   // encontrou! preenche *target
        }
        // Método As() customizado
        if x, ok := err.(interface{ As(any) bool }); ok && x.As(target) {
            return true
        }
        err = Unwrap(err)   // avança o ponteiro
    }
    return false
}

// A type assertion usa reflect internamente → mais caro que errors.Is
// Prefira errors.Is para verificar sentinels (O(n) com comparação barata)
// Use errors.As quando precisar dos CAMPOS do erro concreto
```

### Go vs POSIX — Causalidade de Erros

```
POSIX (C):                          Go:
errno é um único inteiro            error é uma chain de nós

read() → ENOENT (2)                 os.Open() → PathError{
                                        Op:   "open"
                                        Path: "/etc/hosts"
                                        Err:  → syscall.ENOENT
                                    }
                                    embrulhado em:
                                    fmt.Errorf("loadConfig: %w", ...)
                                        → "loadConfig: open /etc/hosts: no such file"

POSIX: informação achatada          Go: cadeia causal completa preservada
       um só valor de errno              cada camada adiciona contexto
       não há "causa raiz"               errors.Is percorre até a raiz
```

---

## 12. Conexão com Sistemas Operacionais

**A chain de erros é uma lista encadeada alocada no heap → [[Gerenciamento de Memória]]**
Cada `fmt.Errorf("...: %w", err)` aloca um `wrapError` no heap com um ponteiro para o nó anterior. A traversal com `errors.Is` e `errors.As` percorre esses ponteiros sequencialmente — pointer chasing clássico, com risco de cache miss se os nós estiverem espalhados na memória.

**`errors.Is` percorre lista encadeada; `errors.As` faz type assertion em cada nó → [[Gerenciamento de Memória]]**
`errors.Is` compara ponteiros (ou chama `Is()`) em cada nó: O(n) no tamanho da chain. `errors.As` usa `reflect` para verificar o tipo de cada nó — operação mais cara que envolve leitura da tabela de tipos em runtime. Ambos seguem o mesmo padrão de traversal de lista com `Unwrap()` como "next pointer".

**`errno` no POSIX é um único valor; Go preserva a cadeia causal completa → [[System Calls]]**
Quando `read(2)` falha com `ENOENT`, você perde o contexto de onde e por quê. Go preserva toda a cadeia: `loadConfig → os.Open("/etc/hosts") → syscall.ENOENT`. `errors.Is(err, syscall.ENOENT)` ainda funciona percorrendo até a raiz — rica semântica causal com compatibilidade com o valor base do POSIX.