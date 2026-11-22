---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: arrays multidimensionais, VLA em parâmetros); experimentos com gcc 13.3, clang 18, UBSan e AddressSanitizer em Linux x86-64"
---
# Matrizes em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3, `clang` 18, `-fsanitize=bounds` e `-fsanitize=address` | Anterior: [[Arrays em C]] | Próxima: [[Strings em C]] | Compare com [[Arrays e Slices]] (Go)

---

# 🔲 O que é uma matriz em C

C **não tem um tipo "matriz"** à parte. Uma matriz é só um **array cujos elementos são arrays** ([[Arrays em C]]):

```c
int m[2][3];        // array de 2 elementos, cada um um int[3]  →  2 linhas × 3 colunas
```

Por isso tudo o que vale para arrays vale aqui: **memória contígua**, tamanho **fixo**, **sem verificação de limites** e **`decay`** para ponteiro ao passar para funções. Esta nota aprofunda o que é específico de **duas ou mais dimensões**: a organização na memória, o custo de percorrer, a passagem para funções e os erros típicos.

> 💡 Leitura da declaração: `int m[2][3]` é "`m` é um array de 2 `int[3]`". A **primeira** dimensão é o número de **linhas**; a **última** é o tamanho de cada **linha**.

---

# 📝 Declaração e inicialização

| Forma | Resultado |
|---|---|
| `int m[2][3] = {{1,2,3},{4,5,6}};` | linhas explícitas: `1 2 3 / 4 5 6` |
| `int p[2][3] = {{1},{4,5}};` | o que faltar é **zerado**: `1 0 0 / 4 5 0` |
| `int z[2][3] = {0};` | **tudo zero** (mesmo idioma de [[Arrays em C]]) |
| `int d[3][3] = {[0][0]=1, [1][1]=1, [2][2]=1};` | **designada** (C99): matriz identidade `100 010 001` |
| `int q[][3] = {{1,2,3},{4,5,6},{7,8,9}};` | **só a primeira** dimensão pode ser omitida: o compilador conta 3 linhas |

Regra: **todas as dimensões, menos a primeira, são obrigatórias** (`int q[][]` é erro). O motivo aparece logo abaixo: sem o tamanho da linha, o compilador não consegue calcular onde cada linha começa.

---

# 🗺️ Como a matriz fica na memória

A memória é uma **faixa linear de bytes**: não existe "linha e coluna" lá. C armazena as linhas **uma depois da outra** (ordem ***row-major***):

```
 m[0][0] m[0][1] m[0][2] │ m[1][0] m[1][1] m[1][2]
    1       2       3    │    4       5       6          → 6 ints seguidos
```

Resultados dos testes:

| Expressão | Valor | Significado |
|---|---|---|
| `sizeof m` | **24** | 6 ints × 4 bytes |
| `sizeof m[0]` | **12** | **uma linha** (3 ints) |
| `sizeof m[0][0]` | **4** | um elemento |
| `&m[1][0] - &m[0][0]` | **3** | a linha 1 começa **3 elementos** depois |
| `&m[0][2] + 1 == &m[1][0]` | **1** (verdadeiro) | a linha 1 começa **logo após** o fim da linha 0 |
| `(char*)(m+1) - (char*)m` | **12 bytes** | somar 1 a `m` **pula uma linha inteira** |

## A conta do endereço

Para achar `m[i][j]`, o compilador calcula, a partir do início:

```
deslocamento (em elementos) = i × COLUNAS + j
```

É por isso que a **quantidade de colunas é indispensável**: sem ela, `i × COLUNAS` é impossível. O número de **linhas** não entra na conta (só serve para evitar ultrapassar o fim). Em 3 dimensões, `int c[2][3][4]`, a ideia se repete: `c[i][j][k]` está em `i×(3×4) + j×4 + k`. Testado: `&c[1][2][3] - &c[0][0][0]` deu **23** = 1×12 + 2×4 + 3.

Como tudo é contínuo, a mesma matriz também pode ser vista como **um array "achatado"** (*flat*) de `lin × col` elementos, acessado por `flat[i * col + j]`. É assim que bibliotecas numéricas e imagens costumam guardar dados. Cuidado: percorrer a matriz inteira via `&m[0][0]` e `flat[k]` funcionou nos testes, mas estritamente `&m[0][0]` aponta para um elemento da **primeira linha**, e o padrão discute se andar além dela por esse ponteiro é permitido; quando possível, prefira índices duplos ou um **bloco único** alocado como tal ([[Alocação Dinâmica em C]]).

