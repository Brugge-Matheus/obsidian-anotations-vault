---
tags:
  - go
  - go/oop
---
# Interfaces

> Interfaces definem comportamento sem especificar implementação. A implementação é implícita — sem declarar `implements`. Internamente, uma interface é um par (tipo, valor) que permite polimorfismo em tempo de execução.
> 

---

## 1. Implementação Interna de Interfaces

Uma variável de interface em Go armazena dois ponteiros:

```
Interface (iface):
┌──────────────┬──────────────┐
│   *itab      │   *data      │
└──────────────┴──────────────┘
       │               │
       ▼               ▼
┌──────────────┐  ┌──────────┐
│ itab:        │  │ valor    │ (copiado para heap se necessário)
│ - *tipo      │  │ concreto │
│ - *interfTipo│  └──────────┘
│ - hash       │
│ - fun[...]   │  ← tabela de ponteiros para métodos
└──────────────┘

Para interface{} / any (eface — empty interface):
┌──────────────┬──────────────┐
│   *_type     │   *data      │
└──────────────┴──────────────┘
```

Isso tem implicações importantes:

- Verificar se uma interface é `nil` **exige que AMBOS os ponteiros sejam nil**
- Chamar um método via interface = deref do itab + call indireto (um nível de indireção)
- Para tipos pequenos (≤ pointer size), o valor é armazenado diretamente no campo `data`

---

## 2. Declaração e Implementação

```go
type Forma interface {
	Area() float64
	Perimetro() float64
}

// Implementação implícita — sem palavra-chave implements
type Retangulo struct{ Largura, Altura float64 }

func (r Retangulo) Area() float64 { 
	return r.Largura * r.Altura 
}

func (r Retangulo) Perimetro() float64 { return 2 * (r.Largura + r.Altura) }

type Circulo struct{ Raio float64 }
func (c Circulo) Area() float64      { return math.Pi * c.Raio * c.Raio }
func (c Circulo) Perimetro() float64 { return 2 * math.Pi * c.Raio }

// Ambos satisfazem Forma — sem nenhuma declaração
var f1 Forma = Retangulo{10, 5}
var f2 Forma = Circulo{7}
```

### Verificar Implementação em Tempo de Compilação

```go
// Padrão para garantir que um tipo satisfaz uma interface (erro de compilação se não)
var _ Forma = Retangulo{}    // verifica Retangulo
var _ Forma = (*Circulo)(nil)  // verifica *Circulo
```

---

## 3. Polimorfismo

```go
func imprimirInfo(f Forma) {
	fmt.Printf("%T: área=%.2f, perímetro=%.2f\n", f, f.Area(), f.Perimetro())
}

formas := []Forma{
	Retangulo{10, 5},
	Circulo{7},
	Retangulo{3, 3},
}

for _, f := range formas {
	imprimirInfo(f)
	// A chamada f.Area() usa dispatch via itab — O(1), um único call indireto
}
```

---

## 4. Interface Vazia — `any`

```go
// interface{} e any são idênticos (any é alias desde Go 1.18)
var v any

v = 42
v = "texto"
v = []int{1, 2, 3}

// any usa eface (sem itab) — mais leve que iface
// Mas operações com any exigem type assertion para recuperar o valor
fmt.Printf("%T %v\n", v, v)

// ⚠️ any perde segurança de tipo em compilação
// Use interfaces específicas sempre que possível
```

---

## 5. Interface Nil vs Ponteiro Nil em Interface

A armadilha mais comum com interfaces em Go:

```go
type Erro interface{ Error() string }

func buscar(id int) *MeuErro {
	if id == 0 {
		return nil   // retorna *MeuErro nil
	}
	return &MeuErro{"não encontrado"}
}

// ❌ Bug clássico
func verificar(id int) Erro {
	err := buscar(id)
	return err   // PROBLEMA: retorna interface com tipo (*MeuErro) e valor nil
	             // A interface NÃO é nil! Tem tipo, mas o valor é nil
}

err := verificar(0)
fmt.Println(err == nil)    // false! ← bug!
fmt.Println(err)           // <nil>  ← confuso

// ✅ Correto: retornar nil diretamente como tipo interface
func verificarCorretto(id int) Erro {
	err := buscar(id)
	if err == nil {
		return nil   // retorna interface nil verdadeira (ambos os ponteiros nil)
	}
	return err
}
```

