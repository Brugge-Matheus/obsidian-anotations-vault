---
tags:
  - go
  - go/funcoes
aliases:
  - Múltiplos retornos
  - Retornos nomeados
  - Naked return
  - (valor, error)
ordem: 17
modulo: "04 - Funções"
---
# Múltiplos Retornos e Erros

> **Nota 17 da trilha de Go.** Em Go, uma função pode devolver **vários resultados** de uma vez. Isso parece um detalhe, mas é a base de um dos traços mais marcantes da linguagem: o padrão **`(valor, error)`**, que substitui as exceções de outras linguagens. Nesta nota veremos múltiplos retornos, retornos nomeados, o `return` "nu", como ignorar resultados, os padrões `(valor, ok)` e `(valor, error)` e uma **primeira visão** de como tratar e propagar erros. O tratamento de erros completo ganha um módulo próprio (06).
>
> **Pré-requisitos:** [[Funções — Fundamentos]].

---

## 🧭 De onde viemos

Em [[Funções — Fundamentos]] escrevemos funções com um único resultado e vimos, no assembly, que o resultado volta num registrador (`AX`). Também já usamos funções com dois resultados sem explicar: `strconv.Atoi` (número e erro), a leitura de map `v, ok := m[k]`. Agora vamos formalizar.

---

## 🔢 1. Vários resultados

A lista de tipos de resultado vai **entre parênteses** (saída real):

```go
func divmod(a, b int) (int, int) {
	return a / b, a % b
}

q, r := divmod(17, 5)
fmt.Println(q, r)          // 3 2
```

Regras:

- O `return` precisa fornecer **todos** os resultados, na ordem
- Quem chama precisa receber **todos** (ou descartar com `_`)
- Não existe "tupla" como tipo: os múltiplos resultados não podem ser guardados numa variável só

```go
x := divmod(17, 5)          // ❌ assignment mismatch: 1 variable but divmod returns 2 values
q, _ := divmod(17, 5)       // ✅ descarta o resto
_, r := divmod(17, 5)       // ✅ descarta o quociente
```

### 1.1 Encadeando chamadas

Um caso especial: se uma função devolve exatamente os valores que outra recebe, dá para passar a chamada direto (saída real):

```go
fmt.Println(divmod(17, 5))          // 3 2   (Println é variádica: recebe os 2 valores)

func soma(a, b int) int { return a + b }
soma(divmod(17, 5))                 // ✅ 3 + 2 = 5
```

### 1.2 Por que Go escolheu isso

Outras linguagens resolvem "devolver mais de uma coisa" de outras formas:

| Linguagem | Como devolve dois valores |
| --- | --- |
| C | Ponteiro de saída: `int divmod(int a, int b, int *resto)` |
| Java | Criar uma classe ou usar arrays/objetos |
| Python | Tupla: `return q, r` (um objeto tupla) |
| PHP | Array: `return [$q, $r]` |
| **Go** | Múltiplos resultados nativos, sem criar objeto algum |

O caso de uso mais importante, de longe, é devolver **um resultado e um erro**.

---

## 🏷️ 2. Retornos nomeados

Os resultados podem ter **nomes**. Eles viram variáveis locais, já inicializadas com o zero value (saída real):

```go
func divmod(a, b int) (q, r int) {
	q = a / b
	r = a % b
	return          // "return nu": devolve os valores atuais de q e r
}

fmt.Println(divmod(17, 5))     // 3 2
```

### 2.1 Para que servem

**1) Documentação:** os nomes aparecem no `go doc` e no editor, explicando o significado de cada resultado:

```go
func minmax(nums ...int) (menor, maior int)        // melhor que (int, int)
func (f *File) Read(b []byte) (n int, err error)   // a própria stdlib usa
```

**2) Modificar o resultado dentro de um `defer`:** só é possível com resultados nomeados (saída real):

```go
func lerConfig() (cfg string, err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("lerConfig: %w", err)   // altera o erro DEPOIS do return
		}
	}()
	_, err = os.ReadFile("/nao/existe.yaml")
	return "", err
}

_, err := lerConfig()
fmt.Println(err)    // lerConfig: open /nao/existe.yaml: no such file or directory
```

