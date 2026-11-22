---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: <stdlib.h> malloc, calloc, realloc, free); glibc (malloc_usable_size); experimentos com gcc 13.3, strace e ulimit em Linux x86-64"
---
# Alocação Dinâmica em C

📚 **Referência:** Padrão ISO C (C17/C23), `<stdlib.h>` e experimentos reais com `gcc` 13.3, `strace` e `ulimit` | Anterior: [[Funções e Ponteiros em C]] | Próxima: [[Stack vs Heap em C]] | Compare com [[Memória e Escape Analysis]] e [[Garbage Collector]] (Go)

---

# 🎯 Por que alocar memória dinamicamente

Até aqui, toda memória vinha de **variáveis declaradas**: o tamanho era fixo na compilação ([[Arrays em C]]) e o objeto **morria ao fim do escopo** ([[Sintaxe, Blocos e Escopo em C]]). Isso não basta quando:

| Necessidade | Por que o array comum falha |
|---|---|
| O tamanho só é conhecido **em tempo de execução** (ler *n* itens de um arquivo) | tamanho fixo; um VLA vive na pilha ([[Arrays em C]]) |
| O dado deve **sobreviver** à função que o criou | variável local some no `return` ([[Funções em C]]) |
| O dado é **grande** | a pilha tem só **8 MiB** ([[Recursão em C]]) |
| A quantidade **cresce** durante o programa (lista, vetor) | não se redimensiona um array |

A solução é pedir memória ao sistema **durante a execução**, numa região chamada ***heap***, e **devolvê-la** quando não for mais necessária. Em C isso é **manual**: **você** decide quando alocar e quando liberar (em Go quem libera é o coletor de lixo, [[Garbage Collector]]). A comparação entre *heap* e pilha é o tema da próxima nota.

---

# 🧰 As quatro funções de `<stdlib.h>`

| Função | Faz | Conteúdo inicial |
|---|---|---|
| `malloc(n)` | reserva **`n` bytes** | **indeterminado** (lixo) |
| `calloc(qtd, tam)` | reserva `qtd × tam` bytes | **zerado** |
| `realloc(p, n)` | **redimensiona** o bloco de `p` para `n` bytes | o conteúdo antigo é **preservado** |
| `free(p)` | **devolve** o bloco | — |

As três primeiras devolvem **`void *`** (que converte sozinho para qualquer ponteiro em C, [[Ponteiros em C]]) ou **`NULL`** se **não houver memória**.

---

# 📦 `malloc` e `free`: o padrão idiomático

```c
int n = 5;
int *v = malloc(n * sizeof *v);        // 1) reserva 5 ints
if (v == NULL) {                        // 2) SEMPRE confira
    perror("malloc");
    return 1;
}
for (int i = 0; i < n; i++) v[i] = (i + 1) * 10;     // 3) usa como array: 10 20 30 40 50
free(v);                                // 4) devolve
v = NULL;                               // 5) evita ponteiro pendurado
```

Cada linha tem uma razão:

- **`sizeof *v`** em vez de `sizeof(int)`: o tamanho acompanha o tipo de `v` ([[Tipos de Dados em C]]);
- **Sem *cast*** no retorno de `malloc` (`(int *)malloc(...)`): em C ele é **desnecessário** e pode esconder um `#include <stdlib.h>` esquecido;
- **Teste `NULL`**: se a alocação falhar e você usar `v`, é desreferência nula (UB, [[Ponteiros em C]]);
- **`v` é um ponteiro**, então `sizeof v` dá **8**, e **não** o tamanho do bloco ([[Arrays e Ponteiros em C]]): guarde `n` à parte;
- **`free(v); v = NULL;`**: depois do `free`, `v` ainda guarda o endereço antigo (**pendurado**). Zerar o ponteiro faz um uso posterior falhar logo (e torna um segundo `free(v)` inofensivo).

**Regras do `free`:** só aceita ponteiros **devolvidos por `malloc`/`calloc`/`realloc`** (ou `NULL`); cada bloco é liberado **uma única vez**; **`free(NULL)` é seguro** e não faz nada (testado). Violar essas regras é **UB** ([[Erros Clássicos de Memória em C]]).

---

# 🧮 `calloc`: zeros e proteção contra estouro

`calloc(qtd, tam)` faz duas coisas que `malloc(qtd * tam)` não faz:

1. **Zera** o bloco (`calloc(4, sizeof *c)` imprimiu `0 0 0 0`);
2. **Verifica** se `qtd × tam` **cabe em `size_t`**.

