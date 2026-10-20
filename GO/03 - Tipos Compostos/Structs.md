---
tags:
  - go
  - go/tipos-compostos
aliases:
  - Structs em Go
  - Struct
  - Struct tags
  - Padding e alinhamento
ordem: 15
modulo: "03 - Tipos Compostos"
---
# Structs

> **Nota 15 da trilha de Go.** Uma **struct** agrupa vários valores, de tipos possivelmente diferentes, sob um único nome: um `Usuario` tem `Nome`, `Email` e `Idade`. É a principal ferramenta para **modelar o domínio** em Go, que não tem classes. Nesta nota veremos como declarar e criar structs, a semântica de valor (cópia), ponteiros para structs, comparação, structs anônimas, **tags**, uma introdução ao **embedding**, o padrão de **construtor** e o **layout na memória** (alinhamento e padding). Métodos e interfaces, que dão comportamento às structs, vêm no módulo 05.
>
> **Pré-requisitos:** [[Ponteiros]], [[Maps]].

---

## 🧭 De onde viemos

Até aqui, cada valor tinha um tipo só: um número, uma string, uma lista de elementos iguais (slice) ou pares chave-valor (map). Mas o mundo real tem entidades com **vários atributos de tipos diferentes**. Poderíamos usar um `map[string]any`, como em linguagens dinâmicas, mas perderíamos a checagem de tipos do compilador. A struct resolve isso: um tipo com campos **nomeados e tipados**.

> 💡 **Struct:** tipo composto formado por uma sequência de **campos**, cada um com nome e tipo. É o equivalente ao `struct` de C, ao "registro" de Pascal, ou a uma classe com só atributos (sem herança).

---

## 🏗️ 1. Declarando

```go
type Usuario struct {
	ID        int64
	Nome      string
	Email     string
	Idade     int
	Ativo     bool
	CriadoEm  time.Time
	Tags      []string
}

type Ponto struct {
	X, Y int                // campos do mesmo tipo podem ser agrupados
}
```

Regras de visibilidade (de [[Convenções da linguagem]]) valem campo a campo:

```go
type Conta struct {
	Titular string    // exportado: outros pacotes leem e escrevem
	saldo   float64   // não exportado: só o pacote da Conta mexe
}
```

---

## 🧰 2. Criando valores

```go
// 1) Literal com nomes de campos (✅ forma recomendada)
u := Usuario{
	Nome:  "Ana",
	Email: "ana@exemplo.com",
	Idade: 30,               // campos omitidos recebem o zero value
}

// 2) Literal posicional (todos os campos, na ordem)
p := Ponto{1, 2}

// 3) Zero value: todos os campos zerados
var vazio Usuario         // Nome "", Idade 0, Tags nil, ...

// 4) Ponteiro para struct
pu := &Usuario{Nome: "Bia"}
```

> ✅ **Use sempre os nomes dos campos**, exceto em structs minúsculas e estáveis (como `Ponto{1, 2}`). O literal posicional quebra quando alguém adiciona ou reordena campos, e o `go vet` avisa quando você usa a forma posicional com structs de outros pacotes.

### 2.1 Acesso a campos

Saída real:

```go
type Endereco struct{ Rua, Cidade string }
type Pessoa struct {
	Nome     string
	Idade    int
	Endereco Endereco        // struct dentro de struct
}

p := Pessoa{Nome: "Ana", Idade: 30}
p.Endereco.Cidade = "Curitiba"
fmt.Printf("%+v\n", p)
// {Nome:Ana Idade:30 Endereco:{Rua: Cidade:Curitiba}}
```

---

## 📋 3. Structs são valores: atribuir copia

Exatamente como arrays (e diferente de objetos em PHP, Ruby, Java ou JavaScript, onde variáveis guardam referências). Saída real:

```go
q := p          // copia TODOS os campos
q.Nome = "Bia"
fmt.Println(p.Nome, q.Nome)    // Ana Bia   ← p não foi alterado
```