Isso funciona porque o `return "", err` primeiro **atribui** os valores aos resultados nomeados, depois roda os `defer`, e só então a função realmente sai. Veremos a ordem exata em [[Defer, Panic e Recover]].

### 2.2 O `return` nu: use com moderação

```go
func processar() (total int, err error) {
	// ... 60 linhas ...
	total = 10
	// ... mais 40 linhas ...
	return            // ❓ o que está sendo devolvido? Precisa rolar a tela para saber
}
```

> ✅ Recomendação da comunidade: use `return` nu **só em funções curtas**. Em funções longas, escreva `return total, err` explicitamente. Ter resultados nomeados **não obriga** a usar `return` nu.

### 2.3 Cuidado com shadowing de resultados nomeados

```go
func buscar() (u *Usuario, err error) {
	if cond {
		u, err := consultar()   // ⚠️ := cria NOVAS variáveis u e err, sombreando os resultados
		_, _ = u, err
		return                  // ❌ o compilador recusa (mensagem real abaixo)
	}
	return
}
```

```
./main.go:10:3: result parameter u not in scope at return
	./main.go:7:3: inner declaration of var u *int
./main.go:10:3: result parameter err not in scope at return
```

O compilador protege contra o pior caso (o `return` nu com resultado sombreado), mas o padrão de shadowing que vimos em [[Sintaxe, Blocos e Escopo]] continua valendo.

---

## ✅ 3. O padrão `(valor, ok)`

Quando a "falha" é simplesmente **"não encontrei"** e não há nada mais a explicar, o idioma é devolver um `bool` como último resultado:

```go
v, ok := m["chave"]            // map: a chave existe?
s, ok := x.(string)            // type assertion: o valor é string?
v, ok := <-ch                  // channel: recebeu um valor ou o channel foi fechado?
```

Nas suas próprias funções:

```go
func (c *Cache) Get(chave string) (valor string, ok bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	valor, ok = c.dados[chave]
	return
}
```

---

## ❌ 4. O padrão `(valor, error)`

Quando algo **pode dar errado** por motivos que valem ser explicados (arquivo inexistente, rede caiu, entrada inválida), a função devolve um **`error`** como **último** resultado:

```go
func dividir(a, b int) (int, error) {
	if b == 0 {
		return 0, errors.New("divisão por zero")   // falhou: valor zero + erro
	}
	return a / b, nil                             // sucesso: valor + nil
}
```

> 💡 **`error`:** um tipo pré-declarado (é uma **interface** com um único método, `Error() string`). O valor `nil` significa "sem erro". Qualquer tipo que tenha o método `Error() string` pode ser usado como erro. Veremos a fundo em [[O Padrão error em Go]].

A forma de usar é **sempre verificar o erro logo depois da chamada** (saída real):

```go
r, err := dividir(10, 0)
if err != nil {
	fmt.Println("erro:", err)      // erro: divisão por zero
	return
}
fmt.Println(r)
```

### 4.1 As convenções

| Convenção | Exemplo |
| --- | --- |
| `error` é sempre o **último** resultado | `func f() (T, error)`, nunca `(error, T)` |
| Com erro, os outros resultados são o zero value (e não devem ser usados) | `return 0, err` / `return nil, err` |
| Sem erro, o erro é `nil` | `return valor, nil` |
| Verifique o erro **antes** de usar o valor | `if err != nil { ... }` |
| Mensagens minúsculas, sem ponto final | `"divisão por zero"` |
| Ao propagar, **adicione contexto** | `fmt.Errorf("ler config: %w", err)` |

> ⚠️ Algumas funções da biblioteca padrão são exceções documentadas e devolvem dados úteis **junto** com um erro. O exemplo clássico é `io.Reader.Read`, que pode devolver `n > 0` bytes lidos **e** `io.EOF` na mesma chamada. Leia a documentação de cada função (ver [[io.Reader e io.Writer]]).

### 4.2 Propagando erros com contexto

Uma função que chama outra normalmente **devolve o erro para cima**, acrescentando o que estava fazendo (saída real):

