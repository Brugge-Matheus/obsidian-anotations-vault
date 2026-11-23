---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (storage duration); Linux (/proc/self/maps, ulimit); glibc (mallinfo2); experimentos com gcc 13.3 em Linux x86-64"
---
# Stack vs Heap em C

📚 **Referência:** Padrão ISO C (duração de armazenamento), `/proc/self/maps`, `mallinfo2` e experimentos reais com `gcc` 13.3 | Anterior: [[Alocação Dinâmica em C]] | Próxima: [[Erros Clássicos de Memória em C]] | Compare com [[Memória e Escape Analysis]] (Go)


> 🧭 **Como ler esta nota:** começa com uma analogia, depois o **mapa real** de uma execução, as características de cada região, um **programa acompanhado linha a linha** (quem vive onde), a fragmentação e um guia de **quando usar cada uma**, fechando com erros comuns e exercícios.
---

# 🗺️ Onde cada coisa vive

Um programa C em execução usa **regiões de memória** diferentes ([[Como C Funciona]] mostra o espaço de endereçamento completo). Esta nota foca nas duas que você controla: a **pilha** (*stack*) e a ***heap***. Mas antes, um mapa real: um programa que imprime o endereço de cada tipo de dado e procura, em `/proc/self/maps`, **a qual região** o endereço pertence:

| O que é | Endereço (exemplo) | Região (permissões) |
|---|---|---|
| código da função | `0x55c308817437` | **texto do executável** (`r-xp`: ler e executar) |
| literal `"abc"` | `0x55c308818061` | **somente leitura** (`r--p`, `.rodata`) |
| `const int` global | `0x55c308818008` | somente leitura (`r--p`) |
| global **inicializado** | `0x55c30881a010` | **dados** (`rw-p`, `.data`) |
| global **zerado** | `0x55c30881a01c` | dados (`rw-p`, `.bss`) |
| `static` **local** | `0x55c30881a014` | **dados** (mesma região das globais) |
| `malloc(16)` | `0x55c30a82c2a0` | **`[heap]`** (`rw-p`) |
| variável **local** | `0x7ffdaadae430` | **`[stack]`** (`rw-p`) |
| `printf` | `0x7ff1af860100` | **biblioteca** `libc.so.6` (`r-xp`) |

```
 endereços ALTOS   ┌──────────────────┐
 (0x7ff...)        │  pilha (stack)   │  cresce para BAIXO ↓
                   │        ↓         │
                   │  bibliotecas     │  libc.so, ld.so
                   │        ↑         │
                   │  heap            │  cresce para CIMA ↑
                   ├──────────────────┤
                   │  .bss  .data     │  globais e static
                   │  .rodata  .text  │  literais e código
 endereços BAIXOS  └──────────────────┘  (0x55...)
```

Os endereços **mudam** a cada execução (ASLR, [[Ponteiros em C]]), mas a **ordem** se mantém: `0x55...` para o executável e a *heap* (perto dele), `0x7f...` para as bibliotecas e a pilha. Cada região tem **permissões**: o código não é gravável e os dados não são executáveis, proteções do sistema ([[Hardware de Proteção]]).

## Uma analogia: mesa de trabalho × armazém

A **pilha** é a sua **mesa de trabalho**: pequena, organizada, e cada tarefa (função) pega um espaço **na ponta** e o **devolve ao terminar**, sem você fazer nada; você só guarda ali o que está usando **agora**. A ***heap*** é um **armazém**: enorme, para guardar coisas **por quanto tempo quiser**, mas você precisa **pedir o espaço** (`malloc`), **anotar onde ficou** (o ponteiro) e **avisar quando liberar** (`free`). Pedir ao armazém **demora mais** que usar a mesa, e se você esquecer de devolver, o espaço fica ocupado.

---

# 📚 A pilha (*stack*)

Já usada em [[Funções em C]]: cada chamada empilha um ***stack frame*** com parâmetros, variáveis locais e o endereço de retorno; ao retornar, o quadro é descartado.

| Característica | Pilha |
|---|---|
| **Quem gerencia** | o **compilador**, **automaticamente** |
| **Vida do dado** | do início ao fim do **bloco/função** ([[Sintaxe, Blocos e Escopo em C]]) |
| **Ordem** | **LIFO** (o último a entrar é o primeiro a sair, [[Pilha (Stack)]]) |
| **Crescimento** | para endereços **menores** (testado: 3 níveis de chamada → 32 bytes mais baixo a cada um) |
| **Custo** | **quase zero** (basta ajustar o registrador `rsp`): ≈ **0,3 ns** num teste |
| **Tamanho** | **pequeno e fixo**: **8 MiB** no Linux (`ulimit -s` = 8192) |

