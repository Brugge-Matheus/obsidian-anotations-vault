---
tags:
  - go
  - go/oop
---
# Métodos e Receptores

> Métodos associam comportamento a tipos. A diferença entre um método e uma função comum é apenas o receptor — um parâmetro extra que aparece antes do nome da função. Qualquer tipo nomeado dentro do mesmo pacote pode ter métodos.
> 

---

## 1. Métodos vs Funções — O Que Muda

```go
// Função comum
func area(r Retangulo) float64 {
	return r.Largura * r.Altura
}

// Mesmo código como método — receptor aparece antes do nome
func (r Retangulo) Area() float64 {
	return r.Largura * r.Altura
}

// Uso
ret := Retangulo{Largura: 10, Altura: 5}
fmt.Println(area(ret))    // chamada como função
fmt.Println(ret.Area())   // chamada como método (açúcar sintático)
// O compilador transforma ret.Area() em Retangulo.Area(ret)
```

---

## 2. Receptor por Valor vs por Ponteiro

Esta é a distinção mais importante de métodos em Go.

### Receptor por Valor — Recebe Cópia

```go
type Retangulo struct {
	Largura, Altura float64
}

func (r Retangulo) Area() float64 {
	return r.Largura * r.Altura
}

func (r Retangulo) Perimetro() float64 {
	return 2 * (r.Largura + r.Altura)
}

// r é uma CÓPIA — qualquer modificação não afeta o original
func (r Retangulo) Escalar(fator float64) Retangulo {
	r.Largura *= fator   // modifica a cópia
	r.Altura *= fator
	return r   // retorna a cópia modificada
}
```

### Receptor por Ponteiro — Recebe Referência

```go
// r é o ORIGINAL — modificações são visíveis fora
func (r *Retangulo) EscalarInPlace(fator float64) {
	r.Largura *= fator
	r.Altura  *= fator
}
```

### Auto-deref e Auto-endereçamento

Go converte automaticamente entre valor e ponteiro ao chamar métodos:

```go
r := Retangulo{10, 5}
rp := &r

// Auto-endereçamento: r.EscalarInPlace(2) → (&r).EscalarInPlace(2)
r.EscalarInPlace(2)   // Go adiciona & automaticamente

// Auto-deref: rp.Area() → (*rp).Area()
fmt.Println(rp.Area())   // Go faz o deref automaticamente

// MAS: isso só funciona para variáveis endereçáveis
// Não funciona com valores temporários:
// Retangulo{10, 5}.EscalarInPlace(2)   ❌ cannot take address of temporary
```

---

## 3. Quando Usar Cada Receptor

A regra é: **seja consistente dentro de um tipo**. Se qualquer método usa ponteiro, use ponteiro em todos:

| Situação | Receptor |
| --- | --- |
| Método modifica o struct | `*T` — obrigatório |
| Struct é grande (> ~64 bytes) | `*T` — evita cópia |
| Tipo tem outros métodos `*T` | `*T` — consistência |
| Struct pequena, método não modifica | `T` — ok |
| Tipo tem semântica de valor (Ponto, Cor, Duração) | `T` |

> 💡 A inconsistência importa porque **o method set de `T` é diferente do de `*T`**: `T` tem apenas métodos com receptor `T`; `*T` tem métodos com receptor `T` E `*T`. Para satisfazer interfaces, isso é crucial.
> 

---

## 4. Method Set — Quais Métodos um Tipo Tem

```go
type Contador struct{ n int }

func (c Contador) Valor() int { return c.n }     // receptor por valor
func (c *Contador) Incrementar() { c.n++ }       // receptor por ponteiro

// Method set de Contador (valor):    {Valor}
// Method set de *Contador (ponteiro): {Valor, Incrementar}

type Incrementavel interface {
	Incrementar()
}

var i Incrementavel

c := Contador{}
// i = c   ❌ Contador não implementa Incrementavel (Incrementar tem receptor *Contador)
i = &c      // ✅ *Contador implementa Incrementavel

// Isso é por que interfaces são geralmente satisfeitas com ponteiros
```

---

## 5. Métodos em Tipos Além de Structs

