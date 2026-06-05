---
tags:
  - go
  - go/oop
---
# Composição vs Herança

> Go não tem herança. Em vez disso, usa **embedding** (composição estrutural) e **interfaces** (composição comportamental). O resultado é mais flexível — você pode compor comportamentos de múltiplas fontes sem as armadilhas da herança múltipla.
> 

---

## 1. O Problema da Herança

```
Herança cria acoplamento rígido:

class Animal {           class Cachorro extends Animal {
  respirar() {}             latir() {}
  dormir() {}            }
}

Problemas:
1. Frágil hierarquia: mudança em Animal quebra todos os subtipos
2. Problema do diamante: múltipla herança cria ambiguidade
3. Rigidez: CachorroRobo precisa latir mas não respirar — hierarquia quebra

Go resolve com composição:
- Você inclui comportamentos que precisa
- Sem hierarquia rígida
- Múltipla "herança" sem problemas
```

---

## 2. Embedding — Campos Promovidos

Ao embutir um tipo sem nomear o campo, seus campos e métodos são **promovidos** ao tipo externo:

```go
type Animal struct {
	Nome string
	Peso float64
}

func (a Animal) Respirar() {
	fmt.Printf("%s está respirando\n", a.Nome)
}

func (a Animal) Dormir() {
	fmt.Printf("%s está dormindo\n", a.Nome)
}

type Cachorro struct {
	Animal   // embedding — os campos e métodos de Animal são promovidos
	Raca    string
}

func (c Cachorro) Latir() {
	fmt.Printf("%s: Au au!\n", c.Nome)   // c.Nome = c.Animal.Nome (promovido)
}

dog := Cachorro{
	Animal: Animal{Nome: "Rex", Peso: 25.0},
	Raca:   "Labrador",
}

// Campos e métodos promovidos — acesso direto
dog.Respirar()       // "Rex está respirando"
dog.Dormir()         // "Rex está dormindo"
dog.Latir()          // "Rex: Au au!"
dog.Nome             // "Rex" — promovido de Animal
dog.Animal.Nome      // "Rex" — acesso explícito também funciona
dog.Animal.Respirar() // também funciona
```

---

## 3. Como Embedding Funciona Internamente

```go
Cachorro em memória:
┌────────────────────────────┐
│ Animal:                    │
│   Nome  string  (ptr+len)  │
│   Peso  float64            │
├────────────────────────────┤
│ Raca    string  (ptr+len)  │
└────────────────────────────┘

dog.Nome     → compila para: dog.Animal.Nome
dog.Respirar → compila para: dog.Animal.Respirar
(zero overhead — é apenas açúcar sintático do compilador)

A struct Animal é literalmente embutida nos bytes da struct Cachorro.
Não é uma referência — é composição estrutural real.
```

---

## 4. Sobrescrita de Métodos Promovidos

```go
type Animal struct{ Nome string }
func (a Animal) Falar() string { return "..." }
func (a Animal) Descrever() string {
	return fmt.Sprintf("Sou %s e falo: %s", a.Nome, a.Falar())
}

type Gato struct {
	Animal
	Cor string
}

// Sobrescreve o método promovido
func (g Gato) Falar() string { return "Miau" }

gato := Gato{Animal: Animal{Nome: "Tom"}, Cor: "cinza"}

fmt.Println(gato.Falar())            // "Miau" — método do Gato
fmt.Println(gato.Animal.Falar())     // "..."  — método original, ainda acessível

// ⚠️ Cuidado: Descrever ainda chama Animal.Falar, não Gato.Falar!
fmt.Println(gato.Descrever())
// "Sou Tom e falo: ..."  ← não "Miau"!
// Go não tem virtual dispatch — sem polimorfismo dinâmico via embedding

// Para polimorfismo verdadeiro, use interfaces:
type Falante interface { Falar() string }
func descrever(nome string, f Falante) string {
	return fmt.Sprintf("Sou %s e falo: %s", nome, f.Falar())
}
fmt.Println(descrever(gato.Nome, gato))   // "Sou Tom e falo: Miau" ✅
```