---

# 🏗️ A *heap*

Vista em [[Alocação Dinâmica em C]]: memória pedida com `malloc` e devolvida com `free`.

| Característica | *Heap* |
|---|---|
| **Quem gerencia** | **você** (`malloc`/`free`) |
| **Vida do dado** | **do `malloc` até o `free`**, independe de escopo e de função |
| **Ordem** | **sem ordem**: blocos alocados e liberados em qualquer sequência |
| **Crescimento** | para endereços **maiores** (3 `malloc(32)` seguidos: `...d8d0`, `...d900`, `...d930`, espaçados por 48 bytes) |
| **Custo** | **maior**: o alocador procura um bloco livre (≈ **8 ns** por `malloc`+`free` no teste) |
| **Tamanho** | **grande**, limitado pela memória disponível: `malloc` + `memset` de **512 MiB** funcionou |

---

# ⚖️ Comparação direta

| | **Pilha** | ***Heap*** |
|---|---|---|
| Alocação | automática (declarar a variável) | manual (`malloc`) |
| Liberação | automática (fim do escopo) | manual (`free`) |
| Velocidade | muito rápida | mais lenta |
| Tamanho | ~8 MiB | gigabytes |
| Vida | limitada ao escopo | até você liberar |
| Fragmentação | **não tem** | **tem** |
| Falha ao esgotar | ***stack overflow***: o programa **morre** (`Segmentation fault`) | `malloc` devolve **`NULL`** (dá para tratar) |
| Erro típico | devolver o endereço de uma **local** (pendurado) | *leak*, *use-after-free*, *double free* |

**Teste de esgotamento:** um array local de **16 MiB** causou `Segmentation fault` (código 139), enquanto `malloc` de 16 MiB e até **512 MiB** foram usados por inteiro sem problema. Na pilha **não há aviso nem como testar** antes: ela simplesmente acaba. Na *heap* o `NULL` é verificável ([[Arrays em C]], [[Recursão em C]]).

---

# 🔗 O ponteiro e o apontado moram em lugares diferentes

Um ponto que confunde: a **variável ponteiro** e o **dado para o qual ela aponta** são objetos **diferentes**, e podem estar em regiões diferentes.

```c
void f(void) {
    int *p = malloc(sizeof *p);     // p: variável LOCAL (pilha)  →  *p: bloco na HEAP
    *p = 7;
    free(p);                        // libera o bloco; o p (pilha) some sozinho no return
}
```

```
   PILHA (f)                 HEAP
  ┌────────────┐          ┌──────────┐
  │ p ─────────┼────────► │    7     │
  └────────────┘          └──────────┘
 some no return           fica até o free()
```

Se a função **sair sem `free`**, o `p` desaparece, mas o **bloco continua reservado** e sem ninguém que o aponte: é o ***memory leak*** ([[Erros Clássicos de Memória em C]]). Já o inverso (devolver um ponteiro para a **pilha**) dá **ponteiro pendurado** ([[Funções e Ponteiros em C]]).

Outros casos: um **array local** (`int v[100]`) e um **VLA** vivem **inteiros na pilha**; um **`static` local** vive na região de **dados** (por isso mantém o valor entre chamadas); **parâmetros** vivem na pilha (ou em registradores, [[Funções em C]]).

---

# 🔬 Um programa, linha a linha: quem vive onde

```c
int global = 1;                                  // (a) região de dados

int conta(void) { static int n = 0; return ++n; }          // (b) n: região de dados
int *cria(void) { int *p = malloc(sizeof *p); *p = 42; return p; }

void f(int a) {                                  // (c) a: pilha (quadro de f)
    int local = a * 2;                           // (d) local: pilha
    int *h = cria();                             // (e) h: pilha;  *h: HEAP
    printf("f: local=%d  *h=%d\n", local, *h);
    free(h);                                     // (f) libera a heap; h some no return
}

int main(void) {
    int x = 10;                                  // (g) x: pilha (quadro de main)
    f(x);
    for (int i = 0; i < 3; i++) printf("conta: %d\n", conta());
    return 0;
}
```

