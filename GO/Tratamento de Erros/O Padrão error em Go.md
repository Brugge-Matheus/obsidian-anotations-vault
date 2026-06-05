---
tags:
  - go
  - go/erros
---
# O Padrão error em Go

> Erros em Go são valores — não exceções. Isso é uma decisão de design fundamental que torna o fluxo de erros explícito e visível. O tipo `error` é uma interface simples, e qualquer tipo que implemente `Error() string` é um erro.
> 

---

## 1. Por Que Valores, Não Exceções

```
Linguagens com exceções (Java, PHP, Python, Ruby):
- Exceção pode ser lançada em qualquer lugar
- Propagação implícita pela call stack
- Fácil ignorar acidentalmente (uncaught exception → crash ou comportamento errado)
- Difícil saber quais funções podem falhar sem ler a documentação

Go com erros como valores:
- Erro é um valor de retorno explícito
- O compilador exige que você receba o retorno (variável declarada mas não usada = erro)
- O fluxo de erros é visível no código
- Fácil de rastrear onde os erros surgem e como são propagados
```

---

## 2. A Interface `error`

```go
// Definida na stdlib — a interface mais simples de Go
type error interface {
	Error() string
}

// Qualquer tipo com Error() string é um error
// Implementação mais simples: errorString (retornado por errors.New)
type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}

// errors.New retorna um ponteiro para errorString
// Usar ponteiro é crucial: dois erros criados com a mesma string são DIFERENTES
err1 := errors.New("algo falhou")
err2 := errors.New("algo falhou")
fmt.Println(err1 == err2)   // false! ponteiros diferentes
```

---

## 3. Criar Erros

```go
import (
	"errors"
	"fmt"
)

// errors.New — mensagem estática, simples
var ErrNaoEncontrado = errors.New("não encontrado")

// fmt.Errorf — mensagem formatada
err := fmt.Errorf("buscarUsuario(%d): usuário não existe", id)

// fmt.Errorf com %w — wrapping (Go 1.13+)
if err := db.Query(); err != nil {
	return fmt.Errorf("buscarUsuario(%d): %w", id, err)
}
```

### Regras de Estilo para Mensagens de Erro

```go
// ✅ Correto
errors.New("arquivo não encontrado")
fmt.Errorf("conexão recusada em %s", host)
fmt.Errorf("timeout após %v", dur)

// ❌ Errado — maiúscula
errors.New("Arquivo não encontrado")

// ❌ Errado — ponto final
errors.New("arquivo não encontrado.")

// ❌ Errado — prefixo "Error:" ou "erro:"
errors.New("Error: arquivo não encontrado")

// Motivo: erros são frequentemente concatenados
// "processar: buscarArquivo: arquivo não encontrado"
// A maiúscula e o ponto ficam estranhos no meio de uma cadeia
```

---

## 4. O Padrão `(valor, error)`

```go
// Toda função que pode falhar retorna error como ÚLTIMO valor
func abrirArquivo(caminho string) (*os.File, error) { ... }
func parsearJSON(dados []byte, v any) error { ... }
func buscarUsuario(id int) (*Usuario, error) { ... }

// Funções que não retornam valor útil
func salvar(u *Usuario) error { ... }

// Verificação imediata — o padrão Go mais importante
f, err := abrirArquivo("config.json")
if err != nil {
	return nil, fmt.Errorf("carregarConfig: %w", err)
}
defer f.Close()
```

---

## 5. Erros Sentinel — Valores Pré-definidos Comparáveis

```go
package meuapp

import "errors"

// Exportados — comparáveis com errors.Is()
// Por convenção: Err + descrição
var (
	ErrNaoEncontrado   = errors.New("não encontrado")
	ErrJaExiste        = errors.New("já existe")
	ErrPermissaoNegada = errors.New("permissão negada")
	ErrEntradaInvalida = errors.New("entrada inválida")
	ErrTimeout         = errors.New("timeout")
)

// Uso
func buscarProduto(id int) (*Produto, error) {
	p, ok := store[id]
	if !ok {
		return nil, fmt.Errorf("buscarProduto(%d): %w", id, ErrNaoEncontrado)
	}
	return p, nil
}

// Verificação
p, err := buscarProduto(99)
switch {
case errors.Is(err, ErrNaoEncontrado):
	http.Error(w, "produto não existe", 404)
case errors.Is(err, ErrPermissaoNegada):
	http.Error(w, "sem permissão", 403)
case err != nil:
	http.Error(w, "erro interno", 500)
default:
	jsonResponse(w, 200, p)
}
```

---

## 6. Propagar com Contexto — A Forma Correta

Cada camada adiciona contexto sem perder a causa raiz:

```go
// Camada de repositório
func (r *Repo) BuscarUsuario(id int) (*Usuario, error) {
	row := r.db.QueryRowContext(ctx, "SELECT * FROM users WHERE id=$1", id)
	if err := row.Scan(&u); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, fmt.Errorf("BuscarUsuario(%d): %w", id, ErrNaoEncontrado)
		}
		return nil, fmt.Errorf("BuscarUsuario(%d): %w", id, err)
	}
	return &u, nil
}

// Camada de serviço
func (s *Service) AtualizarEmail(userID int, email string) error {
	u, err := s.repo.BuscarUsuario(userID)
	if err != nil {
		return fmt.Errorf("AtualizarEmail: %w", err)
	}
	u.Email = email
	if err := s.repo.Salvar(u); err != nil {
		return fmt.Errorf("AtualizarEmail: %w", err)
	}
	return nil
}

// Camada de handler HTTP
func (h *Handler) AtualizarEmail(w http.ResponseWriter, r *http.Request) {
	err := h.svc.AtualizarEmail(userID, email)
	if err != nil {
		slog.Error("falha ao atualizar email", "err", err)   // log completo

		// Resposta para o cliente — apenas informação relevante
		switch {
		case errors.Is(err, ErrNaoEncontrado):
			jsonError(w, 404, "usuário não encontrado")
		case errors.Is(err, ErrEntradaInvalida):
			jsonError(w, 400, "email inválido")
		default:
			jsonError(w, 500, "erro interno")
		}
		return
	}
	w.WriteHeader(http.StatusNoContent)
}

// A chain de erros completa que vai para o log:
// "AtualizarEmail: BuscarUsuario(42): não encontrado"
```

---

## 7. Verificar e Extrair Erros

```go
// errors.Is — verifica se um erro está na chain
errors.Is(err, ErrNaoEncontrado)       // percorre toda a chain

// errors.As — extrai tipo específico da chain
var validErr *ErrValidacao
if errors.As(err, &validErr) {
	// validErr está preenchido com o erro concreto
	fmt.Println(validErr.Campo, validErr.Mensagem)
}

// errors.Unwrap — desembrulha uma camada
inner := errors.Unwrap(err)

// errors.Join — combina múltiplos erros (Go 1.20+)
combined := errors.Join(err1, err2, err3)
```

---

## 8. Padrões de Retorno

```go
// 1. Operação que pode falhar — sem valor útil
func deletar(id int) error { ... }

// 2. Operação que produz valor e pode falhar
func calcular(x int) (int, error) { ... }

// 3. Busca que pode não encontrar — sem erro de sistema
// (Use quando "não encontrar" é estado normal, não erro)
func buscarNoCache(chave string) (*Item, bool) { ... }

// 4. Múltiplos valores de resultado
func parsearData(s string) (time.Time, error) { ... }

// 5. Retornos nomeados — para clareza com múltiplos retornos
func buscarUsuario(id int) (usuario *Usuario, encontrado bool, err error) { ... }
```

---

## 9. Boas Práticas

```go
// ✅ Verificar erro imediatamente
resultado, err := operacao()
if err != nil {
	return fmt.Errorf("contexto: %w", err)
}

// ✅ Usar %w para preservar a chain
return fmt.Errorf("nome_da_funcao(%v): %w", param, err)

// ✅ Erros sentinel para verificação por tipo
var ErrTimeout = errors.New("timeout")

// ✅ Erros customizados para informação estruturada
type ErrValidacao struct {
	Campo    string
	Mensagem string
}

// ✅ Log o erro completo, exponha mensagem amigável ao usuário
slog.Error("operação falhou", "err", err)
http.Error(w, "erro interno", 500)

// ❌ Nunca ignore erros silenciosamente em produção
_, _ = escrever(dados)   // bug esperando acontecer

// ❌ Nunca use panic para erros de negócio esperados
if usuario == nil {
	panic("usuário não encontrado")   // errado — retorne error
}

// ❌ Não repita o contexto do erro desnecessariamente
// Ruim: "erro ao criar usuário: falha ao criar usuário: campo nome vazio"
// Bom:  "criarUsuario: campo 'nome' obrigatório"
```

---

## 10. Como Funciona Internamente

### A Interface `error` na Memória — itab + ponteiro de dados

Em Go, toda interface é representada por dois ponteiros:

```
Variável do tipo error na memória:
┌─────────────────────┬─────────────────────┐
│      *itab          │      *data          │
│  (tipo concreto +   │  (ponteiro para o   │
│   método Error())   │   valor real)       │
└─────────────────────┴─────────────────────┘

Exemplo: err := errors.New("falha")
  itab  → aponta para a tabela de {*errorString, método Error()}
  data  → aponta para errorString{"falha"} no heap

Chamada err.Error():
  1. Go lê *itab para encontrar o endereço do método Error()
  2. Chama Error() passando *data como receiver
  Custo: uma indireção extra comparado a chamada direta de função.
```