---

## 6. Composição de Interfaces

```go
// Interfaces pequenas compostas — o estilo preferido do Go
type Leitor interface {
	Ler() ([]byte, error)
}

type Escritor interface {
	Escrever([]byte) error
}

type Fechavel interface {
	Fechar() error
}

// Composição: combina as três interfaces
type LeitorEscritorFechavel interface {
	Leitor
	Escritor
	Fechavel
}

// Exemplo da stdlib — io package usa esse padrão extensamente:
// io.Reader, io.Writer, io.Closer, io.ReadWriter, io.ReadWriteCloser...
```

---

## 7. Interfaces da Stdlib — As Mais Importantes

```go
// fmt.Stringer — controla impressão
type Stringer interface { String() string }

// error — tratamento de erros
type error interface { Error() string }

// io.Reader — leitura de dados
type Reader interface { Read(p []byte) (n int, err error) }

// io.Writer — escrita de dados
type Writer interface { Write(p []byte) (n int, err error) }

// io.Closer
type Closer interface { Close() error }

// sort.Interface — ordenação customizada
type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

// http.Handler — handler HTTP
type Handler interface { ServeHTTP(ResponseWriter, *Request) }

// context.Context — propagação de cancelamento
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

### Implementar `sort.Interface`

```go
type Pessoa struct{ Nome string; Idade int }

type PorIdade []Pessoa
func (p PorIdade) Len() int           { return len(p) }
func (p PorIdade) Less(i, j int) bool { return p[i].Idade < p[j].Idade }
func (p PorIdade) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

pessoas := []Pessoa{{"Bob", 30}, {"Alice", 25}, {"Carol", 35}}
sort.Sort(PorIdade(pessoas))

// Go 1.21+: sort.Slice com genéricos é mais simples
slices.SortFunc(pessoas, func(a, b Pessoa) int {
	return cmp.Compare(a.Idade, b.Idade)
})
```

---

## 8. Interfaces Pequenas São Melhores

```
"The bigger the interface, the weaker the abstraction."
— Rob Pike
```

```go
// ❌ Interface gigante — difícil de satisfazer, difícil de mockar
type RepositorioUsuario interface {
	Criar(*Usuario) error
	BuscarPorID(int) (*Usuario, error)
	BuscarPorEmail(string) (*Usuario, error)
	Atualizar(*Usuario) error
	Deletar(int) error
	ListarTodos() ([]*Usuario, error)
	Contar() (int, error)
}

// ✅ Interfaces focadas — cada função pede apenas o que precisa
type CriadorUsuario interface { Criar(*Usuario) error }
type LeitorUsuario interface { BuscarPorID(int) (*Usuario, error) }

// A função de envio de boas-vindas só precisa ler
func enviarBoasVindas(leitor LeitorUsuario, id int) error {
	u, err := leitor.BuscarPorID(id)
	// ...
}
// Pode ser testada com um mock de uma única função!
```

---

## 9. Interfaces para Testabilidade

```go
// Interface para dependência externa
type Mailer interface {
	Enviar(dest, assunto, corpo string) error
}

// Implementação real
type SMTPMailer struct{ Host string }
func (m *SMTPMailer) Enviar(dest, assunto, corpo string) error {
	// envia via SMTP...
	return nil
}

// Implementação de teste (mock manual — sem biblioteca)
type MockMailer struct {
	Enviados []string
	Erro     error
}
func (m *MockMailer) Enviar(dest, assunto, corpo string) error {
	if m.Erro != nil {
		return m.Erro
	}
	m.Enviados = append(m.Enviados, dest)
	return nil
}

// Serviço aceita a interface — não a implementação
type ServicoUsuario struct {
	mailer Mailer
}
func (s *ServicoUsuario) Registrar(email string) error {
	return s.mailer.Enviar(email, "Bem-vindo!", "...")
}

// Teste
func TestRegistrar(t *testing.T) {
	mock := &MockMailer{}
	svc := &ServicoUsuario{mailer: mock}

	err := svc.Registrar("alice@ex.com")

	if err != nil || len(mock.Enviados) != 1 || mock.Enviados[0] != "alice@ex.com" {
		t.Fatal("email não enviado corretamente")
	}
}
```

---

## 10. Generics e Interfaces (Go 1.18+)

Com generics, interfaces podem ser usadas como **type constraints** além de polimorfismo dinâmico:

```go
// Interface como constraint — define quais tipos podem ser usados
type Numerico interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64 |
	~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
	~float32 | ~float64
}

