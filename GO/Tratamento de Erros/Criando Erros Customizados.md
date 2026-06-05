---
tags:
  - go
  - go/erros
---
# Criando Erros Customizados

> Erros customizados carregam informação estruturada além de uma string. São essenciais em aplicações reais onde você precisa: distinguir tipos de erro programaticamente, incluir contexto para logging, ou retornar detalhes específicos para o caller.
> 

---

## 1. A Interface `error` Revisitada

```go
// Qualquer tipo com Error() string é um error
type error interface {
	Error() string
}

// Portanto, qualquer struct com esse método é um error
type MeuErro struct {
	Codigo int
	Msg    string
}

func (e *MeuErro) Error() string {
	return fmt.Sprintf("[%d] %s", e.Codigo, e.Msg)
}

// Usar como error — retorno de função
func operacao() error {
	return &MeuErro{Codigo: 404, Msg: "recurso não encontrado"}
}

err := operacao()
fmt.Println(err)   // "[404] recurso não encontrado"
```

> ⚠️ Sempre use **ponteiro** como receptor em erros customizados: `*MeuErro` implementa `error`, `MeuErro` não. Isso garante que `errors.As` funcione corretamente e que dois erros distintos criados com os mesmos valores não sejam iguais por `errors.Is`.
> 

---

## 2. Erro com Campos Estruturados

```go
// Erro de validação com campo específico
type ErrValidacao struct {
	Campo    string
	Valor    any
	Mensagem string
}

func (e *ErrValidacao) Error() string {
	return fmt.Sprintf("validação: campo %q (valor=%v): %s",
		e.Campo, e.Valor, e.Mensagem)
}

// Construtor — evita criar structs diretamente em todo lugar
func errCampoObrigatorio(campo string) error {
	return &ErrValidacao{Campo: campo, Mensagem: "obrigatório"}
}

func errValorInvalido(campo string, valor any, msg string) error {
	return &ErrValidacao{Campo: campo, Valor: valor, Mensagem: msg}
}

// Uso
func validarUsuario(u *Usuario) error {
	if u.Nome == "" {
		return errCampoObrigatorio("nome")
	}
	if len(u.Nome) < 2 {
		return errValorInvalido("nome", u.Nome, "mínimo 2 caracteres")
	}
	if !strings.Contains(u.Email, "@") {
		return errValorInvalido("email", u.Email, "formato inválido")
	}
	return nil
}

// Extrair com errors.As
err := validarUsuario(&Usuario{Nome: "A", Email: "semArroba"})
var valErr *ErrValidacao
if errors.As(err, &valErr) {
	fmt.Printf("Campo: %s\nMotivo: %s\n", valErr.Campo, valErr.Mensagem)
}
```

---

## 3. Erro com Wrapping — Campo `Wrapped`

Para erros que precisam carregar uma causa raiz:

```go
type ErrDB struct {
	Operacao string
	Tabela   string
	Wrapped  error
}

func (e *ErrDB) Error() string {
	if e.Wrapped != nil {
		return fmt.Sprintf("db %s em %q: %v", e.Operacao, e.Tabela, e.Wrapped)
	}
	return fmt.Sprintf("db %s em %q", e.Operacao, e.Tabela)
}

// Implementar Unwrap() para que errors.Is/As percorram a chain
func (e *ErrDB) Unwrap() error {
	return e.Wrapped
}

// Uso
func (r *Repositorio) Buscar(id int) (*Usuario, error) {
	row := r.db.QueryRow("SELECT * FROM usuarios WHERE id = ?", id)
	var u Usuario
	if err := row.Scan(&u.ID, &u.Nome); err != nil {
		return nil, &ErrDB{
			Operacao: "SELECT",
			Tabela:   "usuarios",
			Wrapped:  err,
		}
	}
	return &u, nil
}

// errors.Is percorre até o erro interno
err := repo.Buscar(99)
errors.Is(err, sql.ErrNoRows)   // true — percorreu ErrDB.Unwrap()

var dbErr *ErrDB
errors.As(err, &dbErr)
fmt.Println(dbErr.Tabela)   // "usuarios"
```

---

## 4. Erro HTTP com Código de Status

```go
type ErrHTTP struct {
	Status  int
	Msg     string
	Wrapped error
}

func (e *ErrHTTP) Error() string {
	if e.Wrapped != nil {
		return fmt.Sprintf("HTTP %d: %s: %v", e.Status, e.Msg, e.Wrapped)
	}
	return fmt.Sprintf("HTTP %d: %s", e.Status, e.Msg)
}

func (e *ErrHTTP) Unwrap() error { return e.Wrapped }

// Construtores semânticos
func ErrBadRequest(msg string, cause error) error {
	return &ErrHTTP{Status: 400, Msg: msg, Wrapped: cause}
}

func ErrNotFound(recurso string) error {
	return &ErrHTTP{Status: 404, Msg: recurso + " não encontrado"}
}

func ErrUnauthorized() error {
	return &ErrHTTP{Status: 401, Msg: "não autenticado"}
}

func ErrInternal(cause error) error {
	return &ErrHTTP{Status: 500, Msg: "erro interno", Wrapped: cause}
}

// Handler que converte errors → responses HTTP
func escreverErro(w http.ResponseWriter, err error) {
	var httpErr *ErrHTTP
	if errors.As(err, &httpErr) {
		http.Error(w, httpErr.Msg, httpErr.Status)
		return
	}
	http.Error(w, "erro interno", http.StatusInternalServerError)
}
```

