---
tags:
  - go
  - go/funcoes
aliases:
  - Variádicas
  - Variadic functions
  - "...T"
ordem: 18
modulo: "04 - Funções"
---
# Funções Variádicas

> **Nota 18 da trilha de Go.** Uma função **variádica** aceita **qualquer quantidade** de argumentos do mesmo tipo no seu último parâmetro: `fmt.Println("a", 1, true)`, `append(s, 1, 2, 3)`, `max(3, 9, 1)`. Por baixo, esses argumentos chegam como um **slice**. Nesta nota veremos a declaração com `...T`, como **espalhar** um slice existente com `s...`, a armadilha do compartilhamento de memória, o caso `...any`, exemplos da biblioteca padrão e como o compilador evita alocações.
>
> **Pré-requisitos:** [[Funções — Fundamentos]], [[Arrays e Slices]].

---

## 🧭 De onde viemos

Em [[Funções — Fundamentos]] vimos que Go não tem sobrecarga nem parâmetros com valor padrão. Então como `fmt.Println` aceita 1, 2 ou 10 argumentos? E como `append` aceita um ou vários elementos? A resposta é o parâmetro variádico, que se apoia no que estudamos em [[Arrays e Slices]].

> 💡 **Função variádica:** função cujo último parâmetro aceita zero ou mais argumentos. O nome vem de *variadic* ("de aridade variável"; **aridade** é a quantidade de argumentos de uma função).

---

## ✍️ 1. Declarando e chamando

O último parâmetro recebe `...` antes do tipo. Dentro da função, ele é um **slice** desse tipo (saídas reais):

```go
func somar(nums ...int) int {      // nums é um []int dentro da função
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

fmt.Println(somar(), somar(1), somar(1, 2, 3))   // 0 1 6
```

```go
func info(nums ...int) { fmt.Printf("%v len=%d nil=%v\n", nums, len(nums), nums == nil) }

info()        // [] len=0 nil=true    ← sem argumentos: slice NIL
info(1, 2)    // [1 2] len=2 nil=false
```

Regras:

- Só o **último** parâmetro pode ser variádico, e só um por função
- Chamar sem argumentos variádicos é válido: o slice chega `nil`
- Parâmetros fixos vêm antes:

```go
func log(nivel string, partes ...any) { ... }
func fmt.Printf(format string, a ...any) (n int, err error)
func errors.Join(errs ...error) error
```

---

## 💥 2. Espalhando um slice com `s...`

Se você já tem um slice, passe-o com `...` depois do nome (saída real):

```go
s := []int{4, 5, 6}
fmt.Println(somar(s...))        // 15
somar(s)                        // ❌ cannot use s (variable of type []int) as int value in argument to somar
```

É o mesmo mecanismo que já usamos com `append`:

```go
a := append([]int{1}, []int{2, 3}...)     // [1 2 3]
b := append([]byte("go"), "lang"...)      // caso especial: string espalhada em []byte → "golang"
```

Restrições:

```go
somar(1, 2, s...)       // ❌ não pode misturar valores avulsos com s... no mesmo parâmetro
somar(append([]int{1, 2}, s...)...)   // ✅ junte antes
```

---

## ⚠️ 3. A armadilha: `s...` NÃO copia

Quando você passa valores avulsos, o compilador cria um slice novo. Quando você espalha um slice existente com `s...`, a função recebe **o mesmo slice** (o mesmo array por baixo). Saídas reais:

```go
func zerar(nums ...int) {
	for i := range nums {
		nums[i] = 0
	}
}

s := []int{4, 5, 6}
zerar(s...)
fmt.Println(s)      // [0 0 0]   ← o slice do chamador foi alterado!

t := []int{7, 8}
zerar(t[0], t[1])
fmt.Println(t)      // [7 8]     ← valores avulsos: a função recebeu um slice novo
```

```
zerar(s...)                           zerar(t[0], t[1])
s ──▶ [4 5 6]  ◀── nums               t ──▶ [7 8]
      (mesmo array)                   nums ──▶ [7 8]  (array novo, temporário)
```

