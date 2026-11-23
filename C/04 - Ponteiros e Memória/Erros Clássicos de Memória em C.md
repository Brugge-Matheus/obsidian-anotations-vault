---
tags:
  - c
  - c/memoria
  - c/depuracao
source: "Padrão ISO C (comportamento indefinido); AddressSanitizer/LeakSanitizer (gcc 13.3), Valgrind, gcc -fanalyzer; experimentos reais em Linux x86-64"
---
# Erros Clássicos de Memória em C

📚 **Referência:** Padrão ISO C (comportamento indefinido), AddressSanitizer/LeakSanitizer, Valgrind e `gcc -fanalyzer`; todos os erros abaixo foram **reproduzidos** com `gcc` 13.3 | Anterior: [[Stack vs Heap em C]] | Próxima: *Structs em C* | Compare com [[Garbage Collector]] (Go)


> 🧭 **Como ler esta nota:** primeiro **por que** estes erros são tão perigosos; depois **oito erros**, cada um com **o código, o que acontece sem ferramentas e o que a ferramenta mostra**; em seguida, **como ler um relatório do ASan**, um guia **do sintoma à causa**, o catálogo de **ferramentas**, o **checklist** de prevenção e uma **caça aos bugs** para praticar.
---

# ⚠️ Por que estes erros são especiais

C não verifica limites, não rastreia quem é dono de cada bloco e não coleta lixo ([[Alocação Dinâmica em C]]). Os erros de memória são **comportamento indefinido** (UB, [[Como C Funciona]]), e o pior deles é que **muitas vezes o programa parece funcionar**: a memória é corrompida **em silêncio** e a falha aparece **muito depois**, em outro ponto do código. Essa é uma das maiores fontes de **falhas de segurança** em software escrito em C (o *Heartbleed*, de 2014, foi uma leitura além do limite de um buffer).

Para cada erro, abaixo: **o código**, o que acontece **sem ferramentas** e o que a ferramenta **mostra**. A seção final reúne as ferramentas.

> 💡 **Regra de estudo:** todos os exemplos compilam **sem erro**. É justamente o perigo.

---

# 🧨 1. *Buffer overflow* (estouro de buffer)

Escrever (ou ler) **fora** dos limites de um bloco ou array ([[Arrays em C]], [[Aritmética de Ponteiros em C]]).

```c
int *v = malloc(5 * sizeof *v);
for (int i = 0; i <= 5; i++) v[i] = i;        // i <= 5: escreve v[5], um além do fim
```

| Sem ferramentas | Com `-fsanitize=address` |
|---|---|
| **Nada visível**: o programa imprimiu `0` e terminou normalmente | `heap-buffer-overflow ... WRITE of size 4 ... located 0 bytes after 20-byte region` |

Três variantes, todas detectadas: na ***heap*** (acima), na **pilha** (`stack-buffer-overflow`, visto em [[Arrays em C]]) e em **globais** (`global-buffer-overflow: located 0 bytes after global variable 'tabela' ... of size 16`).

## O clássico *off-by-one* com strings

```c
const char *s = "abcdef";
char *c = malloc(strlen(s));        // 6 bytes: faltou 1 para o '\0'
strcpy(c, s);                       // escreve 7 bytes
```

O ASan acusou `heap-buffer-overflow ... WRITE of size 7 ... located 0 bytes after 6-byte region`; sem ele, o programa **funcionou** (imprimiu `abcdef`). A regra de [[Strings em C]] é: **sempre `strlen(s) + 1`**. Prefira `strdup` ou `snprintf`.

---

# 💀 2. *Use-after-free* (usar depois de liberar)

```c
int *p = malloc(sizeof *p);
*p = 42;
free(p);
printf("%d\n", *p);                 // p ainda guarda o endereço antigo (pendurado)
```

| Sem ferramentas | Com ASan |
|---|---|
| Imprimiu **`1686468499`** (não `42`): o alocador **reaproveitou o bloco** e gravou seus próprios dados ali | `heap-use-after-free ... READ of size 4 ... freed by thread T0 here:` |

A leitura devolveu **lixo**; uma **escrita** poderia corromper outro objeto que ocupasse o bloco reaproveitado. A prevenção está em [[Alocação Dinâmica em C]]: **`free(p); p = NULL;`** (um uso posterior vira `NULL` e falha na hora).