```go
func parsePorta(s string) (int, error) {
	p, err := strconv.Atoi(s)
	if err != nil {
		return 0, fmt.Errorf("porta %q inválida: %w", s, err)   // %w "embrulha" o erro original
	}
	if p < 1 || p > 65535 {
		return 0, fmt.Errorf("porta %d fora da faixa", p)
	}
	return p, nil
}

_, err := parsePorta("80a")
fmt.Println(err)
// porta "80a" inválida: strconv.Atoi: parsing "80a": invalid syntax
```

Repare como a mensagem final conta uma **história** da camada mais externa para a mais interna. É isso que torna erros de Go fáceis de diagnosticar sem stack trace.

Como o erro original foi embrulhado com `%w`, ainda dá para inspecioná-lo (saídas reais):

```go
_, err := lerConfig()
fmt.Println(errors.Is(err, os.ErrNotExist))    // true: "em algum ponto da cadeia é um 'não existe'?"

var ne *strconv.NumError
_, err = parsePorta("x")
fmt.Println(errors.As(err, &ne), ne.Func, ne.Num)   // true Atoi x: extrai o erro de um tipo específico
```

`errors.Is`, `errors.As` e o embrulho com `%w` são o assunto de [[Wrapping e Unwrapping de Erros]].

### 4.3 O formato do código: "caminho feliz" encostado à esquerda

```go
func carregarUsuario(id string) (*Usuario, error) {
	dados, err := buscarNoBanco(id)
	if err != nil {
		return nil, fmt.Errorf("buscar usuário %s: %w", id, err)
	}

	u, err := decodificar(dados)
	if err != nil {
		return nil, fmt.Errorf("decodificar usuário %s: %w", id, err)
	}

	if err := u.Validar(); err != nil {
		return nil, fmt.Errorf("usuário %s inválido: %w", id, err)
	}

	return u, nil
}
```

Cada erro é tratado e devolvido logo; o código "normal" fica alinhado à esquerda e se lê de cima para baixo. É o estilo de **guard clauses** que vimos em [[Controle de Fluxo]].

---

## 🕳️ 5. Ignorando resultados

```go
n, _ := strconv.Atoi("42")         // ignora o erro: só quando há CERTEZA de que não falha
_ = os.Remove(tmp)                 // ignora explicitamente um erro irrelevante
fmt.Println("oi")                  // devolve (n int, err error), e ignorar é aceito por convenção
```

> ⚠️ Ignorar um erro com `_` deve ser uma **decisão consciente**. Linters como `errcheck` (incluído no `golangci-lint`) apontam erros ignorados sem querer. Funções como `fmt.Println` e `strings.Builder.WriteString` são exceções aceitas.

---

## ⚖️ 6. Erros vs exceções

| | Exceções (Java, PHP, Python, Ruby) | Valores de erro (Go) |
| --- | --- | --- |
| Como sinaliza | `throw` interrompe o fluxo e "pula" pela stack | `return ..., err` como qualquer valor |
| Visibilidade | A assinatura nem sempre mostra o que pode falhar | `error` aparece na assinatura |
| Onde é tratado | Em algum `catch` lá em cima (às vezes longe demais) | Logo depois da chamada |
| Custo | Desenrolar a stack é caro | Um `if` e um registrador |
| Risco | Esquecer de tratar → derruba o programa | Esquecer de tratar → bug silencioso (os linters ajudam) |
| Verbosidade | Menor | `if err != nil` repetido |

> 💡 Go também tem um mecanismo parecido com exceções, o `panic`, mas ele é reservado para **erros de programação** e situações irrecuperáveis (índice fora dos limites, nil, invariantes quebradas). Falhas **esperadas** (rede, arquivo, entrada do usuário) são sempre `error`. Veremos a distinção em [[Defer, Panic e Recover]].

---

## 🔬 7. Por dentro: múltiplos resultados são só registradores

Com a ABI de registradores (Go 1.17+), vários resultados voltam em **vários registradores**, sem criar nenhum objeto. Assembly real de uma função que devolve quociente e resto:

```go
//go:noinline
func divmod(a, b int) (int, int) { return a / b, a % b }
```

```
main.divmod:
    TESTQ  BX, BX                ; checagem de divisão por zero (ver Operações Aritméticas)
    JEQ    panico
    ...
    CQO
    IDIVQ  BX                    ; quociente → AX, resto → DX
    MOVQ   DX, BX                ; move o resto para BX: 2º resultado
    RET                          ; resultados: AX (1º) e BX (2º)
```

