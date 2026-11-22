---
tags:
  - c
  - c/fundamentos
  - c/memoria
source: "Padrão ISO C (C17/C23: strings, <string.h>, <ctype.h>, <stdlib.h>, <stdio.h>); experimentos com gcc 13.3, UBSan e AddressSanitizer em Linux x86-64"
---
# Strings em C

📚 **Referência:** Padrão ISO C (C17/C23), `<string.h>`, `<ctype.h>`, `<stdio.h>`, `<stdlib.h>` e experimentos reais com `gcc` 13.3 e `-fsanitize=address` | Anterior: [[Matrizes em C]] | Próxima: [[Funções em C]] | Compare com [[Strings em Profundidade]] (Go)

> 🧭 **Como ler esta nota:** ela vai do mais simples ao mais completo. As seções 1 a 5 ensinam **uma** string; a seção 6 ensina **várias** (listas de nomes, como `names[][100]`); as seções 7 a 12 trazem as ferramentas da biblioteca, os cuidados e os erros comuns. Se for sua primeira vez, leia na ordem.

---

# 🔤 1. O que é uma string em C

**C não tem um tipo "string".** Uma string é uma **convenção** sobre um array de `char` ([[Arrays em C]]):

> 💡 **String em C** = uma sequência de `char` seguida de um byte de valor **zero**, o **terminador nulo** (escrito `'\0'`). Os bytes **antes** dele são o texto; o `'\0'` apenas **marca o fim**.

Pense numa fila de caixas em que a última tem um aviso de "**fim**": quem lê a fila anda caixa por caixa até encontrar o aviso. Não existe uma etiqueta com o total de caixas, então, para saber o tamanho, é preciso **andar até o fim**.

Os valores reais de `char s[] = "oi";`:

```
  índice:      s[0]   s[1]   s[2]
  caractere:   'o'    'i'    '\0'
  valor:       111    105      0          → 3 bytes (sizeof s = 3)
                              ↑ marca o fim
```

Cada caractere é só um **número** (o código dele, em ASCII: `'o'` = 111, `'i'` = 105). Todas as funções de texto dependem do `'\0'`. **Sem ele**, elas continuam lendo a memória adiante até tropeçar num zero qualquer (comportamento indefinido, [[Erros Clássicos de Memória em C]]). Por isso a **regra número 1** das strings é:

> ✅ **Sempre reserve 1 byte a mais para o `'\0'`.** Um texto de 5 letras precisa de um array de, no mínimo, **6**.

## `'a'` não é `"a"`

| | O que é | Tamanho | Exemplo |
|---|---|---|---|
| `'a'` (aspas **simples**) | **um caractere** (um número; tipo `int` em C) | `sizeof('a')` = **4** | `char c = 'a';` |
| `"a"` (aspas **duplas**) | **uma string** (`'a'` + `'\0'`) | `sizeof("a")` = **2** | `char s[] = "a";` |
| `""` | a string **vazia** (só o `'\0'`) | `sizeof("")` = **1** | `char v[] = "";` |

Misturar os dois é um erro clássico: `char c = "a";` não compila direito (atribui um **endereço** a um `char`).

---

# 📝 2. Criando strings

## As formas de declarar

Cada linha abaixo foi compilada e medida (`sizeof` = bytes do array; `strlen` = letras até o `'\0'`):

| Declaração | `sizeof` | `strlen` | O que acontece |
|---|---|---|---|
| `char s[] = "oi";` | **3** | 2 | o compilador **conta** e copia o texto com o `'\0'` (forma mais comum) |
| `char s[10] = "oi";` | **10** | 2 | copia e **zera o resto** (`s[2]` a `s[9]` valem 0) |
| `char s[] = {'o','i','\0'};` | **3** | 2 | igual à primeira, escrita letra por letra (raro) |
| `char s[10] = {0};` | **10** | **0** | string **vazia** com 10 bytes de espaço (idioma para "zerar") |
| `char s[] = {'o','i'};` | **2** | — | **sem `'\0'`: não é uma string!** `strlen` daria lixo |
| `const char *p = "oi";` | **8** | 2 | `p` **aponta** para o texto fixo do programa (veja abaixo) |

Os bytes reais de duas delas:

```
char s[]   = "oi";       111 105   0
char s[10] = "oi";       111 105   0   0   0   0   0   0   0   0     ← sobra zerada
char s[]   = {'o','i'};  111 105                                     ← sem terminador
```

## Tamanho do array × tamanho do texto

Este é o ponto que mais confunde. O **array** tem um tamanho fixo (os bytes reservados); o **texto** dentro dele pode ser menor. O `'\0'` marca onde o texto termina:

```c
char g[10] = "oi";
```
```
 índice:   0    1    2    3    4    5    6    7    8    9
 valor:   'o'  'i'  \0    \0   \0   \0   \0   \0   \0   \0
          └─ texto ─┘└── espaço reservado, sem uso ──────────┘
          strlen(g) = 2                       sizeof g = 10
```

Como o fim é só uma marca, dá para **encurtar** uma string escrevendo um `'\0'` no meio dela:

```c
char u[] = "ola mundo";     // sizeof = 10, strlen = 9
u[3] = '\0';                // troca o espaço por '\0'
// printf("%s", u)  →  "ola"       (strlen = 3, sizeof continua 10)
// os bytes seguintes continuam lá: u + 4 ainda é "mundo"
```

E também **montar** uma string à mão, caractere a caractere (lembrando de fechar com `'\0'`):

```c
char t[6];
t[0] = 'o'; t[1] = 'i'; t[2] = '\0';      // "oi"
```

## Array × ponteiro para literal

Um **literal** (`"texto"` escrito no código) vive numa região **somente leitura** do programa (`.rodata`, [[Variáveis e Constantes em C]]). Há dois jeitos de usá-lo:

| | `char s[] = "oi";` | `const char *p = "oi";` |
|---|---|---|
| O que é `s` / `p` | um **array próprio** (3 bytes, uma cópia) | um **ponteiro** (8 bytes) para o literal |
| Pode alterar as letras? | ✅ `s[0] = 'O';` | ❌ **UB** (na prática, `Segmentation fault`) |
| Pode apontar para outro texto? | ❌ (array não se reatribui) | ✅ `p = "tchau";` |

A explicação do *segfault* está em [[Arrays e Ponteiros em C]]. **Regra prática: texto que será alterado → array; texto fixo → `const char *`.**

## Sequências de escape

Certos caracteres não dá para digitar direto no texto. Usa-se a barra invertida:

| Escape | Significa | | Escape | Significa |
|---|---|---|---|---|
| `\n` | nova linha | | `\\` | barra invertida |
| `\t` | tabulação | | `\"` | aspas duplas |
| `\0` | byte zero (o terminador) | | `\xHH` | byte em hexadecimal (`"\x41"` = `"A"`) |
| `\r` | retorno de carro | | `\ooo` | byte em octal (`"\101"` = `"A"`) |

**Literais vizinhos são concatenados** pelo compilador: `"ab" "cd" "ef"` vale `"abcdef"`. Útil para quebrar textos longos em várias linhas do código.

---

# 🖨️ 3. Mostrando strings

O básico de `<stdio.h>` (o assunto completo ficará em *`stdio`: Entrada e Saída*):

```c
const char *s = "Matheus";
printf("[%s]\n",   s);        // [Matheus]
printf("[%10s]\n", s);        // [   Matheus]   largura 10, alinhado à DIREITA
printf("[%-10s]\n", s);       // [Matheus   ]   alinhado à ESQUERDA
printf("[%.3s]\n", s);        // [Mat]          no máximo 3 caracteres
printf("[%*s]\n", 12, s);     // [     Matheus] largura passada como argumento
puts("oi");                   // escreve "oi" E uma nova linha sozinho
putchar('A');                 // escreve UM caractere
```

| Para mostrar | Use | Observação |
|---|---|---|
| uma **string** | `%s` | espera um `char *` terminado em `'\0'` |
| **um** caractere | `%c` | `printf("%c", 'A')` → `A` |
| o **código** do caractere | `%d` | `printf("%d", 'A')` → `65` |

O mesmo valor, dois formatos: `printf("%c %d %c\n", 'A', 'A', 65)` escreve `A 65 A`. Um `char` é só um número; o `%c` o mostra como letra e o `%d` como número. O `printf("%s", ...)` **para no `'\0'`**: se não existir, ele passa do fim e imprime lixo.

---

# ⌨️ 4. Lendo strings do teclado

Para ler texto, primeiro é preciso **um array vazio com espaço suficiente** (o *buffer*). A leitura **não cria** espaço sozinha.

## `fgets`: a forma recomendada

```c
char nome[16];
fgets(nome, sizeof nome, stdin);           // lê até 15 caracteres ou até o Enter
nome[strcspn(nome, "\n")] = '\0';          // remove o '\n' que o fgets guarda
```

- `fgets(buffer, tamanho, stdin)` lê **uma linha** e **nunca escreve além** do `tamanho` dado (deixa sempre 1 byte para o `'\0'`).
- Se a linha **coube**, o `fgets` guarda também o **Enter** (`'\n'`) no final. A linha com `strcspn(nome, "\n")` (devolve a posição do primeiro `'\n'`) troca esse Enter por `'\0'`.
- Se a linha **não coube**, o `fgets` **corta** e o restante continua esperando na entrada.

Resultados reais (buffer de 16 bytes):

| Entrada digitada | Conteúdo de `nome` | Observação |
|---|---|---|
| `Ana Maria` | `"Ana Maria"` (9 letras) | coube; o `'\n'` foi removido |
| `uma linha bem maior que o buffer` | `"uma linha bem m"` (15 letras) | **cortada** em 15; **sem** `'\n'` no fim, o que denuncia o corte |

## `scanf("%s")`: cuidado

```c
char a[20], b[20];
scanf("%19s %19s", a, b);        // entrada "Ana Maria Souza"  →  a = "Ana", b = "Maria"
```