---

## 5. Erro de Validação com Múltiplos Campos

```go
type CampoErro struct {
	Campo    string `json:"campo"`
	Mensagem string `json:"mensagem"`
}

type ErrValidacaoMultipla struct {
	Campos []CampoErro
}

func (e *ErrValidacaoMultipla) Error() string {
	msgs := make([]string, len(e.Campos))
	for i, c := range e.Campos {
		msgs[i] = fmt.Sprintf("%s: %s", c.Campo, c.Mensagem)
	}
	return "validação falhou: " + strings.Join(msgs, "; ")
}

// Builder — acumula erros durante validação
type Validador struct {
	erros []CampoErro
}

func (v *Validador) Campo(campo, mensagem string) {
	v.erros = append(v.erros, CampoErro{campo, mensagem})
}

func (v *Validador) Erro() error {
	if len(v.erros) == 0 {
		return nil
	}
	return &ErrValidacaoMultipla{Campos: v.erros}
}

// Uso
func validarCadastro(req CadastroRequest) error {
	v := &Validador{}

	if req.Nome == "" {
		v.Campo("nome", "obrigatório")
	} else if len(req.Nome) < 2 {
		v.Campo("nome", "mínimo 2 caracteres")
	}

	if req.Email == "" {
		v.Campo("email", "obrigatório")
	} else if !strings.Contains(req.Email, "@") {
		v.Campo("email", "formato inválido")
	}

	if req.Senha == "" {
		v.Campo("senha", "obrigatória")
	} else if len(req.Senha) < 8 {
		v.Campo("senha", "mínimo 8 caracteres")
	}

	return v.Erro()
}

// Handler que serializa o erro como JSON
func handleCadastro(w http.ResponseWriter, r *http.Request) {
	var req CadastroRequest
	json.NewDecoder(r.Body).Decode(&req)

	if err := validarCadastro(req); err != nil {
		var valErr *ErrValidacaoMultipla
		if errors.As(err, &valErr) {
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusUnprocessableEntity)
			json.NewEncoder(w).Encode(map[string]any{
				"erro":   "validação falhou",
				"campos": valErr.Campos,
			})
			return
		}
		http.Error(w, "erro interno", 500)
	}
}
```

---

## 6. Verificar Interface `Is` Customizada

Para erros que precisam de comparação semântica customizada:

```go
type ErrCodigo struct {
	Codigo int
}

func (e *ErrCodigo) Error() string {
	return fmt.Sprintf("código de erro: %d", e.Codigo)
}

// Implementar Is() para comparação personalizada
func (e *ErrCodigo) Is(target error) bool {
	t, ok := target.(*ErrCodigo)
	if !ok {
		return false
	}
	// Dois ErrCodigo são "iguais" se têm o mesmo código
	return e.Codigo == t.Codigo
}

err := fmt.Errorf("operação: %w", &ErrCodigo{1045})

// errors.Is chama o método Is() do erro
errors.Is(err, &ErrCodigo{1045})   // true
errors.Is(err, &ErrCodigo{9999})   // false
```

---

## 7. Unwrap com Múltiplos Erros (Go 1.20+)

```go
// Para erros que embrulham múltiplos erros
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

// Retornar []error permite que errors.Is/As percorram todos
func (e *ErrMultiplo) Unwrap() []error {
	return e.Erros
}

// errors.Join já faz isso automaticamente:
combined := errors.Join(err1, err2, err3)
// errors.Is(combined, err1) → true
// errors.Is(combined, err2) → true
```

---

## 8. Resumo — Qual Usar em Cada Situação

| Situação | Solução |
| --- | --- |
| Erro simples, mensagem estática | `errors.New("mensagem")` |
| Erro com contexto dinâmico | `fmt.Errorf("contexto: %w", err)` |
| Erro com campos estruturados | Struct customizada com `Error() string` |
| Wrapping com struct customizada | Adicionar campo `Wrapped error`  • `Unwrap() error` |
| Múltiplos erros de validação | `errors.Join` ou struct com `Unwrap() []error` |
| Comparação semântica customizada | Implementar `Is(target error) bool` |
| Extração de tipo específico | `errors.As(err, &target)` |
| Verificar erro na chain | `errors.Is(err, sentinel)` |