```
p ┌──────────────────────┐        q ┌──────────────────────┐
  │ Nome: "Ana"          │          │ Nome: "Bia"          │   duas cópias
  │ Idade: 30            │          │ Idade: 30            │   independentes
  │ Endereco: {..}       │          │ Endereco: {..}       │
  └──────────────────────┘          └──────────────────────┘
```

> ⚠️ **Cópia rasa:** a cópia de uma struct copia os campos "como estão". Se um campo é slice, map ou ponteiro, a cópia copia **o cabeçalho/ponteiro**, e as duas structs passam a **compartilhar** os dados apontados.
> ```go
> a := Usuario{Tags: []string{"go"}}
> b := a
> b.Tags[0] = "rust"       // altera a.Tags também! (mesmo array por baixo)
> b.Tags = append(b.Tags, "c")   // agora pode divergir (append pode realocar)
> ```

O mesmo vale ao passar structs para funções: a função recebe uma cópia. Para que a função altere o original, passe um ponteiro:

```go
func aniversario(p *Pessoa) { p.Idade++ }   // altera o original
func idadeEm(p Pessoa, anos int) int { return p.Idade + anos }  // só lê: valor basta
```

---

## 👉 4. Ponteiros para structs

Como vimos em [[Ponteiros]], o acesso a campos através de ponteiro é **automático** (saída real):

```go
pp := &Ponto{3, 4}
pp.X = 9                 // equivale a (*pp).X = 9
fmt.Println(*pp)         // {9 4}
```

### 4.1 Valor ou ponteiro?

| Use valor (`T`) quando... | Use ponteiro (`*T`) quando... |
| --- | --- |
| A struct é pequena (alguns campos simples) | A struct é grande |
| Ela representa um valor imutável na prática (`Ponto`, `Dinheiro`, `time.Time`) | Ela tem identidade e estado que muda (`Conta`, `Conexao`, `Servidor`) |
| Você quer garantir que ninguém altere o original | Funções/métodos precisam alterá-la |
| Vai usar como chave de map ou comparar com `==` | Contém `sync.Mutex` ou outro campo que não pode ser copiado |

> ✅ Seja **consistente**: se a maioria das funções e métodos de um tipo usa `*T`, use `*T` em todos. Voltaremos a isso em [[Métodos e Receptores]].

---

## 🆕 5. Construtores: funções `NewX`

Go não tem construtores especiais. A convenção é uma função comum, chamada `New` + nome do tipo (ou só `New` se o pacote tem um tipo principal), que **valida e inicializa** os campos:

```go
type Servidor struct {
	host    string
	porta   int
	timeout time.Duration
	rotas   map[string]Handler     // map precisa ser criado!
}

func NewServidor(host string, porta int) (*Servidor, error) {
	if porta <= 0 || porta > 65535 {
		return nil, fmt.Errorf("porta inválida: %d", porta)
	}
	return &Servidor{
		host:    host,
		porta:   porta,
		timeout: 30 * time.Second,     // valor padrão
		rotas:   make(map[string]Handler),
	}, nil
}
```

Use construtor quando:

- Há **invariantes** a garantir (porta válida, campo obrigatório)
- Algum campo precisa de **inicialização** (maps, channels, valores padrão diferentes do zero)
- Os campos são **não exportados** (quem está fora do pacote não conseguiria preenchê-los)

Quando o **zero value já é útil** (como `sync.Mutex` ou `bytes.Buffer`), não precisa de construtor: `var b bytes.Buffer` já está pronto. Esse é o ideal do provérbio *"make the zero value useful"*.

> 💡 Para construtores com muitas opções opcionais, Go tem o padrão **opções funcionais** (`NewServidor(host, WithTimeout(5*time.Second), WithTLS(cfg))`), que veremos em [[Closures e Funções Anônimas]].

---

