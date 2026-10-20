---
tags:
  - go
  - go/erros
aliases:
  - Wrapping de erros
  - errors.Is
  - errors.As
  - errors.Join
  - "%w"
  - Cadeia de erros
ordem: 29
modulo: "06 - Tratamento de Erros"
---
# Wrapping e Unwrapping de Erros

> **Nota 29 da trilha de Go.** Ao propagar um erro, queremos duas coisas que parecem contraditórias: **acrescentar contexto** (o que cada camada estava fazendo) e **preservar a identidade** do erro original (para que alguém lá em cima consiga saber que foi um "arquivo não existe"). A solução de Go, desde a versão 1.13, é **embrulhar** (*wrap*) erros: cada camada cria um erro novo que **contém** o anterior, formando uma **cadeia**. Nesta nota veremos o verbo `%w`, as funções `errors.Unwrap`, `errors.Is`, `errors.As`, `errors.AsType` (1.26) e `errors.Join`, a decisão entre `%w` e `%v`, como a cadeia é percorrida e como isso se compara a outras linguagens.
>
> **Pré-requisitos:** [[O Padrão error em Go]], [[Criando Erros Customizados]].

---

## 🧭 De onde viemos

Em [[O Padrão error em Go]] usamos `errors.Is` e `errors.As` dizendo "veremos o porquê depois". Em [[Criando Erros Customizados]] implementamos `Unwrap()` e `Is()` nos nossos tipos. Agora vamos entender o mecanismo que liga tudo isso: a **cadeia de erros**.

---

## 😩 1. O problema: contexto vs identidade

Antes do Go 1.13, havia um dilema:

```go
// Opção 1: devolver o erro como está → perde o contexto
if err != nil {
	return err          // lá em cima: "no such file or directory" (de qual arquivo? fazendo o quê?)
}

// Opção 2: criar um erro novo com contexto → perde a identidade
if err != nil {
	return fmt.Errorf("carregar config: %v", err)   // %v vira só TEXTO
}
// lá em cima: err == os.ErrNotExist → false. A informação "é um não-existe" se perdeu.
```

Com **wrapping**, o erro novo guarda o texto do contexto **e um ponteiro para o erro original**:

```go
return fmt.Errorf("carregar config: %w", err)   // %w EMBRULHA
// lá em cima: errors.Is(err, os.ErrNotExist) → true, e a mensagem tem o contexto
```

---

## 🎁 2. Embrulhando com `%w`

O verbo `%w` (de *wrap*) só existe em `fmt.Errorf`. Ele formata o erro como `%v` **e** guarda o erro dentro do resultado (saída real):

```go
w := fmt.Errorf("camada2: %w", fmt.Errorf("camada1: %w", ErrA))

for e := error(w); e != nil; e = errors.Unwrap(e) {
	fmt.Printf("%T: %v\n", e, e)
}
// *fmt.wrapError: camada2: camada1: A
// *fmt.wrapError: camada1: A
// *errors.errorString: A
```

```
      w (*fmt.wrapError)
      ┌───────────────────────────────┐
      │ msg: "camada2: camada1: A"    │
      │ err ──────────┐               │
      └───────────────┼───────────────┘
                      ▼
      ┌───────────────────────────────┐
      │ msg: "camada1: A"             │   *fmt.wrapError
      │ err ──────────┐               │
      └───────────────┼───────────────┘
                      ▼
      ┌───────────────────────────────┐
      │ s: "A"                        │   *errors.errorString  (ErrA, a sentinela)
      └───────────────────────────────┘
```

A cadeia é uma **lista ligada** de erros. Cada nó sabe devolver o próximo pelo método `Unwrap() error`. O tipo interno do `fmt` é simplesmente:

```go
type wrapError struct {
	msg string
	err error
}
func (e *wrapError) Error() string { return e.msg }
func (e *wrapError) Unwrap() error { return e.err }
```

### 2.1 Vários `%w` numa mensagem (Go 1.20+)

Saída real:

```go
dupla := fmt.Errorf("salvar: %w; notificar: %w", ErrNaoEncontrado, io.EOF)
fmt.Println(dupla)                                 // salvar: não encontrado; notificar: EOF
fmt.Println(errors.Is(dupla, io.EOF))              // true
fmt.Println(errors.Is(dupla, ErrNaoEncontrado))    // true
```