---

# ⚡ Percorrer por linha × por coluna

Como as linhas são contínuas, **ler na ordem da memória** (linha a linha) é **muito mais rápido** que pular de linha em linha. O processador traz da memória **blocos** (linhas de *cache*, normalmente 64 bytes = 16 `int`), então depois de ler `m[i][0]` os vizinhos `m[i][1]`, `m[i][2]`... já estão no *cache*. Pular `m[0][j]`, `m[1][j]`, `m[2][j]` aproveita **um** elemento por bloco carregado ([[Arrays]]).

```c
for (int i = 0; i < N; i++)          // RÁPIDO: o j (último índice) varia mais rápido
    for (int j = 0; j < N; j++)
        soma += m[i][j];

for (int j = 0; j < N; j++)          // LENTO: salta uma linha inteira (N ints) a cada passo
    for (int i = 0; i < N; i++)
        soma += m[i][j];
```

Medido com uma matriz de **4096 × 4096** `int` (64 MiB), `-O1`, neste ambiente:

| Percurso | Tempo | Soma |
|---|---|---|
| **linha a linha** | ~**9 ms** | igual |
| **coluna a coluna** | ~**128 ms** | igual |

Resultado: **12 a 15 vezes mais lento**, com o **mesmo** resultado e a mesma quantidade de contas. Os números mudam de máquina para máquina, mas a regra é universal: **o último índice deve ser o do laço mais interno**.

---

# 🧮 Operações clássicas

**Transposta:** troca linhas por colunas (`t[j][i] = m[i][j]`); a matriz `2×3` vira `3×2`:

```
1 2 3         1 4
4 5 6   →     2 5
              3 6
```

**Multiplicação** `C = A × B` (`A` é `n×k`, `B` é `k×m`, `C` é `n×m`): cada `C[i][j]` é o produto da linha `i` de A pela coluna `j` de B:

```c
static void multiplica(size_t n, size_t k, size_t m,
                       const int A[n][k], const int B[k][m], int C[n][m]) {
    for (size_t i = 0; i < n; i++)
        for (size_t j = 0; j < m; j++) {
            int s = 0;
            for (size_t p = 0; p < k; p++) s += A[i][p] * B[p][j];
            C[i][j] = s;
        }
}
// A = {{1,2,3},{4,5,6}}, B = {{7,8},{9,10},{11,12}}  →  C = {{58,64},{139,154}}
```

Os tamanhos vêm **antes** da matriz na lista de parâmetros, o que leva ao próximo assunto.

---

# 📞 Passando matrizes para funções

Como em [[Arrays e Ponteiros em C]], a matriz **decai**: a função recebe um **ponteiro para a primeira linha** (`int (*)[3]`), **sem cópia**; alterar `m` lá dentro altera a original. Há três formas de declarar o parâmetro:

| Forma | Quando usar | Observação |
|---|---|---|
| `void f(int m[][3], size_t lin)` | colunas **fixas** e conhecidas | equivale a `int (*m)[3]`; `sizeof m[0]` = 12 |
| `void f(size_t lin, size_t col, int m[lin][col])` | colunas **só conhecidas em tempo de execução** | **VLA como parâmetro**; os tamanhos vêm **antes** |
| `void f(int *flat, size_t lin, size_t col)` | bloco "achatado" | acesso manual `flat[i*col + j]` |

## VLA em parâmetro

```c
void imprime(size_t n, size_t m, int a[n][m]) {
    // sizeof a    = 8   (um ponteiro: o decay continua valendo)
    // sizeof a[0] = 12  (uma linha = m ints, calculado em execução)
    // a[i][j] funciona normalmente
}
```

Resultados reais, chamando a mesma função com a matriz `2×3` e com a transposta `3×2`: `sizeof a` = **8** nos dois casos, `sizeof a[0]` = **12** e **8**: a conta `i × m + j` usa o `m` recebido, **sem depender de uma constante**. O `gcc` ainda avisa se alguém usar `sizeof a`: `-Wsizeof-array-argument`.

**Ordem dos parâmetros:** o tamanho tem que ser declarado **antes** de ser usado: `void g(int a[n][n], size_t n)` é erro de compilação (`'n' undeclared`). Em um protótipo, quando se quer manter a ordem, use `[*]`: `void f(size_t n, size_t m, int a[*][*]);`.

