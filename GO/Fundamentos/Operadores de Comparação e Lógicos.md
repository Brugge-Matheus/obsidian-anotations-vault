---
tags:
  - go
  - go/fundamentos
---
# Operadores de Comparação e Lógicos

> Operadores de comparação e lógicos são a base de toda tomada de decisão em Go. Sempre retornam `bool` e são avaliados em tempo de execução pelo runtime — mas o compilador frequentemente os elimina via constant folding quando os operandos são constantes.
> 

---

## 1. Como o Compilador Trata Operadores

Antes de ver a sintaxe, é importante entender o que acontece por baixo:

```
Código Go               Instrução de máquina (x86-64)
a == b        →         CMP + SETE   (set if equal)
a < b         →         CMP + SETL   (set if less)
a && b        →         avaliação com short-circuit (dois branches)
!a            →         XOR com 1 (ou NOT)
```

Comparações numéricas em Go são **branchless** quando possível — o compilador usa instruções de comparação que não criam branches no pipeline de CPU, evitando branch misprediction. Isso é por que benchmarks de comparação em Go costumam ser extremamente rápidos.

---

## 2. Operadores de Comparação

Comparam dois valores do **mesmo tipo** e retornam `bool`.

| Operador | Instrução SSA/IR | Exemplo | Resultado |
| --- | --- | --- | --- |
| `==` | `OpEq` | `5 == 5` | `true` |
| `!=` | `OpNeq` | `5 != 3` | `true` |
| `<` | `OpLess` | `3 < 5` | `true` |
| `<=` | `OpLeq` | `5 <= 5` | `true` |
| `>` | `OpGreater` | `7 > 3` | `true` |
| `>=` | `OpGeq` | `3 >= 5` | `false` |

```go
a, b := 10, 20

fmt.Println(a == b)   // false
fmt.Println(a != b)   // true
fmt.Println(a < b)    // true
fmt.Println(a <= b)   // true
fmt.Println(a > b)    // false
fmt.Println(a >= b)   // false
```

### Comparação de Strings

Strings são comparadas **lexicograficamente byte a byte** (não caractere a caractere). Internamente Go usa `runtime.cmpstring` que chama `memcmp` otimizado pelo sistema:

```go
fmt.Println("apple" == "apple")   // true
fmt.Println("apple" != "banana")  // true
fmt.Println("apple" < "banana")   // true  — 'a'(97) < 'b'(98) em UTF-8
fmt.Println("b" > "a")            // true
fmt.Println("Go" == "go")         // false — 'G'(71) != 'g'(103)

// Strings com caracteres multibyte — comparação ainda é byte a byte
fmt.Println("café" < "cafê")      // true — 'é'(0xC3A9) < 'ê'(0xC3AA)
```

> ⚠️ Comparação de strings NÃO é O(1) — é O(n) onde n é o comprimento. Para comparação case-insensitive, use `strings.EqualFold` que é otimizado e seguro para Unicode.
> 

### Comparação de Structs

Structs são comparáveis com `==` se **todos os campos forem comparáveis**. A comparação é feita campo a campo na ordem de declaração:

```go
type Ponto struct {
	X, Y int
}

p1 := Ponto{1, 2}
p2 := Ponto{1, 2}
p3 := Ponto{3, 4}

fmt.Println(p1 == p2)   // true
fmt.Println(p1 == p3)   // false
```

> ⚠️ Slices, maps e funções **não são comparáveis** com `==` porque não têm semântica de igualdade de valor bem definida (dois slices com o mesmo conteúdo podem ser vistas diferentes do mesmo array). Use `slices.Equal` (Go 1.21+) ou `reflect.DeepEqual`.
> 

```go
import "slices"

s1 := []int{1, 2, 3}
s2 := []int{1, 2, 3}
fmt.Println(slices.Equal(s1, s2))   // true — Go 1.21+
```

---

## 3. Operadores Lógicos

| Operador | Nome | Comportamento |  |  |
| --- | --- | --- | --- | --- |
| `&&` | AND lógico | `true` somente se **ambos** forem `true` |  |  |
| `\ | \ | ` | OR lógico | `true` se **pelo menos um** for `true` |
| `!` | NOT lógico | Inverte o valor booleano |  |  |

```go
// AND
fmt.Println(true && true)    // true
fmt.Println(true && false)   // false
fmt.Println(false && true)   // false   ← segundo não é avaliado!
fmt.Println(false && false)  // false

// OR
fmt.Println(true || true)    // true    ← segundo não é avaliado!
fmt.Println(true || false)   // true    ← segundo não é avaliado!
fmt.Println(false || true)   // true
fmt.Println(false || false)  // false

// NOT
fmt.Println(!true)   // false
fmt.Println(!false)  // true
```

---

## 4. Avaliação em Curto-Circuito (Short-Circuit)

Go usa **short-circuit evaluation** — o compilador emite um branch condicional que pula a avaliação do segundo operando se o resultado já pode ser determinado pelo primeiro:

```go
// Para &&:
// if (A == false) goto resultado_false
// avalia B
// resultado = B

// Para ||:
// if (A == true) goto resultado_true
// avalia B
// resultado = B
```