---

# 👯 3. *Double free* (liberar duas vezes)

```c
char *p = malloc(16);
free(p);
free(p);                            // segundo free do mesmo bloco
```

| Sem ferramentas | Com ASan |
|---|---|
| O `glibc` **detectou e abortou**: `free(): double free detected in tcache 2` | `attempting double-free on 0x502000000010` (mostra onde alocou e liberou) |

A detecção nativa **não é garantida** (depende da versão e do estado do alocador): corromper as listas internas de blocos livres é uma técnica clássica de **ataque**. Com `p = NULL` após o `free`, o segundo `free(NULL)` é **inofensivo**.

---

# 🚫 4. *Free* inválido

Liberar algo que **não veio** de `malloc`/`calloc`/`realloc`, ou o **meio** de um bloco:

```c
int x = 0;
int *p = malloc(4 * sizeof *p);
free(p + 1);                        // não é o início do bloco
free(&x);                           // é um endereço da PILHA
```

O `glibc` aborta com **`free(): invalid pointer`**; o ASan diz `attempting free on address which was not malloc()-ed ... located 4 bytes inside of 16-byte region`. Só se pode liberar **exatamente** o ponteiro devolvido pela alocação ([[Alocação Dinâmica em C]]): por isso, se você andar com `p++` pelo bloco, **guarde o ponteiro original**.

---

# 🕳️ 5. *Memory leak* (vazamento)

Perder o **único ponteiro** de um bloco alocado, sem liberá-lo:

```c
static void f(void) { char *s = malloc(40); strcpy(s, "perdido"); }   // sai sem free
int main(void) { f(); f(); return 0; }
```

O programa **funciona**: o vazamento não gera erro, só **consome memória**. Em programas de longa duração (servidores) o consumo cresce até esgotar a memória. Quando `f` retorna, o ponteiro `s` (na **pilha**) some e o bloco (na *heap*) fica órfão ([[Stack vs Heap em C]]).

| Ferramenta | Saída real |
|---|---|
| ASan (com o **LeakSanitizer**, ao encerrar) | `Direct leak of 40 byte(s) in 1 object(s)` (×2) → `SUMMARY: 80 byte(s) leaked in 2 allocation(s)` |
| **Valgrind** (`--leak-check=full`) | `total heap usage: 2 allocs, 0 frees, 80 bytes allocated` e `40 bytes in 1 blocks are definitely lost`, com a pilha de chamadas (`f (leak.c:3)`) |

**Causas comuns:** esquecer o `free`; `return` antecipado entre o `malloc` e o `free` (use o `goto` de limpeza, [[Controle de Fluxo em C]]); e **`d = realloc(d, n)`**, em que uma falha devolve `NULL`, sobrescreve `d` e **perde o bloco original** (use um `tmp`).

---

# 🎲 6. Ler memória **não inicializada**

```c
int *p = malloc(sizeof *p);         // conteúdo INDETERMINADO
if (*p > 0) puts("positivo"); else puts("nao positivo");
```

O resultado depende do **lixo** que estava ali: pode variar entre execuções. O ASan **não detecta** este erro. O **Valgrind** sim: `Conditional jump or move depends on uninitialised value(s)`. O `gcc` também avisa neste caso, em tempo de compilação: `'*p' is used uninitialized` (`-Wuninitialized`, com `-O2`). Prevenção: **inicialize** tudo (`calloc`, `= {0}`, `= NULL`), como em [[Variáveis e Constantes em C]].

---

# 🏃 7. Ponteiro para a pilha devolvido (*use-after-return*)

```c
int *f(void) { int x = 7; int *p = &x; return p; }       // x some no return
int main(void) { int *q = f(); printf("%d\n", *q); }
```

Sem ferramentas, o programa **imprimiu `7`**: o dado ainda estava na pilha **por acaso**, e basta outra chamada para sobrescrevê-lo. O ASan o acha se for ativado: `ASAN_OPTIONS=detect_stack_use_after_return=1` → `stack-use-after-return ... READ of size 4`. Com variável direta (`return &x;`) o `gcc` já avisa em compilação (`-Wreturn-local-addr`, [[Funções e Ponteiros em C]]).

---

# 📏 8. Outros erros frequentes

