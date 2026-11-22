---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: conversão de array em ponteiro, parâmetros, arrays multidimensionais); experimentos com gcc 13.3 em Linux x86-64"
---
# Arrays e Ponteiros em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 | Anterior: [[Aritmética de Ponteiros em C]] | Próxima: [[Funções e Ponteiros em C]] | Compare com [[Arrays e Slices]] (Go)

---

# 🧭 Por que esta nota existe

Esta nota **junta** as três anteriores, e cada uma responde a uma pergunta diferente:

| Nota | Pergunta que responde |
|---|---|
| [[Arrays em C]] | O que é um **bloco de elementos** contíguos? |
| [[Ponteiros em C]] | O que é uma variável que guarda um **endereço**? |
| [[Aritmética de Ponteiros em C]] | Como **andar** na memória somando e subtraindo endereços? |
| **Esta nota** | Qual é a **relação** entre um array e um ponteiro? |

A confusão vem de C tratar os dois de forma tão parecida que parecem a mesma coisa. **Não são.** A ideia central é:

> 💡 **Um array NÃO é um ponteiro, mas, na maioria das expressões, o nome do array é convertido automaticamente em um ponteiro para o seu primeiro elemento.** Essa conversão se chama ***decay*** ("decair").

---

# ⚖️ Array × ponteiro: são coisas diferentes

Com `int v[5] = {10,20,30,40,50};` e `int *p = v;`:

| | `v` (array) | `p` (ponteiro) |
|---|---|---|
| O que **é** | **5 `int`s** guardados na memória | **um endereço** (8 bytes) |
| `sizeof` | **20** | **8** |
| Pode mudar para onde "aponta"? | ❌ `v++` → *lvalue required as increment operand* | ✅ `p++` |
| Pode ser atribuído (`v = ...`)? | ❌ (visto em [[Arrays em C]]) | ✅ |
| Tem espaço próprio para dados? | ✅ os elementos **são** o array | ❌ aponta para dados que estão **em outro lugar** |

Os valores `sizeof v = 20` e `sizeof p = 8` foram impressos pelo programa de teste.

---

# 🔁 O *decay*: array vira ponteiro

> **Regra:** numa expressão, um array é convertido em um **ponteiro para o seu primeiro elemento** (tipo `T *`), **exceto** em três casos: como operando de **`sizeof`**, como operando de **`&`**, e como **literal de string** que inicializa um array de `char` (`char s[] = "oi"`).

Por isso as duas linhas abaixo são equivalentes:

```c
int *p = v;          // v decai para &v[0]
int *p = &v[0];      // mesmo resultado (testado: p == &v[0], v == &v[0])
```

E é por isso que `sizeof v` (exceção: **não** decai) dá 20, mas `sizeof(v + 0)` (a soma força o decay) dá **8**, tamanho de um ponteiro.

---

# 🔗 `v[i]` é o mesmo que `*(v + i)`

O padrão **define** o operador de índice **pela aritmética de ponteiros** ([[Aritmética de Ponteiros em C]]):

```
v[i]   ≡   *(v + i)
```

O array decai para ponteiro, soma-se `i` elementos (escalado por `sizeof`) e desreferencia-se. Resultados reais com o array `v` e o ponteiro `p`:

```
p[2] = 30    *(p+2) = 30    v[2] = 30    *(v+2) = 30
```

O índice funciona **igualmente em ponteiros**, e não só em arrays: `p[2]` é "o 3º elemento a partir de `p`". Uma curiosidade consequência da definição: como a soma é comutativa, **`2[v]` também vale `30`** (funciona, mas nunca escreva assim).

> ✅ É a explicação de por que o índice começa em 0 ([[Arrays em C]]) e de por que **não há verificação de limites**: `v[i]` é só uma conta de endereço seguida de leitura.

---

# 📞 Arrays como parâmetros de função

Esta é a consequência prática mais importante do *decay*. Ao passar `v` para uma função, ele **decai** e a função recebe **só o endereço do primeiro elemento**. Por isso, as três declarações abaixo são **idênticas** para o compilador:

```c
void f1(int v[]);     // o tipo do parâmetro é int *
void f2(int v[5]);    // o 5 é IGNORADO; também é int *
void f3(int *v);      // a forma "honesta"
```

O teste confirmou: o `sizeof v` **dentro** das três funções vale **8** (o `gcc` avisa `-Wsizeof-array-argument` nas duas primeiras). Consequências:

- **Sem cópia:** a função **altera o array original**;
- **O tamanho se perde:** passe-o à parte (`size_t n`), como em [[Arrays em C]];
- Se a função só lê, **proteja** com `const`: `void imprime(const int *v, size_t n)` ([[Ponteiros em C]]);
- **Não se pode devolver um array local**: `int *f(void) { int v[3] = {1,2,3}; return v; }` devolve o endereço de uma variável que **deixa de existir** ao sair da função (ponteiro pendurado, [[Sintaxe, Blocos e Escopo em C]]). O `gcc` avisa: `function returns address of local variable` (`-Wreturn-local-addr`).

---

# 📐 `v + 1` × `&v + 1`

Os dois somam 1, mas a conta depende do **tipo**:

| Expressão | Tipo | `+ 1` avança | Teste real |
|---|---|---|---|
| `v` (decai) | `int *` | **um `int`** | **+4 bytes** |
| `&v` (exceção: não decai) | **`int (*)[5]`** (ponteiro para array de 5 `int`) | **o array inteiro** | **+20 bytes** |

