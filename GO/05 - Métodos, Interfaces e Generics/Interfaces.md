---
tags:
  - go
  - go/metodos-interfaces
aliases:
  - Interfaces em Go
  - Interface
  - Polimorfismo em Go
  - Duck typing estrutural
  - itab
ordem: 22
modulo: "05 - Métodos, Interfaces e Generics"
---
# Interfaces

> **Nota 22 da trilha de Go.** Uma **interface** descreve um **comportamento**: um conjunto de métodos. Qualquer tipo que tenha esses métodos **satisfaz** a interface automaticamente, sem declarar nada (`implements` não existe em Go). É assim que Go faz **polimorfismo**: funções que aceitam "qualquer coisa que saiba fazer X". Nesta nota veremos declaração e satisfação implícita, polimorfismo, interfaces da biblioteca padrão (`io.Reader`, `io.Writer`, `fmt.Stringer`, `error`), composição de interfaces, a interface vazia `any`, a famosa **armadilha do nil**, boas práticas de design e, por dentro, como uma interface é representada (o par **tipo + dados** e a **itab**).
>
> **Pré-requisitos:** [[Métodos e Receptores]].

---

## 🧭 De onde viemos

Em [[Métodos e Receptores]] demos comportamento aos tipos e vimos o **method set** de `T` e `*T`. Também esbarramos em interfaces várias vezes: o `error` dos retornos, o `fmt.Stringer` do `String()`, o `any` do `fmt.Println(...any)`, a armadilha do nil em [[Operadores de Comparação e Lógicos]]. Agora vamos entender o mecanismo inteiro.

> 💡 **Interface:** um tipo que especifica um conjunto de assinaturas de métodos. Uma variável de tipo interface pode guardar **qualquer valor** cujo tipo implemente esses métodos.

---

## ✍️ 1. Declarando e satisfazendo

```go
type Forma interface {
	Area() float64
	Perimetro() float64
}

type Retangulo struct{ L, A float64 }
func (r Retangulo) Area() float64      { return r.L * r.A }
func (r Retangulo) Perimetro() float64 { return 2 * (r.L + r.A) }

type Circulo struct{ R float64 }
func (c Circulo) Area() float64        { return math.Pi * c.R * c.R }
func (c Circulo) Perimetro() float64   { return 2 * math.Pi * c.R }
```

Repare: `Retangulo` e `Circulo` **não mencionam** `Forma` em lugar nenhum. Eles satisfazem `Forma` simplesmente porque têm os dois métodos.

> 💡 **Satisfação implícita (tipagem estrutural):** um tipo implementa uma interface se o seu **method set** contém todos os métodos da interface, com as mesmas assinaturas. Não há declaração explícita. É parecido com o *duck typing* de Python e Ruby ("se anda como pato e grasna como pato...") com uma diferença crucial: é verificado pelo **compilador**, não em tempo de execução.

### 1.1 Polimorfismo

Saída real:

```go
formas := []Forma{Retangulo{3, 4}, Circulo{1}}
total := 0.0
for _, f := range formas {
	fmt.Printf("%T área=%.2f\n", f, f.Area())    // a chamada certa é escolhida em tempo de execução
	total += f.Area()
}
fmt.Printf("total=%.2f\n", total)
// main.Retangulo área=12.00
// main.Circulo área=3.14
// total=15.14
```

O laço não sabe (nem quer saber) quais formas existem. Amanhã alguém cria `Triangulo` com os dois métodos, e o código acima funciona sem mudança.

> 💡 **Polimorfismo:** a capacidade de um mesmo código trabalhar com valores de tipos diferentes, através de uma interface comum. A decisão de qual método concreto chamar é tomada em tempo de execução (**despacho dinâmico**).

### 1.2 Por que satisfação implícita é poderosa

Em Java, uma classe precisa declarar `implements Forma`, o que significa que **quem escreve o tipo** precisa conhecer a interface **antes**. Em Go:

- Você pode criar uma interface **depois**, para tipos que já existem, inclusive de **outros pacotes**, sem tocá-los
- Os pacotes ficam **desacoplados**: quem define o tipo não importa quem define a interface
- O consumidor define exatamente o comportamento que precisa

