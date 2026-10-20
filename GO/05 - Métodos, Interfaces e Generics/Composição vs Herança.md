---
tags:
  - go
  - go/metodos-interfaces
aliases:
  - Embedding
  - Composição em Go
  - Herança em Go
  - Campos promovidos
ordem: 23
modulo: "05 - Métodos, Interfaces e Generics"
---
# Composição vs Herança

> **Nota 23 da trilha de Go.** Go **não tem herança**. No lugar dela, oferece duas ferramentas que, combinadas, cobrem os mesmos casos de uso com menos acoplamento: **interfaces** (para polimorfismo) e **embedding** (para reaproveitar campos e métodos). Nesta nota vamos entender os problemas da herança que motivaram essa escolha, como o embedding funciona (campos e métodos **promovidos**, "sobrescrita" por sombreamento, conflitos), a diferença crucial de que **não existe despacho virtual**, embedding de interfaces em structs (decoradores) e como tudo isso é implementado na memória.
>
> **Pré-requisitos:** [[Interfaces]], [[Structs]].

---

## 🧭 De onde viemos

Em [[Structs]] fizemos uma introdução ao embedding: um campo declarado só com o tipo tem seus campos e métodos "promovidos". Em [[Interfaces]] vimos que polimorfismo em Go vem da satisfação implícita de interfaces, não de hierarquias de classes. Agora vamos juntar as duas peças e entender por que Go dispensou a herança.

---

## 🏛️ 1. O problema da herança

Herança mistura duas coisas diferentes num único mecanismo:

1. **Reaproveitar implementação** ("quero os campos e métodos da classe pai")
2. **Ser substituível** ("um `Cachorro` pode ser usado onde se espera um `Animal`")

Isso traz problemas conhecidos:

| Problema | Descrição |
| --- | --- |
| **Classe base frágil** | Mudar um método da classe pai pode quebrar silenciosamente as filhas, que dependiam do comportamento antigo |
| **Hierarquias rígidas** | O mundo raramente é uma árvore: um `PatoDeBorracha` "é um" `Pato` (forma), mas não "é um" `Animal` (não respira) |
| **Problema do diamante** | Com herança múltipla, se B e C herdam de A e D herda de B e C, de quem D herda um método de A? |
| **Acoplamento forte** | A filha depende de detalhes internos da mãe (`protected`), e a mãe não pode evoluir livremente |
| **Herança para reaproveitar código** | Criar uma subclasse só para "pegar" métodos, mesmo sem relação "é um" |

O livro *Design Patterns* (Gang of Four, 1994) já recomendava: **"prefira composição a herança"**. Go levou o conselho ao pé da letra e removeu a herança.

> 💡 **Composição:** construir um tipo **contendo** outros tipos ("tem um"), em vez de **derivar** de outro tipo ("é um"). O comportamento é obtido delegando para as partes.

---

## 🧬 2. Embedding: composição com açúcar sintático

### 2.1 Composição explícita (sem embedding)

```go
type Motor struct{ Potencia int }
func (m *Motor) Ligar() { fmt.Println("vrum") }

type Carro struct {
	motor Motor      // campo nomeado: composição "pura"
	Marca string
}

func (c *Carro) Ligar() { c.motor.Ligar() }   // delegação manual
```

### 2.2 Composição com embedding

Declarando o campo **só com o tipo**, os campos e métodos do tipo embutido são **promovidos**:

```go
type Carro struct {
	Motor            // campo embutido: o nome do campo é "Motor"
	Marca string
}

c := Carro{Motor: Motor{Potencia: 150}, Marca: "Fiat"}
c.Ligar()                // promovido: atalho para c.Motor.Ligar()
fmt.Println(c.Potencia)  // promovido: atalho para c.Motor.Potencia
```

Desde o **Go 1.27**, o literal pode usar o campo promovido diretamente: `Carro{Potencia: 150, Marca: "Fiat"}`.

O embedding é **só um atalho**: o compilador reescreve `c.Ligar()` como `c.Motor.Ligar()`. Não há mágica em tempo de execução.

---