## ⚠️ Matriz **não** é `int **`

```c
void f(int **m);                 // espera um ponteiro para PONTEIROS
int m[2][3] = {{1,2,3},{4,5,6}};
f(m);                            // gcc: warning: incompatible pointer type
```

`m` decai para **`int (*)[3]`** (ponteiro para linha de 3 `int`), **não** para `int **` (ponteiro para ponteiro). As representações são diferentes: na matriz, as linhas estão **dentro** do bloco; em `int **`, cada linha é **outro bloco** apontado por um ponteiro ([[Arrays e Ponteiros em C]]). Ignorar o aviso e rodar o programa lê **bytes de dados como se fossem endereços** e termina em `Segmentation fault`.

## `const` e a pegadinha do C17

`void g(const int m[][3])` recebendo um `int m[2][3]` parece natural, mas **no C17** o `gcc -Wpedantic` avisa: *invalid use of pointers to arrays with different qualifiers in ISO C before C2X*. Em **C23** o aviso **desaparece** (o padrão passou a tratar o `const` do elemento como qualificador do array). Na prática, funciona nos dois.

---

# 🧱 Matriz de tamanho decidido em execução

Quando linhas e colunas só são conhecidas na execução, as duas formas de [[Alocação Dinâmica em C]] (`int **` com um bloco por linha, ou bloco único com `i * col + j`) funcionam. Há uma terceira, que une o melhor das duas: **um único `malloc`, mas com acesso `a[i][j]`**, usando um **ponteiro para VLA**:

```c
size_t lin = 3, col = 4;
int (*a)[col] = malloc(lin * sizeof *a);   // a: ponteiro para "linha de col ints"
if (!a) return 1;
a[2][3] = 23;                              // sintaxe normal de matriz
free(a);                                   // UM free
```

Testado com `gcc` (com ASan e UBSan, sem erros), `clang` e Valgrind (sem vazamentos): `sizeof *a` = **16** (uma linha de 4 `int`), `sizeof a` = **8** (é só um ponteiro) e `&a[1][0] - &a[0][0]` = **4**. O bloco é **contíguo** (bom para o *cache*) e é liberado de uma só vez.

> ⚠️ Esse idioma depende de **VLA**/*variably modified types*, aceitos por `gcc` e `clang`, mas **opcionais** no C11 (e não suportados pelo MSVC). Se a portabilidade importar, use o bloco único com `i * col + j`.

---

# 🧨 Erros típicos

## 1. A vírgula **não** separa índices

```c
printf("%d\n", m[1, 2]);        // NÃO é m[1][2]!
```

Em C, `1, 2` é o **operador vírgula** ([[Operadores em C]]): avalia `1`, descarta e vale `2`. Então `m[1, 2]` é `m[2]`, que é uma **linha** (um `int[3]`, que decai para endereço), e ainda **fora** das 2 linhas existentes. O `gcc` avisa (`left-hand operand of comma expression has no effect` e `format '%d' expects argument of type 'int', but argument 2 has type 'int *'`) e o programa imprimiu **lixo** (`1149974024`). Sempre `m[i][j]`.

## 2. Coluna fora do limite que "cai" na linha seguinte

```c
int m[2][3] = {{1,2,3},{4,5,6}};
m[0][4];          // lê m[1][1] (= 5): não dá erro nenhum
m[0][6];          // fora de TODA a matriz: lixo
```

Sem ferramentas, `m[0][4]` devolveu **5** sem nenhum aviso, porque `4` ainda está dentro dos 24 bytes da matriz. Com `-fsanitize=bounds`, o UBSan acusa ambos: `index 4 out of bounds for type 'int [3]'` e `index 6 out of bounds for type 'int [3]'`, porque **cada índice precisa respeitar o limite da sua própria dimensão**. O ASan só acusa o caso que ultrapassa o **objeto inteiro** (`stack-buffer-overflow` em `m[0][6]`). Por isso: **teste com `-fsanitize=undefined,address`** ([[Erros Clássicos de Memória em C]]).

## 3. Matriz local grande estoura a pilha

Uma `int grande[2048][2048]` **local** (16 MiB) fechou o programa com **`Segmentation fault`** (código 139): a pilha tem 8 MiB ([[Stack vs Heap em C]]). Para matrizes grandes use **`static`**/global (região de dados) ou a ***heap***.

## 4. Outras armadilhas

