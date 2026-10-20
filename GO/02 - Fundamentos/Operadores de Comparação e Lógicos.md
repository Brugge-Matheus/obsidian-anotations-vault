---
tags:
  - go
  - go/fundamentos
aliases:
  - Operadores de comparação
  - Operadores lógicos
  - Curto-circuito
  - Tipos comparáveis
ordem: 9
modulo: "02 - Fundamentos"
---
# Operadores de Comparação e Lógicos

> **Nota 9 da trilha de Go.** Toda decisão de um programa (um `if`, a condição de um `for`, um `case`) depende de uma expressão que resulta em `bool`. Nesta nota veremos os **operadores de comparação**, a regra de **quais tipos podem ser comparados** (e por que slices e maps não podem), os **operadores lógicos** com **avaliação em curto-circuito**, as armadilhas com `NaN` e com interfaces, e o pacote `cmp`. Por fim, vemos como a CPU executa uma comparação.
>
> **Pré-requisitos:** [[Tipos de Dados]], [[Operações Aritméticas]].

---

## 🧭 De onde viemos

Em [[Operações Aritméticas]] os operadores produziam números. Aqui eles produzem **`bool`**, o tipo que estudamos em [[Tipos de Dados]] (aquele sem "truthy/falsy"). Essas expressões booleanas são a matéria-prima da próxima nota, [[Controle de Fluxo]].

---

## ⚖️ 1. Operadores de comparação

| Operador | Significado | Exemplo | Resultado |
| --- | --- | --- | --- |
| `==` | igual | `5 == 5` | `true` |
| `!=` | diferente | `5 != 3` | `true` |
| `<` | menor | `3 < 5` | `true` |
| `<=` | menor ou igual | `5 <= 5` | `true` |
| `>` | maior | `7 > 3` | `true` |
| `>=` | maior ou igual | `3 >= 5` | `false` |

Regras:

- O resultado é sempre um `bool` (na verdade, uma constante booleana sem tipo quando os operandos são constantes)
- Os dois lados precisam ser do **mesmo tipo** (ou um deles ser uma constante sem tipo compatível)

```go
var a int = 10
var b int64 = 10
fmt.Println(a == b)          // ❌ invalid operation: a == b (mismatched types int and int64)
fmt.Println(int64(a) == b)   // ✅ true
fmt.Println(a == 10)         // ✅ 10 é constante sem tipo
```

---

## 🧷 2. Comparável vs ordenado

Nem todo tipo aceita todos os operadores. Go divide os tipos em duas categorias:

> 💡 **Tipo comparável:** aceita `==` e `!=`.
> 💡 **Tipo ordenado:** aceita também `<`, `<=`, `>` e `>=`.

| Tipo | `==` / `!=` | `<` `>` `<=` `>=` | Como compara |
| --- | --- | --- | --- |
| Inteiros, floats | ✅ | ✅ | Pelo valor numérico |
| Strings | ✅ | ✅ | Byte a byte (lexicográfico) |
| `bool` | ✅ | ❌ | — |
| Complexos | ✅ | ❌ | Parte real e imaginária |
| Ponteiros | ✅ | ❌ | Pelo **endereço** |
| Channels | ✅ | ❌ | Pela identidade (mesmo channel criado pelo mesmo `make`) |
| Interfaces | ✅ | ❌ | Tipo dinâmico **e** valor (seção 6) |
| Arrays `[N]T` | ✅ se T for comparável | ❌ | Elemento a elemento |
| Structs | ✅ se todos os campos forem comparáveis | ❌ | Campo a campo |
| **Slices** | ❌ (só com `nil`) | ❌ | — |
| **Maps** | ❌ (só com `nil`) | ❌ | — |
| **Funções** | ❌ (só com `nil`) | ❌ | — |

Saída real dos casos não permitidos:

```go
type T struct{ Tags []string }
a, b := T{}, T{}
_ = a == b
// invalid operation: a == b (struct containing []string cannot be compared)

s1, s2 := []int{}, []int{}
_ = s1 == s2
// invalid operation: s1 == s2 (slice can only be compared to nil)
```

### 2.1 Por que slices, maps e funções não são comparáveis?

Porque não existe uma resposta óbvia para "o que é igual":

- **Slice:** igual seria "mesmo conteúdo" ou "mesma região de memória"? Dois slices podem ver o mesmo array por baixo, e o conteúdo pode mudar a qualquer momento (veremos em [[Arrays e Slices]])
- **Map:** comparar conteúdo exigiria percorrer tudo; comparar identidade seria surpreendente
- **Função:** duas funções com o mesmo código são iguais? E closures com variáveis capturadas diferentes?