| Dado | Onde vive | Quando nasce | Quando morre |
|---|---|---|---|
| `global` | região de **dados** | início do programa | fim do programa |
| `n` (`static` em `conta`) | região de **dados** | início do programa | fim do programa (**por isso `conta()` lembra o valor**) |
| `x` (em `main`) | **pilha** (quadro de `main`) | entrada de `main` | saída de `main` |
| `a`, `local`, `h` (em `f`) | **pilha** (quadro de `f`) | **chamada** de `f` | **retorno** de `f` |
| `*h` (o `int` alocado) | **heap** | `malloc` em `cria` | **`free(h)`** em `f` |

O programa imprimiu `f: local=20  *h=42` e, nas três chamadas de `conta()`, `1`, `2` e `3` (sem erros e sem vazamentos no ASan). Observação: pôr as três chamadas **dentro de um mesmo `printf`** imprimiu `3 2 1`, porque a **ordem de avaliação dos argumentos não é definida** em C ([[Operadores em C]]); por isso o laço. Três lições:

- **O `int` de `*h` sobreviveu a `cria()`**: a função `cria` terminou (seu quadro foi descartado), mas o bloco da *heap* continuou válido, e quem o **recebeu** (`f`) o liberou. É a regra de "quem libera" de [[Alocação Dinâmica em C]].
- **`h` e `*h` são coisas diferentes** (o ponteiro na pilha, o dado na *heap*); ao fim de `f`, `h` some sozinho, mas o bloco só some com o `free`.
- **`n`** não está na pilha, embora seja declarado **dentro** de uma função: `static` o move para a região de dados.

---

# 🧩 Fragmentação da *heap*

A pilha é sempre **LIFO**, então **nunca tem buracos**. Na *heap*, blocos são liberados em **qualquer ordem**, e o espaço livre se **espalha**:

| Tipo | O que é | Teste real |
|---|---|---|
| **Externa** | há memória livre **suficiente no total**, mas em **pedaços pequenos separados**, nenhum grande o bastante | 1000 blocos de 1 KiB, liberando **1 de cada 2**: ~**536 KiB livres** (buracos de 1 KiB). Pedir `malloc(100 KiB)` **fez a *heap* crescer 196 KiB**: os buracos não serviam. Com tudo liberado e contíguo, o mesmo pedido **cresceu 0 KiB** |
| **Interna** | o bloco entregue é **maior que o pedido** | `malloc(25)` devolve **40 bytes** utilizáveis (alinhamento de 16 e metadados, [[Alocação Dinâmica em C]]) |

A fragmentação externa é um problema real em programas de **longa duração** (servidores) e em **microcontroladores** com pouca RAM (por isso o firmware costuma evitar `malloc`). O alocador usa listas de blocos livres e estratégias (primeiro que servir, melhor encaixe) estudadas em [[Gerenciando a Memória Livre]]. Mitigações: **reaproveitar blocos**, alocar objetos de tamanho parecido juntos e liberar **na ordem inversa** da alocação.

---

# 🧭 Quando usar cada uma

| Situação | Use |
|---|---|
| Tamanho **pequeno e conhecido** na compilação; vida curta | **pilha** (variável local, array local) |
| Dado que precisa **sobreviver à função** | ***heap*** (`malloc`) |
| Tamanho **grande** (de centenas de KiB para cima) ou **só conhecido em tempo de execução** | ***heap*** |
| Estrutura que **cresce e encolhe** (lista, vetor) | ***heap*** (`realloc`) |
| Valor global durante todo o programa | **`static`**/global (região de dados) |
| **Muitas** alocações pequenas em laço quente | evite `malloc`: use pilha ou reaproveite blocos |

Em **Go** essa escolha **não é sua**: o compilador decide em tempo de compilação o que vai para a pilha e o que "escapa" para a *heap* (*escape analysis*), e o coletor libera ([[Memória e Escape Analysis]], [[Garbage Collector]]). Em C, **você decide, e o erro é seu**.

---

# 🧯 Erros mais comuns