```go
// Seu pacote precisa só de algo que "saiba ler"
type Leitor interface{ Read(p []byte) (int, error) }

// *os.File, *bytes.Buffer, *strings.Reader, net.Conn, *gzip.Reader...
// TODOS já satisfazem Leitor, sem que seus autores saibam que ele existe.
```

### 1.3 Checagem em tempo de compilação

Como não há `implements`, um idioma garante que um tipo satisfaz uma interface (o compilador avisa se não satisfizer):

```go
var _ Forma = Retangulo{}          // valor
var _ Forma = (*Circulo)(nil)      // ponteiro, sem alocar nada
var _ http.Handler = (*Servidor)(nil)
```

---

## 🧺 2. Method set e interfaces: valor ou ponteiro?

A regra de [[Métodos e Receptores]] decide quem satisfaz:

```go
type Depositor interface{ Depositar(float64) }
func (c *Conta) Depositar(v float64) { ... }    // receptor ponteiro

var d Depositor = Conta{}     // ❌ Conta does not implement Depositor (method Depositar has pointer receiver)
var d Depositor = &Conta{}    // ✅
```

| Métodos declarados com | `T` satisfaz? | `*T` satisfaz? |
| --- | --- | --- |
| Receptor valor `(t T)` | ✅ | ✅ |
| Receptor ponteiro `(t *T)` | ❌ | ✅ |

---

## 📚 3. Interfaces essenciais da biblioteca padrão

A biblioteca padrão é construída em cima de interfaces **minúsculas**:

```go
// io — a base de todo I/O de Go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type Closer interface { Close() error }

// fmt
type Stringer interface { String() string }

// pré-declarada
type error interface { Error() string }

// sort
type Interface interface { Len() int; Less(i, j int) bool; Swap(i, j int) }

// net/http
type Handler interface { ServeHTTP(ResponseWriter, *Request) }
```

O poder disso: qualquer tipo seu que implemente `Write` funciona com **todo** o ecossistema (saída real):

```go
type contador struct{ n int }

func (c *contador) Write(p []byte) (int, error) {   // satisfaz io.Writer
	c.n += len(p)
	return len(p), nil
}

c := &contador{}
io.Copy(c, strings.NewReader("olá, mundo"))   // io.Copy: Reader → Writer
fmt.Fprintf(c, "%d", 12345)                    // fmt escreve em qualquer Writer
fmt.Println(c.n)                               // 16 (11 bytes + 5)
```

`fmt.Fprintf`, `io.Copy`, `json.NewEncoder`, `gzip.NewWriter`, `bufio.NewWriter`... todos aceitam **qualquer** `io.Writer`: um arquivo, uma conexão de rede, um buffer em memória, o seu contador. Veremos esse universo em [[io.Reader e io.Writer]].

> 💡 Isso é a filosofia UNIX ("tudo é arquivo") traduzida em tipos. No UNIX, `read`/`write` funcionam igual para arquivos, pipes, sockets e dispositivos, porque todos são descritores de arquivo (ver [[Arquivos]] e [[Dispositivos de IO]]). Em Go, `Read`/`Write` funcionam igual para qualquer tipo que implemente as interfaces.

---

## 🧱 4. Composição de interfaces

Interfaces podem **embutir** outras (o method set resultante é a união):

```go
type ReadWriter interface {
	Reader
	Writer
}

type ReadWriteCloser interface {
	Reader
	Writer
	Closer
}
```

O pacote `io` define exatamente essas combinações. A ideia é construir interfaces grandes a partir de pequenas, e não o contrário.

---

## 🌀 5. A interface vazia: `any`

```go
type any = interface{}     // alias pré-declarado desde o Go 1.18
```

Uma interface **sem métodos** é satisfeita por **todos** os tipos. Uma variável `any` aceita qualquer valor (saída real):

```go
var a any = 42
fmt.Printf("%v %T\n", a, a)    // 42 int
a = "texto"
a = []int{1, 2}
```

Para usar o valor de volta como o tipo concreto, é preciso uma **type assertion** (`a.(int)`) ou um **type switch**, assunto de [[Type Assertion e Type Switch]].

Onde `any` aparece legitimamente:

- `fmt.Println(a ...any)`, `fmt.Sprintf`
- `json.Unmarshal` para JSON de formato desconhecido (`map[string]any`)
- Containers genéricos **antes** dos generics (hoje, prefira [[Generics]])

