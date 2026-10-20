---
tags:
  - go
  - go/fundamentos
aliases:
  - Ponteiros em Go
  - Pointers
  - nil pointer
  - Passagem por valor
ordem: 11
modulo: "02 - Fundamentos"
---
# Ponteiros

> **Nota 11 da trilha de Go.** Um ponteiro é uma variável que guarda **o endereço de outra variável**. Ponteiros aparecem em tudo daqui para frente (structs, métodos, interfaces, slices, maps, concorrência), então esta nota fecha o módulo de fundamentos construindo a base: o que é um endereço, os operadores `&` e `*`, o valor `nil`, a função `new`, o fato de que **Go sempre passa argumentos por valor**, quando usar ponteiros e o que Go proíbe (aritmética de ponteiros) para ser mais seguro que C.
>
> **Pré-requisitos:** [[Variáveis e constantes]], [[Tipos de Dados]].

---

## 🧭 De onde viemos

Em [[Variáveis e constantes]] vimos que uma variável é um local nomeado na memória e que o compilador decide se ela mora na stack ou na heap. Em [[Tipos de Dados]] vimos que `uintptr` guarda um endereço como número e que `string`, slice e map são pequenos "cabeçalhos" que apontam para dados em outro lugar. Para entender essas peças, precisamos do conceito que liga tudo: **ponteiro**.

---

## 🏠 1. A intuição: endereços

A memória do processo é uma sequência enorme de bytes, cada um com um **endereço** (um número). Você estudou isso em [[Espaços de Endereçamento]] e [[Memória Virtual]]: cada processo enxerga seu próprio espaço de endereços virtuais.

```
endereço        conteúdo
(virtual)
            ┌──────────────┐
0xc000012000│      42      │  ← variável x (int, 8 bytes)
            ├──────────────┤
0xc000012008│ 0xc000012000 │  ← variável p: guarda o ENDEREÇO de x
            ├──────────────┤
0xc000012010│     ...      │
            └──────────────┘
```

> 💡 **Ponteiro:** um valor que é o endereço de memória de outro valor. Uma variável do tipo `*int` guarda o endereço de um `int`. O tipo do ponteiro diz **o que existe no endereço apontado**, para que o compilador saiba quantos bytes ler e como interpretá-los.

Analogia: uma variável é uma **casa**; o valor é **quem mora lá**; o ponteiro é um **papel com o endereço da casa**. Copiar o papel não copia a casa. Quem tem o papel pode ir lá e mudar os móveis.

---

## ✳️ 2. Os dois operadores: `&` e `*`

| Operador | Nome | Lê-se | Resultado |
| --- | --- | --- | --- |
| `&x` | endereço de | "o endereço de x" | um ponteiro `*T` para `x` |
| `*p` | desreferência | "o valor apontado por p" | o próprio valor `T` no endereço |

E `*T` num **tipo** significa "ponteiro para T". Saída real:

```go
x := 42
p := &x                         // p é *int, aponta para x

fmt.Printf("%T %v\n", p, *p)    // *int 42
*p = 99                         // escreve no endereço apontado
fmt.Println(x)                  // 99   ← x mudou através do ponteiro
```

```
depois de  p := &x                 depois de  *p = 99
┌───────┐       ┌────┐             ┌───────┐       ┌────┐
│ p ────┼──────▶│ 42 │ x           │ p ────┼──────▶│ 99 │ x
└───────┘       └────┘             └───────┘       └────┘
```

> ⚠️ O `*` tem dois papéis: em um **tipo** (`var p *int`) significa "ponteiro para"; numa **expressão** (`*p = 1`) significa "desreferenciar". O contexto deixa claro qual é.

### 2.1 Ponteiro para ponteiro

```go
pp := &p        // pp é **int: aponta para p, que aponta para x
**pp = 7
fmt.Println(x)  // 7
```

Existe, mas é raro em Go (em C é comum para "retornar" um ponteiro por parâmetro; em Go usamos múltiplos retornos para isso).

---