- **`sizeof` em parâmetro:** vira o tamanho de um ponteiro (8): passe `lin` e `col`
- **Atribuir/comparar com `=` e `==`:** não copia nem compara conteúdo ([[Arrays em C]]); para copiar, `memcpy(cp, m, sizeof m)` (testado: copiou os 24 bytes)
- **Trocar linha e coluna** na hora de dimensionar (`m[col][lin]`): o erro só aparece na conta do endereço
- **Passar `m` para `int **`** (acima): aviso que **não** deve ser ignorado

---

# 🔤 Matriz de strings

Uma matriz de `char` guarda os **textos dentro dela**; um array de `char *` guarda **endereços** de textos que estão em outro lugar. Essa diferença (`char nm[][6]` × `char *nomes[]`) é o assunto de [[Arrays e Ponteiros em C]] e de [[Strings em C]].

---

# 🆚 Comparação com Go

| | C | Go |
|---|---|---|
| Declaração | `int m[2][3]` | `var m [2][3]int` |
| Atribuição `b = a` | **erro** (não se atribui array) | **copia** a matriz inteira ([[Arrays e Slices]]) |
| Tamanho dinâmico | VLA ou `malloc` | **slice de slices** (`[][]int`): cada linha é **outro** array, **sem** contiguidade garantida |
| Limites | **sem verificação** (UB) | *panic* controlado |

Em Go, uma `[2][3]int` também é contígua e *row-major*; a matriz "do dia a dia" costuma ser um `[][]int`, que lembra o `int **` de C.

---

# ✅ Resumo do Conceito

- Uma matriz em C é um **array de arrays**: `int m[2][3]` = 2 linhas de `int[3]`; **só a primeira dimensão** pode ser omitida na declaração com inicializador
- Memória **contígua** em ordem de **linha** (*row-major*): `m[i][j]` está em `i × COLUNAS + j`; `sizeof m` = 24, `sizeof m[0]` = 12 (uma linha); em 3D, `i×(L×C) + j×C + k`
- **Percorra com o último índice no laço interno**: neste teste, 4096×4096 levou ~**9 ms** por linha e ~**128 ms** por coluna (**12–15× mais lento**), por causa do *cache*
- **Funções:** `m` decai para `int (*)[COLUNAS]` (**sem cópia**, a função altera a original); a **coluna fixa** (`int m[][3]`) ou **VLA como parâmetro** (`size_t n, size_t m, int a[n][m]`, tamanhos **antes**; protótipo com `[*]`)
- **Matriz ≠ `int **`**: tipos incompatíveis (aviso do `gcc`); `const int m[][3]` gera aviso `-Wpedantic` no C17 e não no C23
- **Dinâmica:** `int (*a)[col] = malloc(lin * sizeof *a);` → **um** `malloc`, **um** `free`, acesso `a[i][j]` (depende de VLA); alternativa portável: bloco único `i * col + j`
- **Erros:** `m[1, 2]` **não** é `m[1][2]` (operador vírgula); `m[0][4]` "funciona" lendo a linha seguinte (UBSan detecta, ASan não); matriz local grande **estoura a pilha** (16 MiB → *Segmentation fault*)
- Copiar com `memcpy(cp, m, sizeof m)`; `=` e `==` não servem em arrays

---

# 🔗 Conexões com outras anotações

- [[Arrays em C]] — a base: contiguidade, `sizeof`, limites, VLA e inicialização
- [[Arrays e Ponteiros em C]] — `decay` de `int (*)[3]`, `*(*(m+i)+j)`, `int (*)[3]` × `int *[3]` e matriz de strings
- [[Aritmética de Ponteiros em C]] — "somar 1 a `m` pula uma linha" (escala pelo tamanho do elemento)
- [[Alocação Dinâmica em C]] — matriz dinâmica: `int **` × bloco único × ponteiro para VLA
- [[Stack vs Heap em C]] — por que a matriz local grande estoura a pilha
- [[Erros Clássicos de Memória em C]] — *buffer overflow*, UBSan e ASan na prática
- [[Funções em C]] e [[Funções e Ponteiros em C]] — passagem por valor, parâmetros e `const`
- [[Operadores em C]] — o operador vírgula e a precedência de `[]` e `*`
- [[Strings em C]] — matrizes de `char` como tabelas de textos
- [[Arrays]] — o conceito geral de array e a localidade de *cache* (Estruturas de Dados)
- [[Arrays e Slices]] — a comparação com Go