> ⚠️ *"interface{} says nothing"* (Rob Pike). Um parâmetro `any` não diz nada sobre o que a função espera, e toda checagem vira tempo de execução. Use só quando a função realmente aceita qualquer coisa.

---

## 🕳️ 6. A armadilha do nil (a mais famosa de Go)

Uma interface é nil **somente quando não guarda nem tipo nem valor**. Uma interface que guarda um **ponteiro nil de algum tipo** **não é nil**. Saídas reais:

```go
var f Forma
fmt.Println(f == nil)        // true:  sem tipo, sem valor

f = Retangulo{1, 1}
fmt.Println(f == nil)        // false
```

O bug clássico acontece com `error`:

```go
type MeuErro struct{}
func (*MeuErro) Error() string { return "meu erro" }

func validar(ok bool) error {
	var e *MeuErro           // ponteiro nil do tipo *MeuErro
	if !ok {
		e = &MeuErro{}
	}
	return e                 // ⚠️ devolve uma interface com TIPO *MeuErro e valor nil
}

err := validar(true)         // "deu certo"...
fmt.Println(err == nil, err) // false meu erro    ← 😱 parece que falhou!
```

```
return e  (com e == nil do tipo *MeuErro)

   err (interface error)
   ┌──────────────────┬──────────────────┐
   │ tipo: *MeuErro   │ dados: nil       │   ← tem tipo → err != nil
   └──────────────────┴──────────────────┘

   nil de verdade
   ┌──────────────────┬──────────────────┐
   │ tipo: nil        │ dados: nil       │
   └──────────────────┴──────────────────┘
```

(Repare que até `err.Error()` funcionou: o método tem receptor ponteiro e não acessa campos, então rodou com receptor nil, como vimos em [[Métodos e Receptores]].)

A correção: **devolva `nil` literalmente** no caminho de sucesso, nunca uma variável de tipo concreto (saída real):

```go
func validarCerto(ok bool) error {
	if !ok {
		return &MeuErro{}
	}
	return nil                       // ✅ interface nil de verdade
}
fmt.Println(validarCerto(true) == nil)   // true
```

> ✅ **Regra:** funções que devolvem `error` devem ter **tipo de retorno `error`** (a interface), nunca `*MeuErro`, e devolver `nil` explicitamente.

---

## 🎨 7. Boas práticas de design

### 7.1 Interfaces pequenas

> *"The bigger the interface, the weaker the abstraction."* — Go Proverbs

A maioria das interfaces da biblioteca padrão tem **um ou dois métodos**. Interfaces pequenas são fáceis de implementar, de simular em testes e de combinar.

### 7.2 Defina a interface onde ela é USADA (no consumidor)

Em Java, a interface costuma ficar junto da implementação. Em Go, o idiomático é o **consumidor** declarar a interface mínima de que precisa:

```go
// pacote notificacao (o CONSUMIDOR)
type enviador interface {          // só o que este pacote precisa
	Enviar(para, msg string) error
}

type Servico struct{ env enviador }

// pacote smtp (o PRODUTOR) não sabe nada de "enviador": só tem um tipo com Enviar
```

### 7.3 "Aceite interfaces, devolva structs"

```go
func NovoServico(env enviador) *Servico   // ✅ aceita interface (flexível para quem chama)
                                          // ✅ devolve tipo concreto (quem recebe tem acesso a tudo)
```

Devolver interface esconde métodos e campos do tipo concreto sem necessidade (exceção: quando há várias implementações reais, como `error`, ou para esconder detalhes de propósito).

### 7.4 Não crie interfaces "por precaução"

Não crie uma interface para cada struct "para o futuro" (hábito de Java). Crie quando houver **duas ou mais implementações** ou quando precisar **simular** algo em testes.

### 7.5 Interfaces facilitam testes

```go
type relogio interface{ Agora() time.Time }

type relogioReal struct{}
func (relogioReal) Agora() time.Time { return time.Now() }

type relogioFalso struct{ t time.Time }
func (r relogioFalso) Agora() time.Time { return r.t }   // em testes: tempo fixo e previsível
```

Veremos em [[Testes com go test]].

---

