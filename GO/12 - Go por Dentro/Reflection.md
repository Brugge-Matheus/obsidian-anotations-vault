---
tags:
  - go
  - go/runtime
aliases:
  - reflect
  - Reflexão
  - reflect.Type
  - reflect.Value
  - Struct tags na prática
  - Leis da reflexão
ordem: 53
modulo: "12 - Go por Dentro"
---
# Reflection

> **Nota 53 da trilha de Go.** Go é estaticamente tipado: o compilador conhece o tipo de tudo. Mas `encoding/json` consegue codificar **qualquer** struct sua, e o `fmt.Println` imprime **qualquer** valor. Como, se esses pacotes foram escritos antes de os seus tipos existirem? Com **reflection** (reflexão): a capacidade de um programa **examinar e manipular tipos e valores em tempo de execução**. Nesta nota veremos `reflect.Type` e `reflect.Value`, a diferença entre **tipo** e **kind**, as três **leis da reflexão** (com as mensagens reais de panic quando são violadas), como ler **struct tags** escrevendo um mini-codificador CSV com os iteradores do Go 1.26, chamar métodos por nome, `DeepEqual`, o **custo** medido, quando preferir generics, e de onde o runtime tira essas informações.
>
> **Pré-requisitos:** [[Interfaces]], [[Type Assertion e Type Switch]], [[Structs]], [[Generics]].

---

## 🧭 De onde viemos

Em [[Interfaces]] vimos que uma variável de interface guarda **duas palavras**: um ponteiro para o tipo (ou para a itab) e um ponteiro para os dados. Em [[Type Assertion e Type Switch]] vimos que cada tipo tem um **descritor** único no binário (`*_type`) e que `%T`, `errors.As` e o `reflect` dependem dessa informação de tipos em execução. Em [[Structs]] escrevemos struct tags (`json:"nome"`) e, em [[JSON (marshal e unmarshal)]], vimos o `encoding/json` lendo-as "com o pacote `reflect`". Em [[Iteradores]] mencionamos os iteradores de campos do `reflect` (Go 1.26). Em [[Como Go Funciona]], as seções `.typelink` do binário.

> 💡 **Reflection (reflexão):** mecanismo pelo qual um programa inspeciona a **estrutura** dos seus próprios valores (tipo, campos, métodos, tags) e os manipula **sem conhecer o tipo em tempo de compilação**.

---

## 🪞 1. `reflect.Type` e `reflect.Value`

Tudo começa num valor de interface (`any`), que já carrega o tipo e o dado:

```go
var x any = 3.14
t := reflect.TypeOf(x)    // reflect.Type:  a descrição do TIPO (float64)
v := reflect.ValueOf(x)   // reflect.Value: o VALOR, junto com seu tipo
fmt.Println(t, v.Float()) // float64 3.14
```

| | `reflect.Type` | `reflect.Value` |
| --- | --- | --- |
| Representa | Um tipo | Um valor de algum tipo |
| Exemplos de métodos | `Name`, `Kind`, `NumField`, `Field(i)`, `Method(i)`, `Elem`, `Key`, `Implements` | `Int`, `String`, `Field(i)`, `Index(i)`, `MapRange`, `Set...`, `Call`, `Interface` |
| Obter | `reflect.TypeOf(x)`, `reflect.TypeFor[T]()` (Go 1.22+) | `reflect.ValueOf(x)` |

### 1.1 Tipo vs *kind*

O **tipo** é o nome exato (`main.Celsius`, `time.Duration`); o **kind** é a **categoria** da representação (inteiro, float, struct, ponteiro...). Saída real:

```
int                    Kind=int     Name="int"
string                 Kind=string  Name="string"
main.Celsius           Kind=float64 Name="Celsius"     ← type Celsius float64
[]int                  Kind=slice   Name=""            ← tipos sem nome não têm Name
map[string]bool        Kind=map     Name=""
*main.Usuario          Kind=ptr     Name=""
main.Usuario           Kind=struct  Name="Usuario"
time.Duration          Kind=int64   Name="Duration"
func(...interface {}) (int, error) Kind=func Name=""  ← o tipo de fmt.Println
chan int               Kind=chan    Name=""
```