| Erro | Exemplo | Efeito / aviso |
|---|---|---|
| **`NULL` sem checar** | `char *p = malloc(10); strcpy(p, "oi");` | UB se `malloc` falhar; `gcc -fanalyzer`: `use of possibly-NULL 'p' where non-null expected` |
| **`sizeof` de ponteiro** | `void zera(int v[], size_t n) { memset(v, 0, sizeof v); }` | zera só **8 bytes** (testado: `0 0 1 1 1 1 1 1`); o `gcc` avisa `-Wsizeof-array-argument` e `-Wsizeof-pointer-memaccess` ([[Arrays e Ponteiros em C]]) |
| **Multiplicação que estoura** | `malloc(n * sizeof *p)` com `n` enorme | bloco **menor** que o esperado; use `calloc` ([[Alocação Dinâmica em C]]) |
| **Usar ponteiro após `realloc`** | guardar `p = &v[2]`, depois crescer `v` | o bloco pode **mudar de endereço**: `p` fica pendurado |
| **Índice negativo / `<=`** | `v[-1]`, `for (i = 0; i <= n; i++)` | *buffer overflow* ou *underflow* ([[Controle de Fluxo em C]]) |

---

# 📖 Como ler um relatório do AddressSanitizer

A saída do ASan assusta pelo tamanho, mas **tem sempre a mesma estrutura**. Vamos ler o relatório do *use-after-free* (seção 2), que aqui foi cortado só nas linhas de rodapé:

```
==370==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000010 ...      ← (1) O QUE aconteceu
READ of size 4 at 0x502000000010 thread T0                                             ← (2) a AÇÃO errada
    #0 ... in main /tmp/e12/uaf.c:7                                                    ← (3) ONDE (arquivo:linha)

0x502000000010 is located 0 bytes inside of 4-byte region [0x...010,0x...014)          ← (4) o bloco envolvido
freed by thread T0 here:                                                                ← (5) QUEM liberou
    #0 ... in free ...
    #1 ... in main /tmp/e12/uaf.c:6                                                    ←     (linha do free)

previously allocated by thread T0 here:                                                 ← (6) QUEM alocou
    #1 ... in main /tmp/e12/uaf.c:4                                                    ←     (linha do malloc)

SUMMARY: AddressSanitizer: heap-use-after-free /tmp/e12/uaf.c:7 in main                ← (7) resumo de uma linha
```

Leia **de cima para baixo**: (1) o **tipo** do erro (`heap-use-after-free`, `heap-buffer-overflow`, `stack-buffer-overflow`, `attempting double-free`...); (2) se era **leitura** ou **escrita** e de quantos bytes; (3) a **linha do seu código** onde ocorreu (a primeira linha `#0` que aponta para o **seu** arquivo, não para a biblioteca); (4) onde o endereço cai em relação ao bloco (`0 bytes after 20-byte region` = logo **depois** do fim: *overflow* de um elemento); (5) e (6) a **linha do `free`** e a **do `malloc`**, que contam a história do bloco. Na prática, **vá direto para as linhas `uaf.c:7`, `uaf.c:6` e `uaf.c:4`**: uso, liberação, alocação.

> 💡 O ASan **para no primeiro erro**: corrija, recompile e rode de novo (um programa com vários bugs revela um por vez). Para ver os símbolos e linhas, compile com **`-g`**.

---

# 🩺 Do sintoma à causa

Quando o programa se comporta mal, o **sintoma** sugere por onde começar:

| Sintoma | Causa provável | Ferramenta |
|---|---|---|
| `Segmentation fault` logo ao usar um ponteiro | ponteiro **`NULL`**, não inicializado ou pendurado | ASan, `gdb` (a linha do erro) |
| `free(): double free detected` / `invalid pointer` | *double free* ou `free` de endereço que não veio de `malloc` (seções 3 e 4) | ASan |
| Funciona, mas imprime **lixo** em algum valor | **memória não inicializada** (6) ou *use-after-free* (2) | Valgrind, ASan |
| **O resultado muda** entre execuções iguais | variável não inicializada, ou *overflow* corrompendo vizinhos | Valgrind, `-Wuninitialized`, ASan |
| Um valor "se alterou sozinho" | *buffer overflow* escrevendo no vizinho (1) | ASan, UBSan |
| O programa **gasta mais e mais memória** | ***leak*** (5) | LeakSanitizer, Valgrind |
| Crasha **só às vezes** ou só em outra máquina | UB que depende do lixo/da disposição da memória | ASan + UBSan em **todos** os testes |
| Trava ao entrar numa função recursiva | *stack overflow* ([[Recursão em C]], [[Stack vs Heap em C]]) | `ulimit -s`, `gdb` |