## 🚫 3. `nil`: o ponteiro que não aponta para nada

O zero value de qualquer ponteiro é `nil`:

```go
var p *int
fmt.Println(p == nil)    // true
fmt.Println(*p)          // 💥 panic
```

Desreferenciar `nil` causa um panic (saída real):

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x493989]
```

Repare no `signal SIGSEGV ... addr=0x0`. Isso mostra o mecanismo por baixo (seção 10): o processador tentou ler o endereço 0, que não está mapeado no espaço de endereçamento; a MMU gerou uma falha, o kernel enviou o sinal `SIGSEGV`, e o runtime de Go transformou o sinal num panic.

> ✅ Antes de desreferenciar um ponteiro que pode ser `nil`, teste: `if p != nil { ... }`. É o uso clássico do curto-circuito (`if u != nil && u.Ativo`), que vimos em [[Operadores de Comparação e Lógicos]].

---

## 🆕 4. Criando valores apontados: `&T{}` e `new`

### 4.1 `&` de um literal composto (o mais comum)

```go
p := &Ponto{X: 1, Y: 2}     // cria um Ponto e devolve o endereço dele
```

### 4.2 A função embutida `new`

```go
q := new(int)       // aloca um int com zero value e devolve *int
fmt.Println(*q)     // 0
*q = 10
```

`new(T)` é equivalente a `var v T; return &v`. É pouco usado com structs (onde `&T{}` é mais claro), mas útil para tipos básicos.

**Go 1.26+:** `new` também aceita uma **expressão**, criando a variável já com aquele valor:

```go
idade := new(30)                 // *int apontando para 30
p := Pessoa{Idade: new(calcularIdade(nasc))}   // campo opcional *int em uma linha
```

Antes era preciso uma variável temporária (`v := 30; p.Idade = &v`) ou uma função auxiliar. O caso de uso típico são **campos opcionais** (ponteiros que podem ser `nil`), muito comuns em JSON e APIs (ver [[JSON (marshal e unmarshal)]]).

---

## 📨 5. Go sempre passa por valor

Esta é a regra mais importante da nota:

> **Em Go, toda passagem de argumento e toda atribuição é uma CÓPIA do valor.** Não existe passagem por referência como `&param` em PHP ou `ref` em C#.

O que muda é **o que está sendo copiado**. Se você passa um ponteiro, copia-se o **endereço**, e a função consegue alterar o original (saída real):

```go
func dobrarValor(n int)     { n *= 2 }    // recebe uma CÓPIA do número
func dobrarPonteiro(n *int) { *n *= 2 }   // recebe uma cópia do ENDEREÇO

n := 10
dobrarValor(n)
fmt.Println(n)       // 10   ← a cópia foi dobrada; o original não

dobrarPonteiro(&n)
fmt.Println(n)       // 20   ← a função foi até o endereço e alterou o original
```

```
dobrarValor(n)                       dobrarPonteiro(&n)
┌──────────┐    ┌──────────┐         ┌──────────┐       ┌──────────────┐
│ main: n  │    │ func: n  │         │ main: n  │◀──────┼─ func: n *int│
│   10     │    │ 10 → 20  │         │ 10 → 20  │       │ (cópia do    │
└──────────┘    └──────────┘         └──────────┘       │  endereço)   │
  intocado       cópia descartada                       └──────────────┘
```

O exemplo clássico, a troca de valores:

```go
func trocar(a, b *int) { *a, *b = *b, *a }

a, b := 1, 2
trocar(&a, &b)
fmt.Println(a, b)    // 2 1
```

### 5.1 "Mas slices e maps parecem passados por referência…"

Parecem, mas não são. Um slice ou um map é um **cabeçalho pequeno que contém um ponteiro** para os dados de verdade. Ao passar um slice para uma função, o cabeçalho é copiado, mas o ponteiro de dentro aponta para os **mesmos dados**:

```
main: s ──────┐                      ┌──── func: s (cópia do cabeçalho)
 ┌──────────┐ │                      │ ┌──────────┐
 │ ptr ─────┼─┼──▶ [1][2][3] ◀───────┼─┼── ptr    │
 │ len: 3   │ │   (array na heap)    │ │ len: 3   │
 │ cap: 3   │ │                      │ │ cap: 3   │
 └──────────┘                          └──────────┘
