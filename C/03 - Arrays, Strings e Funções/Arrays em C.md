---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: arrays, inicializadores, VLA); experimentos com gcc 13.3, AddressSanitizer e UBSan em Linux x86-64"
---
# Arrays em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3, `-fsanitize=address` e `-fsanitize=bounds` | Anterior: [[Conversões de Tipo em C]] | Próxima: [[Matrizes em C]] | Compare com [[Arrays e Slices]] (Go) e veja a visão geral em [[Arrays]] (Estruturas de Dados)

> 🧭 **Como ler esta nota:** das seções 1 a 6 você aprende a **criar, preencher e percorrer** um array; da 7 a 10, **como ele fica na memória** e os seus perigos; da 11 a 14, **arrays de texto** (listas de palavras como `char names[][100]`), matrizes, funções e erros comuns. Strings e matrizes têm notas próprias ([[Strings em C]], [[Matrizes em C]]); aqui está a base de tudo.

---

# 🧱 1. O que é um array

> 💡 **Array (vetor):** uma sequência de **N elementos do mesmo tipo**, guardados **lado a lado** na memória, acessados por um **número** chamado **índice**.

Pense num **prédio com apartamentos numerados a partir do zero**: todos têm o **mesmo tamanho**, ficam um depois do outro, e você chega em qualquer um sabendo só o **número**. O prédio não cresce depois de construído, e ninguém impede você de bater na porta do apartamento 99 (que não existe).

```c
int notas[5];            // 5 ints consecutivos: notas[0], notas[1], ... notas[4]
```

Três ideias definem tudo o que vem a seguir:

1. **Tamanho fixo:** o `N` é parte do tipo (`int[5]`) e **não muda** depois de declarado.
2. **Contíguo:** os elementos ocupam **bytes seguidos**, sem lacunas.
3. **Sem rede de proteção:** C **não verifica** se o índice é válido, nem guarda o tamanho junto do array (ao contrário de Go, [[Arrays e Slices]]).

> 💡 **Endereço:** o número que identifica a posição de um byte na memória. Como o array é contíguo, basta conhecer o endereço do **primeiro** elemento e o tamanho do tipo para achar qualquer outro. O operador `&x` dá o endereço de `x`, e o formato `%p` do `printf` o imprime. Guardar e manipular endereços é o assunto de [[Ponteiros em C]] (módulo 04).

---

# ✏️ 2. Declarar, escrever e ler elementos

A declaração é **`tipo nome[quantidade];`**. Para usar um elemento, escreve-se `nome[índice]`, **começando do 0**:

```c
int notas[5];             // reserva espaço para 5 ints (conteúdo ainda indefinido!)

notas[0] = 7;             // ESCREVE no primeiro elemento
notas[1] = 9;
notas[4] = notas[0] + 3;  // pode usar elementos em contas: notas[4] vira 10
int x = notas[1];         // LÊ o segundo elemento (x = 9)
notas[1] += 1;            // funciona como qualquer variável: notas[1] vira 10
```

```
  índice:    0     1     2     3     4         ← o índice é a "posição", e começa em 0
           ┌─────┬─────┬─────┬─────┬─────┐
  valor:   │  7  │  9  │  ?  │  ?  │ 10  │     ← "?" = ainda sem valor (lixo)
           └─────┴─────┴─────┴─────┴─────┘
  O ÚLTIMO índice válido é 4 (tamanho − 1). Não existe notas[5].
```

Cada elemento é uma **variável comum** do tipo do array. O que muda é **como você a nomeia** (pela posição). O índice pode ser **qualquer expressão inteira**: `notas[i]`, `notas[i + 1]`, `notas[n - 1]`. É isso que torna os arrays tão úteis com laços (seção 4).

## Qualquer tipo serve

Um array pode ter elementos de **qualquer tipo** ([[Tipos de Dados em C]]), e o tamanho total é `quantidade × sizeof(tipo)`:

| Declaração | `sizeof` | Conta |
|---|---|---|
| `char c[10]` | **10** | 10 × 1 |
| `int i[10]` | **40** | 10 × 4 |
| `float f[10]` | **40** | 10 × 4 |
| `double d[10]` | **80** | 10 × 8 |
| `long long ll[10]` | **80** | 10 × 8 |

---

# 📝 3. Inicialização

Dá para declarar **e** preencher de uma vez, com chaves:

| Forma | Resultado |
|---|---|
| `int a[5] = {1, 2, 3};` | `1 2 3 0 0` — o que **faltar é zerado** |
| `int b[] = {10, 20, 30, 40};` | o compilador **conta**: `sizeof b` = 16 bytes (4 × 4) |
| `int c[5] = {0};` | **todos zero** (idioma para zerar) |
| `int d[6] = {[1] = 7, [4] = 9};` | **designada** (C99): `0 7 0 0 9 0` |
| `double t[] = {21.5, 23.0, 19.8};` | 3 `double`, `sizeof t` = **24** |
| `char s[] = "oi";` | **3** bytes: `'o'`, `'i'` e o terminador `'\0'` (uma **string**, [[Strings em C]]) |
| `char u[] = {'o', 'i'};` | **2** bytes, **sem** terminador (não é string) |

Pontos de atenção:

- **Sem inicializador**, o conteúdo depende de **onde** o array vive, igual às variáveis ([[Variáveis e Constantes em C]]): **local = lixo** (não zera); **global/`static` = zero** (confirmado: um `int g[4]` global imprimiu `0 0 0 0`). **Sempre inicialize** arrays locais.
- Um inicializador com **mais** valores do que o tamanho (`int v[3] = {1, 2, 3, 4};`) é um erro do programa: o `gcc` **só avisa** (`excess elements in array initializer`) e **ignora** o valor extra. Não ignore o aviso.
- Em `const int tabela[] = {...}` os dados vão para a seção **`.rodata`** (somente leitura); sem `const` e inicializado, para **`.data`**; sem inicializador e global, para **`.bss`**. Conferido com `nm`: `R tabela`, `D dados`, `B zeros` ([[Variáveis e Constantes em C]]).

---

# 🔁 4. Percorrendo um array com laços

O `for` ([[Controle de Fluxo em C]]) e o array são feitos um para o outro: o contador do laço é o **índice**. Exemplo completo, com as contas mais comuns (soma, média, maior valor e busca):

```c
#include <stdio.h>

int main(void) {
    int notas[5] = {7, 9, 6, 10, 8};
    size_t n = sizeof notas / sizeof notas[0];      // 5: quantos elementos (seção 5)

    int soma = 0, maior = notas[0];
    size_t pos_maior = 0;
    for (size_t i = 0; i < n; i++) {                // i vale 0, 1, 2, 3, 4
        soma += notas[i];                           // acumula
        if (notas[i] > maior) {                     // guarda o maior e a posição dele
            maior = notas[i];
            pos_maior = i;
        }
    }
    printf("soma=%d media=%.1f maior=%d (indice %zu)\n",
           soma, (double)soma / (double)n, maior, pos_maior);

    int alvo = 6, achou = -1;                       // busca linear: -1 = não achou
    for (size_t i = 0; i < n; i++)
        if (notas[i] == alvo) { achou = (int)i; break; }
    printf("buscar %d -> indice %d\n", alvo, achou);
    return 0;
}
```

Saída real: `soma=40 media=8.0 maior=10 (indice 3)` e `buscar 6 -> indice 2`. Pontos importantes:

- **`i < n`, nunca `i <= n`**: com `n = 5`, o último índice válido é **4**. Usar `<=` lê `notas[5]`, fora do array (seção 8).
- **`(double)soma / (double)n`**: sem a conversão, `40 / 5` seria divisão **inteira** (em outros casos, perderia a parte decimal; veja [[Conversões de Tipo em C]]).
- **Média de `double`:** `double temps[] = {21.5, 23.0, 19.8};` → soma 64,3, média **21,43**.
- **Alterar todos os elementos** também usa o laço: `for (size_t i = 0; i < n; i++) notas[i] += 1;`.