Qualquer tipo **nomeado** no mesmo pacote pode ter métodos. Tipos baseados em tipos primitivos são muito úteis para adicionar semântica:

```go
// Enum-like com métodos
type DiaSemana int

const (
	Segunda DiaSemana = iota + 1
	Terca
	Quarta
	Quinta
	Sexta
	Sabado
	Domingo
)

func (d DiaSemana) String() string {
	nomes := [...]string{"", "Segunda", "Terça", "Quarta", "Quinta", "Sexta", "Sábado", "Domingo"}
	if d < Segunda || d > Domingo {
		return fmt.Sprintf("DiaSemana(%d)", int(d))
	}
	return nomes[d]
}

func (d DiaSemana) Util() bool {
	return d >= Segunda && d <= Sexta
}

fmt.Println(Quarta)          // "Quarta" (usa String())
fmt.Println(Sabado.Util())   // false

// Método em tipo slice
type Pilha[T any] []T

func (p *Pilha[T]) Push(v T) { *p = append(*p, v) }
func (p *Pilha[T]) Pop() (T, bool) {
	if len(*p) == 0 {
		var zero T
		return zero, false
	}
	s := *p
	v := s[len(s)-1]
	*p = s[:len(s)-1]
	return v, true
}
func (p Pilha[T]) Peek() (T, bool) {
	if len(p) == 0 {
		var zero T
		return zero, false
	}
	return p[len(p)-1], true
}

var st Pilha[int]
st.Push(1); st.Push(2); st.Push(3)
v, ok := st.Pop()
fmt.Println(v, ok)   // 3, true
```

---

## 6. Convenção de Nome do Receptor

Use 1-2 letras representativas do tipo, minúsculas. **Nunca** use `self`, `this` ou `me`:

```go
// ✅ Convenção Go
func (r Retangulo) Area() float64 { ... }
func (u *Usuario) Salvar() error { ... }
func (srv *Servidor) Iniciar() error { ... }

// ❌ Anti-pattern (estilo Python/Ruby)
func (self Retangulo) Area() float64 { ... }
func (this *Usuario) Salvar() error { ... }

// ✅ Consistência — use sempre o mesmo nome para o mesmo tipo
func (u *Usuario) Salvar() error { ... }
func (u *Usuario) Deletar() error { ... }
func (u *Usuario) Validar() error { ... }
```

---

## 7. Method Values e Method Expressions

```go
type Calculadora struct{ acumulador float64 }

func (c *Calculadora) Somar(n float64) { c.acumulador += n }
func (c Calculadora) Valor() float64  { return c.acumulador }

calc := &Calculadora{}

// Method value — função ligada à instância específica
somar := calc.Somar   // tipo: func(float64)
somar(10)
somar(20)
fmt.Println(calc.Valor())   // 30

// Method expression — função que recebe o receptor como primeiro argumento
// Útil para passar como callback genérico
somarExpr := (*Calculadora).Somar   // tipo: func(*Calculadora, float64)
somarExpr(calc, 5)
fmt.Println(calc.Valor())   // 35

// Uso prático: sort.Slice com method expression
sort.Slice(itens, func(i, j int) bool {
	return itens[i].Prioridade() < itens[j].Prioridade()
})
```

---

## 8. Interfaces `fmt.Stringer` e `error`

As duas interfaces mais importantes para métodos:

```go
// fmt.Stringer — controla como o tipo é exibido pelo fmt
type Stringer interface { String() string }

type Temperatura struct {
	Celsius float64
}

func (t Temperatura) String() string {
	return fmt.Sprintf("%.1f°C (%.1f°F)", t.Celsius, t.Celsius*9/5+32)
}

temp := Temperatura{37.0}
fmt.Println(temp)          // 37.0°C (98.6°F)
fmt.Printf("%v\n", temp)  // 37.0°C (98.6°F)
fmt.Printf("%s\n", temp)  // 37.0°C (98.6°F)

// error — implementar erro customizado
type ErrTemperatura struct {
	Valor float64
	Motivo string
}

func (e *ErrTemperatura) Error() string {
	return fmt.Sprintf("temperatura inválida %.1f°C: %s", e.Valor, e.Motivo)
}

func validarTemp(t Temperatura) error {
	if t.Celsius < -273.15 {
		return &ErrTemperatura{t.Celsius, "abaixo do zero absoluto"}
	}
	return nil
}
```