Código genérico baseado em reflection quase sempre faz um `switch v.Kind()`, porque há infinitos tipos mas só 26 *kinds*. `Celsius` e `float64` são tipos diferentes, mas ambos se leem com `v.Float()`.

---

## 📜 2. As três leis da reflexão

Rob Pike resumiu o pacote em três leis (*The Laws of Reflection*, blog oficial de Go, 2011):

| Lei | Em código |
| --- | --- |
| 1. A reflexão vai de um **valor de interface** para um **objeto de reflexão** | `reflect.ValueOf(x)`, `reflect.TypeOf(x)` |
| 2. A reflexão vai de um **objeto de reflexão** de volta para um **valor de interface** | `v.Interface()` (e depois uma type assertion) |
| 3. Para **modificar** um objeto de reflexão, o valor precisa ser **ajustável** (*settable*) | `v.CanSet()`; obtém-se a partir de um **ponteiro** com `.Elem()` |

### 2.1 A terceira lei, na prática

`reflect.ValueOf(c)` recebe uma **cópia** de `c` (passagem por valor, como em qualquer função); alterá-la não teria efeito no original, então o pacote **proíbe**. Saídas reais:

```go
type Config struct {
	Porta int
	host  string
}
c := Config{Porta: 80}
```

```
ValueOf(c).Field(0).SetInt   → panic: reflect: reflect.Value.SetInt using unaddressable value
ValueOf(&c).Elem()...SetInt  → ok
campo não exportado          → panic: reflect: reflect.Value.SetString using value obtained using unexported field
tipo errado                  → panic: reflect: call of reflect.Value.SetString on int Value
c.Porta = 8080
CanSet por valor: false | por ponteiro: true
```

| Regra | Por quê |
| --- | --- |
| Para alterar, passe um **ponteiro** e use `.Elem()` | Só assim o `Value` refere-se à variável original (é **endereçável**) |
| Campos **não exportados** podem ser lidos, mas não alterados | A reflexão respeita o encapsulamento de pacotes (ver [[Criando Pacotes]]) |
| Os métodos de `Value` checam o *kind* **em tempo de execução** | Erros que o compilador pegaria viram **panics** |

É por isso que se escreve `json.Unmarshal(dados, &u)` com **`&`**: sem o ponteiro, o decodificador não teria como preencher `u`.

---

## 🏷️ 3. Lendo struct tags: um mini-codificador CSV

O uso mais comum de reflection é percorrer os campos de uma struct qualquer, guiado pelas tags. Um codificador CSV completo, usando o iterador `Value.Fields()` do **Go 1.26**, com saída real:

```go
type Pedido struct {
	ID      int     `csv:"id"`
	Cliente string  `csv:"cliente"`
	Total   float64 `csv:"total,%.2f"`
	interno string  // não exportado: ignorado
	Notas   string  `csv:"-"` // explicitamente ignorado
}

// LinhaCSV gera uma linha CSV de QUALQUER struct, guiada pelas tags.
func LinhaCSV(x any) (cabecalho, linha string) {
	v := reflect.ValueOf(x)
	if v.Kind() == reflect.Pointer {
		v = v.Elem()
	}
	if v.Kind() != reflect.Struct {
		panic("LinhaCSV: esperava uma struct, veio " + v.Kind().String())
	}
	var nomes, valores []string
	for campo, valor := range v.Fields() { // Go 1.26+: iterador de (StructField, Value)
		if !campo.IsExported() {
			continue
		}
		tag := campo.Tag.Get("csv")
		if tag == "-" {
			continue
		}
		nome, formato, _ := strings.Cut(tag, ",")
		if nome == "" {
			nome = strings.ToLower(campo.Name)
		}
		if formato == "" {
			formato = "%v"
		}
		nomes = append(nomes, nome)
		valores = append(valores, fmt.Sprintf(formato, valor.Interface()))
	}
	return strings.Join(nomes, ","), strings.Join(valores, ",")
}
```

