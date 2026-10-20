---
tags:
  - go
  - go/metodos-interfaces
aliases:
  - Métodos em Go
  - Receptores
  - Receiver
  - Method set
  - Pointer receiver
ordem: 21
modulo: "05 - Métodos, Interfaces e Generics"
---
# Métodos e Receptores

> **Nota 21 da trilha de Go.** Abrimos o módulo de "orientação a objetos à moda Go". Um **método** é uma função ligada a um tipo através de um **receptor**: `func (c *Conta) Depositar(v float64)`. Com métodos, tipos passam a ter **comportamento**, e é isso que permitirá, na próxima nota, que satisfaçam **interfaces**. Veremos a diferença crucial entre **receptor por valor** e **por ponteiro**, o **method set** de cada tipo, métodos em tipos que não são structs, receptores `nil`, method values e expressions, e como o compilador transforma um método numa função comum.
>
> **Pré-requisitos:** [[Structs]], [[Ponteiros]], [[Funções — Fundamentos]].

---

## 🧭 De onde viemos

Em [[Structs]] modelamos **dados** (`Conta` com `Titular` e `saldo`). Em [[Tipos de Dados]] criamos tipos definidos como `type Celsius float64` e dissemos que eles "podem ganhar métodos". Em [[Variáveis e constantes]] escrevemos `func (n Nivel) String() string` sem explicar a sintaxe. Agora vamos explicar: como associar **funções** a **tipos**.

> 💡 **Método:** uma função com um parâmetro especial, o **receptor**, escrito antes do nome. O método "pertence" ao tipo do receptor e é chamado com a sintaxe `valor.Metodo()`.

---

## ✍️ 1. Declarando métodos

```go
type Conta struct {
	Titular string
	saldo   float64
}

//    receptor
//   ┌───────┐
func (c Conta) Saldo() float64 {
	return c.saldo
}

func (c *Conta) Depositar(v float64) {
	c.saldo += v
}
```

```go
c := Conta{Titular: "Ana"}
c.Depositar(100)           // chamada com a sintaxe de ponto
fmt.Println(c.Saldo())     // 100
```

Diferenças em relação a classes de Java/PHP/Ruby:

| Classes | Go |
| --- | --- |
| Métodos declarados **dentro** da classe | Métodos declarados **fora** da struct, em qualquer arquivo **do mesmo pacote** |
| `this`/`self` implícito | Receptor **explícito**, com o nome que você escolher |
| Só classes têm métodos | **Qualquer tipo definido** no pacote pode ter métodos |
| Herança de métodos | Não há herança (há embedding, ver [[Composição vs Herança]]) |

Restrições:

```go
func (d time.Duration) Dobro() time.Duration { return d * 2 }
// ❌ cannot define new methods on non-local type time.Duration
```

- Só se pode declarar métodos para tipos **definidos no mesmo pacote**. Para "estender" um tipo de outro pacote, crie um tipo seu: `type MinhaDuracao time.Duration`
- O tipo base não pode ser ponteiro nem interface (`func (p *int)` não é válido)
- Dentro de um tipo, os nomes de métodos e campos não podem se repetir

### 1.1 Convenção do nome do receptor

```go
func (c *Conta) Depositar(...)     // ✅ 1–2 letras, derivadas do tipo
func (srv *Servidor) Iniciar()     // ✅ abreviação curta
func (this *Conta) Depositar(...)  // ❌ não idiomático
func (self *Conta) Depositar(...)  // ❌ não idiomático
```

Use **o mesmo nome** em todos os métodos do tipo.

---

## 📨 2. Receptor por valor vs por ponteiro

Esta é a decisão mais importante ao escrever métodos. Lembre: **Go sempre passa por valor** ([[Ponteiros]]), e o receptor é só mais um parâmetro.

### 2.1 Receptor por valor recebe uma CÓPIA

Saída real:

```go
func (c Conta) DepositarErrado(v float64) { c.saldo += v }   // altera a cópia

c := Conta{Titular: "Ana"}
c.Depositar(100)          // receptor *Conta: altera o original
c.DepositarErrado(50)     // receptor Conta: altera uma cópia, que é descartada
fmt.Println(c.Saldo())    // 100   ← os 50 sumiram
```