## 🎭 3. "Sobrescrita" é sombreamento, não polimorfismo

Se o tipo externo declara um método com o mesmo nome, ele **esconde** o promovido (saída real):

```go
type Animal struct{ Nome string }
func (a Animal) Som() string        { return "..." }
func (a Animal) Apresentar() string { return "Sou " + a.Nome + " e faço " + a.Som() }

type Cachorro struct {
	Animal
	Raca string
}
func (c Cachorro) Som() string { return "au au" }    // esconde Animal.Som

c := Cachorro{Animal{"Rex"}, "vira-lata"}
fmt.Println(c.Nome, c.Som(), c.Animal.Som())   // Rex au au ...   ← o original continua acessível
```

Agora o ponto **mais importante** da nota (saída real):

```go
fmt.Println(c.Apresentar())
// Sou Rex e faço ...       ← NÃO "au au"!
```

Em Java, `Apresentar()` chamaria o `Som()` sobrescrito da subclasse (**despacho virtual**). Em Go, `Apresentar` é um método de `Animal`, e dentro dele o receptor é um `Animal`. Ele **não sabe** que está embutido num `Cachorro`.

```
c.Apresentar()  →  c.Animal.Apresentar()
                        │
                        └── receptor: a Animal  (só a parte Animal)
                                │
                                └── a.Som()  →  Animal.Som()  → "..."
```

> ⚠️ **Não existe despacho virtual com embedding.** O tipo embutido não conhece o tipo que o contém. Se você precisa que a "base" chame comportamento variável da "derivada" (o padrão *template method*), use uma **interface** explicitamente:
> ```go
> type Falante interface{ Som() string }
> func Apresentar(nome string, f Falante) string { return "Sou " + nome + " e faço " + f.Som() }
> ```

### 3.1 Embedding não cria relação "é um"

```go
var a Animal = Cachorro{}   // ❌ cannot use Cachorro{} as Animal value
```

Um `Cachorro` **contém** um `Animal`; não **é** um `Animal`. A substituibilidade em Go vem só de **interfaces** (saída real):

```go
type Falante interface{ Som() string }
var f Falante = c          // ✅ Cachorro tem Som() (próprio) → satisfaz
fmt.Println(f.Som())       // au au
```

---

## ⚔️ 4. Conflitos de nomes

Se dois tipos embutidos no **mesmo nível** promovem o mesmo nome, acessar esse nome é ambíguo (mensagem real):

```go
type A struct{}
func (A) Nome() string { return "A" }
type B struct{}
func (B) Nome() string { return "B" }

type C struct {
	A
	B
}

var c C
c.Nome()     // ❌ ambiguous selector c.Nome
c.A.Nome()   // ✅ explícito
```

Regras de resolução:

1. Um nome declarado **diretamente** no tipo externo vence qualquer promovido
2. Entre promovidos, vence o de **menor profundidade** (embutido direto vence embutido do embutido)
3. Empate na mesma profundidade: **ambíguo**, mas só é erro se você **usar** o nome

> 💡 Esse é o "problema do diamante" resolvido de forma simples: Go não escolhe por você, obriga a ser explícito.

---

## 🧰 5. Padrões práticos de embedding

### 5.1 Embutir um ponteiro: compartilhar uma dependência

Saída real:

```go
type Logger struct{ prefixo string }
func (l *Logger) Log(msg string) { fmt.Println(l.prefixo + msg) }

type Servico struct {
	*Logger              // embute um PONTEIRO: vários serviços podem compartilhar o mesmo logger
	nome string
}

s := Servico{&Logger{"[svc] "}, "pagamentos"}
s.Log("iniciado")        // [svc] iniciado
```

> ⚠️ Com ponteiro embutido, o zero value tem o ponteiro **nil**: `var s Servico; s.Log("x")` dá panic.

### 5.2 Embutir `sync.Mutex`

Saída real:

```go
type Contador struct {
	sync.Mutex           // promove Lock() e Unlock()
	n int
}

func (c *Contador) Inc() {
	c.Lock()
	defer c.Unlock()
	c.n++
}
```