> ✅ Uma função variádica que **altera** seu parâmetro deve documentar isso, ou melhor, copiar antes (`nums = slices.Clone(nums)`). A regra geral: trate o parâmetro variádico como **somente leitura**.

---

## 🌀 4. `...any`: aceitando qualquer tipo

Com o tipo `any` (interface vazia, ver [[Interfaces]]), a função aceita argumentos de tipos diferentes. É assim que `fmt.Println` funciona:

```go
func Println(a ...any) (n int, err error)
```

Repassando os argumentos para outra função variádica (saída real):

```go
func log(nivel string, partes ...any) {
	fmt.Println("["+strings.ToUpper(nivel)+"]", fmt.Sprint(partes...))
}

log("info", "servidor na porta", 8080, true)
// [INFO] servidor na porta8080 true      ← repare: sem espaço antes do 8080!
```

> ⚠️ Esse resultado ilustra duas coisas: (1) é preciso repassar com `partes...` (sem os `...`, `Sprint` receberia **um** argumento, o slice inteiro); (2) `fmt.Sprint` só insere espaço entre dois operandos quando **nenhum** deles é string. `fmt.Sprintln` sempre insere espaços. Detalhes assim aparecem em `go doc fmt.Sprint`.

E a diferença entre espalhar ou não um `[]any` (saídas reais):

```go
vals := []any{1, "dois", 3.0}
fmt.Println(vals...)    // 1 dois 3     ← três argumentos
fmt.Println(vals)       // [1 dois 3]   ← um argumento: o slice
```

> ⚠️ `...any` abre mão da checagem de tipos do compilador. Se a função só faz sentido para um conjunto de tipos (números, por exemplo), prefira [[Generics]]: `func Somar[T int | float64](nums ...T) T`.

---

## 🧰 5. Variádicas na biblioteca padrão

| Função | Assinatura (resumida) | Uso |
| --- | --- | --- |
| `append` | `append(s []T, elems ...T) []T` | Adicionar elementos |
| `min`, `max` | embutidas, 1+ argumentos | Menor/maior valor |
| `fmt.Println` / `Printf` | `(format string, a ...any)` | Saída formatada |
| `fmt.Errorf` | `(format string, a ...any) error` | Criar erros com contexto |
| `errors.Join` | `(errs ...error) error` | Juntar vários erros |
| `slices.Concat` | `(slices ...S) S` | Concatenar slices |
| `path/filepath.Join` | `(elem ...string) string` | Montar caminhos |
| `exec.Command` | `(name string, arg ...string)` | Executar programas |
| `log/slog.Info` | `(msg string, args ...any)` | Log estruturado com pares chave-valor |

```go
cmd := exec.Command("git", "log", "--oneline", "-5")
caminho := filepath.Join("home", "matheus", "go", "src")
slog.Info("requisição", "método", r.Method, "rota", r.URL.Path, "ms", 12)
```

---

## 🧩 6. Opções funcionais: o padrão variádico mais famoso

Como Go não tem parâmetros opcionais, um padrão muito usado para configurar objetos é receber **opções como funções variádicas**:

```go
srv := NewServidor("localhost",
	WithPorta(9090),
	WithTimeout(5*time.Second),
)

func NewServidor(host string, opts ...Opcao) *Servidor {
	s := &Servidor{host: host, porta: 8080, timeout: 30 * time.Second}   // padrões
	for _, opt := range opts {
		opt(s)                     // cada opção altera um campo
	}
	return s
}
```

Cada `WithX` é uma função que **devolve uma função** (uma closure). O padrão completo é explicado em [[Closures e Funções Anônimas]].

---

## 🔬 7. Por dentro: como o compilador implementa

### 7.1 Valores avulsos viram um array temporário

```go
somar(1, 2, 3)

// o compilador transforma em algo como:
var tmp = [3]int{1, 2, 3}       // array temporário
somar(tmp[:])                   // passa um slice desse array
```

