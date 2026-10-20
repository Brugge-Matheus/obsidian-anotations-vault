---
tags:
  - go
  - go/funcoes
aliases:
  - Closures
  - Funções anônimas
  - Function literals
  - Opções funcionais
  - Functional options
ordem: 19
modulo: "04 - Funções"
---
# Closures e Funções Anônimas

> **Nota 19 da trilha de Go.** Uma **função anônima** é uma função sem nome, escrita no meio do código. Quando ela usa variáveis do escopo em volta, vira uma **closure**: uma função que "carrega junto" as variáveis que capturou, que continuam vivas mesmo depois que a função de fora terminou. Closures são a base de vários padrões importantes de Go: geradores, memoização, middlewares HTTP, **opções funcionais**, callbacks e goroutines. Nesta nota veremos o conceito, a regra de **captura por referência**, a mudança da variável de laço no Go 1.22, os padrões práticos e como uma closure é representada na memória.
>
> **Pré-requisitos:** [[Funções — Fundamentos]], [[Sintaxe, Blocos e Escopo]].

---

## 🧭 De onde viemos

Em [[Funções — Fundamentos]] vimos que funções são valores, com tipo, que podem ser passadas e devolvidas. Em [[Sintaxe, Blocos e Escopo]] vimos que o escopo é **léxico**: um nome é visível dentro do bloco onde foi declarado e nos blocos internos. Closures são o encontro dessas duas ideias: uma função criada dentro de outra enxerga as variáveis de fora, e leva essas variáveis consigo.

---

## ✍️ 1. Funções anônimas (literais de função)

> 💡 **Função anônima (literal de função):** uma expressão que cria uma função sem nome no ponto onde é escrita. Tem a mesma sintaxe de uma declaração, só que sem o nome.

```go
// Guardar numa variável
dobrar := func(n int) int { return n * 2 }
fmt.Println(dobrar(21))                  // 42

// Chamar imediatamente (repare nos parênteses no final)
soma := func(a, b int) int { return a + b }(2, 3)
fmt.Println(soma)                        // 5

// Passar como argumento (o uso mais comum)
slices.SortFunc(pessoas, func(a, b Pessoa) int {
	return cmp.Compare(a.Idade, b.Idade)
})

// Disparar uma goroutine
go func() {
	processar()
}()

// Executar algo ao sair da função
defer func() {
	fmt.Println("limpando")
}()
```

---

## 🎒 2. Closures: funções que capturam variáveis

> 💡 **Closure (fechamento):** uma função junto com o **ambiente** de variáveis que ela referencia de fora do seu próprio corpo. A função "se fecha" sobre essas variáveis, que continuam existindo enquanto a closure existir.

O exemplo clássico, um contador (saída real):

```go
func contador() func() int {
	n := 0                     // variável LOCAL de contador()
	return func() int {        // a função anônima usa n → é uma closure
		n++
		return n
	}
}

c1 := contador()
c2 := contador()
fmt.Println(c1(), c1(), c1(), c2())   // 1 2 3 1
```

O que acontece:

```
contador() já terminou, mas n não morreu: a closure guarda uma referência a ele.

c1 ──▶ ┌─────────────────────────┐        ┌─────────┐
       │ código: func() { n++ }  │        │ n = 3   │   ← o n de c1
       │ ambiente: &n ───────────┼──────▶ └─────────┘
       └─────────────────────────┘

c2 ──▶ ┌─────────────────────────┐        ┌─────────┐
       │ código: (o mesmo)       │        │ n = 1   │   ← cada chamada de contador()
       │ ambiente: &n ───────────┼──────▶ └─────────┘     criou um n NOVO
       └─────────────────────────┘
```

Duas observações:

1. **Estado privado:** ninguém de fora consegue mexer em `n` a não ser chamando a closure. É uma forma de encapsulamento sem struct
2. **Cada chamada de `contador()` cria uma variável `n` nova**, por isso `c1` e `c2` são independentes