## 🧬 8. Interfaces e generics

Desde o Go 1.18, interfaces também servem como **restrições de tipo** em generics, e podem listar **tipos** além de métodos:

```go
type Numero interface {
	~int | ~int64 | ~float64          // conjunto de tipos (só válido como restrição)
}

func Somar[T Numero](xs []T) T { ... }
```

| Use interface (polimorfismo em execução) quando... | Use generics quando... |
| --- | --- |
| Valores de tipos diferentes convivem numa mesma coleção | Todos os elementos têm o mesmo tipo, conhecido em compilação |
| O comportamento (métodos) é o que importa | A estrutura/algoritmo é o mesmo para vários tipos |
| `[]Forma` com retângulos e círculos | `Max[T]`, `Stack[T]`, `slices.Sort` |

Veremos em [[Generics]].

---

## 🔬 9. Por dentro: como uma interface existe na memória

### 9.1 Duas palavras: tipo + dados

Toda variável de interface ocupa **2 palavras** (16 bytes em amd64; saída real: `unsafe.Sizeof(f)` = 16):

```
interface com métodos (iface)              interface vazia (eface, para any)
┌──────────────┬──────────────┐            ┌──────────────┬──────────────┐
│ tab  *itab   │ data pointer │            │ _type *_type │ data pointer │
└──────┬───────┴──────┬───────┘            └──────────────┴──────────────┘
       │              │
       │              └──▶ o valor concreto (ou cópia dele)
       ▼
┌──────────────────────────────────┐
│ itab (tabela da interface)       │
│   inter  → descritor de Forma    │
│   _type  → descritor de Circulo  │
│   hash   (para type switches)    │
│   fun[0] → Circulo.Area          │   ← tabela de métodos, como
│   fun[1] → Circulo.Perimetro     │     uma vtable de C++
└──────────────────────────────────┘
```

> 💡 **itab (interface table):** estrutura criada pelo runtime para cada par **(interface, tipo concreto)** usado no programa. Guarda os descritores dos dois tipos e um array de ponteiros para os métodos concretos, na ordem dos métodos da interface. É calculada uma vez e guardada em cache (ou pré-calculada pelo compilador, na seção `.itablink` que vimos em [[Como Go Funciona]]).

### 9.2 Chamada de método via interface

`f.Area()` com `f` do tipo `Forma` vira:

```
MOVQ  tab(f), AX          ; carrega o ponteiro para a itab
MOVQ  24(AX), CX          ; carrega itab.fun[0] (o endereço de Area do tipo concreto)
MOVQ  data(f), AX         ; o receptor: ponteiro para os dados
CALL  CX                  ; chamada indireta
```

Comparado a uma chamada direta, há duas leituras de memória e um salto indireto, e o compilador **não consegue inlinar** a chamada. São poucos nanossegundos, irrelevantes quase sempre. Quando o compilador consegue provar qual é o tipo concreto, ele faz **devirtualização**: troca a chamada indireta por uma direta (e o PGO ajuda a devirtualizar os casos mais frequentes, ver [[Profiling e Diagnóstico]]).

### 9.3 Guardar um valor em interface pode alocar

O campo de dados é um **ponteiro**. Então, para guardar um valor que não é ponteiro, o runtime precisa colocá-lo em algum lugar e apontar para ele. Saída real de `-gcflags=-m`:

```
./main.go:53:29: Retangulo{...} escapes to heap
./main.go:53:44: Circulo{...} escapes to heap
```

Otimizações do runtime evitam alocação em casos comuns: valores que já são ponteiros, valores de tamanho zero, inteiros pequenos (0–255) e strings/bytes de um caractere usam tabelas estáticas. Por isso `fmt.Println(x)` com muitos argumentos pode alocar, e loggers de alta performance evitam `...any`.

### 9.4 A armadilha do nil, explicada pela memória

Agora fica claro: `err == nil` compara **as duas palavras** com zero. Uma interface com `tab` apontando para a itab de `(*MeuErro, error)` não é zero, mesmo que `data` seja nil.

### 9.5 Comparação com outras linguagens