Com mais de um `%w`, o erro resultante implementa `Unwrap() []error` (vários filhos), e a cadeia vira uma **árvore**.

---

## 🔓 3. `errors.Unwrap`: descer um nível

```go
func Unwrap(err error) error   // devolve o erro "de dentro", ou nil
```

Saída real com um erro de arquivo:

```go
_, err := os.Open("/x")
fmt.Printf("%T | %v\n", err, errors.Unwrap(err))   // *fs.PathError | no such file or directory
fmt.Printf("%T\n", errors.Unwrap(err))             // syscall.Errno
```

> ⚠️ `errors.Unwrap` só funciona com `Unwrap() error` (um filho). Para erros com vários filhos (`errors.Join`, vários `%w`), ele devolve `nil` (saída real: `errors.Unwrap(errors.Join(a, b)) == nil` → `true`). Na prática, você quase nunca chama `Unwrap` diretamente: use `Is` e `As`, que sabem percorrer árvores.

---

## 🔍 4. `errors.Is`: "em algum ponto da cadeia, há este erro?"

```go
func Is(err, target error) bool
```

Percorre a cadeia (ou árvore) e devolve `true` se algum erro:

1. for `==` ao `target`, **ou**
2. tiver um método `Is(error) bool` que devolva `true` para o `target`

Saídas reais:

```go
err := handler("x")     // ErroHTTP → wrapError → ErrNaoEncontrado
errors.Is(err, ErrNaoEncontrado)        // true

err = ler()             // wrapError → *fs.PathError → syscall.ENOENT
errors.Is(err, os.ErrNotExist)          // true  (ENOENT implementa Is → ErrNotExist)
errors.Is(err, fs.ErrNotExist)          // true  (os.ErrNotExist == fs.ErrNotExist)
```

Use para **sentinelas**: `io.EOF`, `os.ErrNotExist`, `sql.ErrNoRows`, `context.Canceled`, `ErrNaoEncontrado`.

---

## 🎣 5. `errors.As`: "há um erro deste TIPO? me dê ele"

```go
func As(err error, target any) bool    // target: ponteiro para uma variável do tipo procurado
```

Percorre a cadeia e, no primeiro erro que pode ser atribuído ao tipo de `*target`, **guarda** esse erro na variável e devolve `true` (saídas reais):

```go
var he *ErroHTTP
if errors.As(err, &he) {                 // ⚠️ passe o ENDEREÇO da variável
	fmt.Println(he.Status)               // 404
}

var pe *fs.PathError
if errors.As(err, &pe) {
	fmt.Println(pe.Op, pe.Path)          // open /nao/existe
}

var t interface{ Temporary() bool }     // também funciona com INTERFACES (comportamento)
if errors.As(err, &t) && t.Temporary() { ... }
```

> ⚠️ `errors.As` dá **panic** se `target` não for um ponteiro não-nil para um tipo que implementa `error` ou para uma interface. O `go vet` detecta os erros mais comuns (como esquecer o `&`).

### 5.1 `errors.AsType` (Go 1.26): a versão genérica

`errors.As` exige declarar a variável antes e passar seu endereço, um resquício de quando Go não tinha generics. Desde o Go 1.26 existe a versão genérica, mais segura e mais rápida:

```go
// antes
var pe *fs.PathError
if errors.As(err, &pe) {
	fmt.Println(pe.Path)
}

// Go 1.26+
if pe, ok := errors.AsType[*fs.PathError](err); ok {
	fmt.Println(pe.Path)
}
```

O tipo é verificado em **compilação** (não dá para passar um tipo que não seja erro) e a variável fica com escopo limitado ao `if`. Veremos o mecanismo de funções genéricas em [[Generics]].

---

## 📚 6. `errors.Join`: juntando erros independentes (Go 1.20+)

Quando várias operações independentes falham, e todas importam:

```go
err := errors.Join(err1, err2, err3)    // ignora os nil; se todos forem nil, devolve nil
```

```go
// fechar vários recursos, sem perder nenhum erro
func (s *Servidor) Fechar() error {
	return errors.Join(
		s.http.Close(),
		s.banco.Close(),
		s.cache.Close(),
	)
}

// juntar o erro principal com o erro do Close (ver Defer, Panic e Recover)
defer func() { err = errors.Join(err, f.Close()) }()
```