---

## 🔗 3. Captura por referência, não por valor

Uma closure em Go captura **a variável**, não uma cópia do valor que ela tinha no momento. Se a variável muda depois, a closure vê o valor novo (saída real):

```go
x := 10
mostrar := func() { fmt.Println("x =", x) }
x = 20
mostrar()          // x = 20   ← não 10!
```

E a closure também pode **alterar** a variável capturada:

```go
total := 0
somar := func(n int) { total += n }
somar(5)
somar(7)
fmt.Println(total)  // 12
```

> 💡 Comparação com outras linguagens: JavaScript e Python também capturam por referência. PHP captura **por valor** por padrão (`function() use ($x)`) e só por referência com `use (&$x)`. C++ deixa escolher (`[x]` ou `[&x]`). Em Go é sempre por referência; para "congelar" um valor, crie uma variável nova: `v := x; f := func() { use(v) }`.

---

## 🔁 4. Closures em laços: a mudança do Go 1.22

Como a captura é por referência, importa **quantas variáveis** um laço cria. Até o Go 1.21 o `for` reutilizava **uma única variável** em todas as voltas, e todas as closures capturavam essa mesma variável. Desde o **Go 1.22**, cada volta cria uma **variável nova** (saída real):

```go
var fs []func() int
for i := range 3 {
	fs = append(fs, func() int { return i * i })
}
for _, f := range fs {
	fmt.Print(f(), " ")
}
// Go 1.22+:  0 1 4
// Go ≤1.21:  4 4 4   ← as 3 closures viam a mesma variável i, que terminou valendo 2
```

O mesmo acontecia com goroutines lançadas no laço, que era o caso mais perigoso:

```go
for _, url := range urls {
	go func() {
		baixar(url)      // Go ≤1.21: várias goroutines baixavam a ÚLTIMA url
	}()                  // Go 1.22+: cada goroutine tem sua url
}
```

Em código antigo você verá os "remendos" da época, hoje desnecessários (e o `go fix` remove):

```go
for _, url := range urls {
	url := url                    // cópia por volta (antigo)
	go func(u string) { ... }(url) // ou passar como argumento (antigo)
}
```

Veremos detalhes da semântica do laço em [[Controle de Fluxo]].

---

## 🧪 5. Padrões práticos

### 5.1 Gerador (estado entre chamadas)

```go
func fibonacci() func() int {
	a, b := 0, 1
	return func() int {
		r := a
		a, b = b, a+b
		return r
	}
}

f := fibonacci()
for range 8 {
	fmt.Print(f(), " ")    // 0 1 1 2 3 5 8 13
}
```

> 💡 Desde o Go 1.23, para sequências o idioma moderno é um **iterador** (`iter.Seq`), consumido com `range`. Iteradores também são closures por dentro. Ver [[Iteradores]].

### 5.2 Memoização

Uma função que recebe outra e devolve uma versão com cache (saída real):

```go
func memoizar(f func(int) int) func(int) int {
	cache := map[int]int{}            // capturado pela closure: sobrevive entre chamadas
	return func(n int) int {
		if v, ok := cache[n]; ok {
			return v
		}
		v := f(n)
		cache[n] = v
		return v
	}
}

chamadas := 0
lento := func(n int) int { chamadas++; return n * n }
rapido := memoizar(lento)
rapido(4); rapido(4); rapido(5)
fmt.Println("chamadas reais:", chamadas)   // chamadas reais: 2
```

> ⚠️ Esse cache não é seguro para uso por várias goroutines (map sem proteção, ver [[Maps]]). Para isso, `sync.Mutex` ou `sync.OnceValue`.

### 5.3 Middleware / decorator

Envolver uma função com comportamento extra, sem alterá-la (saída real):

```go
type Handler func(string) string

func comLog(next Handler) Handler {
	return func(req string) string {
		fmt.Println("→", req)
		resp := next(req)        // chama a função original (capturada)
		fmt.Println("←", resp)
		return resp
	}
}

h := comLog(func(r string) string { return strings.ToUpper(r) })
h("ping")
// → ping
// ← PING
```