---

## 9. Exemplo Completo — Conta Bancária

```go
type ContaBancaria struct {
	id      int64
	titular string
	saldo   float64
}

func NovaConta(titular string, deposito inicial float64) (*ContaBancaria, error) {
	if titular == "" {
		return nil, errors.New("titular obrigatório")
	}
	if inicial < 0 {
		return nil, errors.New("depósito inicial não pode ser negativo")
	}
	return &ContaBancaria{
		id:      gerarID(),
		titular: titular,
		saldo:   inicial,
	}, nil
}

func (c *ContaBancaria) Depositar(valor float64) error {
	if valor <= 0 {
		return fmt.Errorf("depositar: valor deve ser positivo, recebido %.2f", valor)
	}
	c.saldo += valor
	return nil
}

func (c *ContaBancaria) Sacar(valor float64) error {
	if valor <= 0 {
		return fmt.Errorf("sacar: valor deve ser positivo, recebido %.2f", valor)
	}
	if valor > c.saldo {
		return fmt.Errorf("sacar: saldo insuficiente (%.2f < %.2f)", c.saldo, valor)
	}
	c.saldo -= valor
	return nil
}

func (c ContaBancaria) Saldo() float64 { return c.saldo }

func (c ContaBancaria) String() string {
	return fmt.Sprintf("Conta[%d] %s: R$ %.2f", c.id, c.titular, c.saldo)
}
```

---

## 10. Como o Compilador Implementa Métodos Internamente

Go transforma métodos em funções comuns onde o receptor é simplesmente o primeiro argumento. Isso é exatamente o que o compilador C faz com `struct` + ponteiro de função.

```go
// O que você escreve:
func (r Retangulo) Area() float64 {
    return r.Largura * r.Altura
}

// O que o compilador gera internamente (pseudo-código):
func Retangulo·Area(r Retangulo) float64 {   // receptor = 1º parâmetro
    return r.Largura * r.Altura
}

// Chamada: ret.Area()  →  Retangulo·Area(ret)
```

### Receptor por Valor — Passagem na Stack

```
Stack frame de ret.Area():
┌──────────────────────────────────┐  ← SP (stack pointer)
│ cópia de r:                      │
│   Largura float64  (8 bytes)     │
│   Altura  float64  (8 bytes)     │
├──────────────────────────────────┤
│ valor de retorno float64         │
├──────────────────────────────────┤
│ endereço de retorno (caller)     │
└──────────────────────────────────┘

Go usa a calling convention do plano (plan9 ABI com registradores):
- Argumentos pequenos passam em registradores (AX, BX, CX, ...).
- A cópia da struct acontece no momento da chamada.
- Struct grande (> registradores disponíveis) → transborda para a stack.
```

### Receptor por Ponteiro — Apenas o Endereço

```
Stack frame de (&ret).EscalarInPlace(2.0):
┌──────────────────────────────────┐
│ r *Retangulo  (8 bytes — ptr)    │  ← aponta para ret no frame do caller
│ fator float64 (8 bytes)          │
├──────────────────────────────────┤
│ endereço de retorno              │
└──────────────────────────────────┘

Custo: passagem de 8 bytes (ponteiro), independente do tamanho da struct.
Modifica diretamente a memória original — sem cópia.
```

### Analogia com C

```c
// Em C, "métodos" são escritos como funções com ponteiro explícito:
void Retangulo_EscalarInPlace(Retangulo* self, double fator) {
    self->Largura *= fator;
    self->Altura  *= fator;
}

// Go faz exatamente isso, mas com sintaxe de ponto:
// ret.EscalarInPlace(2)  →  compilado como Retangulo·EscalarInPlace(&ret, 2)
```

---

## 11. Method Set e a Construção do itab

Quando um tipo satisfaz uma interface, o runtime Go monta uma **itab** (interface table) que lista os ponteiros de função de cada método. A regra de method set determina se a itab pode ser montada:

```
Method set de T  (valor):    métodos com receptor T
Method set de *T (ponteiro): métodos com receptor T  +  métodos com receptor *T

Interface Incrementavel requer: Incrementar()
  - Contador  → method set = {Valor}         → NÃO satisfaz ✗
  - *Contador → method set = {Valor, Incrementar} → satisfaz ✓

Quando *Contador é atribuído a Incrementavel, o runtime:
1. Localiza (ou cria) a itab para (*Contador, Incrementavel)
2. Armazena o ponteiro da função Incrementar na tabela
3. Atribuição = {itab*, data*} com data* apontando para o Contador

itab para (*Contador, Incrementavel):
┌──────────────────────────────────┐
│ inter   *interfacetype           │  → descreve Incrementavel
│ type    *_type                   │  → descreve *Contador
│ hash    uint32                   │  → para comparações rápidas
│ _       [4]byte                  │
│ fun[0]  uintptr                  │  → &(*Contador).Incrementar
└──────────────────────────────────┘
```

---

## 12. Conexão com Sistemas Operacionais

### Value Receiver e Calling Convention — [[Processadores]]

Go usa uma ABI (Application Binary Interface) baseada em registradores (desde Go 1.17). O receptor de valor é passado como qualquer outro argumento: em registradores para tipos pequenos, na stack para tipos grandes. Isso é idêntico a como o processador lida com qualquer chamada de função — o "método" é apenas açúcar sintático que o compilador desfaz antes de gerar código de máquina.

### Pointer Receiver e Endereçamento de Memória — [[Gerenciamento de Memória]]

Com `*T`, Go passa um ponteiro de 8 bytes (em arquiteturas de 64 bits). O receptor trabalha diretamente com o endereço de memória da struct original — sem cópia. Isso é o mesmo mecanismo que `malloc`/ponteiros em C: o valor real fica em um endereço fixo no heap ou stack, e quem precisa modificá-lo recebe o endereço.

### Método = Função com Primeiro Argumento Implícito — [[Processadores]]

Internamente, `ret.Area()` compila para `Retangulo·Area(ret)`. A ABI do processador não sabe nada de "métodos" — vê apenas uma chamada de função com argumentos em registradores. O compilador Go (assim como o compilador C para `struct + ponteiro`) é quem faz a tradução. Debuggers e profilers veem o símbolo `main.Retangulo.Area` — o ponto na notação indica o receptor.

### Method Set e a itab do Runtime — [[Gerenciamento de Memória]]

As regras de method set (`T` vs `*T`) determinam se o runtime consegue montar a **itab** para um par (tipo concreto, interface). A itab é alocada estaticamente pelo linker quando possível, ou dinamicamente no heap. É essencialmente uma tabela de ponteiros de função — o mesmo conceito de `struct file_operations` no kernel Linux (ver abaixo).

### Dispatch de Métodos via Interface — Analogia com VFS do Kernel — [[System Calls]]

O kernel Linux implementa o VFS (Virtual File System) com uma `struct file_operations` que contém ponteiros de função (`read`, `write`, `ioctl`, `mmap`, etc.). Cada driver de dispositivo ou sistema de arquivos preenche essa struct com suas próprias funções. Chamar `read()` em um arquivo invoca o ponteiro `file_operations->read` do driver — dispatch dinâmico via tabela de funções.

```c
/* kernel Linux — include/linux/fs.h (simplificado) */
struct file_operations {
    ssize_t (*read)  (struct file*, char __user*, size_t, loff_t*);
    ssize_t (*write) (struct file*, const char __user*, size_t, loff_t*);
    int     (*open)  (struct inode*, struct file*);
    int     (*release)(struct inode*, struct file*);
    /* ... dezenas de outros ponteiros de função ... */
};
```

```go
// Go faz exatamente o mesmo com interfaces + métodos:
type File interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
    Close() error
}
// A itab de qualquer tipo que implemente File contém ponteiros para
// Read, Write e Close — estruturalmente idêntico a file_operations.
```

A diferença é que Go verifica em tempo de compilação se o método set está correto, enquanto em C os ponteiros podem ser `NULL` (resultando em kernel panic se chamados).
```