`&v` tem o **mesmo endereço** que `&v[0]` ([[Arrays em C]]), mas **outro tipo**, e o tipo determina o passo da aritmética. Também vale `sizeof(&v)` = 8 (é um ponteiro) e `sizeof(*&v)` = 20 (o array).

---

# 🔲 Arrays multidimensionais e ponteiros

Um `int m[2][3]` é um **array de 2 elementos, cada um sendo um `int[3]`**. Quando decai, o "primeiro elemento" é uma **linha**, então `m` vira um **ponteiro para array de 3 `int`**: `int (*)[3]`.

```c
int m[2][3] = {{1,2,3},{4,5,6}};
sizeof m   // 24 (6 ints)        sizeof *m  // 12 (uma linha)
m[1][2]    // 6
*(*(m + 1) + 2)   // 6   ← a mesma coisa, escrita com ponteiros
```

Leitura: `m + 1` pula **uma linha inteira** (12 bytes), `*(m + 1)` é a linha 1 (que decai para `int *`), `+ 2` anda dois `int` e o `*` final lê. É por isso que, ao passar para funções, a **última dimensão é obrigatória** (o compilador precisa dela para saber o tamanho do passo):

```c
void g(int m[][3], size_t linhas);      // ou, equivalente: void g(int (*m)[3], size_t linhas);
// dentro: sizeof *m = 12, m[1][2] = 6
```

## Array de ponteiros × ponteiro para array

Parênteses mudam tudo, porque `[]` tem precedência **maior** que `*` ([[Operadores em C]]):

| Declaração | Significado | `sizeof` (teste) |
|---|---|---|
| `int *ap[3]` | **array de 3 ponteiros** para `int` | **24** (3 × 8) |
| `int (*pa)[3]` | **ponteiro para** um array de 3 `int` | **8** (um ponteiro) |

A diferença aparece com textos. Uma **matriz** de `char` guarda os bytes **dentro** dela; um **array de ponteiros** guarda só endereços:

```c
const char *nomes[] = {"ana", "bruno"};    // sizeof = 16 (2 ponteiros); os textos ficam em outro lugar
char       nm[][6]  = {"ana", "bruno"};    // sizeof = 12 (2 linhas de 6); os textos estão DENTRO
```

O `char **argv` de `main` ([[Como C Funciona]]) é exatamente um array de ponteiros para strings.

---

# 🔤 O caso clássico: `char s[]` × `char *s`

```c
char a[] = "oi";         // array de 3 chars, COPIADO para a pilha: pode alterar
a[0] = 'O';              // OK → "Oi"       sizeof a = 3

const char *s = "oi";    // ponteiro para o literal, que fica em .rodata (somente leitura)
s[0] = 'O';              // UB → na prática, Segmentation fault     sizeof s = 8
```

Teste real: modificar o literal apontado por `char *s = "oi";` terminou em **`Segmentation fault`** (código 139). Os dois parecem iguais ao **ler**, mas são diferentes: o array **contém** os bytes; o ponteiro **aponta** para bytes que não são seus. Use **`const char *`** para literais (o `gcc` com `-Wwrite-strings` avisa quando falta o `const`). O tema completo está em [[Strings em C]].

---

# ✅ Resumo do Conceito

- **Array ≠ ponteiro**: array = **bloco de elementos** (`sizeof` = N × tamanho); ponteiro = **endereço** (8 bytes); array **não** pode ser incrementado ou atribuído
- ***Decay*:** na maioria das expressões, o array vira **ponteiro para o 1º elemento**; exceções: **`sizeof`**, **`&`** e literal de string que inicializa array de `char`
- **`v[i]` ≡ `*(v + i)`**: o índice é aritmética de ponteiros; vale também para ponteiros (`p[i]`)
- **Parâmetro `int v[]` ≡ `int v[5]` ≡ `int *v`**: não há cópia nem tamanho; passe `n` e use `const` se for só leitura; **não devolva array local**
- **`v + 1`** avança um elemento; **`&v + 1`** avança o **array inteiro** (tipo `int (*)[5]`)
- **Multidimensional:** `m` decai para `int (*)[3]`; `m[i][j]` ≡ `*(*(m + i) + j)`; a **última dimensão** é obrigatória em parâmetros
- **`int *ap[3]`** (array de ponteiros) ≠ **`int (*pa)[3]`** (ponteiro para array); `char *nomes[]` guarda endereços, `char nm[][6]` guarda os bytes
- **`char s[]`** (cópia modificável) ≠ **`const char *s`** (aponta para literal em `.rodata`)

---

# 🔗 Conexões com outras anotações

- [[Arrays em C]] — o array em si: declaração, memória contígua, limites
- [[Ponteiros em C]] — `&`, `*`, `NULL`, `const` e `void *`
- [[Aritmética de Ponteiros em C]] — `p + n`, `p2 - p1` e o limite "um além do fim"
- [[Operadores em C]] — precedência de `[]`, `*` e `&`
- [[Sintaxe, Blocos e Escopo em C]] — ponteiro pendurado ao devolver variável local
- [[Variáveis e Constantes em C]] — `.rodata`, `const` e literais de string
- [[Como C Funciona]] — `argv` e as regiões de memória do processo
- [[Arrays e Slices]] — em Go, o *slice* carrega ponteiro, tamanho e capacidade
- [[Arrays]] — a visão de estruturas de dados (acesso O(1), cache) sobre o mesmo bloco