É exatamente assim que se escrevem middlewares HTTP em Go (log, autenticação, métricas, recuperação de panic):

```go
func comTempo(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		inicio := time.Now()
		next.ServeHTTP(w, r)
		slog.Info("req", "rota", r.URL.Path, "duração", time.Since(inicio))
	})
}
```

Veremos em [[HTTP (net-http)]]. É o mesmo conceito que você viu em [[API Gateway]] e [[Proxy x Proxy Reverso]]: uma camada que intercepta a requisição antes e depois de repassá-la.

### 5.4 Opções funcionais (functional options)

O padrão mais elegante para construtores com muitas configurações opcionais, popularizado por Rob Pike e Dave Cheney. Lembra que Go não tem parâmetros opcionais nem valores padrão? (saída real):

```go
type Servidor struct {
	host    string
	porta   int
	timeout time.Duration
}

type Opcao func(*Servidor)       // uma opção é uma função que altera o Servidor

func WithPorta(p int) Opcao {
	return func(s *Servidor) { s.porta = p }        // closure: captura p
}

func WithTimeout(d time.Duration) Opcao {
	return func(s *Servidor) { s.timeout = d }      // closure: captura d
}

func NewServidor(host string, opts ...Opcao) *Servidor {
	s := &Servidor{host: host, porta: 8080, timeout: 30 * time.Second}   // valores padrão
	for _, o := range opts {
		o(s)                                         // aplica cada opção
	}
	return s
}

s := NewServidor("localhost", WithPorta(9090))
fmt.Printf("%+v\n", *s)
// {host:localhost porta:9090 timeout:30000000000}
```

Por que é bom:

- A chamada é **legível** e **autoexplicativa** (`WithPorta(9090)` em vez de um `9090` solto)
- Adicionar uma opção nova **não quebra** quem já usa o construtor
- Os padrões ficam num lugar só
- Uma opção pode validar, devolver erro (`type Opcao func(*Servidor) error`) ou combinar outras

Bibliotecas famosas usam: gRPC (`grpc.WithTransportCredentials`), Zap, o cliente do MongoDB.

### 5.5 Inicialização preguiçosa com `sync.OnceValue`

```go
var config = sync.OnceValue(func() *Config {   // Go 1.21+
	return carregarConfig()                     // roda UMA vez, na primeira chamada
})

cfg := config()     // seguro com várias goroutines
```

Veremos em [[sync.WaitGroup e sync.Mutex]].

---

## 🔬 6. Por dentro: como uma closure é representada

### 6.1 Um valor de função é um ponteiro para um "funcval"

Todo valor de função em Go é um ponteiro para uma pequena estrutura cujo **primeiro campo** é o endereço do código. Numa closure, depois dele vêm as **variáveis capturadas** (ou ponteiros para elas):

```
c1 := contador()

c1 ─▶ ┌──────────────────────────────┐
      │ fn: main.contador.func1      │  ← endereço do código (o compilador dá esse nome)
      ├──────────────────────────────┤
      │ &n ──────────────┐           │  ← variável capturada (por referência)
      └──────────────────┼───────────┘
                         ▼
                     ┌───────┐
                     │ n = 0 │   ← "moved to heap": precisa sobreviver a contador()
                     └───────┘
```

Assembly real da chamada `c()` e do corpo da closure:

```
; chamada de c() em main:
MOVQ  (AX), CX              ; CX = primeiro campo do funcval: o endereço do código
MOVQ  AX, DX                ; DX = o próprio funcval (o "contexto" da closure)
CALL  CX                    ; chamada indireta

; main.contador.func1  (marcada NEEDCTXT: precisa de contexto)
                            ; pega o ponteiro &n a partir de DX
MOVQ  (CX), AX              ; AX = n
INCQ  AX                    ; n++
MOVQ  AX, (CX)              ; grava n
RET                         ; devolve n em AX
```

