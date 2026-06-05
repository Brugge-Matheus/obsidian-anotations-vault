---
tags:
  - go
  - go/fundamentos
---
# Structs

> Structs são tipos compostos que agrupam campos nomeados. Em Go, structs têm semântica de valor, layout de memória controlável e são a base de toda "orientação a objetos" da linguagem — sem classes, sem herança.
> 

---

## 1. Layout de Memória

O compilador de Go organiza os campos de uma struct na memória seguindo regras de **alignment** (alinhamento). Cada tipo tem um requisito de alinhamento (geralmente igual ao seu tamanho, até 8 bytes):

```go
// Layout ineficiente — padding desperdiça memória
type Ruim struct {
	A bool      // 1 byte + 7 bytes de padding para alinhar B
	B int64     // 8 bytes
	C bool      // 1 byte + 7 bytes de padding
	D int64     // 8 bytes
}
// sizeof(Ruim) = 32 bytes (16 bytes de padding!)

// Layout eficiente — campos maiores primeiro
type Bom struct {
	B int64     // 8 bytes
	D int64     // 8 bytes
	A bool      // 1 byte + 7 bytes de padding (apenas no final)
	C bool      // 1 byte
}
// sizeof(Bom) = 24 bytes (apenas 6 bytes de padding)
```

```go
import "unsafe"

fmt.Println(unsafe.Sizeof(Ruim{}))   // 32
fmt.Println(unsafe.Sizeof(Bom{}))    // 24

// Ver o offset de cada campo
fmt.Println(unsafe.Offsetof(Ruim{}.B))   // 8 (após 1 byte de A + 7 de padding)
fmt.Println(unsafe.Offsetof(Bom{}.B))    // 8 (após os 8 bytes de B)
```

> 💡 Para structs que serão armazenadas em grandes slices ou usadas como chave de map, a ordem dos campos pode ter impacto significativo no uso de memória. O compilador Go **não reordena** campos automaticamente (diferente de C++ com `std::is_standard_layout`).
> 

---

## 2. Definindo Structs

```go
// Forma básica
type Pessoa struct {
	Nome  string
	Idade int
	Email string
}

// Campos do mesmo tipo agrupados
type Retangulo struct {
	Largura, Altura float64   // mesma linha, mesmo tipo
}

// Struct com tipos variados
type Produto struct {
	ID       int64
	Nome     string
	Preco    float64
	Estoque  int
	Ativo    bool
}

// Struct com campos exportados e não-exportados
type ContaBancaria struct {
	ID      int64    // exportado (maiúscula)
	Titular string   // exportado
	saldo   float64  // não exportado — só acessível dentro do pacote
	limite  float64  // não exportado
}
```

---

## 3. Criando Instâncias

```go
// Por nome dos campos — recomendada (resistente a mudanças na struct)
p1 := Pessoa{
	Nome:  "Alice",
	Idade: 30,
	Email: "alice@exemplo.com",
}

// Por posição — frágil, não use em produção
// Se a struct mudar, o código compila mas com bugs
p2 := Pessoa{"Bob", 25, "bob@exemplo.com"}   // ⚠️ evitar

// Campos omitidos recebem zero value
p3 := Pessoa{Nome: "Carol"}   // Idade=0, Email=""

// var — todos os campos com zero value
var p4 Pessoa

// new — retorna *Pessoa com zero value
// Equivale a: pp := &Pessoa{}
pp := new(Pessoa)
fmt.Println(pp)   // &{ 0 }
```

---

## 4. Acesso a Campos e Ponteiros

```go
p := Pessoa{Nome: "Alice", Idade: 30}

// Acesso via ponto
fmt.Println(p.Nome)    // "Alice"
p.Idade = 31           // modificar campo

// Ponteiro para struct — Go faz auto-deref
pp := &p
fmt.Println(pp.Nome)   // equivale a (*pp).Nome
pp.Idade = 32          // equivale a (*pp).Idade = 32

// Endereço de campo
endNome := &p.Nome    // *string apontando para o campo Nome de p
*endNome = "Bob"
fmt.Println(p.Nome)   // "Bob"
```

---

## 5. Semântica de Valor — Cópia ao Atribuir