func Somar[T Numerico](a, b T) T {
	return a + b
}

fmt.Println(Somar(3, 4))      // 7 (int)
fmt.Println(Somar(3.14, 2.0)) // 5.14 (float64)

// O ~ (til) significa "qualquer tipo cujo tipo base seja..."
// ~int inclui type Celsius float64, type MyInt int, etc.

// comparable — constraint embutida para tipos que suportam ==
func Contem[T comparable](slice []T, item T) bool {
	for _, v := range slice {
		if v == item {
			return true
		}
	}
	return false
}
```

---

## 11. Resumo

| Conceito | Detalhe |
| --- | --- |
| Implementação | Implícita — apenas ter os métodos certos |
| Interface nil | Ambos os ponteiros (tipo, valor) devem ser nil |
| Interface com `*T` nil | NÃO é nil — tem tipo, valor nil |
| Method set de `T` | Métodos com receptor `T` |
| Method set de `*T` | Métodos com receptor `T`  • `*T` |
| `any` | Alias de `interface{}` desde Go 1.18 |
| Constraint genérica | `~T` = T e qualquer tipo com base T |
| Performance | Call via interface = 1 indireção (itab) |

---

## 12. Internals Detalhados — Como a Interface Existe na Memória

### iface vs eface

Go tem dois layouts internos de interface:

```
iface — interface com pelo menos um método (e.g. io.Reader):
┌─────────────────────┬─────────────────────┐
│  tab  *itab         │  data  unsafe.Pointer│
└─────────────────────┴─────────────────────┘
  8 bytes                8 bytes
  │                      │
  ▼                      ▼
┌───────────────────┐  ┌────────────────────┐
│ itab:             │  │ valor concreto      │
│  inter *itype     │  │ (copiado p/ heap    │
│  type  *_type     │  │  se > 8 bytes)      │
│  hash  uint32     │  └────────────────────┘
│  fun[0] uintptr   │  ← ponteiro p/ método 0
│  fun[1] uintptr   │  ← ponteiro p/ método 1
│  ...              │
└───────────────────┘

eface — interface{} / any (zero métodos):
┌─────────────────────┬─────────────────────┐
│  _type  *_type      │  data  unsafe.Pointer│
└─────────────────────┴─────────────────────┘
  Sem itab — apenas o descritor de tipo + ponteiro para o dado.
  Mais simples, mas perde informação sobre métodos.
```

### Chamada de Método via Interface — Passo a Passo

```go
var f Forma = Retangulo{10, 5}
area := f.Area()   // como o runtime executa isso?
```

```
1. Carrega f.tab (ponteiro para itab)         → registrador RAX
2. Carrega f.data (ponteiro para Retangulo)   → registrador RBX
3. Lê itab.fun[0] (ponteiro para Area)        → registrador RCX
4. Chama *RCX(RBX)   ← call indireto

Assembly esquemático (amd64):
  MOVQ  f+0(SP), AX    ; AX = *itab
  MOVQ  f+8(SP), BX    ; BX = *data (receptor)
  MOVQ  24(AX), CX     ; CX = itab.fun[0] = &Retangulo.Area
  CALL  CX             ; chama Area com BX como receptor
```

Cada chamada via interface custa: 1 load (itab) + 1 load (fun[N]) + 1 call indireto. O processador não pode predizer o branch facilmente — pode causar branch misprediction em loops críticos de performance.

### O Nil Gotcha — Por Que `(*T)(nil) != nil` em Interface

```
Interface nil verdadeira:         Interface com tipo mas valor nil:
┌──────────┬──────────┐           ┌──────────┬──────────┐
│  nil     │  nil     │           │  *itab   │  nil     │
└──────────┴──────────┘           └──────────┴──────────┘
  == nil → true                     == nil → false ← SURPRESA

// O campo tab não é nil! A interface "sabe" que é *MeuErro,
// mas o ponteiro para o dado concreto é nil.
// Go verifica AMBOS os campos para nil.
```

### Comparação com Outras Linguagens

```
Go interface:         C void*:             C++ virtual:
┌────┬────┐          ┌────┐               ┌──────────┐  vtable:
│itab│data│          │ptr │               │ *vptr────┼──►[*dtor  ]
└────┴────┘          └────┘               │ campos   │  [*method1]
                                          └──────────┘  [*method2]