```
c.DepositarErrado(50)                    c.Depositar(100)
┌─────────────┐   cópia   ┌───────────┐  ┌─────────────┐        ┌──────────────┐
│ c: saldo 0  │ ────────▶ │ saldo 50  │  │ c: saldo 100│ ◀──────┤ c *Conta     │
└─────────────┘           └───────────┘  └─────────────┘        │ (endereço)   │
   intocado          descartada no fim       alterado           └──────────────┘
```

### 2.2 Quando usar cada um

| Use receptor **ponteiro** `(c *T)` quando... | Use receptor **valor** `(c T)` quando... |
| --- | --- |
| O método **altera** o receptor | O método só **lê** e o tipo é pequeno |
| O tipo é **grande** (copiar seria caro) | O tipo é "valor por natureza": `time.Time`, `Ponto`, `Celsius` |
| O tipo contém `sync.Mutex` ou outro campo que não pode ser copiado | Você quer imutabilidade (cada chamada trabalha numa cópia) |
| Outros métodos do tipo já usam ponteiro (**consistência**) | O tipo é um map, func ou channel (já contém ponteiro) |

> ✅ **Regra prática:** na dúvida, use **ponteiro**. E não misture: se algum método do tipo precisa de ponteiro, faça **todos** com ponteiro. A mistura confunde e afeta quais interfaces o tipo satisfaz (seção 3).

### 2.3 Go faz a conversão `&` / `*` automaticamente na chamada

```go
c := Conta{}
c.Depositar(10)      // Depositar tem receptor *Conta, mas c é Conta:
                     // Go transforma em (&c).Depositar(10)

p := &Conta{}
p.Saldo()            // Saldo tem receptor Conta, mas p é *Conta:
                     // Go transforma em (*p).Saldo()
```

Isso só funciona se o valor for **endereçável** (tiver um lugar fixo na memória). Literais e valores dentro de maps não são (mensagens reais):

```go
Conta{}.Depositar(1)
// ❌ cannot call pointer method Depositar on Conta

m := map[string]Conta{}
m["a"].Depositar(1)
// ❌ cannot call pointer method Depositar on Conta
```

Lembra que valores de map não são endereçáveis ([[Maps]])? É o mesmo motivo.

---

## 🧺 3. Method set: quais métodos um tipo "tem"

> 💡 **Method set (conjunto de métodos):** o conjunto de métodos que podem ser chamados num valor de um tipo, **sem a ajuda do `&` automático**. É ele que decide se um tipo satisfaz uma interface.

| Tipo | Method set |
| --- | --- |
| `T` | Métodos com receptor `T` |
| `*T` | Métodos com receptor `T` **e** com receptor `*T` |

```
type Conta struct{...}
func (c Conta)  Saldo()     ← receptor valor
func (c *Conta) Depositar() ← receptor ponteiro

method set de Conta:    { Saldo }
method set de *Conta:   { Saldo, Depositar }
```

Por que essa assimetria? Um `*Conta` sempre consegue obter o valor (`*p`) para chamar `Saldo`. Mas um `Conta` guardado, por exemplo, dentro de uma interface é uma **cópia sem endereço próprio**: chamar `Depositar` nela alteraria uma cópia invisível, o que seria um bug silencioso. Go prefere proibir.

A consequência aparece com interfaces (mensagem real):

```go
type Depositor interface{ Depositar(float64) }

var d Depositor = Conta{}
// ❌ Conta does not implement Depositor (method Depositar has pointer receiver)

var d Depositor = &Conta{}     // ✅ *Conta tem Depositar no seu method set
```

Esse é **o** erro clássico de quem começa com interfaces. Veremos mais em [[Interfaces]].

---

## 🎨 4. Métodos em tipos que não são structs

Qualquer tipo definido pode ter métodos. Saídas reais:

```go
type Celsius float64

func (t Celsius) Fahrenheit() float64 { return float64(t)*9/5 + 32 }

fmt.Println(Celsius(100).Fahrenheit())    // 212
```

```go
type Lista []string

func (l Lista) Juntar() string { return strings.Join(l, ", ") }
func (l *Lista) Add(s string)  { *l = append(*l, s) }   // ponteiro: o append muda o cabeçalho

var l Lista
l.Add("go")
l.Add("rust")
fmt.Println(l.Juntar())                   // go, rust
```

Repare no `Add`: como o `append` pode criar um cabeçalho novo, o método precisa de receptor **ponteiro** para atualizar a variável do chamador (a lição de [[Arrays e Slices]]).

Exemplos famosos na biblioteca padrão:

```go
type Duration int64                  // time.Duration
func (d Duration) Seconds() float64
func (d Duration) String() string    // "1h30m0s"

type HandlerFunc func(ResponseWriter, *Request)   // http.HandlerFunc: um tipo FUNÇÃO com método!
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }

type IP []byte                       // net.IP
func (ip IP) String() string
```

O `http.HandlerFunc` merece atenção: é um tipo **função** que tem um método. Ele permite que uma função comum satisfaça a interface `http.Handler`. Veremos em [[HTTP (net-http)]].

---

## 🚫 5. Receptores `nil`

Um método com receptor ponteiro **pode ser chamado com um ponteiro nil**: o método roda normalmente e recebe `nil`. Só dá panic se o método tentar acessar um campo. Isso permite métodos que tratam o "vazio" com elegância (saídas reais):

```go
type No struct {
	Valor int
	Prox  *No
}

func (n *No) Soma() int {
	if n == nil {          // lista vazia / fim da lista
		return 0
	}
	return n.Valor + n.Prox.Soma()   // chama Soma em Prox mesmo quando Prox é nil
}

lista := &No{1, &No{2, &No{3, nil}}}
fmt.Println(lista.Soma())    // 6

var vazia *No
fmt.Println(vazia.Soma())    // 0   ← sem panic
```

Isso é diferente de Java, onde `null.metodo()` sempre lança `NullPointerException`. A biblioteca padrão usa esse recurso em alguns lugares (por exemplo, métodos de `*bytes.Buffer` e de árvores internas).

---

## 🧷 6. Method values e method expressions

### 6.1 Method value: método "preso" a um valor

`valor.Metodo` (sem parênteses) gera uma **função** com o receptor já fixado (saída real):

```go
dep := c.Depositar       // função func(float64), presa a &c
dep(10)
fmt.Println(c.Saldo())   // 110   ← alterou o c original
```

Muito usado para passar métodos como callbacks:

```go
http.HandleFunc("/saude", srv.handleSaude)     // srv.handleSaude é um method value
time.AfterFunc(5*time.Second, conexao.Fechar)
```

> ⚠️ Para receptor **valor**, o method value **copia** o receptor no momento em que é criado: `f := c.Saldo` congela o saldo daquele instante.

### 6.2 Method expression: método como função comum

`Tipo.Metodo` gera uma função cujo **primeiro parâmetro é o receptor** (saídas reais):

```go
f := (*Conta).Depositar      // func(*Conta, float64)
f(&c, 5)

g := Conta.Saldo             // func(Conta) float64
fmt.Println(g(c))            // 115

fmt.Printf("%T | %T\n", f, g)
// func(*main.Conta, float64) | func(main.Conta) float64
```

Isso revela a verdade: **um método é só uma função cujo primeiro argumento é o receptor**.

---

## 🖨️ 7. Dois métodos especiais: `String()` e `Error()`

Alguns nomes de método têm significado para a biblioteca padrão, porque correspondem a **interfaces** conhecidas:

```go
type Ponto struct{ X, Y int }

func (p Ponto) String() string {              // interface fmt.Stringer
	return fmt.Sprintf("(%d, %d)", p.X, p.Y)
}

fmt.Println(Ponto{1, 2})    // (1, 2)   ← o fmt chama String() automaticamente
```

```go
type ErroValidacao struct{ Campo string }

func (e *ErroValidacao) Error() string {       // interface error
	return "campo inválido: " + e.Campo
}
```

> ⚠️ **Recursão infinita no `String()`:** usar `%v` ou `%s` do próprio receptor dentro de `String()` chama `String()` de novo, até estourar a stack:
> ```go
> func (p Ponto) String() string { return fmt.Sprintf("%v", p) }   // 💥 stack overflow
> ```
> Formate os **campos** (`p.X`, `p.Y`), ou converta para um tipo sem o método.

Veremos essas interfaces em [[Interfaces]] e [[O Padrão error em Go]].

---

## 🔬 8. Por dentro: método é função com um argumento a mais

O compilador transforma cada método numa função comum, cujo nome inclui o tipo:

```
func (c *Conta) Depositar(v float64)   →   função  main.(*Conta).Depositar(c *Conta, v float64)
func (c Conta) Saldo() float64         →   função  main.Conta.Saldo(c Conta) float64

c.Depositar(100)                       →   main.(*Conta).Depositar(&c, 100)
```

São esses nomes que aparecem em stack traces e perfis (`main.(*Conta).Depositar`). Na ABI de registradores ([[Funções — Fundamentos]]):