E `somar()` sem argumentos passa um slice `nil` (sem criar array).

### 7.2 Quase sempre sem alocação

Se a função não guarda o slice em lugar nenhum, o array temporário fica **na stack**. Saída real de `go build -gcflags=-m`:

```
./main.go:16:36: ... argument does not escape
./main.go:29:13: ... argument does not escape
```

Mas se a função guarda o slice (num campo, numa global, numa goroutine) ou o converte para interface, o compilador precisa colocá-lo na heap:

```
./main.go:19:56: fmt.Sprint(partes...) escapes to heap
```

Com `...any`, existe ainda o custo de colocar cada argumento numa interface (veremos em [[Interfaces]] por que isso às vezes aloca). Por isso, em código muito quente, loggers de alta performance evitam `...any` e usam funções tipadas (`slog.Int("ms", 12)`, `zap.String(...)`).

### 7.3 Comparação com C

C também tem variádicas (`printf`), mas sem saber o **tipo** nem a **quantidade** dos argumentos:

```c
int soma(int n, ...) {            // é preciso passar n "na mão"
    va_list args;
    va_start(args, n);
    int total = 0;
    for (int i = 0; i < n; i++)
        total += va_arg(args, int);   // se o chamador passou um double, lixo (comportamento indefinido)
    va_end(args);
    return total;
}
```

Em Go, os argumentos chegam como um slice **tipado**, com **tamanho** conhecido e **checagem de limites**. Os bugs clássicos de `printf` de C (formato que não casa com os argumentos, o que já causou falhas de segurança) são pegos pelo compilador (tipos) e pelo `go vet` (verbos do `fmt`).

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Esquecer `...` ao passar um slice | `cannot use s (type []int) as int` | `f(s...)` |
| Esquecer `...` ao repassar `...any` | A função recebe um único argumento (o slice) | `fmt.Sprint(partes...)` |
| Alterar o parâmetro variádico | Altera o slice do chamador quando foi passado com `s...` | Trate como somente leitura ou clone |
| Misturar valores e `s...` | Erro de compilação | Junte com `append` antes |
| `...any` onde caberia um tipo | Perde a checagem de tipos | Tipo concreto ou generics |
| Supor que `fmt.Sprint` separa tudo com espaço | Texto colado | Use `Sprintln` ou formate explicitamente |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Declaração | `func f(fixos T, varios ...U)`; só o último parâmetro |
| Dentro da função | O parâmetro é um `[]U` (nil se não houver argumentos) |
| Espalhar | `f(s...)` passa o **mesmo** slice (sem cópia) |
| Avulsos | `f(a, b, c)` cria um array temporário (geralmente na stack) |
| `...any` | Aceita qualquer tipo (como `fmt.Println`); perde segurança de tipos |
| Repassar | Sempre com `...` |
| Padrão famoso | Opções funcionais (`NewX(..., WithY(...))`) |
| vs C | Tipado, com tamanho e checagem de limites |

---

## 🔗 Conexões

**Dentro de Go**
- O slice por trás do parâmetro → [[Arrays e Slices]]
- `any` e o custo de interfaces → [[Interfaces]]
- Opções funcionais com closures → [[Closures e Funções Anônimas]]
- Variádicas genéricas → [[Generics]]
- `errors.Join` → [[Wrapping e Unwrapping de Erros]]
- Escape analysis do array temporário → [[Memória e Escape Analysis]]

**Sistemas Operacionais**
- [[A Linguagem C]] — `va_list`, `printf` e a falta de segurança de tipos nas variádicas de C
- [[Syscalls para Gerenciamento de Processos]] — `exec.Command(nome, args...)` monta o `argv` passado ao `execve`
- [[Interpretador de Comandos Shell]] — o shell também transforma uma linha em uma lista variável de argumentos para o programa

---

⬅️ **Anterior:** [[Múltiplos Retornos e Erros]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Closures e Funções Anônimas]] ➡️