## 🟰 6. Comparando structs

Uma struct é comparável com `==` se **todos** os seus campos forem comparáveis. A comparação é campo a campo (saída real):

```go
fmt.Println(Ponto{1, 2} == Ponto{1, 2})    // true
```

```go
type T struct{ Tags []string }
T{} == T{}    // ❌ invalid operation: struct containing []string cannot be compared
```

Structs comparáveis podem ser **chaves de map** (`map[Ponto]string`, como vimos em [[Maps]]). Para comparar structs com campos não comparáveis, escreva um método `Equal` ou use `reflect.DeepEqual` (comum em testes).

---

## 🕶️ 7. Structs anônimas

Uma struct pode ser usada sem dar nome ao tipo (saída real):

```go
cfg := struct {
	Host string
	Port int
}{"localhost", 8080}
fmt.Println(cfg)          // {localhost 8080}
```

Usos típicos:

```go
// Casos de teste em tabela (muito comum, ver Testes com go test)
testes := []struct {
	entrada  string
	esperado int
}{
	{"1", 1},
	{"42", 42},
}

// Decodificar só parte de um JSON, sem criar tipo
var resp struct {
	Status string `json:"status"`
}
json.Unmarshal(dados, &resp)
```

---

## 🏷️ 8. Tags de struct

Uma **tag** é uma string de metadados anexada a um campo. O compilador ignora; bibliotecas leem em tempo de execução via **reflection** (saída real):

```go
type Usuario struct {
	ID    int64  `json:"id"`
	Nome  string `json:"nome" db:"nome_completo" validate:"required"`
	Senha string `json:"-"`                     // "-" = nunca serializar
	Email string `json:"email,omitempty"`       // omitir se vazio
}

f, _ := reflect.TypeOf(Usuario{}).FieldByName("Nome")
fmt.Println(f.Tag.Get("json"), f.Tag.Get("db"))   // nome nome_completo
```

O formato convencional é `chave:"valor"` separados por espaço. Bibliotecas comuns que usam tags:

| Chave | Biblioteca | Para quê |
| --- | --- | --- |
| `json` | `encoding/json` | Nome do campo no JSON, `omitempty`, `omitzero`, `-` |
| `xml`, `yaml`, `toml` | serializadores | Idem para outros formatos |
| `db` | `sqlx`, drivers | Nome da coluna no banco |
| `validate` | `go-playground/validator` | Regras de validação |
| `env` | bibliotecas de configuração | Nome da variável de ambiente |

> ⚠️ Erros de digitação em tags (`json:nome` sem aspas, `jsom:"nome"`) não são erro de compilação: a tag simplesmente é ignorada. O `go vet` verifica o formato das tags conhecidas.

Veremos tags na prática em [[JSON (marshal e unmarshal)]] e reflection em [[Reflection]].

---

## 🧬 9. Embedding (introdução)

Um campo declarado **só com o tipo**, sem nome, é um **campo embutido** (*embedded field*). Os campos e métodos do tipo embutido são **promovidos**: podem ser acessados como se fossem da struct externa (saída real):

```go
type Base struct{ ID int }

func (b Base) Descrever() string { return fmt.Sprintf("ID=%d", b.ID) }

type Usuario struct {
	Base             // campo embutido (o nome do campo é "Base")
	Nome string
}

u := Usuario{Base: Base{ID: 7}, Nome: "Caio"}
fmt.Println(u.ID, u.Descrever(), u.Base.ID)   // 7 ID=7 7
```

```
Usuario na memória:
┌────────────────────────┐
│ Base ┌───────────────┐ │   u.ID  é atalho para  u.Base.ID
│      │ ID: 7         │ │   u.Descrever() é atalho para u.Base.Descrever()
│      └───────────────┘ │
│ Nome: "Caio"           │
└────────────────────────┘
```