## ⚠️ Percorrer de trás para frente: a armadilha do `size_t`

`size_t` ([[Tipos de Dados em C]]) é **sem sinal**: ele **nunca** é negativo. Portanto:

```c
for (size_t i = n - 1; i >= 0; i--)        // ERRADO: i >= 0 é SEMPRE verdadeiro
    printf("%d ", v[i]);
```

Quando `i` chega a 0 e faz `i--`, ele **dá a volta** para o maior valor possível, e o laço **nunca termina**: no teste, depois de `30 20 10` o programa passou a imprimir **lixo** (`-548394256 0 3 ...`), lendo **antes** do array. O `gcc` avisa: `comparison of unsigned expression in '>= 0' is always true` (`-Wtype-limits`). Duas formas corretas (ambas imprimiram `30 20 10`):

```c
for (size_t i = n; i-- > 0; )              // testa i > 0 ANTES de decrementar
    printf("%d ", v[i]);

for (int i = (int)n - 1; i >= 0; i--)      // ou usa int (com cast) e evita o problema
    printf("%d ", v[i]);
```

---

# 📏 5. Descobrindo o tamanho: o idioma `sizeof`

O array **não carrega** o número de elementos, mas `sizeof` (calculado na compilação) permite deduzi-lo **no mesmo escopo** em que ele foi declarado:

```c
int b[] = {10, 20, 30, 40};
sizeof b                    // 16  (bytes totais)
sizeof b[0]                 // 4   (bytes de um elemento)
sizeof b / sizeof b[0]      // 4   (número de elementos)
```

É comum embrulhar isso numa macro (ver *O Pré-processador*): `#define TAM(v) (sizeof(v) / sizeof((v)[0]))`. O resultado é `size_t`, então use `size_t` no índice: `for (size_t i = 0; i < TAM(b); i++)`, o que evita o aviso `-Wsign-compare` ([[Conversões de Tipo em C]]).

> ⚠️ Esse truque **só funciona onde o array foi declarado**. Dentro de uma função que **recebeu** o array, o `sizeof` devolve **8** (veja a seção 10).

---

# ⌨️ 6. Preenchendo o array com dados do usuário

Para guardar valores digitados, **declare o array com o tamanho máximo** e use um **contador** do que foi realmente lido. Nunca leia mais valores do que cabem:

```c
#define MAX 5

int v[MAX];
size_t n = 0;                                    // quantos valores foram lidos
while (n < MAX && scanf("%d", &v[n]) == 1)       // para ao encher OU ao falhar a leitura
    n++;
printf("li %zu valores:", n);
for (size_t i = 0; i < n; i++) printf(" %d", v[i]);
```

- **`n < MAX` vem primeiro**: o `&&` só tenta ler se **ainda há espaço** (curto-circuito, [[Operadores em C]]).
- **`scanf` devolve quantos valores leu** (aqui, `1` se deu certo): `== 1` detecta entrada inválida.
- **`&v[n]`** é o **endereço** do elemento `n`, onde o `scanf` vai gravar.

Testes reais: entrada `4 8 15 abc 23` → `li 3 valores: 4 8 15` (parou no `abc`); entrada `1 2 3 4 5 6 7` → `li 5 valores: 1 2 3 4 5` (parou ao **encher**, sem estourar). Depois da leitura, **use `n`, não `MAX`**: só os primeiros `n` elementos têm valor válido.

---

# 🗺️ 7. Como o array fica na memória

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

# 🚧 8. Índices fora dos limites

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

# 🚫 9. Arrays não se atribuem nem se comparam com `=` e `==`

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

# 📞 10. Passando arrays para funções

Ao passar um array para uma função, **não é feita uma cópia**: a função recebe o **endereço do primeiro elemento**. Duas consequências:

```c
void preenche(int v[], size_t n, int valor) {          // altera o array ORIGINAL
    for (size_t i = 0; i < n; i++) v[i] = valor * (int)(i + 1);
}
int soma(const int v[], size_t n) {                    // só lê: const protege
    int s = 0;
    for (size_t i = 0; i < n; i++) s += v[i];
    return s;
}

int v[4];
preenche(v, 4, 10);                                    // v = 10 20 30 40
printf("soma=%d  sizeof v=%zu\n", soma(v, 4), sizeof v);   // soma=100  sizeof v=16
```

- A função **altera o array original** (não existe cópia): por isso `preenche` funciona sem devolver nada;
- O tamanho se **perde**: dentro da função, `sizeof(v)` vale **8** (um endereço), **não** 16. O padrão em C é **passar o tamanho como outro parâmetro** (`size_t n`). O `gcc` avisa quem tenta usar `sizeof` ali (`-Wsizeof-array-argument`);
- Use **`const`** (`const int v[]`) quando a função **só lê**: o compilador recusa qualquer escrita por engano;
- Uma função **não pode devolver** um array local (ele some no `return`); devolva por parâmetro (como `preenche`) ou use alocação dinâmica ([[Alocação Dinâmica em C]]).

O mecanismo por trás disso (chamado ***decay***) é o tema de [[Arrays e Ponteiros em C]].

---

# 🔤 11. Arrays de caracteres e de textos

## Um texto: array de `char`

Um array de `char` terminado por `'\0'` é uma **string** ([[Strings em C]]): `char s[] = "oi";` ocupa 3 bytes. Tudo o que vale para arrays (índice, limites, sem `=` nem `==`) vale para ela também.

## Vários textos: lista de palavras

Para guardar **várias palavras** (nomes, comandos, meses), usa-se um **array de arrays de `char`**, ou seja, uma matriz em que **cada linha é uma string**:

```c
char names[][100] = {"Matheus", "Sarah", "Carlos", "Teste"};
```

Leia como "uma lista de strings, cada uma com espaço para **100** bytes"; a primeira dimensão fica em branco e o compilador a **conta** (4 palavras). Resultados reais:

| Expressão | Valor | Significado |
|---|---|---|
| `sizeof names` | **400** | 4 linhas × 100 bytes |
| `sizeof names[0]` | **100** | **uma** linha (uma palavra com espaço de 100 bytes) |
| `sizeof names / sizeof names[0]` | **4** | quantidade de palavras |
| `names[1]` | `"Sarah"` | a linha 1 **é** uma string completa |
| `names[1][0]` e `names[0][6]` | `'S'` e `'s'` | dois índices: **[qual palavra][qual letra]** |

Em `names[i]`, o primeiro índice escolhe a **palavra**; um segundo índice, `names[i][j]`, escolhe a **letra** dentro dela. Percorrer a lista com o mesmo laço da seção 4:

```c
size_t qtd = sizeof names / sizeof names[0];                 // 4
for (size_t i = 0; i < qtd; i++)
    printf("%zu: %s (%zu letras)\n", i, names[i], strlen(names[i]));
// 0: Matheus (7 letras)    1: Sarah (5 letras)    2: Carlos (6 letras)    3: Teste (5 letras)
```

Duas tarefas típicas, com `strlen` e `strcmp` de `<string.h>`:

```c
size_t maior = 0;                                            // posição da palavra mais longa
for (size_t i = 1; i < qtd; i++)
    if (strlen(names[i]) > strlen(names[maior])) maior = i;
// maior = 0  →  "Matheus"

int pos = -1;                                                // busca por conteúdo: strcmp, NUNCA ==
for (size_t i = 0; i < qtd; i++)
    if (strcmp(names[i], "Carlos") == 0) { pos = (int)i; break; }
// pos = 2
```

> ⚠️ Para **comparar palavras** use **`strcmp`**, não `==` (compararia endereços, seção 9). E para **trocar** o texto de uma posição, **`strcpy`/`snprintf`**, não `names[1] = "Ana"` (atribuir a array é erro de compilação).

## A outra forma: array de ponteiros