```
id,cliente,total
7,Ana,99.90

campo Total: tipo=float64 offset=24 tag="csv:\"total,%.2f\""
```

É, em miniatura, o que `encoding/json`, `encoding/xml`, drivers de banco (`sqlx`, ORMs), validadores e bibliotecas de configuração fazem. Repare na convenção das tags: `chave:"valor"` separadas por espaço, o primeiro elemento é o **nome**, as opções vêm depois de vírgulas, e `"-"` significa "ignore". O `StructField` também revela o **offset** do campo dentro da struct (24 bytes aqui: `ID` ocupa 8 e o `string` `Cliente` ocupa 16), o mesmo layout de [[Structs]].

> 💡 Os iteradores do Go 1.26: `Type.Fields()`, `Type.Methods()`, `Type.Ins()`, `Type.Outs()` (parâmetros e resultados de funções), `Value.Fields()` e `Value.Methods()`. Antes deles, o padrão era `for i := range t.NumField() { f := t.Field(i) ... }`, que continua válido.

---

## ☎️ 4. Chamando métodos e comparando valores

Saídas reais:

```go
type Usuario struct{ Nome string; Idade int }
func (u Usuario) Saudacao(prefixo string) string { return prefixo + ", " + u.Nome }

u := Usuario{"Ana", 30}
m := reflect.ValueOf(u).MethodByName("Saudacao")
res := m.Call([]reflect.Value{reflect.ValueOf("Olá")})
fmt.Println("chamada por nome:", res[0])      // chamada por nome: Olá, Ana

fmt.Println(reflect.TypeFor[Usuario](), reflect.TypeFor[error]().Kind())
// main.Usuario interface

a := map[string][]int{"x": {1, 2}}
b := map[string][]int{"x": {1, 2}}
fmt.Println(reflect.DeepEqual(a, b))          // true (== nem compila para maps e slices)
```

| Recurso | Uso |
| --- | --- |
| `MethodByName`, `Call` | Sistemas de plugins e RPC (o `net/rpc` descobre métodos assim) |
| `reflect.MakeFunc` | Criar funções em tempo de execução (raro) |
| `reflect.New(t)`, `MakeSlice`, `MakeMap` | Criar valores de um tipo conhecido só em execução (o decodificador JSON faz isso) |
| `reflect.DeepEqual` | Comparação profunda, útil em testes (mas veja as armadilhas) |
| `t.Implements(iface)` | Saber se um tipo satisfaz uma interface |

---

## 🐢 5. Quanto custa

Medição real (Go 1.27): ler o campo `Y` de um `*Ponto`:

```
direto: p.Y                               0.97 ns/op    0 allocs/op
reflect: v.Field(1).Int()                 3.68 ns/op    0 allocs/op
reflect: v.FieldByName("Y").Int()        66.00 ns/op    0 allocs/op
reflect: ValueOf(p).Elem()... (tudo)     79.78 ns/op    0 allocs/op
reflect: Interface() de volta            27.78 ns/op    1 allocs/op
```

| Fonte do custo | Detalhe |
| --- | --- |
| Busca por **nome** | `FieldByName` compara strings: ~70× mais lento que o acesso direto |
| Verificações em execução | Cada método confere o *kind*, a endereçabilidade, a exportação |
| Voltar para `any` | `Interface()` pode **alocar** (empacotar o valor numa interface) |
| Sem inlining nem otimização | O compilador não enxerga através do `reflect` |