---

## 5. Múltiplo Embedding — A Vantagem sobre Herança

```go
type Nadador struct{}
type Corredor struct{}
type Voador struct{}

func (n Nadador) Nadar() string { return "nadando" }
func (r Corredor) Correr() string { return "correndo" }
func (v Voador) Voar() string { return "voando" }

// Triatlo sem herança múltipla — sem ambiguidade
type Pato struct {
	Nadador
	Corredor
	Voador
	Nome string
}

donald := Pato{Nome: "Donald"}
fmt.Println(donald.Nadar())   // "nadando"
fmt.Println(donald.Correr())  // "correndo"
fmt.Println(donald.Voar())    // "voando"

// Robô que corre mas não respira
type Robo struct {
	Corredor
	Modelo string
}

r := Robo{Modelo: "T-800"}
fmt.Println(r.Correr())   // "correndo"
```

---

## 6. Embedding de Interfaces — Composição Comportamental

```go
// Interfaces podem embutir outras interfaces
type Leitor interface {
	Ler(p []byte) (n int, err error)
}

type Escritor interface {
	Escrever(p []byte) (n int, err error)
}

type Fechavel interface {
	Fechar() error
}

// Composição de interfaces
type LeitorEscritorFechavel interface {
	Leitor
	Escritor
	Fechavel
}

// Embedding de interface em struct — força implementação
type BufferLimitado struct {
	io.Reader   // embedding da interface
	limite int
}

func (b *BufferLimitado) Read(p []byte) (int, error) {
	if len(p) > b.limite {
		p = p[:b.limite]
	}
	return b.Reader.Read(p)   // delega para o Reader embebido
}
```

---

## 7. Composição com Injeção de Dependência

O padrão real de uso em produção — compor comportamentos via interfaces:

```go
// Interfaces pequenas e focadas
type Logger interface {
	Info(msg string, args ...any)
	Error(msg string, args ...any)
}

type Metricas interface {
	Registrar(nome string, valor float64)
}

// Serviço compõe comportamentos via embedding de interfaces
type ServicoUsuario struct {
	logger   Logger    // injetado
	metricas Metricas  // injetado
	repo     RepositorioUsuario
}

func (s *ServicoUsuario) Criar(req CriarUsuarioRequest) (*Usuario, error) {
	inicio := time.Now()

	u, err := s.repo.Inserir(req)
	if err != nil {
		s.logger.Error("falha ao criar usuário", "err", err)
		return nil, fmt.Errorf("criar usuário: %w", err)
	}

	s.metricas.Registrar("usuarios.criados", 1)
	s.logger.Info("usuário criado", "id", u.ID, "elapsed", time.Since(inicio))
	return u, nil
}

// Para testes: injetar implementações simples
type loggerSilencioso struct{}
func (l *loggerSilencioso) Info(msg string, args ...any)  {}
func (l *loggerSilencioso) Error(msg string, args ...any) {}

svc := &ServicoUsuario{
	logger:   &loggerSilencioso{},   // sem ruído nos testes
	metricas: &metricasNoop{},
	repo:     mockRepo,
}
```

---

## 8. Herança vs Composição — Comparação Direta

```go
// Como seria herança em Go (não existe, mas ilustrativo)
// PHP:
// class Veiculo { protected $vel; public function acelerar($v) { $this->vel += $v; } }
// class Carro extends Veiculo { public function buzinar() { } }

// Equivalente com composição em Go:
type Veiculo struct {
	Velocidade float64
}

func (v *Veiculo) Acelerar(delta float64) {
	v.Velocidade += delta
}

func (v Veiculo) VelocidadeAtual() float64 {
	return v.Velocidade
}

type Carro struct {
	Veiculo   // composição — não herança
	Marca string
}

func (c Carro) Buzinar() string {
	return fmt.Sprintf("%s: beep!", c.Marca)
}

// Uso idêntico ao que seria com herança
carro := Carro{Marca: "Toyota"}
carro.Acelerar(100)                    // método promovido
fmt.Println(carro.VelocidadeAtual())   // 100
fmt.Println(carro.Buzinar())           // Toyota: beep!
```