```c
const char *nomes[] = {"Matheus", "Sarah", "Carlos", "Teste"};      // sizeof = 32 (4 ponteiros × 8)
```

Aqui o array guarda só **endereços** de textos fixos (somente leitura), em vez dos **bytes** dos textos (400 bytes na matriz). Cada forma tem vantagens (alterar as letras, ler do teclado, tamanho do espaço, ordenar com `qsort`): a comparação completa, a leitura com `fgets`, a lista dinâmica com `strdup` e a ordenação estão em [[Strings em C]] (seção 6: *Arrays de strings*).

---

# 🔲 12. Arrays com mais de uma dimensão

Um array de arrays. `int m[2][3]` é **2 linhas × 3 colunas** e fica na memória **linha após linha** (*row-major*): `m[i][j]` está em `i × colunas + j`. O que é específico dessa organização (percurso que respeita o *cache*, passagem para funções, erros típicos) está em [[Matrizes em C]]; a lista de palavras `names[][100]` da seção anterior é um caso particular.

---

# 🎚️ 13. Tamanho variável: VLA e limites da pilha

- **VLA** (*variable length array*, C99; opcional desde o C11): `int n = 3; int vla[n];` — o tamanho vem de uma variável em tempo de execução (`sizeof vla` deu **12**). Não aceita inicializador e vive na **pilha**.
- **Limite da pilha:** arrays locais grandes estouram a pilha. Um `char grande[16 * 1024 * 1024]` local (16 MiB) causou **`Segmentation fault`** (código 139) com a pilha padrão de **8 MiB** (`ulimit -s` = 8192).

Para tamanho **decidido em tempo de execução** ou **grande**, a solução é a **alocação dinâmica** ([[Alocação Dinâmica em C]]; onde cada coisa vive: [[Stack vs Heap em C]]).

---

# 🧯 14. Erros mais comuns

| Erro | Exemplo | Efeito | Correção |
|---|---|---|---|
| Laço com `<=` | `for (i = 0; i <= n; i++)` | lê/escreve `v[n]`, fora do array | `i < n` |
| Achar que começa em 1 | `v[1]` como primeiro elemento | pula o primeiro; o último vira `v[n]` | o primeiro é `v[0]` |
| Não inicializar array local | `int v[5]; soma += v[0];` | **lixo** nas contas | `int v[5] = {0};` |
| Contagem regressiva com `size_t` | `for (size_t i = n-1; i >= 0; i--)` | laço infinito e leitura fora | `for (size_t i = n; i-- > 0; )` |
| `sizeof` dentro de função | `sizeof v` com `int v[]` como parâmetro | **8**, não o tamanho do array | passe `n` como parâmetro |
| Copiar com `=` | `b = a;` | erro de compilação | `memcpy(b, a, sizeof a)` |
| Comparar com `==` | `if (a == c)` | compara **endereços** | `memcmp` (ou `strcmp` para texto) |
| Ler mais valores do que cabem | `scanf` em laço sem `n < MAX` | *buffer overflow* | `while (n < MAX && scanf(...) == 1)` |
| Mais valores no inicializador | `int v[3] = {1,2,3,4};` | valor extra ignorado (aviso) | corrija o tamanho (ou omita: `v[]`) |
| Array local enorme | `int v[10000000];` | `Segmentation fault` (pilha) | `static`, global ou `malloc` |

Para detectar o que escapar: `-Wall -Wextra` ([[Convenções e Estilo]]) e `-fsanitize=address,undefined` ([[Erros Clássicos de Memória em C]]).

---

# 🧪 15. Para praticar

Tente resolver sem olhar; depois compile com `-Wall -Wextra -fsanitize=address,undefined` para conferir.