```

Por isso a função consegue alterar os **elementos**, mas não consegue mudar o tamanho do slice do chamador (um `append` dentro da função altera só a cópia do cabeçalho). Esse é um dos pontos mais importantes de [[Arrays e Slices]].

> 💡 Muitos textos chamam slices, maps e channels de "tipos de referência". A especificação de Go **não** usa esse termo. É mais preciso dizer: são valores que **contêm ponteiros** internamente.

---

## 🧱 6. Ponteiros para structs

Com structs, Go faz a desreferência **automaticamente** no acesso a campos (saída real):

```go
type Ponto struct{ X, Y int }

pt := &Ponto{1, 2}
pt.X = 10                 // açúcar sintático para (*pt).X = 10
fmt.Println(pt.X, (*pt).Y, *pt)   // 10 2 {10 2}
```

Em C você escreveria `pt->x`; em Go o `.` funciona tanto para valores quanto para ponteiros. O mesmo vale para chamadas de método (veremos em [[Métodos e Receptores]]).

---

## 🤔 7. Quando usar ponteiros

| Use ponteiro quando... | Exemplo |
| --- | --- |
| A função precisa **alterar** o valor do chamador | `func (c *Conta) Depositar(v int)` |
| O valor é **grande** e copiá-lo seria caro | structs com muitos campos ou arrays grandes |
| É preciso representar **ausência** de valor | campo opcional `Idade *int` (nil = "não informado") |
| O valor **não pode ser copiado** | tipos que contêm `sync.Mutex` |
| É preciso **compartilhar** o mesmo objeto entre várias partes | um cache, uma conexão |

| Prefira valor quando... | Exemplo |
| --- | --- |
| O tipo é pequeno e imutável na prática | `time.Time`, `Ponto{X, Y}`, números, strings |
| Você quer garantir que ninguém altere o original | passar uma cópia é uma proteção |
| É um slice, map, channel ou função | já contêm ponteiros internos; `*[]int` quase nunca é necessário |

> ⚠️ "Ponteiro é sempre mais rápido" é **mito**. Copiar uma struct pequena (até algumas dezenas de bytes) é baratíssimo e cabe em registradores. Já um ponteiro pode fazer o valor **escapar para a heap** (seção 9), o que custa uma alocação e trabalho extra para o GC. Meça antes de otimizar (ver [[Benchmarks]]).

---

## 🛡️ 8. O que Go proíbe (e por quê)

### 8.1 Sem aritmética de ponteiros

Em C, `p + 1` aponta para o próximo elemento, e você pode andar livremente pela memória. Em Go:

```go
p := &x
p++           // ❌ invalid operation: p++ (non-numeric type *int)
p = p + 8     // ❌
```

Essa proibição elimina, de uma vez, as classes de vulnerabilidade mais comuns de C: *buffer overflow*, leitura fora dos limites, ponteiros "soltos" apontando para o meio de outro objeto. Para percorrer sequências, usa-se slices, que têm **checagem de limites** (veremos em [[Arrays e Slices]]).

> 💡 Quando é realmente necessário (interoperar com C, estruturas binárias, otimizações extremas), existe o pacote `unsafe`, cujo nome já avisa. Veremos em [[unsafe e cgo]].

### 8.2 Retornar o endereço de uma variável local é SEGURO

Em C isto é um bug grave (*dangling pointer*):

```c
int* novo_contador() {
    int c = 0;
    return &c;      // 💥 c some quando a função retorna; o ponteiro aponta para lixo
}
```

Em Go, o mesmo código é **correto e idiomático** (saída real):

```go
func novoContador() *int {
	c := 0
	return &c       // ✅ o compilador percebe e coloca c na heap
}