| Linguagem | Mecanismo | Onde fica a tabela de métodos |
| --- | --- | --- |
| C++ | Funções virtuais + vtable | No **objeto** (ponteiro oculto em toda instância) |
| Java | Interfaces + vtables/itables | No **objeto** (cabeçalho do objeto) |
| Go | Interfaces + itab | Na **variável de interface**, só quando é usada como interface |
| C (kernel Linux) | Struct de ponteiros para função | Explícita, montada à mão |

Em Go, uma struct comum **não carrega custo nenhum** por poder satisfazer interfaces: a itab só aparece quando o valor é colocado numa variável de interface.

O kernel Linux usa exatamente essa ideia em C. O **VFS** (sistema de arquivos virtual) define uma "interface" para arquivos como uma struct de ponteiros de função, e cada sistema de arquivos (ext4, procfs, pipes, sockets) preenche a sua:

```c
struct file_operations {                      // ≈ uma interface de Go
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    int     (*release)(struct inode *, struct file *);   // ≈ Close
    ...
};
```

É por isso que a syscall `read` funciona igual para um arquivo, um pipe ou um socket. Você estudou essas syscalls em [[Syscalls para Gerenciamento de Arquivos]].

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Devolver ponteiro nil tipado como `error` | `err != nil` mesmo sem erro | Retorno do tipo `error` e `return nil` explícito |
| `var i I = T{}` com métodos de ponteiro | `does not implement (method has pointer receiver)` | `&T{}` |
| Interfaces enormes | Difíceis de implementar e simular | Divida em interfaces pequenas |
| Interface definida no produtor "por precaução" | Abstração inútil, acoplamento | Defina no consumidor, quando precisar |
| `any` em todo lugar | Perde a checagem de tipos | Tipos concretos, interfaces específicas ou generics |
| Comparar interfaces com tipos não comparáveis | panic em execução | Evite `==` entre `any` de conteúdo desconhecido |
| Esperar inlining em chamadas via interface | Código mais lento em laço quente | Tipo concreto no laço crítico (depois de medir) |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Interface | Conjunto de métodos; descreve comportamento |
| Satisfação | **Implícita**: ter os métodos basta; checada pelo compilador |
| Polimorfismo | Código que funciona com qualquer tipo que satisfaça a interface |
| Method set | Métodos com receptor ponteiro só são satisfeitos por `*T` |
| Essenciais | `io.Reader`, `io.Writer`, `io.Closer`, `fmt.Stringer`, `error`, `http.Handler` |
| Composição | Interfaces embutem interfaces |
| `any` | Interface vazia; aceita tudo; use com moderação |
| Armadilha do nil | Interface com ponteiro nil tipado ≠ nil |
| Design | Pequenas, no consumidor, "aceite interfaces, devolva structs" |
| Por dentro | 2 palavras (itab/tipo + dados); itab ≈ vtable; chamada indireta; pode alocar |

---

## 🔗 Conexões

**Dentro de Go**
- Method sets e receptores → [[Métodos e Receptores]]
- Embutindo interfaces e structs → [[Composição vs Herança]]
- Recuperando o tipo concreto → [[Type Assertion e Type Switch]]
- Interfaces como restrições de tipo → [[Generics]]
- `error` como interface → [[O Padrão error em Go]]
- O ecossistema `io.Reader`/`io.Writer` → [[io.Reader e io.Writer]]
- `http.Handler` → [[HTTP (net-http)]]
- Interfaces para testes → [[Testes com go test]]
- Inspeção de tipos em execução → [[Reflection]]

**Sistemas Operacionais**
- [[Arquivos]] e [[Syscalls para Gerenciamento de Arquivos]] — "tudo é arquivo": a mesma interface `read`/`write` para arquivos, pipes e sockets
- [[Dispositivos de IO]] — drivers expõem operações padronizadas (struct de ponteiros de função), a "interface" do kernel
- [[Micronúcleos]] e [[O Modelo Cliente-Servidor]] — contratos bem definidos entre componentes desacoplados

**Nginx**
- [[Socket]] — um socket é um descritor de arquivo; em Go, `net.Conn` satisfaz `io.Reader` e `io.Writer`
- [[Proxy x Proxy Reverso]] — um proxy só precisa de "algo que leia" e "algo que escreva": `io.Copy(destino, origem)`

---

⬅️ **Anterior:** [[Métodos e Receptores]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Composição vs Herança]] ➡️