---

## 9. Como Funciona Internamente

### Dispatch via itab — Como o Método `Error()` é Chamado

```
Quando você retorna &MeuErro{...} como error:

  variável error:
  ┌───────────────┬───────────────┐
  │  *itab        │  *data        │
  │  {*MeuErro,   │  → &MeuErro{ │
  │   Error:0x..} │    Codigo:404,│
  └───────────────┴───────────────┘
                        Codigo: 404
                        Msg: "não encontrado"

  Chamada err.Error():
    1. Lê itab → encontra endereço do método Error() de *MeuErro
    2. Chama Error() com data como receiver
    3. Retorna a string formatada

  Isso é polimorfismo em runtime — o compilador não sabe em
  tempo de compilação qual tipo concreto está atrás do error.
```

### Por Que Usar Ponteiro como Receptor

```go
// MeuErro implementa error via PONTEIRO (*MeuErro)
func (e *MeuErro) Error() string { ... }

// Consequência: o valor zero de error é nil (ponteiro nulo)
// Se você usasse valor (MeuErro, não *MeuErro):
var err error = MeuErro{}   // NÃO é nil mesmo que "vazio"!
// Isso causaria o famoso bug do "interface nil != nil":

func retornarErro() error {
    var e *MeuErro = nil
    return e   // BUG: interface não-nil com data pointer nil!
}
// errors.Is/As precisam do ponteiro para funcionar corretamente
```

### Erro com Contexto — Analogia com Structs de Syscall

```
POSIX (C) — errno é apenas um inteiro:
  read() falha → errno = ENOENT (2)
  Você sabe QUE deu erro mas não TEM contexto (qual path? qual operação?)

os.PathError em Go — struct com contexto completo:
  type PathError struct {
      Op   string  // "open", "read", "stat"
      Path string  // "/etc/hosts"
      Err  error   // o errno subjacente: syscall.ENOENT
  }

  Heap depois de os.Open("/nao/existe"):
  ┌──────────────────────────────┐
  │  *PathError                  │
  │  Op:   "open"                │
  │  Path: "/nao/existe"         │
  │  Err:  → *os.oserror{ENOENT} │
  └──────────────────────────────┘

  Go transformou um inteiro errno numa struct rica com contexto.
  errors.As(err, &pathErr) → acessa Op e Path diretamente.
```

### A Chain de Erros como Lista Encadeada no Heap

```
fmt.Errorf("camada2: %w", fmt.Errorf("camada1: %w", erroRaiz))

Heap:
  ┌─────────────────┐    Unwrap()    ┌─────────────────┐    Unwrap()    ┌─────────┐
  │ wrapError       │ ─────────────► │ wrapError       │ ─────────────► │ *err    │
  │ msg:"camada2"   │                │ msg:"camada1"   │                │ "raiz"  │
  │ err: *          │                │ err: *          │                └─────────┘
  └─────────────────┘                └─────────────────┘

  Cada nó aloca memória no heap.
  fmt.Errorf com %w cria um wrapError interno com ponteiro para o próximo.
  errors.Is() percorre essa lista: nó1 → nó2 → nó3 → nil
```

---

## 10. Conexão com Sistemas Operacionais

**Erros customizados usam dispatch via itab — alocação e indireção de ponteiros → [[Gerenciamento de Memória]]**
Qualquer struct que implemente `Error() string` e seja armazenada numa variável `error` vive no heap. O runtime usa a tabela itab para encontrar o método correto em tempo de execução — o mesmo mecanismo de polimorfismo que o kernel usa para dispatch de drivers (via tabelas de ponteiros de função, como `struct file_operations` no Linux).

**Erros com campos contextuais espelham como syscalls retornam erros estruturados → [[System Calls]]**
`errno` no POSIX é um inteiro simples — você sabe que ocorreu `ENOENT` mas não sabe qual path causou o erro. `os.PathError{Op, Path, Err}` resolve exatamente isso: carrega o contexto completo. Toda a stdlib do Go segue esse padrão de enriquecer os erros de syscall com estrutura.

**A chain de erros com %w é uma lista encadeada no heap → [[Gerenciamento de Memória]]**
Cada `fmt.Errorf("...: %w", err)` aloca um novo nó no heap contendo um ponteiro para o próximo erro. Percorrer a chain via `errors.Is()` é traversal de lista encadeada — pointer chasing que pode causar cache misses se a chain for muito longa.

**Go oferece erros estruturados onde o OS só oferece inteiros → [[System Calls]]**
No POSIX, `errno` é um inteiro. Go transforma esse inteiro num tipo rico: `syscall.Errno` implementa `error`, e pacotes como `os` e `net` embrulham `Errno` em structs com contexto. O programador Go opera num nível de abstração muito acima do `int` que o kernel retorna.