> 💡 Em amd64, o registrador `DX` é o **registrador de contexto**: toda chamada por valor de função coloca ali o ponteiro para o funcval, e a closure usa esse ponteiro para achar suas variáveis capturadas. Uma função comum (sem captura) também é chamada assim, só que simplesmente ignora o `DX`.

### 6.2 Capturar faz a variável escapar (às vezes)

Saída real de `go build -gcflags=-m`:

```
./main.go:10:2: moved to heap: n
./main.go:11:9: func literal escapes to heap
```

Se a closure **sobrevive** à função que a criou (é devolvida, guardada, passada para uma goroutine), as variáveis capturadas vão para a **heap**. Se a closure é usada só localmente (por exemplo, num `slices.SortFunc` dentro da mesma função), tudo pode ficar na stack e ser até inlinado.

Uma otimização: se a variável capturada **nunca é alterada** depois da captura e é pequena, o compilador pode copiar o **valor** para dentro do funcval em vez de um ponteiro, já que o efeito observável é o mesmo.

### 6.3 Desde o Go 1.27: nomes estáveis

O compilador dá nomes às funções anônimas (`main.contador.func1`), que aparecem em stack traces e perfis. No Go 1.27 esses nomes ficaram mais simples e estáveis mesmo com inlining, e literais idênticos podem compartilhar o mesmo código.

---

## ⚠️ 7. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Esperar captura por valor | Closure vê o valor mais recente | Copie para uma variável nova antes de capturar |
| Código antigo (go ≤ 1.21) com closures/goroutines em laço | Todas veem o último valor | Atualize a linha `go` do `go.mod` |
| Closure compartilhada entre goroutines alterando variável capturada | Condição de corrida | Mutex, atomic ou channels |
| Closure que captura objeto grande e vive muito | Memória retida (o GC não libera o capturado) | Capture só o necessário |
| `defer` com closure em laço longo | Defers se acumulam até a função terminar | Extraia o corpo do laço para uma função |
| Chamar a closure sem `()` | `go func() {...}` sem `()` não compila | `go func() { ... }()` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Função anônima | `func(params) result { ... }` como expressão |
| Closure | Função + variáveis capturadas do escopo externo |
| Captura | **Por referência**: vê e altera a variável original |
| Ciclo de vida | Variáveis capturadas vivem enquanto a closure existir |
| Laços | Go 1.22+: uma variável nova por iteração |
| Padrões | Gerador, memoização, middleware, opções funcionais, `sync.OnceValue` |
| Representação | Ponteiro para funcval {código, capturas}; contexto no registrador `DX` |
| Custo | Capturas que sobrevivem vão para a heap |

---

## 🔗 Conexões

**Dentro de Go**
- Funções como valores → [[Funções — Fundamentos]]
- Variável de laço por iteração → [[Controle de Fluxo]]
- Closures com `defer` e `recover` → [[Defer, Panic e Recover]]
- Closures em goroutines → [[Goroutines]]
- Iteradores (`iter.Seq`) são closures → [[Iteradores]]
- Middlewares HTTP → [[HTTP (net-http)]]
- `sync.Once`, `sync.OnceValue` → [[sync.WaitGroup e sync.Mutex]]
- Opções variádicas → [[Funções Variádicas]]
- Escape para a heap → [[Memória e Escape Analysis]]

**Sistemas Operacionais**
- [[Race Condition]] — closures executadas por várias goroutines compartilham as variáveis capturadas
- [[Processadores]] — chamadas indiretas e o registrador de contexto

**Nginx**
- [[API Gateway]] e [[Proxy x Proxy Reverso]] — o conceito de middleware (interceptar antes/depois e repassar) é o mesmo de uma cadeia de closures

---

⬅️ **Anterior:** [[Funções Variádicas]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Defer, Panic e Recover]] ➡️