| Problema | Detalhe |
|---|---|
| **Para no espaço** | `%s` lê **uma palavra** por vez: `"Ana Maria Souza"` virou só `"Ana"` e `"Maria"` |
| **Sem limite, estoura** | `scanf("%s", b)` com `char b[4]` e entrada `abcdefghij` → o ASan acusa `stack-buffer-overflow ... WRITE of size 11` |
| **Solução** | sempre informe o **limite** no formato: `%19s` para um array de 20 (sobra 1 para o `'\0'`) |

Para ler uma **linha com espaços** com `scanf`, existe `scanf("%39[^\n]", b)` (lê tudo até o Enter, no máximo 39 caracteres): a entrada `Ana Maria Souza` foi lida inteira. Mesmo assim, **prefira `fgets`**.

> ⛔ **`gets` não existe mais.** Ele lia sem nenhum limite e foi **removido do C11**: o `gcc` avisa `implicit declaration of function 'gets'` e o *linker* ainda diz "*the `gets' function is dangerous and should not be used*". Nunca use.

---

# 🚶 5. Percorrendo uma string

Como o fim é marcado pelo `'\0'`, o laço é: **ande até encontrá-lo**. A condição `s[i] != '\0'` equivale a `s[i]` (o zero é "falso", [[Controle de Fluxo em C]]).

```c
char s[] = "banana";
size_t n = 0;
for (size_t i = 0; s[i] != '\0'; i++)       // para no terminador
    if (s[i] == 'a') n++;
// n = 3
```

Inverter uma string (copiando para outro array do mesmo tamanho):

```c
char t[] = "abc";
char r[sizeof t];                    // 4 bytes: "abc" + '\0'
size_t L = strlen(t);                // 3
for (size_t i = 0; i < L; i++) r[i] = t[L - 1 - i];
r[L] = '\0';                         // NÃO esqueça de fechar!
// r = "cba"
```

Com ponteiro, a mesma coisa fica `for (const char *c = s; *c; c++)`: o ponteiro anda de caractere em caractere ([[Aritmética de Ponteiros em C]]). O passo `r[L] = '\0'` é o que mais se esquece ao montar strings à mão.

---

# 📚 6. Arrays de strings: várias strings juntas

Uma string é um array de `char`. Uma **lista de strings** (nomes, comandos, meses) é, portanto, um **array de arrays de `char`**: uma **matriz** ([[Matrizes em C]]). Há **duas** formas de fazer isso, com consequências diferentes.

## 6.1 Forma A: matriz de `char` (`names[][100]`)

```c
char names[3][100] = {"Ana", "Bruno", "Carla"};
```

Leia como "3 strings, cada uma com espaço para **100** bytes". Cada **linha** da matriz é **um array de `char`**, ou seja, **uma string**:

```
           coluna: 0    1    2    3    4   ...  99
 names[0]  →      'A'  'n'  'a'  \0   \0   ...  \0       ← "Ana"    (3 letras, 100 bytes reservados)
 names[1]  →      'B'  'r'  'u'  'n'  'o'   ...  \0       ← "Bruno"
 names[2]  →      'C'  'a'  'r'  'l'  'a'   ...  \0       ← "Carla"

 os textos estão DENTRO da matriz, em linhas de 100 bytes, uma depois da outra
```

Resultados reais:

| Expressão | Valor | Significado |
|---|---|---|
| `sizeof names` | **300** | 3 linhas × 100 bytes |
| `sizeof names[0]` | **100** | **uma** linha |
| `sizeof names / sizeof names[0]` | **3** | quantidade de strings |
| `names[1]` | `"Bruno"` | a linha 1 **é** uma string (`strlen` = 5) |
| `names[1][0]`, `names[1][2]` | `'B'`, `'u'` | índice duplo: **[qual string][qual letra]** |
| `names[1] - names[0]` | **100** | as linhas distam **100 bytes** (endereços reais: `...d130`, `...d194`, `...d1f8`) |

Em `names[i]` o primeiro índice escolhe a **string** e, se houver um segundo, `names[i][j]` escolhe a **letra** dentro dela. Cada linha pode ser usada com **qualquer** função de string, passando `names[i]`:

```c
strcpy(names[1], "Beatriz");          // substitui o texto da linha 1
strcat(names[1], " Lima");            // "Beatriz Lima"
names[2][0] = 'c';                    // altera uma letra: "carla"
printf("%s | %s | %s\n", names[0], names[1], names[2]);
// Ana | Beatriz Lima | carla
```

Como o espaço **pertence à matriz**, pode alterar tudo (diferente de um literal). Só não pode **atribuir** o texto com `=`: `names[0] = "Ana";` é **erro de compilação** (`assignment to expression with array type`). O mesmo vale para qualquer array: `char s[10]; s = "oi";` não compila. Use `strcpy` ou `snprintf`.

### Vantagens e custos

| ✅ Vantagem | ⚠️ Custo |
|---|---|
| Os textos **estão dentro** da matriz: dá para **alterá-los** e **copiar** com `strcpy` | **Desperdício**: os 3 nomes ocupam **16** bytes (com os `'\0'`), mas a matriz reserva **300** |
| **Bloco contíguo**, sem `malloc` nem `free` | **Tamanho máximo fixo**: texto maior que 99 letras **não cabe** |
| Ideal para **ler do teclado** (cada linha é um *buffer* pronto) | Estourar uma linha **invade a seguinte** (veja o aviso abaixo) |