---

## 9. Aceitar Interfaces, Retornar Tipos Concretos

A regra de ouro de Go para APIs:

```go
// ❌ Acoplado — força o caller a usar PostgresDB especificamente
func criarServico(db *PostgresDB) *ServicoUsuario { ... }

// ✅ Flexível — aceita qualquer implementação
type DB interface {
	Query(ctx context.Context, sql string, args ...any) (Rows, error)
	Exec(ctx context.Context, sql string, args ...any) error
}

func criarServico(db DB) *ServicoUsuario { ... }
// Funciona com PostgresDB, MySQL, SQLite, mock, etc.

// Retornar tipo concreto (não interface) permite evolução da API
// sem quebrar código existente
func novoServico(db DB) *ServicoUsuario {   // retorna *ServicoUsuario, não interface
	return &ServicoUsuario{db: db}
}
```

---

## 10. Quando Usar Embedding vs Interface

| Situação | Embedding | Interface |
| --- | --- | --- |
| Reutilizar campos | ✅ — campos são promovidos | ❌ — interfaces não têm campos |
| Reutilizar implementação | ✅ — métodos são promovidos | ❌ — só define comportamento |
| Polimorfismo dinâmico | ❌ — sem virtual dispatch | ✅ — dispatch em runtime |
| Testar com mock | Difícil | ✅ — qualquer struct que implemente |
| Múltipla "herança" | ✅ — sem ambiguidade se nomes únicos | ✅ — compose interfaces |
| Extender tipo externo | ❌ — não pode adicionar métodos a tipos externos | ✅ — criar nova interface que inclui |

---

## 11. Como Embedding Funciona Internamente — Layout de Memória

```go
type Animal struct {
    Nome string   // 16 bytes (ptr + len)
    Peso float64  // 8 bytes
}

type Cachorro struct {
    Animal        // embedding — embutido nos primeiros bytes
    Raca string   // 16 bytes após Animal
}
```

```
Layout de Cachorro na memória (offset em bytes):
┌──────────────────────────────────────┐
│ offset  0: Animal.Nome.ptr  (8 b)    │
│ offset  8: Animal.Nome.len  (8 b)    │
│ offset 16: Animal.Peso      (8 b)    │
├──────────────────────────────────────┤
│ offset 24: Raca.ptr         (8 b)    │
│ offset 32: Raca.len         (8 b)    │
└──────────────────────────────────────┘
Total: 40 bytes

dog.Nome     →  *(*string)(unsafe.Pointer(&dog) + 0)
dog.Peso     →  *(*float64)(unsafe.Pointer(&dog) + 16)
dog.Raca     →  *(*string)(unsafe.Pointer(&dog) + 24)

Acesso = aritmética de ponteiro pura.
Zero overhead — sem indireção, sem ponteiros extras, sem vtable.
```

O tipo embutido está **dentro** da struct, não referenciado por ponteiro. Isso significa que `Cachorro` e `Animal` compartilham os mesmos bytes — `&dog.Animal == (*Animal)(unsafe.Pointer(&dog))`.

### Métodos Promovidos — O Compilador Gera Wrappers

```go
// Você escreve:
dog.Respirar()

// O compilador gera (em tempo de compilação, sem custo em runtime):
func (c Cachorro) Respirar() {   // wrapper gerado automaticamente
    c.Animal.Respirar()          // delega ao método do tipo embutido
}

// Para pointer receiver:
func (c *Cachorro) Respirar() {
    c.Animal.Respirar()   // ou (&c.Animal).Respirar() se necessário
}
```

Esses wrappers são candidatos à inlining pelo compilador — se `Animal.Respirar()` é pequena, o compilador pode inlinar tudo, resultando em zero overhead.

### Múltiplo Embedding — Sem Conflito de Offset