A estratégia das bibliotecas sérias é **refletir uma vez por tipo e guardar o resultado**: o `encoding/json` analisa cada tipo na primeira vez (quais campos, nomes, opções), monta uma função de codificação especializada e a guarda num cache (`sync.Map`, ver [[sync.WaitGroup e sync.Mutex]]); nas chamadas seguintes, usa o equivalente ao `Field(i)` por índice, a linha de 3,7 ns.

> 💡 Reflection também tem um custo de **tamanho de binário**: se o programa usa `Value.Method`/`MethodByName` com nomes não constantes, o linker não pode descartar métodos não usados de nenhum tipo exportado, porque qualquer um pode ser chamado por nome.

---

## ⚖️ 6. Reflection ou generics?

| Situação | Use |
| --- | --- |
| O mesmo algoritmo para vários tipos **conhecidos em compilação** (somar, ordenar, `Map`/`Filter`) | **Generics** (ver [[Generics]]): verificado pelo compilador, rápido |
| Comportamento que depende de um conjunto de métodos | **Interfaces** |
| Tipos **desconhecidos** até a execução, estrutura arbitrária (codificar qualquer struct, preencher a partir de um banco, ler tags) | **Reflection** |
| Gerar código especializado na compilação | **`go generate`** (ver [[Build, Flags e Cross-compilation]]): rápido como código escrito à mão |

> ✅ "Clear is better than clever. Reflection is never clear." (um dos *Go Proverbs*, ver [[Convenções da linguagem]]). Reflection é a ferramenta certa para **bibliotecas de infraestrutura**; em código de aplicação, quase sempre há uma alternativa mais simples.

---

## 🔬 7. Por dentro: de onde vem a informação

### 7.1 O descritor de tipo

Para cada tipo usado no programa, o compilador grava no binário um **descritor**. O de Go 1.27 (`internal/abi/type.go`, simplificado):

```go
type Type struct {
	Size_       uintptr  // tamanho em bytes
	PtrBytes    uintptr  // quantos bytes do início podem conter ponteiros (para o GC)
	Hash        uint32   // hash do tipo (maps, type switches)
	TFlag       TFlag
	Align_      uint8
	FieldAlign_ uint8
	Kind_       Kind     // o kind: struct, ptr, int...
	Equal       func(unsafe.Pointer, unsafe.Pointer) bool   // como comparar com ==
	GCData      *byte    // mapa de quais palavras são ponteiros
	Str         NameOff  // o nome, como deslocamento numa tabela de strings
	PtrToThis   TypeOff  // o tipo *T, se existir
}
// tipos compostos estendem esta struct: StructType tem a lista de campos
// (nome, tipo, offset, tag); FuncType, os parâmetros; e assim por diante
```

Repare que o **mesmo** descritor serve a três clientes: o **GC** (`PtrBytes`, `GCData`, ver [[Garbage Collector]]), o **runtime** (`Equal` e `Hash` para maps e `==` em interfaces, ver [[Maps]]) e o **`reflect`**. Os nomes e tags ficam em tabelas compactadas e são referenciados por deslocamentos (`NameOff`), para economizar espaço.

### 7.2 Do `any` ao `reflect.Value`

```
var x any = Ponto{1, 2}            eface: [ _type ──▶ descritor de main.Ponto | data ──▶ {1, 2} ]
                                                       (Size_=16, Kind_=struct, campos X, Y)
reflect.ValueOf(x)
   → reflect.Value{ typ_: o mesmo *abi.Type, ptr: data, flag: kind + "endereçável?" + "veio de campo não exportado?" }
```

`reflect.TypeOf` só **lê a primeira palavra** da interface ([[Interfaces]]); `reflect.ValueOf` guarda as duas, mais *flags* que implementam as regras da seção 2.1. Não há mágica: é a mesma informação que a type assertion já usava ([[Type Assertion e Type Switch]]), exposta como API.

### 7.3 Paralelo com outras linguagens e com o SO