- Receptor **ponteiro**: o endereço vai em `AX`, `v` em `X0` (float)
- Receptor **valor**: os **campos** da struct são espalhados pelos registradores (ou copiados para a stack, se a struct for grande)

Chamada direta de método (`c.Depositar(...)`) é tão rápida quanto chamada de função, e pode ser inlinada. O custo extra só aparece quando o método é chamado **através de uma interface** (despacho dinâmico, sem inlining), como veremos na próxima nota.

### 8.1 Por que ponteiro não é "sempre mais rápido"

- Receptor valor pequeno: os campos vão em registradores; nada escapa
- Receptor ponteiro: se o valor estiver na stack, tudo bem; mas guardar o valor numa interface ou passar seu endereço adiante pode fazê-lo **escapar para a heap**

A escolha entre `T` e `*T` deve ser guiada primeiro pela **semântica** (o método altera? o tipo pode ser copiado?), e só depois por medição (ver [[Benchmarks]]).

### 8.2 Comparação com C

Em C, a "orientação a objetos" é feita exatamente assim, à mão:

```c
struct conta { double saldo; };
void conta_depositar(struct conta *c, double v) { c->saldo += v; }   // "método" = função + ponteiro
conta_depositar(&c, 100);
```

Go só adiciona a sintaxe `c.Depositar(100)`, a ligação ao tipo e os method sets. O kernel Linux é cheio desse padrão (funções que recebem `struct file *`, `struct task_struct *`).

---

## ⚠️ 9. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Receptor valor num método que altera | Alterações perdidas | Receptor ponteiro |
| Misturar receptores valor e ponteiro | Confusão sobre method set e interfaces | Seja consistente (geralmente ponteiro) |
| `var i Iface = T{}` com métodos de ponteiro | `does not implement (method has pointer receiver)` | `&T{}` |
| Chamar método de ponteiro em valor de map ou literal | `cannot call pointer method` | Variável intermediária ou map de ponteiros |
| Método em tipo de outro pacote | `cannot define new methods on non-local type` | Defina um tipo próprio |
| `String()` usando `%v` do próprio receptor | Stack overflow | Formate os campos |
| Copiar struct com mutex via receptor valor | O `go vet` avisa (`copylocks`) | Receptor ponteiro |
| Method value com receptor valor | Captura uma cópia antiga | Use ponteiro ou chame direto |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Método | Função com receptor: `func (r T) Nome()` |
| Onde | Qualquer arquivo do mesmo pacote do tipo |
| Que tipos | Qualquer tipo definido no pacote (struct, slice, func, número...) |
| Receptor valor | Recebe cópia; não altera o original |
| Receptor ponteiro | Altera o original; evita cópia |
| Chamada | `&`/`*` automáticos se o valor for endereçável |
| Method set | `T`: só métodos de valor; `*T`: todos |
| Receptor nil | Permitido para receptor ponteiro |
| Method value | `v.M` → função presa ao receptor |
| Method expression | `T.M` → função com o receptor como 1º parâmetro |
| Especiais | `String()` (fmt.Stringer), `Error()` (error) |
| Por dentro | `main.(*Conta).Depositar(c, v)`: função comum |

---

## 🔗 Conexões

**Dentro de Go**
- Métodos são a base das interfaces → [[Interfaces]]
- Métodos promovidos por embedding → [[Composição vs Herança]]
- Métodos com parâmetros de tipo (Go 1.27) → [[Generics]]
- `Error()` e erros customizados → [[Criando Erros Customizados]]
- `ServeHTTP` e `http.HandlerFunc` → [[HTTP (net-http)]]
- Receptores e mutex → [[sync.WaitGroup e sync.Mutex]]
- Endereçabilidade em maps → [[Maps]]

**Sistemas Operacionais**
- [[A Linguagem C]] — "métodos" em C são funções que recebem um ponteiro para a struct, exatamente o que o compilador de Go gera
- [[Implementação de Processos]] — o kernel opera sobre structs (o PCB) passando ponteiros para funções, o mesmo padrão de receptor ponteiro
- [[Sistemas Monolíticos]] — a organização de um núcleo em procedimentos que operam sobre estruturas compartilhadas

**Algoritmos e Estruturas de Dados**
- [[Linked Lists]] e [[Árvores Binárias]] — métodos com receptor nil tratam o caso base (lista vazia, folha) com elegância

---

⬅️ **Anterior:** [[Defer, Panic e Recover]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Interfaces]] ➡️