Isso é **garantido pela spec** — não é uma otimização opcional. Você pode depender disso para segurança:

```go
// Proteção contra nil pointer — padrão idiomático
var usuario *Usuario
if usuario != nil && usuario.Ativo {
	fmt.Println("Usuário ativo")
}
// Se usuario == nil, usuario.Ativo nunca é acessado → sem panic

// Lazy evaluation — só chama a função cara se necessário
if resultado, ok := cache.Get(chave); ok || carregarDoBanco(&resultado, chave) {
	usar(resultado)
}
```

> 💡 O curto-circuito com `||` também é usado para defaults seguros: `valor := entrada || "padrão"` não existe em Go (só em linguagens dinâmicas), mas o padrão equivalente é `if entrada == "" { entrada = "padrão" }`.
> 

---

## 5. Precedência dos Operadores

A ordem completa de precedência em Go (maior para menor):

```
Unários:    +  -  !  ^  *  &  <-
Nível 5:    *  /  %  <<  >>  &  &^
Nível 4:    +  -  |  ^
Nível 3:    ==  !=  <  <=  >  >=
Nível 2:    &&
Nível 1:    ||   (menor precedência)
```

Entre os lógicos:

```go
// ! tem maior precedência
fmt.Println(!false && true)    // true  — (!false) && true
fmt.Println(!(false && true))  // true  — !(false)

// && tem maior precedência que ||
fmt.Println(false || true && false)    // false — false || (true && false)
fmt.Println((false || true) && false)  // false — true && false = false

// Regra de ouro: use parênteses para clareza
aprovado := (idade >= 18 && temRenda) || (temFiador && score > 700)
```

---

## 6. Comparação com Zero Values

Padrão idiomático de Go — verificar se um valor foi inicializado:

```go
var nome string
var contador int
var ativo bool
var ponteiro *int
var canal chan int
var slice []int
var mapa map[string]int

// Zero values de cada tipo
fmt.Println(nome == "")         // true
fmt.Println(contador == 0)      // true
fmt.Println(!ativo)             // true (ativo == false)
fmt.Println(ponteiro == nil)    // true
fmt.Println(canal == nil)       // true
fmt.Println(slice == nil)       // true
fmt.Println(mapa == nil)        // true

// Cuidado: slice vazio não é nil
s := make([]int, 0)
fmt.Println(s == nil)   // false! s é não-nil com len=0
fmt.Println(len(s) == 0)   // true — forma mais robusta de verificar "vazio"
```

---

## 7. Operador `==` em Interfaces

Comparação de interfaces verifica **tipo E valor** ao mesmo tempo:

```go
var a interface{} = 42
var b interface{} = 42
var c interface{} = "42"

fmt.Println(a == b)   // true  — mesmo tipo (int) e mesmo valor (42)
fmt.Println(a == c)   // false — tipos diferentes (int vs string)

// Armadilha: interface com ponteiro nil
var p *int = nil
var i interface{} = p

fmt.Println(p == nil)   // true  — ponteiro é nil
fmt.Println(i == nil)   // false! — interface tem tipo (*int), valor nil
                        // interface só é nil quando tipo E valor são nil
```

---

## 8. Comparações em Contextos de Performance

```go
// Para comparação de strings longas — considere comparar o comprimento primeiro
func iguaisOtimizado(a, b string) bool {
	return len(a) == len(b) && a == b
	// Se os comprimentos forem diferentes, evita o memcmp
}

// Para comparação de structs grandes — use campos discriminantes primeiro
type Usuario struct {
	ID    int64
	Nome  string
	Email string
	// ... outros campos
}

func (u Usuario) Igual(outro Usuario) bool {
	return u.ID == outro.ID   // ID é suficiente — evita comparar strings
}
```

---

## 9. Resumo Visual

```
Comparação:   ==  !=  <  <=  >  >=   →  sempre retorna bool
                                        strings: O(n), structs: campo a campo

Lógicos:      &&  (AND)   true somente se AMBOS forem true
              ||  (OR)    true se PELO MENOS UM for true
              !   (NOT)   inverte o bool

Precedência:  !  >  &&  >  ||

Short-circuit (garantido pela spec):
  A && B  →  se A == false, B não é avaliado
  A || B  →  se A == true,  B não é avaliado

Interfaces:   == verifica tipo E valor simultaneamente
              interface só é nil quando ambos (tipo, valor) são nil
```

---

## 10. Por Dentro: Comparações, Branches e Hardware

### Comparadores → CMP + flags EFLAGS no x86-64

```
Go source         x86-64 assembly
──────────        ────────────────────────────────
a == b      →     CMP  rax, rbx      ; subtrai e seta flags
                  SETE al            ; al = 1 se ZF=1 (equal), 0 caso contrário

a < b       →     CMP  rax, rbx
                  SETL al            ; al = 1 se SF≠OF (signed less)

a > b       →     CMP  rax, rbx
                  SETG al            ; al = 1 se ZF=0 e SF=OF (signed greater)

// CMP é basicamente SUB sem armazenar o resultado — apenas atualiza EFLAGS:
//   ZF (Zero Flag)     = 1 se resultado == 0
//   SF (Sign Flag)     = 1 se resultado < 0
//   OF (Overflow Flag) = 1 se overflow ocorreu
//   CF (Carry Flag)    = 1 se borrow ocorreu (unsigned)

// Branchless comparison (sem branch):
//   SETE, SETL, SETG escrevem 0 ou 1 no registro de destino
//   sem pular nenhuma instrução → sem pipeline stall
```