Em vez de escolher uma semântica que surpreenderia metade das pessoas, Go proíbe. Para comparar conteúdo, use as funções explícitas:

```go
slices.Equal(s1, s2)        // conteúdo de slices (Go 1.21+)
maps.Equal(m1, m2)          // conteúdo de maps (Go 1.21+)
bytes.Equal(b1, b2)         // []byte
reflect.DeepEqual(x, y)     // qualquer coisa, recursivo (lento; mais usado em testes)
```

> 💡 Ser comparável também é o que permite um tipo ser **chave de map** (veremos em [[Maps]]) e usar a restrição `comparable` em [[Generics]].

---

## 🔤 3. Comparando strings

Strings são comparadas **byte a byte**, como em `memcmp` de C. Saída real:

```go
fmt.Println("apple" < "banana")     // true   'a'(97) < 'b'(98)
fmt.Println("Go" == "go")           // false  'G'(71) ≠ 'g'(103)
fmt.Println("Zebra" < "abacate")    // true   ← maiúsculas vêm antes (Z=90 < a=97)
fmt.Println("café" < "cafê")        // true   é = C3 A9 < ê = C3 AA (bytes UTF-8)
fmt.Println("10" < "9")             // true   ← '1' < '9' (não é comparação numérica!)
```

Consequências:

- É **sensível a maiúsculas**. Para ignorar: `strings.EqualFold("Go", "GO")` → `true` (correto também para Unicode)
- A ordem é a dos **bytes**, não a ordem alfabética de um idioma. Para ordenar nomes em português corretamente ("Álvaro" perto de "Alberto"), use o pacote `golang.org/x/text/collate`
- Números em string são comparados como texto (`"10" < "9"`)
- O custo é **O(n)** no tamanho do prefixo comum (comprimentos diferentes são detectados rápido para `==`)

---

## 🌊 4. Comparando floats: o caso `NaN`

Saída real:

```go
nan := math.NaN()
fmt.Println(nan < 1, nan > 1, nan == nan, nan != nan)
// false false false true
```

`NaN` não é menor, nem maior, nem igual a nada, **nem a si mesmo**. Isso quebra intuições:

```go
x := math.NaN()
if x == x { }           // falso!  (é inclusive um truque antigo para testar NaN)
if x < 0 || x >= 0 { }  // falso!  "todo número é negativo ou não-negativo" não vale
```

Além disso, como vimos em [[Tipos de Dados]], nunca compare floats calculados com `==`: use uma tolerância.

---

## 🔀 5. Operadores lógicos

| Operador | Nome | Resultado |
| --- | --- | --- |
| `&&` | E (AND) | `true` só se **os dois** forem `true` |
| `\|\|` | OU (OR) | `true` se **pelo menos um** for `true` |
| `!` | NÃO (NOT) | inverte |

```
 A      B     │  A && B   A || B   !A
──────────────┼──────────────────────
 false  false │  false    false    true
 false  true  │  false    true     true
 true   false │  false    true     false
 true   true  │  true     true     false
```

Os operandos precisam ser `bool` (ou tipos definidos com subjacente `bool`). Não existe `and`/`or`/`not` como palavras, nem XOR lógico (use `a != b` para dois bools).

---

## ⚡ 6. Avaliação em curto-circuito

> 💡 **Curto-circuito (short-circuit evaluation):** o operando da direita só é avaliado se for necessário para decidir o resultado. Em `A && B`, se `A` é `false`, `B` nem é executado. Em `A || B`, se `A` é `true`, `B` nem é executado.

Isso é **garantido pela especificação** (não é uma otimização opcional), e você pode e deve contar com isso:

```go
// 1. Proteção contra nil: o acesso à direita só acontece se for seguro
if usuario != nil && usuario.Ativo { }

// 2. Proteção contra índice fora dos limites
if i < len(s) && s[i] == alvo { }

// 3. Evitar trabalho caro
if emCache(chave) || carregarDoBanco(chave) { }   // só vai ao banco se não estava no cache

// 4. Divisão segura
if n != 0 && total/n > limite { }
```

> ⚠️ A ordem importa. `if usuario.Ativo && usuario != nil` dá panic quando `usuario` é `nil`, porque o lado esquerdo é avaliado primeiro.

> ⚠️ Cuidado com **efeitos colaterais** no lado direito: se você escreve `if ok || registrar(evento)`, o `registrar` não roda quando `ok` é `true`. Se a função precisa rodar sempre, chame antes do `if`.

---

## 🧩 7. Comparando interfaces

Uma interface guarda dois componentes: o **tipo dinâmico** e o **valor dinâmico**. A comparação `==` entre interfaces verifica os dois. Saídas reais:

```go
var a, b any = 42, 42
var c any = int64(42)
fmt.Println(a == b, a == c)    // true false   ← int(42) ≠ int64(42): tipos diferentes
```

### 7.1 A armadilha do nil dentro da interface

```go
var p *int              // ponteiro nil
var i any = p           // interface com tipo *int e valor nil
fmt.Println(p == nil, i == nil)    // true false   😱
```

```
i == nil exige os DOIS componentes nulos:

  interface nil de verdade      interface i (guardando um *int nil)
  ┌────────┬────────┐           ┌────────┬────────┐
  │ tipo:  │ valor: │           │ tipo:  │ valor: │
  │  nil   │  nil   │           │  *int  │  nil   │   ← tem tipo, então NÃO é nil
  └────────┴────────┘           └────────┴────────┘
```

Esse comportamento é a origem de um bug famoso com o tipo `error` (que é uma interface). Vamos estudar em detalhes em [[Interfaces]] e [[O Padrão error em Go]].

### 7.2 Comparar interfaces pode dar panic

Se as duas interfaces guardam o **mesmo tipo dinâmico** e esse tipo **não é comparável**, a comparação compila mas falha em tempo de execução (saída real):

```go
var s1, s2 any = []int{1}, []int{1}
fmt.Println(s1 == s2)
// panic: runtime error: comparing uncomparable type []int
```

O compilador não tem como saber o que vai estar dentro da interface, então a verificação fica para o runtime.

---

## 📐 8. Precedência

Os operadores desta nota ficam nos **três níveis mais baixos** da tabela que vimos em [[Operações Aritméticas]]:

```
Nível 5:  *  /  %  <<  >>  &  &^      ← aritméticos e bits primeiro
Nível 4:  +  -  |  ^
Nível 3:  ==  !=  <  <=  >  >=        ← depois as comparações
Nível 2:  &&                          ← depois o E
Nível 1:  ||                          ← por último o OU
(! é unário: vem antes de todos)
```

```go
a+b > c*d && x != 0 || flag
// lido como:  (((a+b) > (c*d)) && (x != 0)) || flag

!ok && pronto       // (!ok) && pronto
!(ok && pronto)     // nega o resultado inteiro

aprovado := (idade >= 18 && temRenda) || (temFiador && score > 700)   // ✅ parênteses deixam claro
```

> 💡 **Leis de De Morgan**, úteis para simplificar condições: `!(a && b) == !a || !b` e `!(a || b) == !a && !b`.

---

## 📦 9. O pacote `cmp` (Go 1.21+)

Saídas reais:

```go
import "cmp"

cmp.Compare(1, 2)          // -1   (a < b → -1, a == b → 0, a > b → +1)
cmp.Compare("b", "a")      //  1
cmp.Less(1.0, math.NaN())  // false
cmp.Compare(math.NaN(), 1.0)   // -1   ← cmp trata NaN como MENOR que tudo (ordem total)

// cmp.Or: retorna o primeiro valor que não é o zero value (Go 1.22+)
cmp.Or("", "", "padrão")   // "padrão"
cmp.Or(porta, 8080)        // porta se ≠ 0, senão 8080
```

- `cmp.Compare` é a função de três vias usada por `slices.SortFunc` e similares (veremos em [[Arrays e Slices]])
- Diferente dos operadores `<` e `==`, `cmp.Compare` e `cmp.Less` definem uma **ordem total** para floats (com `NaN` antes de todos), o que torna a ordenação estável mesmo com `NaN`
- `cmp.Or` é o jeito idiomático de "valor padrão" (o `valor || "padrão"` de outras linguagens)

---

## 🔬 10. Por dentro: como a CPU compara

Uma comparação em x86-64 é feita pela instrução `CMP`, que **subtrai** os operandos sem guardar o resultado, apenas atualizando o registrador de **flags**:

```
a < b   (int)
────────────────────────────────────────────────
CMPQ  BX, AX        ; calcula AX - BX, descarta o resultado,
                    ; atualiza as flags: ZF (zero), SF (sinal),
                    ; OF (overflow), CF (carry/empréstimo)
SETLT CL            ; CL = 1 se "less than" (SF ≠ OF), senão 0
                    ; ← assim um bool é produzido SEM desvio

if a < b { ... }
────────────────────────────────────────────────
CMPQ  BX, AX
JGE   pula_corpo    ; desvia se a >= b (condição invertida)
...corpo...
pula_corpo:
```

| Flag | Significa | Usada em |
| --- | --- | --- |
| ZF (Zero) | resultado = 0 | `==`, `!=` |
| SF (Sign) | resultado negativo | `<`, `>` com sinal |
| OF (Overflow) | estouro com sinal | `<`, `>` com sinal (SF ≠ OF) |
| CF (Carry) | empréstimo sem sinal | `<`, `>` sem sinal |

