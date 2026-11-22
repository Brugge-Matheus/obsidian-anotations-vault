---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: arrays, inicializadores, VLA); experimentos com gcc 13.3, AddressSanitizer e UBSan em Linux x86-64"
---
# Arrays em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3, `-fsanitize=address` e `-fsanitize=bounds` | Anterior: [[Conversões de Tipo em C]] | Próxima: [[Matrizes em C]] | Compare com [[Arrays e Slices]] (Go) e veja a visão geral em [[Arrays]] (Estruturas de Dados)

---

# 🧱 O que é um array

> 💡 **Array (vetor):** uma sequência de **N elementos do mesmo tipo**, guardados **lado a lado** na memória, acessados por um **índice**.

```c
int notas[5];            // 5 ints consecutivos: notas[0] ... notas[4]
```

Três ideias definem tudo o que vem a seguir:

1. **Tamanho fixo:** o `N` é parte do tipo (`int[5]`) e **não muda** depois de declarado.
2. **Contíguo:** os elementos ocupam **bytes seguidos**, sem lacunas.
3. **Sem rede de proteção:** C **não verifica** se o índice é válido, nem guarda o tamanho junto do array (ao contrário de Go, [[Arrays e Slices]]).

> 💡 **Endereço:** o número que identifica a posição de um byte na memória. Como o array é contíguo, basta conhecer o endereço do **primeiro** elemento e o tamanho do tipo para achar qualquer outro. O operador `&x` dá o endereço de `x`, e o formato `%p` do `printf` o imprime. Guardar e manipular endereços é o assunto de [[Ponteiros em C]] (módulo 04); por ora basta saber que `&x` dá o endereço de `x`.

---

# 📝 Declaração e inicialização

| Forma | Resultado |
|---|---|
| `int a[5] = {1, 2, 3};` | `1 2 3 0 0` — o que **faltar é zerado** |
| `int b[] = {10, 20, 30, 40};` | o compilador **conta**: `sizeof b` = 16 bytes (4 × 4) |
| `int c[5] = {0};` | **todos zero** (idioma para zerar) |
| `int d[6] = {[1] = 7, [4] = 9};` | **designada** (C99): `0 7 0 0 9 0` |
| `char s[] = "oi";` | **3** bytes: `'o'`, `'i'` e o terminador `'\0'` ([[Strings em C]]) |
| `char t[] = {'o', 'i'};` | **2** bytes, **sem** terminador |

Valores reais conferidos acima. Pontos de atenção:

- **Sem inicializador**, o conteúdo depende de **onde** o array vive, igual às variáveis ([[Variáveis e Constantes em C]]): **local = lixo** (não zera); **global/`static` = zero** (confirmado: `global[4]` imprimiu `0 0 0 0`).
- Um inicializador com **mais** valores do que o tamanho é **erro de compilação**.
- Em `const int tabela[] = {...}` os dados vão para a seção **`.rodata`** (somente leitura); sem `const` e inicializado, para **`.data`**; sem inicializador e global, para **`.bss`**. Conferido com `nm`: `R tabela`, `D dados`, `B zeros` (veja [[Variáveis e Constantes em C]]).

---

# 📏 Descobrindo o tamanho: o idioma `sizeof`

O array **não carrega** o número de elementos, mas `sizeof` (calculado na compilação, [[Tipos de Dados em C]]) permite deduzi-lo **no mesmo escopo** em que ele foi declarado:

```c
int b[] = {10, 20, 30, 40};
sizeof b                    // 16  (bytes totais)
sizeof b[0]                 // 4   (bytes de um elemento)
sizeof b / sizeof b[0]      // 4   (número de elementos)
```

É comum embrulhar isso numa macro (ver *O Pré-processador*): `#define TAM(v) (sizeof(v) / sizeof((v)[0]))`. O resultado é `size_t`, então use `size_t` no índice: `for (size_t i = 0; i < TAM(b); i++)`, o que evita o aviso `-Wsign-compare` ([[Conversões de Tipo em C]]).

> ⚠️ Esse truque **só funciona onde o array foi declarado**. Dentro de uma função que **recebeu** o array, o `sizeof` devolve **8** (veja *Passando para funções*).

---

# 🗺️ Como o array fica na memória

Para `int v[4] = {1, 2, 3, 4};` os endereços reais impressos foram:

```
&v[0] = ...0720       ┌────┬────┬────┬────┐
&v[1] = ...0724       │ 1  │ 2  │ 3  │ 4  │   cada int ocupa 4 bytes
&v[2] = ...0728       └────┴────┴────┴────┘
&v[3] = ...072c      0720 0724 0728 072c   (endereços, em hexadecimal)
```

- Cada elemento está **4 bytes** depois do anterior (`sizeof(int)`); a diferença `(char *)&v[1] - (char *)&v[0]` deu **4**.
- **`&v` e `&v[0]` são o mesmo endereço** (`0x...0720`): o array **começa** onde começa seu primeiro elemento (o que isso implica está em [[Arrays e Ponteiros em C]]).
- O endereço do elemento `i` é: **endereço de `v[0]` + `i × sizeof(elemento)`**. É por isso que o índice começa em **0**: ele é o **deslocamento** (*offset*) a partir do início.

---

# 🚧 Índices fora dos limites

Um array `int v[5]` tem índices **0 a 4**. Usar `v[5]` (ou `v[-1]`) lê ou escreve **fora** do array: é **comportamento indefinido** (UB, [[Como C Funciona]]). Nenhum erro é emitido pelo programa. Teste real com `i = 5`, lendo e escrevendo `v[i]`:

| Como foi compilado | O que aconteceu |
|---|---|
| Sem opções | **Nada visível**: imprimiu `32767` (lixo vizinho) e terminou com código `0` |
| `-fsanitize=bounds` (UBSan) | `index 5 out of bounds for type 'int [5]'` |
| `-fsanitize=address` (ASan) | `stack-buffer-overflow ... READ of size 4` |
| `-O2 -Wall` com índice **constante** (`v[7]`) | `-Warray-bounds`: *array subscript 7 is above array bounds of 'int[5]'* |

O silêncio é o perigo: ler fora devolve lixo; **escrever** fora **corrompe** outras variáveis, o endereço de retorno e abre brechas de segurança (*buffer overflow*, em [[Erros Clássicos de Memória em C]]). Durante o desenvolvimento, compile com `-fsanitize=address,undefined`, como sugerido em [[Convenções e Estilo]].

> ✅ **Regra:** o laço correto sobre `n` elementos é `i < n` (**nunca `<=`**), e o índice válido máximo é **`n - 1`**.

---

# 🚫 Arrays não se atribuem nem se comparam com `=` e `==`

```c
int a[3] = {1, 2, 3}, b[3];
b = a;                         // ERRO: assignment to expression with array type
```

O array **não é um valor** que se possa copiar com `=`. Para copiar use um **laço** ou `memcpy` (de `<string.h>`): `memcpy(b, a, sizeof a);` (testado: `b` ficou `1 2 3`).

Comparar `a == c` **compila**, mas **compara endereços**, não conteúdos: com dois arrays de valores iguais deu `0` (o `gcc` avisa `-Warray-compare`). Para comparar conteúdo: `memcmp(a, c, sizeof a) == 0` (deu `1`).

| | Go ([[Arrays e Slices]]) | C |
|---|---|---|
| `b = a` | **copia** o array inteiro | **erro de compilação** |
| `a == c` | compara **elementos** | compara **endereços** |
| Índice inválido | *panic* controlado | **UB** |

---

# 📞 Passando para funções (aviso)

Ao passar um array para uma função, **não é feita uma cópia**: a função recebe o **endereço do primeiro elemento**. Duas consequências, vistas no teste:

```c
void dobra(int v[], size_t n) {
    // sizeof(v) aqui vale 8 (tamanho de um endereço), NÃO 16
    for (size_t i = 0; i < n; i++) v[i] *= 2;     // altera o array ORIGINAL
}
// fora: sizeof(v) = 16 ; depois de dobra(v, 4): 2 4 6 8
```

- A função **altera o array original** (não existe cópia);
- O tamanho se **perde**: por isso o padrão em C é **passar o tamanho como outro parâmetro** (`size_t n`). O `gcc` avisa quem tenta usar `sizeof` ali (`-Wsizeof-array-argument`: *'sizeof' on array function parameter 'v' will return size of 'int *'*).

O mecanismo por trás disso (chamado ***decay***) é o tema de [[Arrays e Ponteiros em C]].

---

# 🔲 Arrays multidimensionais