---

# 🛠️ As ferramentas

| Ferramenta | Como usar | O que acha | Custo |
|---|---|---|---|
| **AddressSanitizer** (ASan) | `gcc -g -fsanitize=address prog.c` | *overflow* (heap, pilha, global), *use-after-free*, *double/invalid free*; **vazamentos** (LeakSanitizer); *use-after-return* com a variável de ambiente acima | ~2× mais lento; precisa **recompilar** |
| **UBSan** | `-fsanitize=undefined` | `signed overflow`, índices fora de limites de arrays, deslocamentos inválidos ([[Operadores em C]]) | pequeno |
| **Valgrind** | `valgrind --leak-check=full ./prog` | vazamentos, leitura de **memória não inicializada**, acessos inválidos; **sem recompilar** | 10–50× mais lento |
| **`gcc -fanalyzer`** | compilar com a flag | análise **estática**: `double-free`, `use-after-free`, `malloc-leak`, `possible-null-dereference`, `free-of-non-heap` | só na compilação |
| **Avisos** | `-Wall -Wextra` ([[Convenções e Estilo]]) | `-Wuninitialized`, `-Wreturn-local-addr`, `-Warray-bounds`, `-Wstringop-overflow` | nenhum |

O `-fanalyzer` apontou **estaticamente** (sem executar) os casos de *double free* (`double-'free' of 'p'`), *use-after-free* (`use after 'free' of 'p'`), vazamento (`leak of 's'`) e `free` de variável da pilha (`'free' of '&x' which points to memory on the stack`).

> ✅ **Hábito de desenvolvimento:** compile e **rode os testes** com `-g -fsanitize=address,undefined` (e passe por `valgrind` de vez em quando). Em produção, desligue-os (custo e riscos), mas **cada erro achado na fase de testes** é um a menos em campo.

---

# 🛡️ Como evitar: checklist

| Prática | Previne |
|---|---|
| **Inicialize** variáveis e ponteiros (`= NULL`, `calloc`, `= {0}`) | uso de lixo (6) |
| `p = malloc(n * sizeof *p);` e **teste `NULL`** | *NULL* sem checar, tamanho errado |
| **Cada `malloc` tem um `free`** e um dono documentado ([[Alocação Dinâmica em C]]) | vazamento (5), double free (3) |
| **`free(p); p = NULL;`** | use-after-free (2), double free (3) |
| **Passe o tamanho** junto de arrays e buffers; confira índices | *buffer overflow* (1) |
| **`snprintf`**, `strncat` e `strlen + 1` em vez de `strcpy` ([[Strings em C]]) | estouro de strings |
| **Nunca devolva** o endereço de uma variável local | use-after-return (7) |
| **Um só ponto de saída** de limpeza (`goto fim;`) | vazamento em saídas antecipadas |
| **Escopo mínimo**: alloc e free na mesma função/nível quando possível | confusão de propriedade |
| Rode **ASan/UBSan/Valgrind** nos testes e **`-fanalyzer`** na compilação | tudo o que escapou à revisão |

Linguagens com **coletor de lixo e verificação de limites**, como Go, eliminam a maior parte desses erros por construção ([[Garbage Collector]]); o preço é perder o controle fino da memória. Em C, a **disciplina e as ferramentas** ocupam esse lugar.

---

# 🕵️ Caça aos bugs

O programa abaixo **compila sem erros** e tem **três** defeitos de memória. Tente achá-los antes de ler a resposta:

```c
char *copia_nome(const char *s) {
    char *r = malloc(strlen(s));
    strcpy(r, s);
    return r;
}
int main(void) {
    char *a = copia_nome("Ana");
    char *b = copia_nome("Bia");
    free(a);
    printf("%s e %s\n", a, b);
    return 0;
}
```

**Resposta (conferida com o ASan, um erro por vez):**