Para tipos **sem sinal**, o compilador usa outras instruções (`SETCS`, `JCC`...) que olham a flag CF. É por isso que o tipo importa até na hora de comparar.

### 10.1 O curto-circuito vira desvios

```go
if p != nil && p.Ativo { corpo() }
```

```
TESTQ AX, AX          ; p == nil ?
JEQ   fim             ; se nil, pula TUDO (o lado direito nunca é lido)
CMPB  (AX), $0        ; lê p.Ativo (só chega aqui se p != nil)
JEQ   fim
CALL  corpo
fim:
```

Cada `&&`/`||` vira um **desvio condicional**. O preditor de desvios da CPU aprende o padrão: se `p` quase nunca é `nil`, o custo é praticamente zero. Quando a condição é imprevisível (50/50 aleatório), cada erro de predição custa ~15–20 ciclos (o pipeline é esvaziado). Nesses casos o compilador às vezes usa instruções sem desvio (`CMOV`, `SETcc`).

> 💡 Você viu pipeline e predição de desvios em [[Processadores]]. A lição prática: em código muito quente, condições previsíveis são mais baratas que condições aleatórias. Mas isso só importa depois de medir (ver [[Benchmarks]]).

### 10.2 Comparação de strings e structs

- `s1 == s2` com strings: primeiro compara os **tamanhos** (diferentes → falso imediatamente), depois chama uma rotina otimizada de comparação de memória (`runtime.memequal`)
- `s1 < s2`: chama `runtime.cmpstring`, que compara blocos de bytes com instruções vetoriais
- Structs e arrays pequenos: o compilador gera comparações campo a campo, ou uma comparação de memória direta quando o layout permite (sem padding com lixo no meio)

---

## ⚠️ 11. Armadilhas

| Armadilha | Exemplo | Correção |
| --- | --- | --- |
| Comparar slices/maps com `==` | `s1 == s2` não compila | `slices.Equal`, `maps.Equal` |
| Ordem errada no curto-circuito | `u.Ativo && u != nil` | Teste o nil **antes** |
| Efeito colateral no lado direito | `ok \|\| salvar()` | Chame `salvar()` fora do `if` |
| `NaN` com `==` | `x == math.NaN()` sempre falso | `math.IsNaN(x)` |
| Strings "numéricas" | `"10" < "9"` é `true` | Converta para número antes |
| Maiúsculas/minúsculas | `"Go" == "go"` é `false` | `strings.EqualFold` |
| Interface com ponteiro nil | `i == nil` é `false` | Ver [[Interfaces]] |
| Interface com tipo não comparável | panic em tempo de execução | Evite `==` com `any` de conteúdo desconhecido |

---

## 📋 Resumo

| Tópico | Ponto-chave |
| --- | --- |
| Comparação | `==` `!=` `<` `<=` `>` `>=`, mesmo tipo dos dois lados |
| Comparáveis | Básicos, ponteiros, channels, interfaces, arrays e structs de comparáveis |
| Não comparáveis | Slices, maps e funções (só com `nil`) |
| Ordenados | Números e strings |
| Strings | Byte a byte, sensível a maiúsculas |
| `NaN` | Diferente de tudo, inclusive de si mesmo |
| Lógicos | `&&`, `\|\|`, `!` só com `bool` |
| Curto-circuito | Garantido; usado para proteger acessos |
| Interfaces | `==` compara tipo e valor; pode dar panic com tipos não comparáveis |
| `cmp` | `Compare`, `Less` (ordem total), `Or` (valor padrão) |
| Por dentro | `CMP` + flags; `SETcc` para gerar bool; desvios para `if` e curto-circuito |

---

## 🔗 Conexões

**Dentro de Go**
- Usando as condições → [[Controle de Fluxo]]
- Por que slices não são comparáveis → [[Arrays e Slices]]
- Chaves de map precisam ser comparáveis → [[Maps]]
- Igualdade de structs → [[Structs]]
- O nil dentro de interfaces → [[Interfaces]] e [[O Padrão error em Go]]
- A restrição `comparable` → [[Generics]]

**Sistemas Operacionais**
- [[Processadores]] — flags da ALU, instruções de desvio, pipeline e predição de desvios
- [[Bits e Bytes]] — por que a comparação de strings é byte a byte e como UTF-8 afeta a ordem

**Algoritmos e Estruturas de Dados**
- [[Algoritmos de Ordenação]] — toda ordenação depende de uma função de comparação (`cmp.Compare`)
- [[Algoritmos de Busca]] — busca binária exige um tipo ordenado

---

⬅️ **Anterior:** [[Operações Aritméticas]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Controle de Fluxo]] ➡️