> ⚠️ **Estourar uma linha corrompe a próxima, em silêncio.** Com `char names[2][8]`, `strcpy(names[0], "Maximiliano")` (12 bytes com o `'\0'`) escreve os 8 bytes da linha 0 e **continua na linha 1**: depois disso `printf("%s", names[1])` imprime `ano`. O `-fsanitize=address` e o `-fsanitize=undefined` **não reclamam**, porque tudo continua **dentro da matriz inteira**. Proteja-se com `snprintf(names[0], sizeof names[0], ...)` ou `fgets` com o tamanho da linha.

### Lendo vários nomes do teclado

```c
#define MAX 3        // quantos nomes
#define TAM 20       // bytes por nome (incluindo o '\0')

char names[MAX][TAM];
size_t qtd = 0;
while (qtd < MAX) {
    printf("Nome %zu: ", qtd + 1);
    if (fgets(names[qtd], sizeof names[qtd], stdin) == NULL) break;   // fim da entrada
    names[qtd][strcspn(names[qtd], "\n")] = '\0';                      // tira o Enter
    qtd++;
}
for (size_t i = 0; i < qtd; i++)
    printf("%zu) %s (%zu letras)\n", i + 1, names[i], strlen(names[i]));
```

Entrada `Ana`, `Bruno Souza`, `Carla`; saída real:

```
1) Ana (3 letras)
2) Bruno Souza (11 letras)
3) Carla (5 letras)
```

Repare em `sizeof names[qtd]`: é o tamanho de **uma linha** (20), exatamente o que o `fgets` precisa. O contador `qtd` guarda **quantas linhas foram realmente preenchidas** (o resto da matriz está sem uso).

## 6.2 Forma B: array de ponteiros (`char *nomes[]`)

```c
const char *nomes[] = {"Ana", "Bruno", "Carla"};
```

Aqui **não há espaço de texto dentro do array**: ele guarda só **3 ponteiros** (3 endereços), cada um apontando para um literal que está em outro lugar do programa:

```
 nomes (array de 3 ponteiros, 24 bytes)         literais em .rodata (somente leitura)
 ┌───────────┐
 │ nomes[0] ─┼───────────────────────────────►  "Ana"   \0
 ├───────────┤
 │ nomes[1] ─┼───────────────────────────────►  "Bruno" \0
 ├───────────┤
 │ nomes[2] ─┼───────────────────────────────►  "Carla" \0
 └───────────┘
```

| Expressão | Valor | Significado |
|---|---|---|
| `sizeof nomes` | **24** | 3 ponteiros × 8 bytes |
| `sizeof nomes[0]` | **8** | **um ponteiro** (e **não** o tamanho do texto!) |
| `sizeof nomes / sizeof nomes[0]` | **3** | quantidade de strings |
| endereços de `nomes[0..2]` | `...6020`, `...6024`, `...602a` | os textos estão **colados** em `.rodata`, **sem** espaço fixo por nome |

Aqui os textos **ocupam só o que precisam** (nada de 100 bytes por nome), mas são **somente leitura**: `nomes[1][0] = 'b';` é **UB** (*segfault*). O que **dá** para trocar é **qual texto cada posição aponta**:

```c
nomes[1] = "Bia";            // OK: o ponteiro agora aponta para outro literal
```

É por isso que se escreve `const char *`: o `const` impede, em tempo de compilação, alterar as letras por engano. É a forma ideal para **tabelas fixas**: nomes de meses, comandos, mensagens de erro. E é exatamente o tipo de `char **argv` do `main` ([[Como C Funciona]]).

## 6.3 Comparando as duas formas

| | **A:** `char names[3][100]` | **B:** `const char *nomes[3]` |
|---|---|---|
| O que o array guarda | os **bytes dos textos** | **endereços** dos textos |
| `sizeof` (3 nomes) | **300** | **24** |
| Altera as letras? | ✅ sim | ❌ não (literais) |
| `x[i] = "texto"` | ❌ erro de compilação | ✅ troca o ponteiro |
| Copiar texto para a posição | `strcpy(names[i], "...")` | não se copia: troca-se o ponteiro |
| Cada string tem limite? | ✅ **99 letras** | não (cada literal tem o seu tamanho) |
| Ler do teclado direto | ✅ `fgets(names[i], sizeof names[i], stdin)` | ❌ não há espaço para guardar o que for lido |
| Quando usar | dados que **mudam** ou vêm do **usuário** | tabelas **fixas** de texto |

> 💡 **Teste mental:** "preciso **escrever** nesses textos?" Se sim → **matriz de `char`** (A). Se são só consulta → **array de ponteiros para `const char`** (B).

## 6.4 Forma C: array de ponteiros com texto alocado (`char **`)

Quando a lista **cresce**, ou cada texto precisa ser **seu e alterável**, dá para combinar as duas ideias: um array de ponteiros em que cada ponteiro aponta para um texto **copiado na *heap***. A função `strdup` aloca o tamanho exato e copia:

```c
const char *origem[] = {"Ana", "Bruno Souza", "Carla"};
size_t n = sizeof origem / sizeof origem[0];

char **copias = malloc(n * sizeof *copias);          // n ponteiros
for (size_t i = 0; i < n; i++)
    copias[i] = strdup(origem[i]);                   // cópia do texto, tamanho exato
copias[1][0] = 'b';                                  // agora PODE alterar: "bruno Souza"

for (size_t i = 0; i < n; i++) free(copias[i]);      // libera cada texto...
free(copias);                                        // ...e depois o array
```

Testado com ASan e UBSan, sem erros nem vazamentos. Regra de ouro: **um `free` para cada `malloc`/`strdup`**, e liberar os textos **antes** do array que guarda os ponteiros ([[Alocação Dinâmica em C]], [[Erros Clássicos de Memória em C]]). O `strdup` é do POSIX (e do C23); com `-std=c17` estrito é preciso `#define _POSIX_C_SOURCE 200809L` antes dos `#include`.

## 6.5 Passando listas de strings para funções

O array decai para ponteiro ([[Arrays e Ponteiros em C]]), então a função **precisa receber a quantidade**:

```c
// Forma A: a última dimensão (100) é obrigatória no parâmetro (veja Matrizes em C)
void lista_a(const char n[][100], size_t qtd) {
    for (size_t i = 0; i < qtd; i++) printf("%zu: %s\n", i, n[i]);
}

// Forma B: um ponteiro para ponteiros de texto constante
void lista_b(const char *const n[], size_t qtd) {
    for (size_t i = 0; i < qtd; i++) printf("%zu: %s\n", i, n[i]);
}
```

Nos dois casos, **dentro** da função o corpo é idêntico: `n[i]` é uma string. E o `n[i]` pode ser passado a qualquer função de texto, como `maiuscula(names[1])`. Chamadas testadas: `lista_a(names, 2)` e `lista_b(nomes, 2)` imprimiram `0: Ana / 1: Bruno` e `0: Carla / 1: Dani`.

## 6.6 Ordenando uma lista de strings

`qsort` ([[Funções e Ponteiros em C]]) precisa de uma função de comparação que receba **ponteiros para os elementos**. Os elementos de cada forma são diferentes, então a função também é:

```c
// Forma A: cada elemento é um char[100]; o ponteiro para ele JÁ é um char* (o texto)
int cmp_a(const void *a, const void *b) { return strcmp(a, b); }
qsort(names, 4, sizeof names[0], cmp_a);

// Forma B: cada elemento é um char*; o ponteiro para ele é um char** → um '*' a mais
int cmp_b(const void *a, const void *b) {
    return strcmp(*(const char *const *)a, *(const char *const *)b);
}
qsort(nomes, 4, sizeof nomes[0], cmp_b);
```

Com `{"Carla", "Ana", "Daniel", "Bruno"}`, as duas versões deram `Ana Bruno Carla Daniel`. Note como a **forma A** é mais simples: o texto está **dentro** do elemento. Na **forma B** é preciso "abrir" o ponteiro (`*`) antes do `strcmp`. Na forma A, o `qsort` **troca de lugar os 100 bytes** de cada linha; na B, só os **8 bytes** dos ponteiros.

---

# 📏 7. `strlen` × `sizeof`

Dois números diferentes, que se confundem:

| | Mede | Para `char g[10] = "oi";` |
|---|---|---|
| **`sizeof g`** | o **tamanho do array** (bytes reservados), em tempo de compilação | **10** |
| **`strlen(g)`** | o **texto**: caracteres **até o `'\0'`**, sem contá-lo, em tempo de execução | **2** |

O `strlen` (de `<string.h>`) **percorre** a string byte a byte, então custa **O(n)**: chamá-lo repetidamente na condição de um laço é um desperdício. Calcule **uma vez** e guarde em uma variável (`size_t L = strlen(t);`, como no exemplo de inverter a string). Escrever o seu próprio é um bom exercício:

```c
size_t tam(const char *s) {
    size_t n = 0;
    while (s[n] != '\0') n++;          // para no terminador
    return n;
}
// tam("linguagem") → 9
```

O parâmetro é `const char *` (a função só lê), como visto em [[Ponteiros em C]]. Cuidado também: **dentro de uma função**, `sizeof` de um parâmetro `char s[]` vale **8** (é um ponteiro). Para uma string, o certo é usar `strlen`.

---

# 🧰 8. A biblioteca `<string.h>`