| # | Bug | Onde | Como o ASan acusa | Correção |
|---|---|---|---|---|
| 1 | **faltou 1 byte** para o `'\0'`: `malloc(strlen(s))` | `copia_nome` | `heap-buffer-overflow ... WRITE` | `malloc(strlen(s) + 1)` |
| 2 | **uso depois do `free`**: `printf` de `a` | `main` | `heap-use-after-free ... READ` | não usar `a` depois do `free(a)` |
| 3 | **vazamento**: `b` nunca é liberado | `main` | `LeakSanitizer: detected memory leaks` (ao encerrar) | `free(b);` antes do `return` |

Sem ferramentas, o programa original **rodou até o fim sem nenhuma mensagem** (imprimiu lixo no lugar do primeiro texto, e `Bia` no segundo): é exatamente por isso que se testa com `-fsanitize=address`. Depois de corrigir os três, o ASan não acusa nada e o programa termina com código `0`.

---

# 🧪 Para praticar

1. Reproduza cada um dos oito erros desta nota em um arquivo pequeno e compile com `-fsanitize=address,undefined`. Para cada um, identifique no relatório as linhas do **uso**, do **`free`** e do **`malloc`**.
2. Escreva uma função com **três saídas antecipadas** (`return` no meio) depois de dois `malloc`, e depois reescreva com o padrão **`goto fim;`** de limpeza único.
3. Reescreva `copia_nome` de duas formas seguras: com `strdup` e com `snprintf`.
4. Escreva um programa com um **vazamento** dentro de um laço (um `malloc` por volta, sem `free`) e use `valgrind --leak-check=full` para ver o total perdido. Quantos bytes foram perdidos?
5. Pegue um programa de uma nota anterior (por exemplo, a lista de nomes de [[Strings em C]]) e rode-o sob ASan + UBSan: apareceu algum aviso que você não esperava?

---

# ✅ Resumo do Conceito

- Erros de memória em C são **UB** e **silenciosos**: o programa pode "funcionar" e falhar só depois, em outro lugar; são grande fonte de falhas de segurança
- ***Buffer overflow*** (heap, pilha, global; `<=`, falta do `+1` do `'\0'`): sem ferramenta **nada acontece**; ASan: `heap-buffer-overflow`
- ***Use-after-free***: leu **lixo** (`1686468499`); ASan: `heap-use-after-free`; use **`p = NULL`** após o `free`
- ***Double free*** e ***free* inválido** (`p + 1`, `&x`): o `glibc` aborta (`double free detected`, `invalid pointer`) **às vezes**; ASan sempre detecta
- ***Leak***: não gera erro, só consome memória (LeakSanitizer: `80 byte(s) leaked`; Valgrind: `definitely lost`); cuidado com `d = realloc(d, ...)`
- **Memória não inicializada:** só Valgrind (e `-Wuninitialized` às vezes) acham; ASan não
- **Devolver endereço de local**: "funcionou" por acaso (`7`); ASan com `detect_stack_use_after_return=1`
- **Ferramentas:** **ASan** + **UBSan** (`-g -fsanitize=address,undefined`), **Valgrind**, **`-fanalyzer`**, `-Wall -Wextra`; rode-as nos testes
- **Prevenção:** inicializar, testar `NULL`, `sizeof *p`, passar tamanhos, `snprintf`, `free` + `NULL`, dono definido, limpeza única

---

# 🔗 Conexões com outras anotações

- [[Alocação Dinâmica em C]] — as funções cujo mau uso gera estes erros e as regras do `free`
- [[Stack vs Heap em C]] — onde cada bloco vive e por que o vazamento é da *heap*
- [[Arrays em C]] e [[Aritmética de Ponteiros em C]] — limites, índices e "um além do fim"
- [[Ponteiros em C]] e [[Funções e Ponteiros em C]] — ponteiro nulo, selvagem e pendurado; retorno de local
- [[Strings em C]] — `strcpy` × `snprintf` e o terminador
- [[Controle de Fluxo em C]] — `goto` de limpeza e *off-by-one* em laços
- [[Como C Funciona]] e [[O Compilador]] — UB, *sanitizers* e `-fsanitize=undefined`
- [[Convenções e Estilo]] — `-Wall -Wextra` e o uso de ferramentas
- [[Garbage Collector]] e [[Memória e Escape Analysis]] — como Go evita esta classe de erros