```go
p1 := Pessoa{Nome: "Alice", Idade: 30}
p2 := p1   // cópia COMPLETA de todos os campos — sem referência compartilhada

p2.Nome = "Bob"
fmt.Println(p1.Nome)   // "Alice" — não foi alterado
fmt.Println(p2.Nome)   // "Bob"

// Para compartilhar o mesmo objeto, use ponteiro
p3 := &p1
p3.Nome = "Carol"
fmt.Println(p1.Nome)   // "Carol" — foi alterado!
```

> 💡 A cópia ao atribuir é uma proteção: você nunca precisa "clonar" uma struct explicitamente. Mas para structs grandes (dezenas de campos), passe por ponteiro em funções para evitar copiar muitos bytes.
> 

---

## 6. Structs como Parâmetros de Função

```go
// Por valor — recebe uma cópia (não modifica o original)
func envelhecer(p Pessoa) Pessoa {
	p.Idade++
	return p   // retorna a cópia modificada
}

// Por ponteiro — modifica o original diretamente
func envelhecerInPlace(p *Pessoa) {
	p.Idade++
}

alice := Pessoa{Nome: "Alice", Idade: 30}

nova := envelhecer(alice)
fmt.Println(alice.Idade)   // 30 — não mudou
fmt.Println(nova.Idade)    // 31

envelhecerInPlace(&alice)
fmt.Println(alice.Idade)   // 31 — mudou!
```

---

## 7. Comparação de Structs

Structs são comparáveis com `==` se **todos os campos forem comparáveis**. A comparação é campo a campo, na ordem de declaração:

```go
type Ponto struct{ X, Y int }

p1 := Ponto{1, 2}
p2 := Ponto{1, 2}
p3 := Ponto{3, 4}

fmt.Println(p1 == p2)   // true — todos os campos iguais
fmt.Println(p1 == p3)   // false

// Struct como chave de map (todos os campos devem ser comparáveis)
visitas := map[Ponto]int{}
visitas[Ponto{0, 0}]++

// Struct com slice — NÃO é comparável
type Config struct {
	Tags []string   // slice não é comparável
}
// c1 == c2   ❌ erro de compilação

// Use reflect.DeepEqual para comparação profunda (lenta)
import "reflect"
c1 := Config{Tags: []string{"a", "b"}}
c2 := Config{Tags: []string{"a", "b"}}
fmt.Println(reflect.DeepEqual(c1, c2))   // true
```

---

## 8. Struct Anônima

Útil para estruturas temporárias ou de uso único — muito comum em testes:

```go
// Variável com struct anônima
ponto := struct {
	X, Y int
}{X: 10, Y: 20}

// Table-driven tests — padrão idiomático
casos := []struct {
	entrada  int
	esperado int
	nome     string
}{
	{0, 0, "zero"},
	{2, 4, "positivo"},
	{-1, 1, "negativo"},
}

for _, tc := range casos {
	t.Run(tc.nome, func(t *testing.T) {
		got := quadradoAbsoluto(tc.entrada)
		if got != tc.esperado {
			t.Errorf("quadradoAbsoluto(%d) = %d, want %d", tc.entrada, got, tc.esperado)
		}
	})
}

// Struct anônima em JSON — sem precisar criar um tipo nomeado
response := struct {
	Status  string `json:"status"`
	Versao  string `json:"versao"`
}{
	Status: "ok",
	Versao: "1.0.0",
}
json.NewEncoder(w).Encode(response)
```

---

## 9. Tags de Struct

Tags são **metadados em tempo de compilação** anexados a campos. São strings brutas que bibliotecas leem via reflection em runtime:

```go
import (
	"encoding/json"
	"reflect"
)

type Usuario struct {
	ID    int    `json:"id" db:"id"`
	Nome  string `json:"nome" db:"nome" validate:"required,min=2,max=100"`
	Email string `json:"email,omitempty" db:"email" validate:"required,email"`
	Senha string `json:"-" db:"senha_hash"`    // json:"-" = nunca serializa
}

// Como as tags são lidas em runtime
t := reflect.TypeOf(Usuario{})
campo := t.Field(1)   // campo Nome
fmt.Println(campo.Tag.Get("json"))      // "nome"
fmt.Println(campo.Tag.Get("validate"))  // "required,min=2,max=100"

// Opções JSON importantes
type Exemplo struct {
	A string  `json:"a"`             // nome customizado
	B string  `json:"b,omitempty"`   // omite se zero value
	C string  `json:"-"`             // nunca serializa
	D int     `json:",string"`       // serializa número como "42" (string)
	E string  `json:",omitempty"`    // omite, nome original "E"
}
```