> 💡 **Go 1.27+:** num literal de struct, a chave pode ser qualquer **seletor de campo válido**, inclusive um campo promovido. Então dá para escrever `Usuario{ID: 7, Nome: "Caio"}` em vez de `Usuario{Base: Base{ID: 7}, Nome: "Caio"}`.

⚠️ **Embedding não é herança.** `Usuario` não "é uma" `Base`: não dá para passar um `Usuario` onde se espera uma `Base`. É **composição** com açúcar sintático. Isso fica muito mais interessante com métodos e interfaces, e é o assunto de [[Composição vs Herança]].

---

## ∅ 10. A struct vazia `struct{}`

Uma struct sem campos ocupa **zero bytes** (saída real: `unsafe.Sizeof(struct{}{})` = 0). Usos:

```go
set := map[string]struct{}{}        // conjunto: só as chaves importam (ver Maps)
done := make(chan struct{})         // sinalização: só o evento importa (ver Channels)
type semEstado struct{}             // tipo que só existe para ter métodos
```

---

## 📐 11. Layout na memória: alinhamento e padding

Os campos de uma struct ficam **na ordem declarada**, contíguos, mas com **buracos** (padding) inseridos pelo compilador para respeitar o **alinhamento** de cada tipo, que vimos em [[Tipos de Dados]]. Saída real:

```go
type Ruim struct {
	A bool     // 1 byte
	B int64    // 8 bytes: precisa começar em endereço múltiplo de 8
	C bool     // 1 byte
}
type Bom struct {
	B int64
	A bool
	C bool
}

unsafe.Sizeof(Ruim{})        // 24
unsafe.Sizeof(Bom{})         // 16
unsafe.Offsetof(Ruim{}.B)    // 8
unsafe.Offsetof(Ruim{}.C)    // 16
```

```
Ruim (24 bytes)
offset: 0    1                        8                       16   17                      24
       ┌────┬────────────────────────┬────────────────────────┬────┬───────────────────────┐
       │ A  │   padding (7 bytes)    │         B (8)          │ C  │   padding (7 bytes)   │
       └────┴────────────────────────┴────────────────────────┴────┴───────────────────────┘

Bom (16 bytes)
offset: 0                       8    9   10                     16
       ┌────────────────────────┬────┬────┬──────────────────────┐
       │         B (8)          │ A  │ C  │  padding (6 bytes)   │
       └────────────────────────┴────┴────┴──────────────────────┘
```

Por que existe padding no **final** de `Ruim`? Porque num array `[]Ruim`, o próximo elemento também precisa ter seu `B` alinhado em 8. O tamanho de uma struct é sempre múltiplo do maior alinhamento entre seus campos.

> ✅ **Regra prática:** ordenar campos do **maior para o menor** minimiza o padding. Importa quando há **milhões** de instâncias (caches, slices enormes). Para structs comuns, prefira a ordem que deixa o código mais legível. O linter `fieldalignment` aponta structs que poderiam ser menores.

> 💡 **Por que o hardware exige alinhamento?** A CPU lê a memória em blocos alinhados (palavras, linhas de cache de 64 bytes). Um `int64` desalinhado pode atravessar dois blocos e exigir dois acessos; em algumas arquiteturas causa exceção. Operações **atômicas** de 64 bits exigem alinhamento até em amd64 (ver [[sync.WaitGroup e sync.Mutex]]). Você estudou barramentos e acesso à memória em [[Barramentos]] e [[Memória]].

### 11.1 Structs vs a forma como C organiza

O layout de Go segue as mesmas regras de C na mesma plataforma (campos em ordem, alinhamento natural). A diferença é que Go **não garante** isso pela especificação, então para trocar structs com C use `cgo`, que cuida da compatibilidade (ver [[unsafe e cgo]]).

---

## 🔬 12. Por dentro: structs, stack e cópia