A mensagem junta todas com quebras de linha, e `errors.Is`/`As` encontram qualquer uma delas (vimos em [[Criando Erros Customizados]]).

---

## ⚖️ 7. `%w` ou `%v`? Uma decisão de API

Embrulhar com `%w` torna o erro interno **parte da sua API**: quem chama pode passar a depender dele com `errors.Is`/`As`. Às vezes você **não** quer isso.

| Use `%w` quando... | Use `%v` quando... |
| --- | --- |
| A causa é algo que o chamador **deve** poder distinguir (`ErrNaoEncontrado`, `context.Canceled`, `fs.ErrNotExist`) | A causa é um **detalhe de implementação** (o erro específico do driver PostgreSQL, de uma biblioteca HTTP interna) |
| Você está dentro do mesmo módulo/sistema | Você está na fronteira de uma biblioteca pública e quer liberdade para trocar a implementação |
| O erro é de uma camada que você controla | Você vai **traduzir** o erro para uma sentinela sua |

Saída real da diferença:

```go
semW := fmt.Errorf("contexto: %v", ErrNaoEncontrado)
errors.Is(semW, ErrNaoEncontrado)      // false: virou só texto
```

Um padrão comum na camada de repositório: **traduzir** o erro do banco para uma sentinela do seu domínio:

```go
func (r *Repo) Buscar(ctx context.Context, id string) (*Usuario, error) {
	err := r.db.QueryRowContext(ctx, q, id).Scan(&u.Nome, &u.Email)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, ErrNaoEncontrado                       // sentinela do SEU pacote
	}
	if err != nil {
		return nil, fmt.Errorf("buscar usuário %s: %v", id, err)   // detalhe do banco: não embrulha
	}
	return &u, nil
}
```

Assim, o resto do sistema depende de `ErrNaoEncontrado`, não do `database/sql`, e você pode trocar o banco sem quebrar ninguém.

---

## 🗺️ 8. Boas práticas de wrapping

```go
// ✅ cada camada acrescenta O QUE fazia e COM QUÊ, uma vez
return fmt.Errorf("processar pedido %d: %w", id, err)

// ❌ redundante: a mensagem já diz que falhou
return fmt.Errorf("erro ao processar pedido: falhou: %w", err)

// ❌ repetir informação que a camada de baixo já incluiu
// (os.Open já põe o caminho na mensagem: "open app.yaml: ...")
return fmt.Errorf("abrir app.yaml: %w", err)    // vira "abrir app.yaml: open app.yaml: ..."
return fmt.Errorf("carregar config: %w", err)   // ✅ acrescenta o PROPÓSITO

// ❌ embrulhar duas vezes na mesma função
```

| Faça | Evite |
| --- | --- |
| Acrescente contexto ao **devolver** | Embrulhar e também registrar em log (ver "log uma vez" em [[O Padrão error em Go]]) |
| Compare com `errors.Is` | `err == ErrX` |
| Extraia com `errors.As` / `errors.AsType` | Type assertion direta `err.(*T)` |
| Decida conscientemente entre `%w` e `%v` | Embrulhar tudo com `%w` "por padrão" numa biblioteca pública |

---

## 🔬 9. Por dentro: como `Is` e `As` percorrem a cadeia

A implementação de `errors.Is` é uma **busca em profundidade** (pré-ordem) na árvore de erros. Simplificada:

```go
func Is(err, target error) bool {
	if err == nil || target == nil {
		return err == target
	}
	comparavel := reflectlite.TypeOf(target).Comparable()
	for {
		if comparavel && err == target {           // 1) igualdade direta
			return true
		}
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
			return true                            // 2) igualdade personalizada
		}
		switch x := err.(type) {
		case interface{ Unwrap() error }:          // 3a) um filho: desce
			err = x.Unwrap()
			if err == nil {
				return false
			}
		case interface{ Unwrap() []error }:        // 3b) vários filhos: recursão em cada um
			for _, e := range x.Unwrap() {
				if Is(e, target) {
					return true
				}
			}
			return false
		default:
			return false
		}
	}
}
```

Observações:

- Cada nível custa algumas **type assertions** para interfaces (as que vimos em [[Type Assertion e Type Switch]]); cadeias têm tipicamente 2 a 5 níveis, então o custo é desprezível fora de laços muito quentes
- A ordem de busca em árvores é **profundidade primeiro**: o primeiro filho e todos os seus descendentes, depois o segundo filho...
- `errors.As` faz o mesmo percurso, mas em vez de `==` testa se o erro é **atribuível** ao tipo do alvo (usando reflection leve) e copia o valor encontrado

### 9.1 Comparação com outras linguagens

| Linguagem | Mecanismo | Equivalente ao `Unwrap` |
| --- | --- | --- |
| Java | Encadeamento de exceções: `new X("contexto", causa)` | `getCause()`; o stack trace mostra "Caused by: ..." |
| Python | `raise X("contexto") from causa` | `__cause__` |
| Rust | `Result<T, E>` + `?` + crates como `anyhow`/`thiserror` | `source()` |
| C / POSIX | `errno` global, sem cadeia | Nenhum: o contexto é perdido ou vai só para logs |
| **Go** | `fmt.Errorf("contexto: %w", err)` | `Unwrap()` |

A diferença conceitual: em Java e Python o encadeamento acontece no mecanismo de **exceções**; em Go, é só uma **estrutura de dados** (uma lista ligada de valores), coerente com a ideia de que "erros são valores".

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `%v` quando queria `%w` | `errors.Is` não encontra a causa | Use `%w` |
| `%w` em biblioteca pública expondo detalhes | Chamadores dependem da sua implementação | `%v` ou traduza para sentinela |
| `errors.As(err, pe)` sem `&` | panic ou erro do `go vet` | `errors.As(err, &pe)`, ou `errors.AsType` |
| `errors.Unwrap` em erro de `Join` | Devolve nil | Use `Is`/`As` |
| `%w` fora de `fmt.Errorf` | `Sprintf` mostra `%!w(...)` | `%w` só no `Errorf` |
| Contexto repetido em todas as camadas | Mensagens longas e redundantes | Cada camada acrescenta só o seu propósito |
| Comparar com `==` | Falha com erros embrulhados | `errors.Is` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Wrapping | Erro novo que contém o anterior: contexto + identidade |
| `%w` | Embrulha em `fmt.Errorf`; vários `%w` desde 1.20 |
| Cadeia | Lista ligada (ou árvore) via `Unwrap() error` / `Unwrap() []error` |
| `errors.Unwrap` | Desce um nível (só para um filho) |
| `errors.Is` | Procura um **valor** (sentinela) em toda a cadeia; respeita métodos `Is` |
| `errors.As` | Procura um **tipo** e extrai; aceita interfaces |
| `errors.AsType` | Versão genérica de `As` (Go 1.26) |
| `errors.Join` | Junta erros independentes (Go 1.20) |
| `%w` vs `%v` | `%w` faz a causa virar API; `%v` esconde |
| Por dentro | Busca em profundidade com type assertions a cada nível |

---

## 🔗 Conexões

**Dentro de Go**
- Filosofia e sentinelas → [[O Padrão error em Go]]
- `Unwrap`, `Is` e erros múltiplos nos seus tipos → [[Criando Erros Customizados]]
- `%w` e verbos do `fmt` → [[Strings em Profundidade]]
- `errors.Join` com `Close` adiado → [[Defer, Panic e Recover]]
- `errors.AsType` e funções genéricas → [[Generics]]
- Type assertions usadas na travessia → [[Type Assertion e Type Switch]]
- `context.Canceled` / `DeadlineExceeded` embrulhados → [[context.Context]]

**Sistemas Operacionais**
- [[System Calls]] — o `errno` de C não tem cadeia; o Go preserva o código original (`syscall.Errno`) no fim da cadeia
- [[Syscalls para Gerenciamento de Arquivos]] — `ENOENT`, `EACCES`, `EEXIST`: os códigos que viram `fs.ErrNotExist`, `fs.ErrPermission`, `fs.ErrExist`

**Algoritmos e Estruturas de Dados**
- [[Linked Lists]] — a cadeia de erros é uma lista ligada
- [[Árvores Binárias]] e [[Grafos]] — com vários `%w` ou `Join`, a cadeia vira uma árvore percorrida em profundidade

---

⬅️ **Anterior:** [[Criando Erros Customizados]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Criando Pacotes]] ➡️