Os resultados seguem a mesma sequência de registradores dos argumentos (`AX`, `BX`, `CX`, ...). Por isso devolver `(valor, error)` é praticamente gratuito: um `error` é uma interface (2 palavras: ponteiro de tipo e ponteiro de dados), que ocupa mais dois registradores.

### 7.1 A herança das system calls

O padrão `(valor, error)` é uma versão melhorada da convenção das syscalls de UNIX que você estudou em [[System Calls]] e [[Syscalls para Gerenciamento de Arquivos]]:

```
C / POSIX:                                   Go:
int fd = open("a.txt", O_RDONLY);            f, err := os.Open("a.txt")
if (fd == -1) {                              if err != nil {
    // o motivo está numa variável global        // o motivo está NO PRÓPRIO valor err
    perror("open");   // errno = ENOENT          fmt.Println(err)
}                                            }
```

| Problema da convenção de C | Solução de Go |
| --- | --- |
| O valor de erro (`-1`) se mistura com valores válidos | Erro num resultado separado |
| `errno` é global: qualquer chamada posterior pode sobrescrevê-lo | O erro viaja junto com o resultado |
| Só um código numérico | Uma mensagem com contexto, que pode embrulhar outros erros |

No Linux, a syscall de verdade devolve o resultado num registrador e, em caso de erro, um número negativo (`-ENOENT`). O pacote `syscall` do Go transforma isso num `syscall.Errno`, e o pacote `os` embrulha num `*os.PathError` com a operação e o caminho (`open /nao/existe.yaml: no such file or directory`).

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Usar o valor sem checar o erro | Valor zero/inválido usado como se fosse bom | `if err != nil` antes de usar |
| Ignorar erro com `_` sem pensar | Falhas silenciosas | Trate ou documente o motivo; use `errcheck` |
| `return` nu em função longa | Difícil saber o que é devolvido | Retorne explicitamente |
| Shadowing de `err` com `:=` em bloco interno | Erro "some" | Use `=` (ver [[Sintaxe, Blocos e Escopo]]) |
| Devolver erro sem contexto | "no such file or directory" sem saber qual | `fmt.Errorf("contexto: %w", err)` |
| Erro como primeiro resultado | Fora da convenção, confunde linters e pessoas | `error` sempre por último |
| Usar `panic` para falhas esperadas | Programa cai por um arquivo ausente | Devolva `error` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Múltiplos resultados | `func f() (A, B)`; quem chama recebe todos ou descarta com `_` |
| Encadear | `g(f())` quando os tipos casam |
| Resultados nomeados | Documentam e permitem alterar o resultado num `defer` |
| `return` nu | Só em funções curtas |
| `(valor, ok)` | "Achei ou não achei" |
| `(valor, error)` | Falhas explicáveis; `error` por último; `nil` = sucesso |
| Tratamento | Verificar logo; propagar com contexto via `%w` |
| Inspeção | `errors.Is`, `errors.As` (módulo 06) |
| Por dentro | Resultados em registradores (`AX`, `BX`...); herança do `errno` de C |

---

## 🔗 Conexões

**Dentro de Go**
- O tipo `error` em profundidade → [[O Padrão error em Go]]
- Tipos de erro próprios → [[Criando Erros Customizados]]
- `%w`, `errors.Is`, `errors.As`, `errors.Join` → [[Wrapping e Unwrapping de Erros]]
- Alterar resultados nomeados com `defer`; `panic` vs `error` → [[Defer, Panic e Recover]]
- Vírgula ok em maps, type assertions e channels → [[Maps]], [[Type Assertion e Type Switch]], [[Channels]]
- `io.EOF`: erro que não é falha → [[io.Reader e io.Writer]]

**Sistemas Operacionais**
- [[System Calls]] e [[Syscalls para Gerenciamento de Arquivos]] — retorno `-1` + `errno`, a convenção que inspirou `(valor, error)`
- [[Processadores]] — os resultados voltam em registradores, sem acesso à memória

---

⬅️ **Anterior:** [[Funções — Fundamentos]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Funções Variádicas]] ➡️