Conecta com [[Processadores]] — EFLAGS register, instruções CMP/SET*, branch prediction, instruction pipeline.

### Short-circuit `&&` e `||`: branches no código gerado

```go
// if a > 0 && b > 0 { ... }
// compila para (pseudo-assembly):

CMP   rax, 0         ; a > 0?
JLE   label_false    ; se NÃO → pula (short-circuit: b não é avaliado)
CMP   rbx, 0         ; b > 0?
JLE   label_false    ; se NÃO → pula
; corpo do if
JMP   label_end
label_false:
; else
label_end:

// Se a condição é previsível (sempre true ou sempre false):
//   branch predictor acerta → sem pipeline stall (~1 ciclo)
//
// Se a condição é aleatória (50/50):
//   branch predictor erra ~50% → pipeline flush (~15 ciclos de penalidade)
//   neste caso, branchless (CMOV) seria mais rápido
```

Conecta com [[Processadores]] — branch prediction, pipeline stalls, conditional jumps (JLE, JGE).

### Boolean: por que `bool` ocupa 1 byte e não 1 bit

```
Em Go: bool ocupa 1 byte (8 bits) — mesmo que armazene 0 ou 1

Por quê não 1 bit?
  CPUs modernas são byte-addressable (endereçáveis por byte)
  → menor unidade de memória endereçável é 1 byte
  → não existe instrução para ler/escrever 1 bit isolado em memória
  
  Para ler 1 bit em posição arbitrária precisaria:
    1. ler o byte inteiro
    2. aplicar máscara (AND com bitmask)
    3. shift para isolar o bit
  → 3 instruções vs 1 instrução (carregar 1 byte)

  Além disso: alinhamento
    struct { a bool; b int64 }
    → sem alinhamento: b precisaria de acesso desalinhado (mais lento)
    → com 1 byte para bool: compilador pode inserir 7 bytes de padding
                            para alinhar b em offset múltiplo de 8

Layout na memória:
  var b bool  →  1 byte: 0x00 (false) ou 0x01 (true)
  struct { A bool; B int64 }
  → [A: 1 byte][padding: 7 bytes][B: 8 bytes] = 16 bytes total
```

Conecta com [[Processadores]] — byte addressability, memory alignment, struct padding.

### Comparação de interfaces: tipo E valor

```
Interface em Go — layout na memória (2 words de 64 bits):
  ┌──────────────────┬──────────────────┐
  │  *itab           │  *data           │
  │  (type + methods)│  (value pointer) │
  └──────────────────┴──────────────────┘

i == j   →  compara os dois campos:
              (*itab_i == *itab_j)   AND   (*data_i == *data_j)
           se itab diferente → false imediatamente (tipos diferentes)
           se mesmo itab → compara os valores apontados por *data

Por que a armadilha com nil?
  var p *int = nil
  var i interface{} = p

  i está assim na memória:
  ┌──────────────────┬──────────────────┐
  │  *itab (*int)    │  nil             │
  └──────────────────┴──────────────────┘
  
  i == nil?  →  nil interface = {itab: nil, data: nil}
  i tem itab != nil → i != nil   ← ARMADILHA!
```

Conecta com [[Gerenciamento de Memória]] — layout de interface na memória (itab comparison), pointer equality vs value equality.

---

## 11. Conexão com Sistemas Operacionais

- **[[Processadores]]**: Comparações compilam para `CMP` + `SET*` (branchless) ou `CMP` + `J*` (branches). O registrador `EFLAGS` guarda o resultado da comparação (ZF, SF, CF, OF). Short-circuit `&&`/`||` gera branches — a branch prediction unit da CPU prevê o caminho mais provável para manter o pipeline cheio.

- **[[Processadores]]** (branch prediction): Short-circuit evaluation gera código de branch. Se a condição é previsível (ex: verificação de nil antes de desreferenciar — quase sempre "não nil"), o branch predictor acerta → sem pipeline stall. Condições 50/50 aleatórias → mispredict penalty de ~15 ciclos.

- **[[Processadores]]** (alinhamento): `bool` em Go ocupa 1 byte por byte-addressability — CPUs não têm instrução para ler/escrever bits individuais na memória. Structs com `bool` podem ter bytes de padding inseridos pelo compilador para manter alinhamento de outros campos.

- **[[Gerenciamento de Memória]]**: Comparação de interfaces (`==`) compara dois pointers: `*itab` (tipo) e `*data` (valor). Uma interface com tipo definido mas valor nil (`var p *int = nil; var i interface{} = p`) não é `== nil` porque seu `*itab` é não-nulo. Isso é uma fonte clássica de bugs com erros retornados como interface.