Explícito,           Sem tipo —           Implícito,
verificável          cast manual          por herança

Java interface:
┌──────────────────────────────────────────┐
│ header (8b) │ vtable ptr │ campos...     │
└──────────────────────────────────────────┘
Vtable por herança — cada objeto carrega ponteiro.
Go: a itab é por (tipo, interface) — não por objeto.
```

---

## 13. Conexão com Sistemas Operacionais

### Layout de Memória da Interface — [[Gerenciamento de Memória]]

Uma interface Go ocupa exatamente 16 bytes (dois ponteiros de 8 bytes em 64-bit). O campo `data` pode apontar para o heap (se o valor não cabe em um ponteiro) ou armazenar o valor inline (se cabe em 8 bytes, como `int`, `float64`, ponteiros). O garbage collector rastreia o campo `data` como uma raiz de GC — precisa saber que é um ponteiro, não um inteiro. Essa distinção usa o mapa de ponteiros embutido no `_type`.

### Dispatch Dinâmico e Predição de Branch — [[Processadores]]

Chamar `i.Method()` compila para um **call indireto** — o processador não sabe para onde vai até ler o valor de `itab.fun[N]` em runtime. CPUs modernas têm preditores de branch para calls indiretos (BTB — Branch Target Buffer), mas se o mesmo site de chamada dispacha para muitos alvos diferentes (e.g., loop com 10 tipos distintos), o preditor falha, custando ~10-15 ciclos por misprediction. Isso é o mesmo problema de vtables em C++.

### Empty Interface e o Sistema de Tipos do Runtime — [[Gerenciamento de Memória]]

O campo `_type` de um `eface` (empty interface) aponta para um descritor de tipo que o runtime usa para: reflection, comparação, hashing, GC (mapa de ponteiros), impressão com `%v`. Esse descritor é alocado estaticamente no segmento de dados do binário pelo compilador — não no heap. É análogo aos símbolos de tipo em tabelas DWARF que debuggers usam para inspecionar variáveis.

### Interface Nil Gotcha e Semântica de Ponteiros — [[Gerenciamento de Memória]]

A armadilha `(*T)(nil) != nil` quando armazenado em interface existe porque Go codifica o tipo no primeiro campo da interface. Um ponteiro nulo em C (`NULL`) e um "ponteiro nulo tipado" são indistinguíveis em C — ambos são `(void*)0`. Go os distingue explicitamente: `(nil, nil)` é interface nil; `(*MeuErro, nil)` não é. Essa semântica está diretamente ligada a como o runtime precisa saber **o tipo** para chamar métodos ou fazer type assertion.

### Interfaces do Kernel Linux — VFS como Go Interface em C — [[System Calls]]

O VFS do kernel Linux usa `struct file_operations` e `struct inode_operations` — structs de ponteiros de função — para implementar dispatch polimórfico entre sistemas de arquivos (ext4, tmpfs, proc, NFS). É conceitualmente idêntico à itab de Go:

```c
/* Kernel — struct file_operations (simplificada) */
struct file_operations {
    ssize_t (*read)   (struct file*, char __user*, size_t, loff_t*);
    ssize_t (*write)  (struct file*, const char __user*, size_t, loff_t*);
    int     (*open)   (struct inode*, struct file*);
    int     (*release)(struct inode*, struct file*);
    long    (*ioctl)  (struct file*, unsigned int, unsigned long);
};

/* Quando você chama read(fd, buf, n) no userspace:
   syscall → sys_read() → file->f_op->read(file, buf, n, &pos)
                                    ^
                          call indireto via tabela de funções
                          (exatamente como itab.fun[N] em Go) */
```

```go
// Go — io.Reader é a mesma abstração em nível de linguagem:
type Reader interface {
    Read(p []byte) (n int, err error)
}
// itab para (*os.File, io.Reader).fun[0] = &os.File.Read
// que internamente chama a syscall read(2).
```

A camada Go (interface) e a camada do kernel (file_operations) usam o mesmo padrão de design: uma tabela de ponteiros de função associada ao tipo concreto, preenchida em "tempo de registro" (compilação em Go, `module_init()` no kernel).