```go
// errors.New retorna *errorString — PONTEIRO deliberado
// Isso garante que dois erros com a mesma string sejam DIFERENTES
err1 := errors.New("falha")
err2 := errors.New("falha")

// Na memória:
// err1.data → 0xC000010020 → errorString{"falha"}
// err2.data → 0xC000010040 → errorString{"falha"}  ← endereço diferente!
fmt.Println(err1 == err2)   // false — ponteiros diferentes
```

### O Padrão `(valor, error)` — Espelho das Syscalls

O padrão de retorno múltiplo de Go imita diretamente a convenção de syscalls do Linux:

```
Linux syscall (C):
  ssize_t n = read(fd, buf, count);
  if (n == -1) {
      // errno contém o código de erro (ENOENT, EACCES, etc.)
      perror("read");
  }

Go:
  n, err := f.Read(buf)
  if err != nil {
      // err contém o erro como valor explícito
  }

Diferença crucial: errno em C é uma variável GLOBAL (thread-local no POSIX).
Go passa o erro como VALOR DE RETORNO — sem estado global, sem condição de corrida.
```

### Por Que Go Evita Exceções — Custo de Controle de Fluxo

```
Fluxo normal (return com error):
  funcA() → funcB() → funcC() → retorna error
  Cada retorno usa o mecanismo normal de retorno de função
  O processador prediz bem esse fluxo → cache quente

Exceções (throw/catch):
  funcA() → funcB() → funcC() → LANÇA exceção
  O runtime precisa fazer "stack unwinding":
    1. Percorrer a call stack buscando um handler catch
    2. Destruir frames de stack (chamar destructors em C++)
    3. Transferir controle para o catch
  Custo: muito mais caro que um return normal
  O processador NÃO prediz bem — branch misprediction

Go deliberadamente escolheu o caminho de menor custo de CPU.
panic/recover existe mas é reservado para erros IRRECUPERÁVEIS.
```

### Erros Sentinel — Comparação por Ponteiro

```go
// Definidos uma única vez:
var ErrNaoEncontrado = errors.New("não encontrado")

// ErrNaoEncontrado → *errorString em endereço fixo no data segment

// errors.Is(err, ErrNaoEncontrado) compara PONTEIROS, não strings:
// err.data == ErrNaoEncontrado?  → O(1), uma comparação de ponteiro

// Análogo aos constantes errno do POSIX:
// #define ENOENT  2   // No such file or directory
// #define EACCES 13   // Permission denied
// Também são inteiros comparados diretamente — mesma ideia, semântica equivalente
```

---

## 11. Conexão com Sistemas Operacionais

**`error` como interface — representação de dois ponteiros → [[Gerenciamento de Memória]]**
Toda interface em Go ocupa dois ponteiros (itab + data pointer) no heap ou stack. Entender como a memória representa interfaces explica por que `errors.New` retorna um ponteiro e por que comparar `err1 == err2` compara endereços, não conteúdo.

**Padrão `(valor, error)` espelha a convenção de syscalls → [[System Calls]]**
No Linux, toda syscall retorna o resultado em `rax`; se negativo, é um código `-errno`. Go replica essa semântica com retorno múltiplo explícito: `n, err := f.Read(buf)` mapeia diretamente para `(result, errno)` de `read(2)`.

**`errno` em C é global/thread-local; `error` em Go é um valor → [[Threads POSIX]]**
Em programas multithread C, `errno` é thread-local (definido via `__thread`), mas ainda é estado implícito — uma função que esquece de verificar `errno` antes de chamar outra pode perder o erro. Em Go, o erro é passado explicitamente como valor de retorno: sem estado compartilhado, sem condição de corrida entre goroutines.

**Erros sentinel como constantes de ponteiro → [[System Calls]]**
`io.EOF`, `os.ErrNotExist` são endereços fixos comparados em O(1), exatamente como `ENOENT=2`, `EEOF` no POSIX. A diferença: Go usa ponteiros tipados; POSIX usa inteiros — mas a semântica de "valor especial pré-definido comparável" é idêntica.

**Go evita exceções para não pagar o custo de stack unwinding → [[Processadores]]**
Exceções (como em Java/C++) exigem que o CPU percorra e desmonte a call stack ao propagar um erro — operação cara que quebra o pipeline de instruções e causa branch misprediction. Go usa retornos normais que o preditor de branch do processador lida bem, mantendo o pipeline quente.