```go
type Pato struct {
    Nadador    // offset 0 (tamanho de Nadador)
    Corredor   // offset após Nadador
    Voador     // offset após Corredor
    Nome string
}
// Cada tipo embutido ocupa seu espaço sequencial na struct.
// Sem ponteiros extras — composição estrutural real.
```

---

## 12. Embedding no Kernel Linux — container_of

O kernel Linux usa extensivamente o mesmo padrão de embedding que Go. A macro `container_of` é o equivalente ao que o compilador Go faz automaticamente:

```c
/* Lista duplamente encadeada genérica do kernel (include/linux/list.h) */
struct list_head {
    struct list_head *next, *prev;
};

/* Qualquer struct pode "herdar" list_head por embedding: */
struct tarefa {
    int   pid;
    char  nome[16];
    struct list_head lista;   /* embutido — mesmo padrão do Go */
};

/* Para obter o ponteiro da tarefa a partir do list_head embutido: */
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))

/* Go faz isso automaticamente:
   dog.Animal  ↔  container_of(list_ptr, struct tarefa, lista)
   Ambos são aritmética de ponteiro sobre o offset do campo. */
```

---

## 13. Conexão com Sistemas Operacionais

### Embedding = Aritmética de Ponteiro — [[Processadores]]

Acessar `dog.Nome` é literalmente `load [base + 0]` em assembly — o compilador conhece o offset de cada campo em tempo de compilação e gera uma instrução de acesso à memória com deslocamento fixo. Não há ponteiro extra, não há desvio de execução. A CPU busca o dado diretamente do endereço calculado: `endereço_do_cachorro + offset(Animal.Nome)`. Esse é o padrão mais eficiente possível — sem indireção.

### Zero Overhead vs C++ Virtual Inheritance — [[Processadores]]

Em C++, herança virtual adiciona um ponteiro `vptr` em cada objeto e um nível de indireção para acessar campos de classes base. Em Go, embedding não gera nenhum ponteiro extra — é apenas offset fixo. Isso tem implicações de cache: a struct inteira (incluindo o tipo embutido) está em memória contígua, maximizando o uso de cache lines da CPU. Uma cache line de 64 bytes pode conter vários campos de diferentes tipos embutidos sem nenhum miss adicional.

### Métodos Promovidos e Inlining — [[Processadores]]

Os wrappers gerados pelo compilador para métodos promovidos são funções normais que apenas delegam a chamada. O compilador Go é agressivo com inlining: se o método do tipo embutido for pequeno, o wrapper inteiro desaparece e o código é inserido diretamente no call site. O resultado é idêntico a ter escrito o método diretamente no tipo externo — zero custo em runtime.

### Kernel Linux usa Composição, não Herança — [[System Calls]]

O kernel Linux é escrito em C (sem herança) e usa embedding de structs extensivamente para abstração. O padrão `container_of` permite obter o ponteiro da struct contêiner a partir de um campo embutido — exatamente o que Go faz automaticamente. Drivers de dispositivo definem `struct file_operations` (tabela de ponteiros de função) e a embutem ou associam à sua struct principal — composição comportamental, idêntico a como Go compõe interfaces.

```c
/* Driver de dispositivo no kernel — composição sem herança: */
struct meu_dispositivo {
    struct cdev       cdev;           /* embutido — como Go embedding */
    struct file_operations fops;      /* tabela de funções — como itab */
    spinlock_t        lock;
    void __iomem      *base_addr;
};
/* Não há hierarquia de classes — apenas structs compostas. */
```

### Favor Composition over Inheritance — Lição do Kernel — [[System Calls]]

O "Fragile Base Class Problem" (mudança na classe base quebra todas as subclasses) não existe no kernel Linux nem em Go porque ambos usam composição. Quando um novo sistema de arquivos é adicionado ao kernel, ele implementa `struct file_operations` do zero — não herda de um "sistema de arquivos base". Da mesma forma, em Go, um novo tipo apenas implementa os métodos de uma interface — sem herança, sem acoplamento à implementação de outros tipos.