---

## 10. Embedding — Composição de Structs

Embedding inclui os campos e métodos de um tipo dentro de outro, promovendo-os ao nível externo. É a base da "herança" em Go:

```go
type Endereco struct {
	Rua    string
	Cidade string
	Estado string
}

func (e Endereco) Formatado() string {
	return fmt.Sprintf("%s, %s - %s", e.Rua, e.Cidade, e.Estado)
}

type Pessoa struct {
	Nome     string
	Idade    int
	Endereco         // embedding — sem nome de campo
}

p := Pessoa{
	Nome:  "Alice",
	Idade: 30,
	Endereco: Endereco{
		Rua:    "Rua das Flores, 123",
		Cidade: "São Paulo",
		Estado: "SP",
	},
}

// Campos e métodos promovidos
fmt.Println(p.Rua)             // "Rua das Flores, 123" — promovido!
fmt.Println(p.Formatado())     // "Rua das Flores, 123, São Paulo - SP" — promovido!
fmt.Println(p.Endereco.Rua)   // também funciona (acesso explícito)
```

### Sobrescrita de Métodos Promovidos

```go
type Animal struct{ Nome string }
func (a Animal) Falar() string { return "..." }

type Gato struct {
	Animal
	Cor string
}

// Sobrescreve o método promovido
func (g Gato) Falar() string { return "Miau" }

cat := Gato{Animal: Animal{Nome: "Tom"}, Cor: "cinza"}
fmt.Println(cat.Falar())          // "Miau" — versão do Gato
fmt.Println(cat.Animal.Falar())   // "..."  — versão do Animal (ainda acessível)
```

### Embedding de Interface

É possível embutir uma interface dentro de uma struct — o tipo deve implementar os métodos da interface:

```go
type Leitor interface{ Ler() []byte }

type LeitorComBuffer struct {
	Leitor           // embedding de interface
	buffer []byte
}

// LeitorComBuffer precisa implementar Leitor, ou fornecer um campo
// que o satisfaça em tempo de construção
```

---

## 11. Padrões Comuns

### Constructor Function

```go
type Servidor struct {
	host    string
	porta   int
	timeout time.Duration
}

// Retorna ponteiro — indica que o objeto tem identidade e será modificado
func NovoServidor(host string, porta int) (*Servidor, error) {
	if host == "" {
		return nil, errors.New("host obrigatório")
	}
	if porta < 1 || porta > 65535 {
		return nil, fmt.Errorf("porta inválida: %d", porta)
	}
	return &Servidor{
		host:    host,
		porta:   porta,
		timeout: 30 * time.Second,   // valor padrão
	}, nil
}
```

### Implementar `fmt.Stringer`

```go
type Ponto struct{ X, Y float64 }

func (p Ponto) String() string {
	return fmt.Sprintf("(%.2f, %.2f)", p.X, p.Y)
}

p := Ponto{3.14159, 2.71828}
fmt.Println(p)            // (3.14, 2.72) — usa String()
fmt.Printf("%v\n", p)    // (3.14, 2.72)
fmt.Printf("%+v\n", p)   // {X:3.14159 Y:2.71828} — ignora String()
fmt.Printf("%#v\n", p)   // main.Ponto{X:3.14159, Y:2.71828}
```

### Struct Vazia — Zero Bytes

```go
// struct{} é o único tipo com tamanho zero em Go
// Todos os valores struct{} compartilham o mesmo endereço (runtime.zerobase)
fmt.Println(unsafe.Sizeof(struct{}{}))   // 0

// Usos práticos:
// 1. Set com map[T]struct{} (sem overhead de valor)
visitado := map[string]struct{}{}
visitado["alice"] = struct{}{}

// 2. Sinal em channel (sem dado, apenas sinalização)
done := make(chan struct{})
go func() {
	// trabalho...
	close(done)   // sinaliza sem enviar dados
}()
<-done

// 3. Implementar interface vazia para "marcador de tipo"
type Marker interface{ isMarker() }
type MarcadorConcreto struct{}
func (MarcadorConcreto) isMarker() {}
```