| Linguagem / sistema | Informação de tipos em execução |
| --- | --- |
| C | Nenhuma: o binário não sabe o que é uma struct (só o DWARF de depuração sabe, ver [[Build, Flags e Cross-compilation]]) |
| C++ | RTTI limitado (`typeid`, `dynamic_cast`), sem lista de campos |
| Java / C# | Reflexão completa, inclusive carregar classes em execução |
| **Go** | Reflexão de tipos, campos, métodos e tags; **sem** criar tipos novos nomeados nem carregar código em execução |
| Depuradores (gdb, Delve) | Usam DWARF, não o `reflect` |

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `ValueOf(x)` e tentar alterar | `using unaddressable value` | `ValueOf(&x).Elem()` |
| Alterar campo não exportado | Panic | Exporte o campo, ou não use reflection para isso |
| Chamar `Int()` num `string` | `call of reflect.Value.Int on string Value` | `switch v.Kind()` antes |
| `FieldByName` num laço quente | ~70× mais lento | Descubra os índices uma vez e guarde (cache por tipo) |
| Esquecer o caso do ponteiro | `Kind() == ptr`, `NumField` dá panic | `if v.Kind() == reflect.Pointer { v = v.Elem() }` |
| `DeepEqual` com valores que contêm funções, ou `nil` vs vazio | `false` inesperado (`[]int{}` ≠ `nil`) | Comparações específicas (`slices.Equal`, `maps.Equal`, `cmp` em testes) |
| Tag mal formada (`json:nome` sem aspas) | Ignorada em silêncio | `go vet` (analisador `structtag`) |
| Reflection onde generics resolveriam | Código lento e frágil | Generics |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Reflection | Examinar e manipular tipos e valores em execução |
| `Type` / `Value` | `TypeOf`/`TypeFor` e `ValueOf`; começam de um `any` |
| Tipo vs kind | Nome exato vs categoria; código reflexivo faz `switch Kind()` |
| Leis | Interface → reflexão; reflexão → interface (`Interface()`); alterar exige valor ajustável |
| Alterar | Ponteiro + `Elem()`; só campos exportados |
| Tags | `Field.Tag.Get("chave")`; convenção `nome,opções` e `"-"` |
| Iteradores (1.26) | `Type.Fields/Methods/Ins/Outs`, `Value.Fields/Methods` |
| Custo | `Field(i)` ~4× o direto; `FieldByName` ~70×; cache por tipo |
| Quando usar | Tipos desconhecidos em compilação; senão generics ou interfaces |
| Por dentro | O descritor `abi.Type` que o GC, o runtime e o `reflect` compartilham |

---

## 🔗 Conexões

**Dentro de Go**
- As duas palavras de uma interface e a itab → [[Interfaces]]
- Descritores de tipo e type switches → [[Type Assertion e Type Switch]]
- Tags e layout de structs → [[Structs]]
- O maior usuário de reflection da stdlib → [[JSON (marshal e unmarshal)]]
- A alternativa verificada em compilação → [[Generics]]
- Iteradores (`range` sobre `Fields()`) → [[Iteradores]]
- Cache por tipo com `sync.Map` → [[sync.WaitGroup e sync.Mutex]]
- O mapa de ponteiros do GC no descritor → [[Garbage Collector]]
- `Equal`/`Hash` usados por maps → [[Maps]]
- Encapsulamento por exportação → [[Criando Pacotes]]
- "Reflection is never clear" → [[Convenções da linguagem]]
- Geração de código como alternativa → [[Build, Flags e Cross-compilation]]
- Ir além das regras com `unsafe` → [[unsafe e cgo]]

**Sistemas Operacionais**
- [[A Linguagem C]] — sem informação de tipos em execução
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — o que vai para o binário e o que fica só na compilação
- [[Paravirtualização e a Máquina Virtual Java]] — a JVM, com reflexão e carregamento de classes em execução

---

⬅️ **Anterior:** [[Modelo de Memória]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[unsafe e cgo]] ➡️