> ⚠️ Isso **exporta** `Lock` e `Unlock` como parte da API do seu tipo (qualquer um pode travar seu contador por fora). Para tipos exportados, prefira um campo **nomeado e não exportado**: `mu sync.Mutex`.

### 5.3 Embutir uma interface: decoradores com pouco código

Embutir uma **interface** numa struct dá à struct todos os métodos da interface, delegando para o valor guardado. Você então "sobrescreve" só o que quer modificar (saída real):

```go
type contaBytes struct {
	io.Reader        // qualquer Reader
	n int
}

func (c *contaBytes) Read(p []byte) (int, error) {   // intercepta só o Read
	n, err := c.Reader.Read(p)
	c.n += n
	return n, err
}

cb := &contaBytes{Reader: strings.NewReader("olá, embedding")}
io.ReadAll(cb)
fmt.Println(cb.n)        // 15
```

Muito usado para "envolver" um `http.ResponseWriter` e capturar o status code num middleware, ou envolver uma conexão para medir tráfego.

> ⚠️ Se o campo interface for nil, chamar um método promovido dá panic. E cuidado: o tipo externo **perde** interfaces opcionais que o interno tinha (um `http.ResponseWriter` embutido que também era `http.Flusher` deixa de parecer `Flusher`). Veremos a checagem disso em [[Type Assertion e Type Switch]].

### 5.4 Embedding de interfaces em interfaces

Como vimos em [[Interfaces]]:

```go
type ReadWriter interface {
	io.Reader
	io.Writer
}
```

---

## 🔁 6. Traduzindo padrões de herança para Go

| Em linguagens com herança | Em Go |
| --- | --- |
| Classe abstrata com métodos abstratos | Interface |
| Subclasse que reaproveita campos/métodos | Struct com embedding |
| Subclasse "é um" a superclasse (substituibilidade) | O tipo satisfaz uma interface |
| Método sobrescrito chamado pela base (*template method*) | A "base" recebe uma interface com o comportamento variável |
| `super.metodo()` | `t.TipoEmbutido.Metodo()` explícito |
| Herança múltipla | Embutir vários tipos (conflitos resolvidos explicitamente) |
| Injeção de dependência via construtor | Campos de tipo interface preenchidos no construtor `NewX` |

Exemplo de injeção de dependência, o padrão mais comum em serviços Go:

```go
type Repositorio interface {
	Buscar(id string) (*Usuario, error)
}

type ServicoUsuario struct {
	repo Repositorio            // depende do COMPORTAMENTO, não de um banco específico
	log  *slog.Logger
}

func NovoServicoUsuario(repo Repositorio, log *slog.Logger) *ServicoUsuario {
	return &ServicoUsuario{repo: repo, log: log}
}
// Em produção: um repositório PostgreSQL. Em testes: um repositório em memória.
```

---

## 🔬 7. Por dentro: embedding na memória

### 7.1 O tipo embutido é um campo comum, no início ou no meio da struct

```
type Cachorro struct { Animal; Raca string }

Cachorro na memória:
offset 0  ┌────────────────────────────┐
          │ Animal                     │
          │   Nome string (16 bytes)   │   ← c.Nome é c.Animal.Nome: offset 0
offset 16 ├────────────────────────────┤
          │ Raca string (16 bytes)     │
          └────────────────────────────┘
```

Acessar um campo promovido é **somar um deslocamento constante**, calculado em compilação. Custo zero, exatamente como acessar um campo normal.

### 7.2 Métodos promovidos e wrappers

Para que `Cachorro` satisfaça interfaces com os métodos promovidos de `Animal`, o compilador gera pequenas **funções-ponte** (wrappers) quando necessário:

```
// gerado automaticamente (conceitualmente):
func (c Cachorro) Apresentar() string { return c.Animal.Apresentar() }
```

Elas aparecem em stack traces com a marca `<autogenerated>`. Quando a chamada é direta (não via interface), o compilador nem usa o wrapper: chama `Animal.Apresentar` passando `c.Animal`.

### 7.3 Comparação: herança em C++ vs embedding em Go