O segundo ponto é uma **falha de segurança clássica**: numa multiplicação em `size_t` que **dá a volta** ([[Tipos de Dados em C]]), `malloc` reserva **menos** do que o esperado e o programa depois escreve fora do bloco. Teste real: com `a = SIZE_MAX/2 + 2` e `b = 2`, o produto `a * b` deu **2** (o tamanho "mágico" que `malloc` receberia), enquanto `calloc(SIZE_MAX/2, 4)` devolveu **`NULL`**, detectando o estouro. O `gcc` até avisa em constantes (`-Walloc-size-larger-than`).

> ✅ Para blocos de **`n` elementos** com `n` vindo de fora (arquivo, rede, usuário), prefira `calloc(n, sizeof *p)` ou **confira a multiplicação** antes do `malloc`.

---

# 🔄 `realloc`: crescer e encolher

`realloc(p, novo)` tenta **mudar o tamanho** do bloco, **preservando o conteúdo** (até o menor dos dois tamanhos). Se não há espaço contíguo ao lado, **aloca outro bloco, copia e libera o antigo**: o **endereço pode mudar**, então **todo ponteiro antigo para dentro do bloco fica inválido**.

```c
int *tmp = realloc(d, ncap * sizeof *d);   // NUNCA: d = realloc(d, ...)
if (tmp == NULL) {                          // falhou: d continua válido!
    free(d);
    return 1;
}
d = tmp;                                    // só agora troca
```

**Por que um `tmp`?** Em caso de falha `realloc` devolve `NULL` **e não libera o bloco original**. Com `d = realloc(d, ...)`, o `NULL` **sobrescreve `d`** e o bloco antigo **vaza** (fica sem dono). Casos especiais: `realloc(NULL, n)` equivale a `malloc(n)` (testado); evite `realloc(p, 0)`: o comportamento variava entre implementações e passou a ser **indefinido** no C23.

## Exemplo: vetor que cresce (dobrando a capacidade)

Guardando `len` (usado) e `cap` (reservado), cresce-se `cap` **dobrando** quando `len == cap`. Teste: 1000 inserções com `cap` indo de 4 até 1024 (9 reservas no total) e **apenas 1** movimentação do bloco além da alocação inicial:

```c
if (len == cap) {
    size_t ncap = cap ? cap * 2 : 4;
    int *tmp = realloc(d, ncap * sizeof *d);
    if (!tmp) { free(d); return 1; }
    d = tmp;  cap = ncap;
}
d[len++] = i;
```

Dobrar a capacidade (em vez de somar 1 a cada vez) faz o custo **amortizado** de cada inserção ser O(1) ([[Complexidade de Algoritmos]]), pois poucas realocações caras se diluem em muitas inserções. É a estrutura por trás de listas dinâmicas em várias linguagens (o *slice* do Go, em [[Arrays e Slices]]).

---

# 👤 Quem libera? Propriedade da memória

C não rastreia donos. A regra é **combinar e documentar**: **quem aloca deve saber quem vai liberar**. Convenções comuns:

| Situação | Quem libera |
|---|---|
| A função **devolve** memória alocada (`char *copia = strdup(s);`) | o **chamador** |
| Uma função recebe um ponteiro e **só usa** | **ninguém** (continua com o dono) |
| Um par `criar()` / `destruir()` | quem chamou `criar()` chama `destruir()` |

`strdup` (padrão a partir do C23, antes extensão POSIX) **aloca e copia** uma string: `char *copia = strdup(s)` deu um endereço **diferente** de `s`, e **precisa de `free(copia)`** ([[Strings em C]]). Outro padrão é a função que **aloca por parâmetro**, usando `T **` ([[Funções e Ponteiros em C]]): `int criar(int **out, size_t n)`, devolvendo *status* e entregando o ponteiro em `*out`.

Quando uma função tem **vários pontos de falha** depois de alocar recursos, centralize a limpeza com `goto` ([[Controle de Fluxo em C]]).

---

# 💥 Quando a alocação falha

`malloc` devolve `NULL` se **não conseguir**. Testes reais:

| Situação | Resultado |
|---|---|
| `malloc((size_t)-1)` (pedido absurdo) | **`NULL`** |
| Laço pedindo blocos de 10 MiB com o limite de memória virtual em 200 MiB (`ulimit -v 204800`) | **`NULL` após 190 MiB** |

No Linux, o sistema costuma permitir ***overcommit***: o `malloc` "dá certo" mesmo sem RAM suficiente e o problema só aparece ao **usar** a memória (o *kernel* pode então encerrar o processo). **Nem por isso se deve deixar de testar `NULL`**: em outros sistemas, em contêineres com limite e em microcontroladores a falha é comum ([[Container vs VM]]).

---