c := novoContador()
*c++
*c++
fmt.Println(*c)     // 2
```

O compilador faz **escape analysis** e decide colocar `c` na heap, onde ele vive enquanto alguém tiver um ponteiro para ele. O GC libera quando ninguém mais aponta. Saída real de `go build -gcflags=-m`:

```
./main.go:11:2: moved to heap: c
```

### 8.3 Comparação de ponteiros compara endereços

```go
u, v := 5, 5
fmt.Println(&u == &v, u == v)    // false true
```

Dois ponteiros são iguais se apontam para **a mesma variável**. Ponteiros não são ordenados (`p1 < p2` não compila).

---

## 🔬 9. Por dentro: ponteiros, stack, heap e GC

### 9.1 Um ponteiro é só um endereço de 8 bytes

Em amd64, todo ponteiro ocupa 8 bytes, não importa para o que aponta. Para a CPU é um número usado em instruções de acesso à memória:

```
*p = 99      →    MOVQ  $99, (AX)     ; escreve 99 no endereço que está em AX
x := *p      →    MOVQ  (AX), BX      ; lê o valor no endereço que está em AX
```

### 9.2 Ponteiros guiam o escape analysis

```
func f() int  { x := 1; return x }    → x fica na STACK (o valor é copiado no retorno)
func g() *int { x := 1; return &x }   → x vai para a HEAP (o endereço sobrevive à função)
```

A regra simplificada: **se um ponteiro para a variável pode sobreviver ao fim da função** (retornado, guardado numa global, num campo de struct da heap, num channel, capturado por uma goroutine), a variável vai para a heap. Estudaremos todas as regras em [[Memória e Escape Analysis]].

### 9.3 Ponteiros são o que o GC segue

O garbage collector descobre o que está vivo **seguindo ponteiros** a partir das raízes (variáveis globais, stacks das goroutines, registradores). Tudo que não for alcançado é lixo:

```
 raízes (stack, globais)
        │
        ▼
     ┌─────┐      ┌─────┐
     │  A  │ ───▶ │  B  │      A e B: alcançáveis → VIVOS
     └─────┘      └─────┘
                               C: ninguém aponta → LIXO (será liberado)
     ┌─────┐
     │  C  │
     └─────┘
```

Por isso Go precisa saber **exatamente onde há ponteiros** na memória (lembra das seções `.data` vs `.noptrdata` em [[Como Go Funciona]]?), e por isso a aritmética de ponteiros livre é incompatível com um GC preciso. Veremos em [[Garbage Collector]].

> 💡 O GC de Go **não move** objetos da heap (não é compactador). Mas as **stacks das goroutines** podem ser movidas quando crescem, e o runtime atualiza os ponteiros para dentro delas. É mais um motivo para nunca guardar endereços como `uintptr`.

---

## 🧯 10. Por dentro: como um nil vira panic

O caminho completo, do `*p` até a mensagem de panic:

```
1. Seu código:        fmt.Println(nilp.X)      (nilp == nil)
2. Instrução:         MOVQ (AX), BX            com AX = 0x0
3. Hardware (MMU):    a página do endereço 0 não está mapeada
                      → falha de página (page fault)
4. Kernel:            o endereço é inválido para o processo
                      → envia o sinal SIGSEGV à thread
5. Runtime do Go:     o tratador de SIGSEGV (instalado na inicialização,
                      lembra dos 114 rt_sigaction em "Como Go Funciona"?)
                      vê que a falha foi num endereço baixo, dentro de código Go
                      → converte em panic: "invalid memory address or
                        nil pointer dereference"
6. Panic:             desenrola a stack executando defers; se ninguém
                      recuperar, imprime o stack trace e sai com código 2