| Erro | Exemplo | Efeito | Correção |
|---|---|---|---|
| Devolver o endereço de uma local | `int *f(void) { int x; return &x; }` | ponteiro **pendurado** (a pilha foi reaproveitada) | devolver por valor ou usar `malloc` |
| Array local enorme | `int v[10000000];` | `Segmentation fault` (pilha de 8 MiB) | `static`, global ou `malloc` |
| Recursão sem fim ou muito funda | `f` chama `f` sem caso base | *stack overflow* | caso base; iterar ([[Recursão em C]]) |
| Esquecer o `free` | alocar na função e sair | ***leak*** | liberar (ou devolver ao chamador) |
| Achar que `static` local mora na pilha | `static int n` | confunde o tempo de vida | `static` vive o programa todo |
| Alocar muita coisa pequena em laço quente | `malloc` + `free` por iteração | lento (≈ 8 ns contra ≈ 0,3 ns) | reaproveitar um bloco ou usar a pilha |
| Misturar `free` com variáveis da pilha | `free(&x)` | abortado (*invalid pointer*) | só `free` do que veio de `malloc` |

---

# 🧪 Para praticar

1. Escreva um programa com uma variável **global**, uma **local de `main`**, uma **`static` local** e um `malloc`, e **imprima o endereço** de cada uma (`%p`, com `(void *)`). Quais ficam perto de quais? Rode duas vezes (o que muda?).
2. Escreva `int *dobro(int x)` que devolve o dobro **de duas formas**: errada (endereço de uma local) e certa (`malloc`). Compile com `-Wall` e veja o aviso da errada.
3. Compare o tempo de somar 1 milhão de vezes usando um **buffer local** de 64 bytes × um **`malloc`/`free`** de 64 bytes a cada volta (com `clock()`).
4. Declare um `char grande[16 * 1024 * 1024];` **local**, execute e observe o `Segmentation fault`; depois mude para `static` ou `malloc` e confirme que funciona.
5. Para cada dado a seguir, diga se vai para a pilha, a *heap* ou a região de dados: um contador de chamadas que persiste, o resultado de uma conta dentro de uma função, o conteúdo de um arquivo lido por inteiro, uma tabela fixa de nomes de meses.

---

# ✅ Resumo do Conceito

- Regiões reais: **código** (`r-xp`), **literais e `const`** (`r--p`), **globais e `static`** (`.data`/`.bss`, `rw-p`), **heap** (`[heap]`), **bibliotecas** e **pilha** (`[stack]`, endereços altos); a pilha cresce **para baixo** e a *heap* **para cima**
- **Pilha:** automática, LIFO, vida = escopo, **quase gratuita** (≈ 0,3 ns), **~8 MiB**, esgotar = **`Segmentation fault`**
- ***Heap*:** manual (`malloc`/`free`), vida = até o `free`, **mais lenta** (≈ 8 ns), **grande** (512 MiB ok no teste), esgotar = **`NULL`**
- **Ponteiro e apontado** são objetos distintos: `int *p = malloc(...)` tem `p` na **pilha** e o bloco na **heap**; sair sem `free` = ***leak***; devolver endereço de local = **ponteiro pendurado**
- **`static` local** vive na região de **dados**; arrays locais e VLAs, na **pilha**
- **Fragmentação externa** (livre total suficiente, mas em buracos: `malloc(100 KiB)` cresceu a *heap* com 536 KiB livres) e **interna** (`malloc(25)` → 40 bytes); a pilha **não fragmenta**
- Pilha para o **pequeno e curto**; *heap* para o **que sobrevive, é grande ou cresce**; em Go o compilador decide (*escape analysis*)

---

# 🔗 Conexões com outras anotações

- [[Alocação Dinâmica em C]] — `malloc`, `realloc`, `free` e o que o alocador faz por dentro
- [[Funções em C]] e [[Recursão em C]] — *stack frames*, crescimento da pilha e estouro por recursão
- [[Arrays em C]] — array local grande estoura a pilha; VLA
- [[Ponteiros em C]] — endereços virtuais, ASLR e ponteiro pendurado
- [[Funções e Ponteiros em C]] — devolver ponteiro para local × para heap
- [[Variáveis e Constantes em C]] — `.data`, `.bss`, `.rodata` e `static`
- [[Como C Funciona]] — o espaço de endereçamento completo do processo
- [[Espaços de Endereçamento]], [[Memória Virtual]] e [[Gerenciando a Memória Livre]] — o suporte do sistema operacional
- [[Pilha (Stack)]] — a estrutura de dados LIFO
- [[Memória e Escape Analysis]] e [[Garbage Collector]] — o contraste com Go
