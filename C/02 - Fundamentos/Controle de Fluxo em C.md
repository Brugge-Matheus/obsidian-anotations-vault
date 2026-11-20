---
tags:
  - c
  - c/fundamentos
source: "Padrão ISO C (C17/C23: instruções de seleção, iteração e salto); experimentos com gcc 13.3 e clang 18 em Linux x86-64"
---
# Controle de Fluxo em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 e `clang` 18 | Anterior: [[Operadores em C]] | Próxima: [[Conversões de Tipo em C]] | Compare com [[Controle de Fluxo]] (Go)

---

# 🧭 O que é fluxo de controle

> 💡 **Fluxo de controle:** a **ordem** em que as instruções são executadas. Por padrão é **sequencial** (de cima para baixo); as estruturas de controle **desviam** esse caminho para **escolher** entre alternativas ou **repetir** trechos.

| Tipo | Estruturas em C |
|---|---|
| **Seleção** (decidir) | `if` / `else`, `switch` |
| **Iteração** (repetir) | `while`, `do ... while`, `for` |
| **Salto** (desviar) | `break`, `continue`, `goto`, `return` |

Todas decidem com base em uma **condição**, que em C é **qualquer expressão escalar**: **0 é falso** e **qualquer outro valor é verdadeiro** ([[Tipos de Dados em C]]). Por isso `while (n)`, `if (ptr)` e `if (!erro)` são válidos, e as condições costumam usar os operadores de [[Operadores em C]]. O corpo de cada estrutura é **uma instrução**, quase sempre um **bloco** `{ }` ([[Sintaxe, Blocos e Escopo em C]]).

---

# 🔀 `if` / `else`

```c
if (idade >= 18) {
    puts("maior");
} else if (idade >= 16) {
    puts("16 ou 17");
} else {
    puts("menor");
}
```

- Os **parênteses** da condição são **obrigatórios**;
- A cadeia `else if` é avaliada **de cima para baixo** e executa **apenas o primeiro** ramo verdadeiro (com `idade = 17`: imprime `16 ou 17`);
- **Use sempre chaves.** Sem elas, o `else` pende do `if` mais próximo (*dangling else*) e uma linha duplicada vira o bug do `goto fail`; ambos estão em [[Sintaxe, Blocos e Escopo em C]] e [[Convenções e Estilo]]. O mesmo vale para o `;` perdido em `if (x);`.
- Prefira **retornos antecipados** (*guard clauses*) a `if` aninhados: `if (erro) return -1;` no início deixa o caminho principal sem recuos.

---

# 🎛️ `switch`

Escolhe entre **vários valores fixos** de uma expressão **inteira**:

```c
static const char *dia(int d) {
    switch (d) {
    case 1:
    case 7:
        return "fim de semana";
    case 2: case 3: case 4: case 5: case 6:
        return "dia util";
    default:
        return "invalido";
    }
}
// dia(1) → "fim de semana" ; dia(3) → "dia util" ; dia(9) → "invalido"
```

## Regras

| Regra | Detalhe (erro real do `gcc`) |
|---|---|
| A expressão deve ser **inteira** (`int`, `char`, `enum`...) | `switch (1.0)` → `switch quantity not an integer`; **strings também não** (use `strcmp` com `if`) |
| Cada `case` precisa de uma **constante inteira** | literais, `enum` e `#define` servem; **`const int N` não**: `case label does not reduce to an integer constant` (veja [[Variáveis e Constantes em C]]) |
| Valores **não podem repetir** | `duplicate case value` |
| `default` é **opcional** e pode ficar em qualquer posição | roda quando nenhum `case` combina |

> 💡 O `switch` funciona como um **salto para um rótulo**: o programa pula para o `case` certo e **continua dali em diante**. Sendo constantes, o compilador pode gerar uma **tabela de saltos** em vez de comparações em sequência (mais sobre isso na seção *Por dentro*).

## O *fall-through* (cair no próximo `case`)

Sem `break`, a execução **continua** nos `case` seguintes. Com `n = 1, 2, 3`:

```c
case 1: printf("um ");          // sem break: cai para o case 2
case 2: printf("dois ");  break;
default: printf("outro ");
```

Resultado real: `um dois`, `dois`, `outro`. Isso permite **agrupar rótulos** (como `case 1: case 7:` acima, que é intencional), mas **esquecer o `break` é erro clássico**. O `-Wextra` avisa (`-Wimplicit-fallthrough`: *this statement may fall through*). Quando a queda é **intencional**, marque-a e o aviso some:

```c
case 1: printf("um ");
        [[fallthrough]];                       // C23
        // ou: __attribute__((fallthrough));   // GNU, funciona em C17 no gcc/clang
case 2: ...
```

## Declarações dentro de um `case`

Antes do **C23**, um rótulo precisa ser seguido de uma **instrução**, não de uma declaração. O `gcc` 13 aceita `case 1: int x = 5;`, mas o `-Wpedantic` avisa (o `clang` diz *label followed by a declaration is a C23 extension*). A forma portável é **abrir um bloco**:

```c
case 1: {
    int x = 5;
    printf("%d\n", x);
    break;
}
```

---

# 🔁 Laços de repetição

| Estrutura | Testa a condição | Executa o corpo | Use quando |
|---|---|---|---|
| `while (c) { }` | **antes** | **0 ou mais** vezes | não se sabe quantas vezes repetir |
| `do { } while (c);` | **depois** | **pelo menos 1** vez | o corpo deve rodar ao menos uma vez (menus, validação de entrada) |
| `for (i; c; p) { }` | **antes** | **0 ou mais** vezes | contagem ou percurso com início, condição e passo claros |

```c
int i = 3;
while (i > 0) { printf("%d ", i); i--; }              // 3 2 1
int k = 10;
do { printf("k=%d ", k); k++; } while (k < 3);       // k=10  (roda 1 vez mesmo com condição falsa)
for (int j = 0; j < 3; j++) printf("j=%d ", j);      // j=0 j=1 j=2
```

## Anatomia do `for`

```c
for (inicialização; condição; passo) corpo
//   1 (uma vez)     2 (a cada volta, antes)   4 (a cada volta, depois do corpo)
```

A ordem é: **inicialização** → testa **condição** → **corpo** → **passo** → testa a condição de novo. É equivalente a um `while` com a inicialização antes e o passo no fim do corpo.

- Qualquer uma das três partes pode ser **omitida**: `for (;;)` é um **laço infinito** (idioma comum, encerrado por `break` ou `return`);
- Desde o **C99** a variável pode ser declarada no próprio `for`, e seu **escopo é só o laço** ([[Sintaxe, Blocos e Escopo em C]]);
- O operador **vírgula** ([[Operadores em C]]) permite várias variáveis: `for (int p = 0, q = 10; p < q; p += 4, q -= 1)` percorre `(0,10)` e `(4,9)` e para em `(8,8)`.

## Armadilhas reais de laços

| Armadilha | O que acontece | Aviso do `gcc` |
|---|---|---|
| **Contagem regressiva sem sinal** `for (size_t i = n - 1; i >= 0; i--)` | `i >= 0` é **sempre verdadeiro** (um `unsigned` nunca fica negativo): laço **infinito** | `-Wtype-limits` (*comparison of unsigned expression in '>= 0' is always true*) |
| `for (int i = 0; i < strlen(s); i++)` | compara `int` com `size_t` ([[Conversões de Tipo em C]]) e **recalcula** `strlen` a cada volta (custo O(n) por volta) | `-Wsign-compare` |
| `while (c);` ou `for (...);` | o `;` vira corpo **vazio**; o bloco seguinte roda fora do laço | `-Wempty-body`, `-Wmisleading-indentation` |
| `<=` em vez de `<` (*off-by-one*) | lê/escreve **um elemento além** do fim de um *array* | nenhum; use [[Convenções e Estilo]] e *sanitizers* |
| Modificar a variável de controle no corpo | laço difícil de entender | — |

> ✅ Para um *array* de `n` elementos o padrão é `for (size_t i = 0; i < n; i++)`. Para contar de trás para frente com `size_t`: `for (size_t i = n; i-- > 0; )`.

---

# ⏭️ `break` e `continue`

| Instrução | Efeito |
|---|---|
| `break;` | **sai** do laço (`while`, `do`, `for`) **ou do `switch`** mais interno |
| `continue;` | **pula** o resto desta volta e vai para a **próxima** (no `for`, executa o passo antes) |

```c
for (int j = 0; j < 6; j++) {
    if (j == 1) continue;     // pula o 1
    if (j == 4) break;        // encerra o laço no 4
    printf("%d ", j);         // 0 2 3
}
```

## Duas pegadinhas

1. **`break` só sai da estrutura mais interna.** Em dois `for` aninhados, ele encerra só o de dentro (testado: `(0,0) (1,0) (2,0)`).
2. **`break` dentro de um `switch` que está dentro de um laço sai do `switch`, não do laço:**

```c
for (int j = 0; j < 4; j++) {
    switch (j) {
    case 1: break;                  // sai só do switch
    default: printf("j=%d ", j);
    }
}
// j=0 j=2 j=3   ← o for continuou
```

Para sair de **vários níveis** use uma variável-sinalizador (*flag*), mova o laço para uma **função** e use `return`, ou use `goto` (a seguir).

---

# 🦘 `goto`

Salta **incondicionalmente** para um **rótulo** (*label*) da **mesma função** (os rótulos têm escopo de função, visto em [[Sintaxe, Blocos e Escopo em C]]):

```c
    if (erro) goto fim;
    ...
fim:
    limpar();
```

Usado sem critério, produz código "espaguete" (a crítica clássica de Dijkstra). Mas há **um uso legítimo e muito comum**: a **limpeza centralizada de recursos** quando há vários pontos de falha, padrão do kernel Linux:

```c
int processar(const char *caminho) {
    int status = -1;
    FILE *f = NULL;
    char *buf = NULL;

    f = fopen(caminho, "r");
    if (f == NULL) goto fim;

    buf = malloc(64);
    if (buf == NULL) goto fim;

    if (fgets(buf, 64, f) == NULL) goto fim;
    printf("leu: %s", buf);
    status = 0;

fim:
    free(buf);                    // free(NULL) é seguro
    if (f != NULL) fclose(f);
    return status;
}
```

Testado: com um arquivo existente retorna `0`; com `"/nao/existe"` retorna `-1`, **sem vazar nada**. Todos os caminhos passam pelo mesmo trecho de limpeza (evita repetir `free`/`fclose` em cada `if`). Regras práticas: salte **para frente**, **dentro da mesma função**, e **nunca para dentro** do escopo de um *array* de tamanho variável (VLA, [[Variáveis e Constantes em C]]).

---

# ⚙️ Por dentro: tudo vira comparação e salto

Para a CPU **não existem** `if` nem `for`: só instruções que **comparam** e **saltam** (`cmp`, `jmp`, `jl`...). É o que o compilador gera ([[Fases da Compilação]], `gcc -S`). Para este laço:

```c
int soma(int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += i;
    return s;
}
```

o `gcc -O0` produz (sintaxe Intel, trecho do corpo):

```asm
    mov  DWORD PTR -8[rbp], 0     ; s = 0
    mov  DWORD PTR -4[rbp], 0     ; i = 0
    jmp  .L2                      ; salta para o teste da condição
.L3:                              ; ── corpo ──
    mov  eax, DWORD PTR -4[rbp]
    add  DWORD PTR -8[rbp], eax   ; s += i
    add  DWORD PTR -4[rbp], 1     ; i++
.L2:                              ; ── condição ──
    mov  eax, DWORD PTR -4[rbp]
    cmp  eax, DWORD PTR -20[rbp]  ; compara i com n
    jl   .L3                      ; se i < n, volta ao corpo
```

O `for` virou **um salto ao teste, o corpo, o passo, e um salto condicional de volta**. É também por isso que o `switch` com valores consecutivos pode virar uma **tabela de saltos**: um único salto indireto em vez de várias comparações. Com `-O2` ([[O Compilador]]) o compilador reorganiza ou elimina laços, mas o **comportamento observável** é o mesmo (regra *as-if*, [[Como C Funciona]]). O papel do contador de programa e dos desvios na CPU está em [[Processadores]].

---

# ✅ Resumo do Conceito

- **Fluxo de controle** = ordem de execução; em C a condição é **qualquer escalar** (0 falso, resto verdadeiro) e o corpo é uma instrução (use **chaves** sempre)
- **`if`/`else if`** avalia de cima para baixo e executa **só o primeiro** ramo verdadeiro
- **`switch`**: expressão **inteira**, `case` com **constantes** (`const int` não vale), sem repetições; **sem `break` há *fall-through*** (`-Wimplicit-fallthrough`; marque o intencional com `[[fallthrough]]`); use **chaves** para declarar variáveis num `case`
- **`while`** testa antes, **`do-while`** executa ao menos uma vez, **`for`** = inicialização; condição; passo (partes opcionais, `for (;;)` é infinito, variável de controle com escopo do laço)
- **Armadilhas:** contagem regressiva com `unsigned` (`i >= 0` sempre verdadeiro), `strlen` na condição, `;` após `while`/`for`/`if`, *off-by-one*
- **`break`** sai do laço **ou `switch`** mais interno (`break` num `switch` dentro de `for` **não** sai do `for`); **`continue`** vai para a próxima volta
- **`goto`**: só dentro da função; uso legítimo é a **limpeza centralizada de recursos**
- No código de máquina tudo é **`cmp` + salto** (`jmp`, `jl`...); `switch` pode virar tabela de saltos

---

# 🔗 Conexões com outras anotações

- [[Operadores em C]] — relacionais, lógicos e o operador vírgula usados nas condições
- [[Tipos de Dados em C]] — 0 falso/resto verdadeiro, `size_t` e `unsigned`
- [[Conversões de Tipo em C]] — por que `i < strlen(s)` e `i >= 0` com `unsigned` enganam
- [[Sintaxe, Blocos e Escopo em C]] — blocos, *dangling else*, escopo de função dos rótulos
- [[Variáveis e Constantes em C]] — `enum`/`#define` como `case`, VLAs e `goto`
- [[Convenções e Estilo]] — chaves obrigatórias (`goto fail`), `-Wextra` e avisos
- [[Fases da Compilação]] e [[O Compilador]] — `gcc -S` e otimização de laços
- [[Processadores]] — instruções de comparação e salto na CPU
- [[Controle de Fluxo]] — o equivalente em Go (o `switch` não cai no próximo `case` e o `for` substitui `while` e `do-while`)