| Função | Faz | Cuidado |
|---|---|---|
| `strlen(s)` | tamanho do texto | O(n) |
| `strcmp(a, b)` | **compara** conteúdo: `0` se iguais, **negativo/positivo** conforme a ordem | use só o **sinal** (`< 0`, `== 0`, `> 0`) |
| `strcpy(dst, src)` | **copia** `src` (com `'\0'`) para `dst` | **não verifica** se cabe |
| `strcat(dst, src)` | **acrescenta** `src` ao fim de `dst` | **não verifica** se cabe |
| `strncpy`, `strncat` | versões "com limite" | `strncpy` tem **armadilha** (abaixo) |
| `strchr(s, c)` / `strrchr` | 1ª / última ocorrência do caractere (ou `NULL`) | devolve um **endereço dentro de `s`**, não uma cópia |
| `strstr(s, sub)` | 1ª ocorrência da substring (ou `NULL`) | idem |
| `strtok(s, sep)` | **divide** a string em pedaços | **altera** a string original (veja abaixo) |
| `strdup(s)` | **cria uma cópia** na *heap* | você precisa dar `free` |
| `memcpy`, `memmove`, `memset`, `memcmp` | operam em **bytes**, **sem** olhar o terminador | `memcpy` com áreas que **se sobrepõem** é UB; use `memmove` |

Resultados reais: `strcmp("banana", "banana")` → igual; `("banana", "bananas")` → **negativo** (mais curta vem antes); `("banana", "abacate")` → **positivo**. `strchr("um dois tres", 'd')` devolveu `"dois tres"` e `strchr(..., 'z')` devolveu `NULL`. `memmove(b + 2, b, 4)` em `"abcdef"` deu `"ababcd"` (correto mesmo com sobreposição).

## Comparar: `==` não serve

```c
char a[] = "ana", b[] = "ana";
a == b            // 0 (falso!)  compara ENDEREÇOS dos dois arrays
strcmp(a, b) == 0 // 1 (verdadeiro)  compara o TEXTO
```

Dois arrays distintos têm endereços distintos, mesmo com texto igual. O `gcc` avisa `-Warray-compare` ([[Arrays em C]]). Com ponteiros para literais iguais (`const char *p = "ana", *q = "ana";`) o teste deu `p == q` verdadeiro, mas só porque o compilador **juntou** os dois literais iguais; **isso não é garantido**. Para comparar **conteúdo**, sempre `strcmp`.

## Dividir um texto: `strtok`

```c
char linha[] = "ana,bruno,carla";
for (char *t = strtok(linha, ","); t; t = strtok(NULL, ","))
    printf("[%s] ", t);                   // [ana] [bruno] [carla]
```

O `strtok` troca cada separador por `'\0'` **dentro da própria string**: depois do laço, `linha` ficou `"ana"` (o primeiro `','` virou `'\0'`). Por isso **não funciona com literais** (`const char *`) e **destrói** o original; guarde uma cópia se precisar dele. Ele também mantém estado interno (não serve em programas com várias *threads*; existe `strtok_r`).

## Concatenar e copiar sem estourar

`strcpy`/`strcat` copiam **até o `'\0'` da origem**, sem saber o tamanho do destino. Com origem maior que o destino, **escrevem fora do array** (*buffer overflow*, [[Arrays em C]]). Teste real: `char buf[4]; strcpy(buf, "abcdefgh");` → o ASan acusa `stack-buffer-overflow ... WRITE of size 9`.

A resposta moderna é **`snprintf`** (de `<stdio.h>`): **nunca** escreve além do tamanho dado e **sempre termina** a string.

```c
char buf[8];
int n = snprintf(buf, sizeof buf, "%s-%d", "abcdef", 12345);
// buf = "abcdef-"  (7 caracteres + '\0')     n = 12  (tamanho que SERIA necessário)
```

Se `n >= sizeof buf`, houve **truncamento**: o retorno permite detectá-lo. Para montar textos, **prefira `snprintf`**; para copiar, `snprintf(dst, sizeof dst, "%s", src)` também funciona. O `gcc -Wall` até avisa em tempo de compilação quando o texto é maior que o destino (`-Wformat-truncation`: *directive output truncated writing 14 bytes into a region of size 8*).

> ⚠️ **A armadilha do `strncpy`:** apesar do nome, ele **não é um `strcpy` seguro**. Se a origem **não cabe**, copia `n` bytes e **não escreve o `'\0'`**: com `char t[4]` e `strncpy(t, "abcdef", 4)`, o conteúdo ficou `a b c d` **sem terminador**. Se a origem é **curta**, ele **preenche o resto com zeros** (custo desnecessário em buffers grandes). O `gcc` avisa: `-Wstringop-truncation`. Se usar, termine à mão (`t[sizeof t - 1] = '\0';`).

---

# 🔁 9. Strings e funções

Como o array decai para ponteiro, uma função de texto recebe **`char *`** (ou `const char *`) e, como a string sabe onde termina, **não precisa do tamanho**:

```c
void maiuscula(char *s) {                 // altera o texto: char *
    for (; *s; s++) *s = (char)toupper((unsigned char)*s);
}
size_t conta_vogais(const char *s) {      // só lê: const char *
    size_t n = 0;
    for (; *s; s++) if (strchr("aeiouAEIOU", *s)) n++;
    return n;
}
// maiuscula(names[1]) → "BRUNO"       conta_vogais("Bruno Souza") → 5
```

Regras: use **`const char *`** quando a função só **lê** (o compilador então recusa qualquer escrita por engano); a função **altera o original**, sem cópia; e **nunca devolva um array local**:

```c
char *ruim(void) { char b[10] = "oi"; return b; }   // b some no return!
```