| | C++ (herança com métodos virtuais) | Go (embedding) |
| --- | --- | --- |
| Ponteiro para vtable em cada objeto | Sim (8 bytes por instância) | Não |
| Chamada de método "herdado" | Pode ser virtual (indireta) | Sempre direta (resolvida em compilação) |
| Layout | Base no início, vptr oculto | Campo comum, sem nada oculto |
| Polimorfismo | Embutido na hierarquia | Separado, via interfaces (itab só quando usado) |

### 7.4 O kernel Linux faz o mesmo em C

O kernel não tem herança, mas usa composição o tempo todo, e uma macro famosa para "voltar" da parte para o todo:

```c
struct list_head { struct list_head *next, *prev; };

struct tarefa {
    int id;
    struct list_head lista;      // "embutido": a tarefa pode estar numa lista
};

// container_of: dado um ponteiro para o campo, calcula o endereço da struct que o contém
struct tarefa *t = container_of(ponteiro_lista, struct tarefa, lista);
//                 = (struct tarefa *)((char *)ponteiro_lista - offsetof(struct tarefa, lista))
```

Go faz o caminho contrário automaticamente (do todo para a parte, somando o offset) e **proíbe** o caminho de volta, que exige aritmética de ponteiros (ver [[Ponteiros]]). A lista ligada genérica do kernel é composição pura: qualquer struct que embute um `list_head` pode participar de listas. Você viu a importância de listas no gerenciamento de processos em [[Implementação de Processos]].

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Esperar despacho virtual | Método da "base" chama o método da base, não o "sobrescrito" | Use uma interface explícita |
| Tratar embedding como "é um" | `cannot use Cachorro as Animal` | Use interfaces para substituibilidade |
| Embutir tipo com métodos indesejados | API pública poluída (`Lock`/`Unlock` expostos) | Campo nomeado não exportado |
| Ponteiro embutido nil | panic ao chamar método promovido | Inicialize no construtor |
| Interface embutida nil | panic | Inicialize; documente |
| Dois embutidos com o mesmo método | `ambiguous selector` | Acesso explícito ou defina o método no tipo externo |
| Perder interfaces opcionais ao embrulhar | `w.(http.Flusher)` falha | Implemente/repasse os métodos opcionais |
| Hierarquias profundas de embedding | Difícil saber de onde vem cada método | Prefira composição rasa e explícita |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Go não tem herança | Usa interfaces (polimorfismo) + embedding (reuso) |
| Embedding | Campo só com o tipo; campos e métodos promovidos |
| Promoção | Atalho de compilação: `c.Ligar()` = `c.Motor.Ligar()` |
| "Sobrescrita" | Sombreamento; o original continua acessível |
| Sem despacho virtual | O tipo embutido não conhece quem o contém |
| Sem "é um" | Substituibilidade só via interfaces |
| Conflitos | Mesmo nível = ambíguo; resolva explicitamente |
| Padrões | Ponteiro compartilhado, mutex, interface embutida (decorador), DI |
| Por dentro | Offset constante; wrappers gerados; sem vtable por objeto |

---

## 🔗 Conexões

**Dentro de Go**
- A base do embedding → [[Structs]]
- Polimorfismo e satisfação implícita → [[Interfaces]]
- Métodos promovidos e method sets → [[Métodos e Receptores]]
- Checar interfaces opcionais de um valor embrulhado → [[Type Assertion e Type Switch]]
- Embutir mutex → [[sync.WaitGroup e sync.Mutex]]
- Decorar `http.ResponseWriter` em middlewares → [[HTTP (net-http)]]
- Injeção de dependência e testes → [[Testes com go test]]

**Sistemas Operacionais**
- [[Sistemas Monolíticos]], [[Sistema de Camadas]] e [[Micronúcleos]] — organizar um sistema em componentes com contratos claros, em vez de hierarquias rígidas
- [[Implementação de Processos]] — estruturas do kernel compostas por outras (listas, filas), à moda de `container_of`
- [[A Linguagem C]] — composição de structs e ponteiros para função, a "orientação a objetos" manual de C

---

⬅️ **Anterior:** [[Interfaces]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Type Assertion e Type Switch]] ➡️