---

## 12. Quando Usar Valor vs Ponteiro

| Situação | Use Valor (`T`) | Use Ponteiro (`*T`) |
| --- | --- | --- |
| Struct pequena (≤ 2-3 campos) | ✅ | Desnecessário |
| Struct grande | ❌ (cópia cara) | ✅ |
| Método modifica o struct | ❌ | ✅ obrigatório |
| nil é estado válido | ❌ | ✅ |
| Comparação com `==` | ✅ (compara campos) | ❌ (compara endereços) |
| Chave de map | ✅ (se comparável) | ❌ (compara endereços) |
| Implementar interface com métodos ptr | — | ✅ obrigatório |

---

## Conexão com Sistemas Operacionais

### Regras de alinhamento de memória — [[Processadores]]

As regras de padding mostradas na seção 1 (`Ruim` vs `Bom`) existem porque o **barramento de memória** da CPU transfere dados alinhados à sua word size. Em [[Processadores]] vimos que o barramento de dados conecta CPU e RAM, e uma transferência é mais eficiente quando o dado começa num endereço múltiplo do seu tamanho. Um `int64` em endereço ímpar pode exigir **dois ciclos de barramento** em vez de um, ou até causar uma exceção de hardware em arquiteturas como ARM sem suporte a acesso não-alinhado. O compilador Go insere padding automaticamente para garantir que cada campo esteja no endereço correto.

### Padding e largura do barramento — [[Processadores]]

O padding de struct também tem relação com o alinhamento de **cache lines** (64 bytes em x86-64). Quando uma struct tem seu tamanho otimizado (como `Bom` com 24 bytes vs `Ruim` com 32 bytes), mais instâncias cabem em uma única cache line. Em [[Processadores]] vimos que o processador opera em blocos alinhados — structs com padding excessivo podem fazer com que campos frequentemente acessados juntos fiquem em cache lines diferentes, aumentando o número de cache misses.

### Layout de struct e hierarquia de memória — [[Memória]]

Para structs armazenadas em grandes slices (e.g., `[]Produto` com milhões de elementos), o tamanho de cada struct afeta diretamente quantos elementos cabem em uma cache line. Em [[Memória]] estudamos a hierarquia de memória: quanto menor a struct, mais elementos por cache line, melhor a localidade espacial e menor a pressão sobre a cache L1/L2. Reordenar campos para minimizar padding é uma otimização de baixo nível equivalente à escolha de layout de dados em C para sistemas de alta performance.

### Campos contíguos na memória — [[Espaços de Endereçamento]]

Os campos de uma struct ficam em endereços **contíguos** no espaço de endereçamento virtual do processo (com exceção do padding). Em [[Espaços de Endereçamento]] vimos que o espaço de endereçamento virtual é linear — um array de bytes endereçável. Uma struct é exatamente um "recorte" desse espaço: `unsafe.Offsetof` retorna o deslocamento em bytes de cada campo a partir do início da struct, que é o endereço base da instância.

### Structs em C vs structs em Go — [[A Linguagem C]]

Em [[A Linguagem C]] vimos que C também possui structs com as mesmas regras de alinhamento e padding. A diferença em Go é que o compilador **não reordena** os campos (mantém a ordem de declaração, assim como C), o que garante compatibilidade de layout na interface com código C via `cgo`. Structs decoradas com tags específicas podem ser usadas diretamente em chamadas de sistema ou em interfaces com bibliotecas C, desde que o layout de memória seja idêntico.

### Semântica de valor (cópia) vs ponteiro — [[Processos]]

Passar uma struct por valor copia todos os bytes dos campos (seção 6). Isso é análogo à semântica de argumentos por cópia em C. Em [[Processos]] o conceito de **copy-on-write (COW)** aparece no contexto de fork: o SO evita copiar páginas de memória até que uma escrita ocorra. Go não usa COW para structs — a cópia é sempre imediata e completa, o que garante que modificar uma cópia nunca afeta o original. Passar por ponteiro (`*Struct`) é a forma de obter semântica de referência, equivalente a passar `struct *p` em C — ambos evitam a cópia e permitem modificar o dado original.