O `gcc` avisa: `-Wreturn-local-addr` (*function returns address of local variable*). Para devolver texto, ou **receba o buffer do chamador** (`void f(char *saida, size_t n)`), ou devolva memória da ***heap*** (`strdup`/`malloc`), com a regra de "quem libera" de [[Alocação Dinâmica em C]] ([[Funções e Ponteiros em C]]).

---

# 🔠 10. Caracteres: `<ctype.h>`

Como `char` é um número, as funções de `<ctype.h>` classificam e convertem um caractere por vez: `isalpha`, `isdigit`, `isspace`, `isupper`, `islower`, `toupper`, `tolower`. Elas **esperam um valor representável em `unsigned char`** (ou `EOF`), então **converta**, como explicado em [[Conversões de Tipo em C]]:

```c
char up[] = "Ola, C99!";
for (size_t i = 0; up[i]; i++)                    // para no '\0'
    up[i] = (char)toupper((unsigned char)up[i]);
// "OLA, C99!"
```

Note a condição `up[i]`: o **terminador vale 0 (falso)**, então o laço para sozinho.

---

# 🌍 11. Bytes × caracteres: o problema do UTF-8

Em C, `char` é **um byte**, e o `strlen` conta **bytes**, não caracteres visíveis. Textos em **UTF-8** (o padrão do Linux, da web e do Go) usam **de 1 a 4 bytes** por caractere:

| Texto | `strlen` (bytes) | Caracteres visíveis |
|---|---|---|
| `"ola"` | 3 | 3 |
| `"olá"` | **4** | 3 (o `á` ocupa **2** bytes) |
| `"日本"` | **6** | 2 (3 bytes cada) |

Consequências: `strlen` **não** é a "quantidade de letras"; `toupper` **não** converte `á`; indexar `s[i]` pode cair **no meio** de um caractere. Funciona bem para ASCII; para Unicode de verdade é preciso uma biblioteca (ou `wchar_t`/`<uchar.h>`). **Dimensione os arrays em bytes**: um nome de 20 letras acentuadas pode precisar de 40 bytes ou mais. Em Go a `string` é tratada como UTF-8 de forma nativa ([[Strings em Profundidade]]).

---

# 🔢 12. Texto ↔ número

| Direção | Ferramenta | Observação |
|---|---|---|
| texto → `int` | `atoi(s)` | **sem detecção de erro**: `atoi("abc")` e `atoi("0")` dão ambos **0** |
| texto → `long` | **`strtol(s, &fim, base)`** | informa **onde parou** (`fim`) e sinaliza estouro com `errno` |
| número → texto | `snprintf(buf, n, "%d", v)` | veja a seção 8 |

Teste real do `strtol`: `strtol("123abc", &fim, 10)` devolveu `123` e `fim` ficou apontando para `"abc"` (o que **sobrou**); com `"99999999999999999999"` devolveu `LONG_MAX` e `errno == ERANGE`. Por isso, **prefira `strtol`** (família `strtoul`, `strtod`) para entradas de usuário, verificando `fim` e `errno`. O assunto completo fica em *stdlib e Utilitários*.

---

# 🧯 13. Erros mais comuns (e como evitar)

| Erro | Exemplo | Efeito | Correção |
|---|---|---|---|
| Esquecer o byte do `'\0'` | `char s[3] = "oi!";` (precisa de 4) | texto sem terminador | `char s[] = "oi!";` (o compilador conta) |
| Usar `==` para comparar texto | `if (a == "sim")` | compara **endereços** | `strcmp(a, "sim") == 0` |
| Atribuir texto a array | `s = "oi";` / `names[0] = "Ana";` | **erro de compilação** | `strcpy` / `snprintf` |
| Alterar um literal | `char *p = "oi"; p[0] = 'O';` | `Segmentation fault` | `char p[] = "oi";` |
| `strcpy`/`strcat` sem checar | origem maior que o destino | *buffer overflow* | `snprintf(dst, sizeof dst, ...)` |
| `scanf("%s")` sem limite | entrada maior que o array | *buffer overflow* | `%19s` ou `fgets` |
| `gets` | qualquer uso | removido do C11 | `fgets` |
| Esquecer o `'\n'` do `fgets` | comparar `"sim"` com `"sim\n"` | `strcmp` ≠ 0 | `names[i][strcspn(names[i], "\n")] = '\0';` |
| `sizeof` em vez de `strlen` | `sizeof p` com `char *p` | 8, não o tamanho do texto | `strlen(p)` |
| Devolver array local | `return b;` | ponteiro pendurado | receber o buffer ou usar `strdup` |
| Esquecer o `'\0'` ao montar à mão | `r[L]` sem fechar | lixo no fim | `r[L] = '\0';` |
| Estourar uma linha da matriz | `strcpy(names[0], longo)` | corrompe `names[1]`, **sem aviso** das ferramentas | `snprintf(names[0], sizeof names[0], ...)` |

Para detectar o que escapar: compile com `-Wall -Wextra` ([[Convenções e Estilo]]) e teste com `-fsanitize=address,undefined` ([[Erros Clássicos de Memória em C]]).