1. Declare `int v[8]` com valores à sua escolha e imprima a **soma**, o **menor** valor e quantos são **pares** (`v[i] % 2 == 0`).
2. Escreva `void inverte(int v[], size_t n)` que inverte o array **no próprio lugar** (troque `v[i]` com `v[n-1-i]` até a metade).
3. Leia até **10 inteiros** do teclado (parando ao encher ou em entrada inválida) e imprima só os que são **maiores que a média**.
4. Dado `char names[][100] = {"Matheus", "Sarah", "Carlos", "Teste"}`, escreva uma função que diz **se um nome existe** na lista (use `strcmp`) e outra que devolve a **posição** da palavra mais longa.
5. Escreva `int conta(const int v[], size_t n, int alvo)` que devolve **quantas vezes** `alvo` aparece.

---

# ✅ Resumo do Conceito

- **Array** = N elementos do **mesmo tipo**, **contíguos**, de **tamanho fixo**, indexados a partir de **0** (último índice = `n - 1`)
- **Declarar:** `tipo nome[N];` → `nome[i]` lê e escreve; cada elemento é uma variável comum; `sizeof` = `N × sizeof(tipo)` (`int[10]` = 40, `double[10]` = 80)
- **Inicialização parcial zera o resto** (`{1, 2, 3}` em `int[5]` → `1 2 3 0 0`); `{0}` zera tudo; `[i] = v` é designada (C99); local sem inicializador = **lixo**, global = **zero**; valores a mais no inicializador: o `gcc` só avisa
- **Laços:** `for (size_t i = 0; i < n; i++)`, **`<`**, nunca `<=`; contagem regressiva com `size_t`: **`for (size_t i = n; i-- > 0; )`** (o `i >= 0` nunca termina)
- **Número de elementos:** `sizeof v / sizeof v[0]` (só no escopo da declaração); com dados do usuário, **use um contador** (`n < MAX && scanf(...) == 1`)
- Elemento `i` fica em **endereço de `v[0]` + `i × sizeof(elemento)`**; `&v` e `&v[0]` são o mesmo endereço
- **Sem verificação de limites:** `v[n]` é **UB**; detecte com `-fsanitize=address,undefined` e `-Warray-bounds`
- **Não se copia com `=`** (erro) nem se compara com `==` (compara endereços): use `memcpy` e `memcmp`
- **Em funções**, o array chega como **endereço**: sem cópia (a função altera o original), tamanho perdido (`sizeof` = 8), passe `n` à parte; **`const`** quando só lê
- **Lista de palavras:** `char names[][100] = {"Matheus", "Sarah", ...}` é um array de strings (`sizeof` = 400; `names[i]` é a palavra `i`; `names[i][j]` é a letra); comparar com `strcmp`; alternativa `const char *nomes[]` (32 bytes, só endereços), detalhada em [[Strings em C]]
- **Multidimensional** = array de arrays em **ordem de linha** ([[Matrizes em C]]); **VLA** vive na pilha (**8 MiB**); para grandes ou variáveis, use alocação dinâmica

---

# 🔗 Conexões com outras anotações

- [[Tipos de Dados em C]] — `sizeof`, `size_t` e tamanho dos tipos
- [[Variáveis e Constantes em C]] — valor inicial, `.data`/`.bss`/`.rodata` e VLA
- [[Conversões de Tipo em C]] — `size_t` × `int` no índice dos laços e a divisão inteira da média
- [[Controle de Fluxo em C]] — o `for` que percorre o array e o *off-by-one*
- [[Operadores em C]] — curto-circuito do `&&` na leitura com limite
- [[Strings em C]] — arrays de `char`, listas de strings, `fgets`, `strcmp` e `strdup`
- [[Matrizes em C]] — arrays com 2 ou mais dimensões
- [[Arrays e Ponteiros em C]] — o *decay* e a relação entre `v[i]` e `*(v+i)`
- [[Como C Funciona]] — UB e regiões de memória
- [[Convenções e Estilo]] — *sanitizers* e `-Wall -Wextra`
- [[Erros Clássicos de Memória em C]] — *buffer overflow* e como detectá-lo
- [[Arrays e Slices]] — o equivalente em Go (valor copiável, com verificação de limites)
- [[Arrays]] — a visão de estruturas de dados (acesso O(1), cache) sobre o mesmo bloco contíguo