```

O endereço 0 (e as primeiras páginas) **nunca** são mapeados exatamente para que acessos via ponteiro nulo falhem de forma previsível. Você estudou o mecanismo de falha de página em [[Memória Virtual]] e [[Paginação]], e a proteção de memória em [[Hardware de Proteção]].

> 💡 Compare com a divisão por zero, que vimos em [[Operações Aritméticas]]: lá o compilador insere uma **checagem explícita** antes do `IDIV`. Para nil, o compilador geralmente confia na **falha de hardware** (é de graça quando o ponteiro não é nil). Só quando o campo acessado fica muito longe do início do objeto (offsets grandes, que poderiam cair numa página mapeada) o compilador insere uma checagem explícita.

---

## ⚠️ 11. Armadilhas

| Armadilha | Exemplo | Correção |
| --- | --- | --- |
| Desreferenciar nil | `var p *T; p.Campo` | Verificar `p != nil` ou garantir inicialização |
| Achar que Go passa por referência | Alterar parâmetro não-ponteiro e esperar efeito | Passe `*T` se precisa alterar |
| Ponteiro para slice/map sem necessidade | `func f(s *[]int)` | Passe o slice; retorne o novo slice se precisar crescer |
| Ponteiro "por performance" em tipos pequenos | `func area(p *Ponto)` | Valor costuma ser mais rápido (sem escape) |
| Ponteiro para a variável do range (código antigo) | `ptrs = append(ptrs, &v)` com go ≤ 1.21 | Go 1.22+ resolve; ou use `&s[i]` |
| Guardar endereço como `uintptr` | O GC não enxerga | Use ponteiros de verdade |
| Copiar struct com `sync.Mutex` | Duas cópias do mutex | Passe `*T` (o `go vet` avisa) |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Ponteiro | Valor que guarda o endereço de outro valor; tipo `*T` |
| `&x` | Endereço de `x` |
| `*p` | Valor no endereço (ler ou escrever) |
| `nil` | Zero value de ponteiros; desreferenciar causa panic |
| `new(T)` | Aloca zero value e devolve `*T`; Go 1.26+: `new(expr)` |
| `&T{...}` | Forma idiomática de criar e apontar para structs |
| Passagem | **Sempre por valor**; com ponteiro, copia-se o endereço |
| Slices/maps | Cabeçalhos com ponteiros internos, não "referências" |
| Structs | `p.Campo` desreferencia automaticamente |
| Aritmética | Proibida (só com `unsafe`) |
| Retornar `&local` | Seguro: escape analysis manda para a heap |
| GC | Segue ponteiros a partir das raízes para achar o que está vivo |
| nil por dentro | Falha de página no endereço 0 → `SIGSEGV` → panic |

---

## 🔗 Conexões

**Dentro de Go**
- Cabeçalhos de slices com ponteiro interno → [[Arrays e Slices]]
- Ponteiros para structs e construtores → [[Structs]]
- Receptores por ponteiro → [[Métodos e Receptores]]
- Funções e passagem de argumentos → [[Funções — Fundamentos]]
- Escape analysis em profundidade → [[Memória e Escape Analysis]]
- Como o GC usa os ponteiros → [[Garbage Collector]]
- Panics e recuperação → [[Defer, Panic e Recover]]
- Aritmética de ponteiros com `unsafe` → [[unsafe e cgo]]

**Sistemas Operacionais**
- [[Espaços de Endereçamento]] e [[Memória Virtual]] — o ponteiro guarda um endereço **virtual**; a MMU traduz para físico
- [[Paginação]] — a página do endereço 0 não mapeada é o que torna o nil detectável
- [[Hardware de Proteção]] — a MMU e o modo usuário impedem acesso fora do espaço do processo
- [[Syscalls Diversas]] — sinais (`SIGSEGV`) como forma do kernel notificar o processo
- [[A Linguagem C]] — ponteiros de C, aritmética de ponteiros e ausência de GC: tudo que Go restringe para ganhar segurança

**Algoritmos e Estruturas de Dados**
- [[Linked Lists]] e [[Árvores Binárias]] — estruturas encadeadas são construídas com ponteiros (`prox *No`)

---

⬅️ **Anterior:** [[Controle de Fluxo]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Strings em Profundidade]] ➡️