---

# 🧪 14. Para praticar

Tente resolver sem olhar; depois compile com `-Wall -Wextra -fsanitize=address,undefined` para conferir.

1. Escreva `void inverte(char *s)` que inverte a string **no próprio array** (troque `s[i]` com `s[L-1-i]` até a metade).
2. Leia **5 nomes** com `fgets` em um `char names[5][30]` e mostre o **maior** (compare com `strlen`).
3. Ordene os nomes do exercício 2 com `qsort` e `strcmp` (use `cmp_a` da seção 6.6).
4. Dada `const char *meses[] = {"jan", "fev", ...}`, escreva uma função `const char *nome_mes(int n)` que devolve `"invalido"` se `n` estiver fora de 1 a 12 (lembre que o índice é `n - 1`).
5. Conte quantas palavras há em uma frase (conte as transições de espaço para letra, com `isspace`).

---

# ✅ Resumo do Conceito

- **String em C** = array de `char` terminado por **`'\0'`** (valor 0); **não guarda o tamanho**; reserve **sempre 1 byte a mais**
- **`'a'`** é um caractere (`int`, 4 bytes); **`"a"`** é uma string (2 bytes); `""` tem 1
- **Declarar:** `char s[] = "oi"` (3 bytes, conta sozinho), `char s[10] = "oi"` (resto zerado), `char s[10] = {0}` (vazia); `{'o','i'}` **não é string**; **`const char *p = "oi"`** aponta para texto fixo (alterar é UB)
- **`sizeof`** = tamanho do array (10 em `char g[10] = "oi"`); **`strlen`** = texto até o `'\0'` (2), custa **O(n)**; escrever `'\0'` no meio **encurta** a string
- **Mostrar:** `%s`, `%c`, `%d` (código), `%10s`, `%-10s`, `%.3s`, `puts`; **ler:** **`fgets`** + `strcspn` para tirar o `'\n'`; `scanf("%s")` para no espaço e **precisa de limite** (`%19s`); `gets` não existe
- **Lista de strings:**
  - **`char names[N][TAM]`** (matriz de `char`): textos **dentro** do array, **alteráveis**, tamanho fixo por linha, `names[i]` é a string `i`, copia-se com `strcpy`; estourar uma linha **corrompe a seguinte** sem aviso
  - **`const char *nomes[]`** (array de ponteiros): só **endereços** de literais, **somente leitura**, troca-se o ponteiro; ideal para tabelas fixas (`argv`)
  - **`char **` com `strdup`**: cada texto na *heap*, alterável e de tamanho exato; um `free` por texto e outro pelo array
- **`qsort`:** na forma A, `strcmp(a, b)` direto; na forma B, `strcmp(*(char **)a, *(char **)b)`
- **`==` compara endereços**: para conteúdo use **`strcmp`** (confie só no **sinal**); **`strcpy`/`strcat` não verificam limites**; **`strncpy` pode não terminar**; a ferramenta segura é **`snprintf`**
- **`strtok`** divide **alterando** a string; **`strdup`** copia para a *heap*; **`memmove`** aceita sobreposição, `memcpy` não
- **Funções:** `char *` altera, **`const char *`** só lê; **nunca devolva** um array local
- **`<ctype.h>`** exige `(unsigned char)`; o laço `while (s[i])` para sozinho no terminador
- **UTF-8:** `strlen` conta **bytes** (`"olá"` = 4); `toupper` só serve a ASCII
- **`atoi`** não detecta erro; use **`strtol`** com `fim` e `errno`

---

# 🔗 Conexões com outras anotações

- [[Arrays em C]] — strings são arrays: memória contígua, limites e `memcpy`
- [[Matrizes em C]] — `names[][100]` é uma matriz: `sizeof`, linhas contíguas e passagem para funções
- [[Arrays e Ponteiros em C]] — `char s[]` × `char *s`, o *segfault* ao alterar literais e `char nm[][6]` × `char *nomes[]`
- [[Ponteiros em C]] — `const char *` em parâmetros e retorno de `strchr`/`strstr`
- [[Aritmética de Ponteiros em C]] — percorrer strings com ponteiros (`while (*e) e++`)
- [[Funções e Ponteiros em C]] — `qsort`, *callbacks* e retorno de ponteiros
- [[Alocação Dinâmica em C]] — `malloc`, `strdup` e `free` para listas de strings na *heap*
- [[Erros Clássicos de Memória em C]] — *buffer overflow*, ASan e a leitura sem `'\0'`
- [[Variáveis e Constantes em C]] — literais e a seção `.rodata`
- [[Tipos de Dados em C]] — `char`, `size_t` e `sizeof`
- [[Conversões de Tipo em C]] — `(unsigned char)` com `ctype` e o `char` negativo
- [[Controle de Fluxo em C]] — laços sobre strings e `strlen` na condição
- [[Convenções e Estilo]] — `-Wall -Wextra` avisa `-Wstringop-truncation`, `-Wformat-truncation` e `-Warray-compare`
- [[Strings em Profundidade]] — o equivalente em Go (UTF-8 nativo, imutável, com tamanho)