Um array de arrays. `int m[2][3]` é **2 linhas × 3 colunas**:

```c
int m[2][3] = {{1, 2, 3},
               {4, 5, 6}};
m[1][2]              // 6  (linha 1, coluna 2)
```

Na memória **não há "linhas e colunas"**: tudo é **uma sequência contínua**, com a ordem **linha por linha** (***row-major***):

```
 m[0][0] m[0][1] m[0][2] │ m[1][0] m[1][1] m[1][2]
    1       2       3    │    4       5       6        → 6 ints seguidos
```

Resultados reais: `sizeof(m)` = **24**, `sizeof(m[0])` = **12** (uma linha), `sizeof(m[0][0])` = 4. Logo, linhas = `sizeof m / sizeof m[0]` = 2 e colunas = `sizeof m[0] / sizeof m[0][0]` = 3. E `&m[1][0]` está **4 bytes** depois de `&m[0][2]`: a linha 1 começa **logo após** o fim da linha 0.

O elemento `m[i][j]` está no deslocamento **`i × colunas + j`** (elementos) a partir do início. Em C a **última dimensão é essencial** para o compilador fazer essa conta: ao passar para uma função, só a **primeira** dimensão pode ser omitida: `void f(int m[][3], size_t linhas)`. O assunto completo (VLA em parâmetros, custo do percurso, erros típicos) está em [[Matrizes em C]].

---

# 🎚️ Tamanho variável: VLA e limites da pilha

- **VLA** (*variable length array*, C99; opcional desde o C11): `int n = 3; int vla[n];` — o tamanho vem de uma variável em tempo de execução (`sizeof vla` deu **12**). Não aceita inicializador e vive na **pilha**.
- **Limite da pilha:** arrays locais grandes estouram a pilha. Um `char grande[16 * 1024 * 1024]` local (16 MiB) causou **`Segmentation fault`** (código 139) com a pilha padrão de **8 MiB** (`ulimit -s` = 8192).

Para tamanho **decidido em tempo de execução** ou **grande**, a solução é a **alocação dinâmica** ([[Alocação Dinâmica em C]]; onde cada coisa vive: [[Stack vs Heap em C]]).

---

# ✅ Resumo do Conceito

- **Array** = N elementos do **mesmo tipo**, **contíguos**, de **tamanho fixo**, indexados a partir de **0**
- **Inicialização parcial zera o resto** (`{1, 2, 3}` em `int[5]` → `1 2 3 0 0`); `{0}` zera tudo; `[i] = v` é designada (C99); local sem inicializador = **lixo**, global = **zero**
- **Número de elementos:** `sizeof v / sizeof v[0]` (só no escopo da declaração); use `size_t` no índice
- Elemento `i` fica em **endereço de `v[0]` + `i × sizeof(elemento)`**; `&v` e `&v[0]` são o mesmo endereço
- **Sem verificação de limites:** `v[n]` é **UB**; detecte com `-fsanitize=address,undefined` e `-Warray-bounds`
- **Não se copia com `=`** (erro) nem se compara com `==` (compara endereços): use `memcpy` e `memcmp`
- **Em funções**, o array chega como **endereço**: sem cópia, tamanho perdido (`sizeof` = 8), passe `n` à parte
- **Multidimensional** = array de arrays em **ordem de linha** (*row-major*); `m[i][j]` está em `i × colunas + j`; a **última dimensão** é obrigatória em parâmetros
- **VLA** existe mas vive na pilha (**8 MiB**); para grandes ou variáveis, use alocação dinâmica

---

# 🔗 Conexões com outras anotações

- [[Tipos de Dados em C]] — `sizeof`, `size_t` e tamanho dos tipos
- [[Variáveis e Constantes em C]] — valor inicial, `.data`/`.bss`/`.rodata` e VLA
- [[Conversões de Tipo em C]] — `size_t` × `int` no índice dos laços
- [[Controle de Fluxo em C]] — o `for` que percorre o array e o *off-by-one*
- [[Como C Funciona]] — UB e regiões de memória
- [[Convenções e Estilo]] — *sanitizers* e `-Wall -Wextra`
- [[Arrays e Slices]] — o equivalente em Go (valor copiável, com verificação de limites)
- [[Arrays]] — a visão de estruturas de dados (acesso O(1), cache) sobre o mesmo bloco contíguo