- Uma struct pequena que não escapa vive **na stack** (ou inteiramente em **registradores**) e não gera trabalho para o GC
- Passar uma struct por valor copia seus bytes; para structs de até algumas palavras, o compilador passa os campos **em registradores** (a ABI de Go desde 1.17 usa registradores para argumentos e retornos)
- Uma struct grande copiada em um laço quente pode pesar; aí o ponteiro compensa (meça com [[Benchmarks]])
- O GC precisa saber **quais campos são ponteiros**. Por isso o compilador gera, para cada tipo, um mapa de bits ("onde há ponteiros"). Uma struct só com campos numéricos é "sem ponteiros" e o GC nem precisa olhar dentro dela

```go
type SemPonteiros struct{ X, Y, Z float64 }        // o GC ignora o conteúdo
type ComPonteiros struct{ Nome string; Pai *No }   // o GC precisa escanear
```

---

## ⚠️ 13. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Esperar semântica de referência | Alterar a cópia não muda o original | Passe `*T` quando precisar alterar |
| Cópia rasa de campos slice/map | Duas structs compartilham dados | Copie os campos explicitamente |
| Literal posicional | Quebra ao adicionar campos | Use nomes de campos |
| Campo map não inicializado | panic ao escrever | Inicialize no construtor |
| Copiar struct com `sync.Mutex` | Dois mutexes independentes; o `go vet` avisa | Use `*T` |
| Comparar struct com slice | Erro de compilação | Método `Equal` ou `reflect.DeepEqual` |
| Erro de digitação em tag | Tag ignorada silenciosamente | `go vet` |
| Tratar embedding como herança | Tipo não é aceito onde se espera a "base" | Use interfaces (ver [[Interfaces]]) |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Struct | Campos nomeados e tipados; modela entidades |
| Criação | Literal com nomes, zero value, `&T{...}` |
| Semântica | Valor: atribuir e passar **copiam** (cópia rasa) |
| Ponteiro | `p.Campo` desreferencia automaticamente |
| Construtor | Função `NewX` que valida e inicializa |
| Comparação | `==` se todos os campos forem comparáveis |
| Anônima | Para testes em tabela, JSON pontual, dados locais |
| Tags | Metadados lidos por reflection (`json`, `db`...) |
| Embedding | Campos e métodos promovidos; composição, não herança |
| `struct{}` | Zero bytes; sets e sinais |
| Layout | Ordem declarada + padding de alinhamento; ordenar do maior para o menor economiza |

---

## 🔗 Conexões

**Dentro de Go**
- Dando comportamento às structs → [[Métodos e Receptores]]
- Embedding em profundidade → [[Composição vs Herança]]
- Structs que satisfazem interfaces → [[Interfaces]]
- Tags na prática → [[JSON (marshal e unmarshal)]]
- Como bibliotecas leem tags → [[Reflection]]
- Opções funcionais para construtores → [[Closures e Funções Anônimas]]
- Testes em tabela com structs anônimas → [[Testes com go test]]
- Stack, heap e escape de structs → [[Memória e Escape Analysis]]

**Sistemas Operacionais**
- [[A Linguagem C]] — structs de C, o mesmo modelo de layout com alinhamento
- [[Implementação de Processos]] — a tabela de processos do SO é um array de structs (o PCB: estado, registradores, ponteiros de memória, arquivos abertos)
- [[Memória]] e [[Barramentos]] — por que o hardware exige alinhamento e como o padding nasce disso

**Redes**
- [[PDU (Protocol Data Unit)]] e [[TCP-IP]] — cabeçalhos de protocolos são estruturas com campos de tamanho fixo, naturalmente modelados como structs (com cuidado com padding e ordem de bytes)

**Algoritmos e Estruturas de Dados**
- [[Linked Lists]] e [[Árvores Binárias]] — nós são structs com campos ponteiro (`type No struct { Valor int; Prox *No }`)

---

⬅️ **Anterior:** [[Maps]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Funções — Fundamentos]] ➡️