# ⚙️ O que acontece por dentro

O `malloc` **não** chama o sistema operacional a cada pedido. Ele gerencia a *heap* por conta própria:

- **Pede memória ao SO em blocos grandes** (chamadas de sistema `brk` e `mmap`) e **subdivide**. No `strace` de `malloc(100)`, o `brk` **aumentou a *heap* em 132 KiB de uma vez**;
- Ao `free`, o bloco vai para uma **lista de blocos livres** para ser reaproveitado, e **não** volta necessariamente ao SO ([[Gerenciando a Memória Livre]]; a memória em si é [[Memória Virtual]] paginada);
- **O bloco real é maior que o pedido** (glibc, `malloc_usable_size`): `malloc(1)` dá **24** bytes utilizáveis, `malloc(25)` dá **40**, e todo bloco é **alinhado a 16 bytes**. Cada bloco também carrega **metadados** (tamanho etc.) antes dele.

**Custo:** `malloc` + `free` de 64 bytes custou **cerca de 7 a 9 ns** (5 milhões de repetições), contra **~0,3 ns** de um buffer local (ordem de grandeza neste ambiente, com `-O1`): alocar é **barato, mas muito mais caro que a pilha**. Evite `malloc` em laços quentes; reaproveite blocos.

---

# 🗂️ Matriz dinâmica: duas formas

| Forma | Alocação | Acesso | Liberação |
|---|---|---|---|
| **Array de ponteiros** (`int **m`) | 1 `malloc` para as linhas + 1 `calloc` **por linha** | `m[i][j]` | 1 `free` por linha + 1 do array |
| **Bloco único** (`int *flat`) | **1** `calloc(lin * col, sizeof *flat)` | `flat[i * col + j]` | **1** `free` |

Os dois funcionaram (`m[2][3] = 7` e `flat[2*col+3] = 7`). O **bloco único** é preferível: menos chamadas, **memória contígua** (melhor para a *cache*, [[Arrays]]) e **um** `free` só. É a mesma conta `i × colunas + j` de [[Arrays em C]].

---

# ✅ Resumo do Conceito

- **Alocação dinâmica** serve quando o tamanho é só conhecido na execução, o dado deve **sobreviver à função**, é **grande** ou **cresce**; a memória vem da ***heap*** e é **manual**
- **`malloc(n)`** (lixo), **`calloc(q, t)`** (zera e **checa overflow**), **`realloc(p, n)`** (redimensiona; **pode mover**), **`free(p)`**
- Padrão: **`p = malloc(n * sizeof *p)`**, **teste `NULL`**, sem *cast*, `free(p); p = NULL;`; `free(NULL)` é seguro; `sizeof p` é 8, **guarde o tamanho**
- **`realloc` sempre com `tmp`**: `d = realloc(d, ...)` vaza o bloco em caso de falha; ponteiros antigos ficam inválidos se o bloco se move
- **Vetor dinâmico:** dobrar a capacidade dá custo **amortizado O(1)** por inserção
- **Propriedade:** documente **quem libera**; `criar`/`destruir`, "quem recebe memória alocada libera"
- **Falha:** `malloc` devolve `NULL` (testado: pedido absurdo e limite de 200 MiB); no Linux o *overcommit* adia o problema, mas **sempre** teste
- **Por dentro:** a *heap* cresce em blocos grandes (`brk` de 132 KiB), blocos livres são reaproveitados, o bloco real é **maior e alinhado a 16 bytes**; `malloc`+`free` ≈ **8 ns** × **0,3 ns** da pilha
- **Matriz dinâmica:** prefira **um bloco** com `i * col + j`

---

# 🔗 Conexões com outras anotações

- [[Funções e Ponteiros em C]] — funções que devolvem ponteiros alocados e `T **` como saída
- [[Ponteiros em C]] — `void *`, `NULL` e ponteiro pendurado depois do `free`
- [[Arrays em C]] e [[Arrays e Ponteiros em C]] — o array de tamanho fixo que a alocação dinâmica supera
- [[Strings em C]] — `strdup` e buffers alocados
- [[Tipos de Dados em C]] — `size_t` e overflow da multiplicação
- [[Controle de Fluxo em C]] — `goto` para limpeza de vários recursos
- [[Recursão em C]] — o limite de 8 MiB da pilha
- [[Complexidade de Algoritmos]] e [[Arrays]] — custo amortizado e *cache*
- [[Gerenciando a Memória Livre]] e [[Memória Virtual]] — como o sistema controla blocos livres e páginas
- [[Memória e Escape Analysis]] e [[Garbage Collector]] — o contraste com Go (alocação decidida pelo compilador e coleta automática)
