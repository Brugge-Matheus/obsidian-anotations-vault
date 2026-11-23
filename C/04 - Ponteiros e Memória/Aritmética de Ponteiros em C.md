---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: aritmética e comparação de ponteiros, ptrdiff_t); experimentos com gcc 13.3 e AddressSanitizer em Linux x86-64"
---
# Aritmética de Ponteiros em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 e `-fsanitize=address` | Anterior: [[Ponteiros em C]] | Próxima: [[Arrays e Ponteiros em C]] | Compare com [[Ponteiros]] (Go, que **não** tem aritmética de ponteiros)

> 🧭 **Como ler esta nota:** a ideia central cabe numa frase (**somar 1 a um ponteiro anda 1 *elemento*, não 1 byte**). O restante mostra o que se pode fazer com isso: operações permitidas, o limite do "um além do fim", a pegadinha de `*p++`, percursos clássicos (busca, cópia, inversão) e os erros comuns.

---

# ➕ O que é aritmética de ponteiros

Já sabemos que um ponteiro guarda um endereço ([[Ponteiros em C]]) e que os elementos de um array ficam **lado a lado** ([[Arrays em C]]). A **aritmética de ponteiros** é o conjunto de operações que permite **andar** por essa memória: avançar para o **próximo elemento**, voltar, medir distâncias e comparar posições.

> 💡 **A regra central:** somar `n` a um ponteiro **não soma `n` bytes**; soma **`n` elementos**, ou seja, `n × sizeof(tipo apontado)` bytes. É por isso que o **tipo** do ponteiro importa tanto.

Resultado real com `p` apontando para o início de cada array:

| Ponteiro | `p + 1` avança | Porque |
|---|---|---|
| `int *` | **4 bytes** | `sizeof(int)` = 4 |
| `char *` | **1 byte** | `sizeof(char)` = 1 |
| `double *` | **8 bytes** | `sizeof(double)` = 8 |

Assim, com `int v[5] = {10, 20, 30, 40, 50}` e `int *pi = v;`, o compilador traduz `pi + 1` em "endereço de `pi` + 4" e `*(pi + 1)` vale **20**; `*(pi + 4)` vale 50. A fórmula geral:

```
endereço de (p + n)  =  endereço de p  +  n × sizeof(*p)
```

É exatamente a conta que o compilador faz para acessar `v[n]` (a ligação completa está em [[Arrays e Ponteiros em C]]).

## Vendo os endereços de verdade

Para `int v[5]` e `p = v`, imprimir `(void *)(v + i)` mostra os endereços avançando **de 4 em 4**, e a distância em bytes até `p`:

```
v + 0 = ...fba0    v + 1 = ...fba4    v + 2 = ...fba8    v + 3 = ...fbac    v + 4 = ...fbb0
distância em bytes até p:   0              4                  8                 12                16
```

Em hexadecimal, `a0 → a4 → a8 → ac → b0`: cada passo soma `4` (que é `sizeof(int)`). Se `v` fosse um array de `double`, os passos seriam de **8** em **8**. O compilador faz essa multiplicação sozinho: você escreve `p + 1` e ele entende "o próximo elemento".

---

# 🧮 As operações permitidas

| Operação | Resultado | Exemplo |
|---|---|---|
| `p + n`, `n + p` | ponteiro `n` elementos **adiante** | `pi + 2` aponta para `30` |
| `p - n` | ponteiro `n` elementos **atrás** | `pi - 1` |
| `p++`, `p--`, `p += n`, `p -= n` | **move** o próprio `p` | `p++` passa ao próximo |
| `p2 - p1` | **distância em elementos** (`ptrdiff_t`) | `&v[3] - &v[0]` → **3** |
| `p1 < p2`, `==`, `!=`... | **compara posições** | `ini < fim` |

O teste real, com `p = v` e `int` de 4 bytes: `p++` → `20`; `p += 2` → `40`; `p--` → `30`. E `int a = *q++;` (lê o apontado **e depois** avança `q`, precedência em [[Operadores em C]]) deu `10`, ficando `q` em `20`.

**O que não existe:** somar dois ponteiros, multiplicá-los ou dividi-los não tem sentido, e o compilador recusa:

```
int *r = p + q;    // error: invalid operands to binary + (have 'int *' and 'int *')
int *s = p * 2;    // error: invalid operands to binary * (have 'int *' and 'int')
```

## Subtração: distância em elementos, não em bytes

```c
int *ini = &v[0], *fim = &v[3];
fim - ini                       // 3      (elementos, tipo ptrdiff_t)
(char *)fim - (char *)ini       // 12     (bytes: 3 × 4)
```

O resultado tem o tipo **`ptrdiff_t`** (com sinal, 8 bytes aqui; formato `%td`, [[Tipos de Dados em C]]). Para obter **bytes**, converta os ponteiros para `char *` antes de subtrair.

---

# 🧩 A pegadinha: `*p++`, `*++p`, `(*p)++`, `++*p`

Os operadores `*` (desreferência) e `++` se combinam de formas que parecem iguais e **fazem coisas diferentes**, porque o `++` tem **precedência maior** que o `*` ([[Operadores em C]]). Com `int v[5] = {10, 20, 30, 40, 50}` e `p = v` antes de cada linha (resultados reais):

| Expressão      | Lê-se    | O que faz                                                      | Resultado                             |
| -------------- | -------- | -------------------------------------------------------------- | ------------------------------------- |
| `a = *p++`     | `*(p++)` | **lê** o apontado (`10`) **e depois** avança `p`               | `a = 10`, `*p` agora é `20`           |
| `b = *++p`     | `*(++p)` | **avança** `p` **e depois** lê                                 | `b = 20`, `*p` é `20`                 |
| `c = (*p)++`   |          | lê o dado (`10`), **soma 1 no dado** (o ponteiro **não** anda) | `c = 10`, `v[0]` virou `11`, `p == v` |
| `e = ++*p`     | `++(*p)` | soma 1 no dado **e depois** lê                                 | `e = 11`, `v[0] = 11`                 |
| `d = *(p + 1)` |          | lê o vizinho **sem mover** `p`                                 | `d = 20`, `p == v`                    |

A diferença-chave é **quem recebe o `++`**: o **ponteiro** (`p++`, anda de posição) ou o **dado** (`(*p)++`, altera o valor). O uso mais comum é `*p++`, para ler e já andar, como em laços de cópia (abaixo). Em caso de dúvida, use parênteses ou separe em duas linhas.

---

# 🚧 Limites: a regra do "um além do fim"

A aritmética só é **definida** enquanto o ponteiro permanece **dentro do mesmo array**, com **uma exceção**: o ponteiro pode apontar para **uma posição depois do último elemento** (*one-past-the-end*).

```
   v[0]  v[1]  v[2]  v[3]  v[4]  │ (além do fim)
  ┌─────┬─────┬─────┬─────┬─────┐
  │ 10  │ 20  │ 30  │ 40  │ 50  │
  └─────┴─────┴─────┴─────┴─────┘
   ▲                             ▲
   v                           v + 5   ← pode ser FORMADO e COMPARADO, nunca DESREFERENCIADO
```

| Ação | Situação |
|---|---|
| Ponteiro de `v` até `v + 5` (para `int v[5]`) | **válido** |
| **Desreferenciar** `*(v + 5)` | **UB** (o ASan acusa `stack-buffer-overflow ... READ of size 4`) |
| Formar `v + 6` ou `v - 1` | **UB**, mesmo sem desreferenciar |
| Subtrair/comparar ponteiros de **arrays diferentes** | **UB** |

Essa exceção existe para tornar o **laço com ponteiros** natural: o ponteiro de parada é o "fim" que nunca é lido. Testado, `v + 5 - v` deu **5**.

> ⚠️ Os mesmos perigos de [[Arrays em C]] valem aqui: C **não verifica** nada. Ultrapassar os limites com ponteiros é a origem do *buffer overflow* ([[Erros Clássicos de Memória em C]]).

---

# 🔄 Percorrendo memória com ponteiros

Duas formas de somar os elementos, equivalentes:

```c
// por índice
for (size_t i = 0; i < 5; i++) soma += v[i];

// por ponteiro
for (int *it = v; it < v + 5; it++) soma += *it;      // soma = 150
```

A segunda usa a regra do "um além do fim" (`v + 5` é o limite) e é o estilo de muita biblioteca C. Como as duas geram código parecido, **escolha a mais legível**; em geral, índices para percursos simples e ponteiros quando há **dois cursores** ou **ponteiro que anda sem índice**.

## Exemplo: inverter um array no lugar (dois ponteiros)

```c
void inverte(int *ini, int *fim) {
    while (ini < fim) {
        int t = *ini;  *ini = *fim;  *fim = t;
        ini++;  fim--;
    }
}
// v = {10,20,30,40,50};  inverte(v, v + 4);  →  50 40 30 20 10
```

`ini` anda para frente, `fim` para trás, e a **comparação de ponteiros** (`ini < fim`) decide quando se cruzam.

## Exemplo: o tamanho de uma sequência terminada por zero

```c
const char *s = "linguagem";
const char *e = s;
while (*e) e++;               // avança até o byte zero (o '\0' final)
// e - s  →  9
```

É a lógica do `strlen` ([[Tipos de Dados em C]] mostra o `'\0'` em `char s[] = "oi"`; o assunto completo está em [[Strings em C]]).

---

# 🧰 Mais padrões clássicos

## Buscar e devolver um ponteiro

Uma função pode devolver **um ponteiro para o elemento achado** (ou `NULL` se não achar). A **posição** sai da subtração:

```c
const int *busca(const int *ini, const int *fim, int alvo) {
    for (const int *p = ini; p < fim; p++)
        if (*p == alvo) return p;                // achou: devolve o endereço do elemento
    return NULL;                                  // não achou
}

const int *r = busca(v, v + 5, 40);
// r != NULL  →  índice = r - v = 3;     busca(v, v + 5, 99) devolve NULL
```

Note a assinatura `(inicio, fim)`: ela **não precisa de `n`** e é o desenho da biblioteca padrão de C++. O `const` diz que a busca só lê o array.

## Copiar um texto com dois ponteiros

```c
char *copia(char *d, const char *s) {
    char *ini = d;
    while ((*d++ = *s++) != '\0') ;              // copia o byte, avança os dois, para após copiar o '\0'
    return ini;
}
// char dst[16]; copia(dst, "ola");  →  dst = "ola"
```

Em cada volta: `*s++` lê um byte e avança `s`; `*d++ = ...` grava e avança `d`; o valor da atribuição é o byte copiado, e o laço acaba **depois** de copiar o `'\0'`. É uma versão simples do `strcpy` ([[Strings em C]]), e tem o mesmo defeito: **não verifica se `d` tem espaço** (seção seguinte, e [[Erros Clássicos de Memória em C]]).

## Percorrer de trás para frente

```c
for (int *q = v + 4; q >= v; q--) printf("%d ", *q);       // 50 40 30 20 10
```

Funciona porque `q` termina em `v - 1`, que **é formado** quando o laço faz o último `q--` antes de testar `q >= v`: tecnicamente isso é **UB** (antes do início do array), embora normalmente funcione. Um jeito sem esse problema é usar índice com `size_t i = n; i-- > 0` ([[Arrays em C]]).

---

# 📐 Deslocamento em bytes e `void *`

Para andar **byte a byte** num objeto qualquer, converta para `unsigned char *` (cada passo = 1 byte):

```c
unsigned x = 0x04030201;
*((unsigned char *)&x + 2)      // 03  (o 3º byte; ordem little-endian de Ponteiros em C)
```

Já o **`void *`** não tem tamanho de elemento definido, então a aritmética **não faz parte do padrão**. O `gcc` aceita como extensão (tratando-o como `char *`), mas avisa com `-Wpedantic`/`-Wpointer-arith`: *pointer of type 'void \*' used in arithmetic*. Para ser portável, converta antes para `char *`.

---

# 🧯 Erros mais comuns

| Erro | Exemplo | Efeito | Correção |
|---|---|---|---|
| Esperar soma em **bytes** | achar que `p + 1` soma 1 byte num `int *` | acessa o lugar errado | `p + 1` é **um elemento** |
| Andar além do fim | `p = p + 5; *p` (`int v[3]`) | UB (ASan: `stack-buffer-overflow`) | respeitar `v + n` como limite |
| Desreferenciar o "um além do fim" | `*(v + n)` | UB | só **comparar** com `v + n` |
| Subtrair ponteiros de arrays diferentes | `&a[0] - &b[0]` | UB | só dentro do mesmo array |
| Confundir `*p++` com `(*p)++` | | anda o ponteiro em vez de alterar o dado | parênteses |
| Somar ponteiros | `p + q` | erro de compilação | só `p ± inteiro` e `p - p` |
| Usar `void *` com aritmética | `vp + 1` | extensão do `gcc`, não portável | converter para `char *` |
| Perder o ponteiro inicial | `while (*p) p++;` e depois `free(p)` | o `free` recebe o endereço **errado** | guardar `ini = p` antes |

---

# 🧪 Para praticar

1. Com `int v[6] = {5, 3, 8, 1, 9, 2}`, escreva uma função que devolve um **ponteiro para o maior elemento** (usando o padrão `(ini, fim)`) e imprima o **índice** com `r - v`.
2. Escreva `size_t meu_strlen(const char *s)` usando **só ponteiros** (sem índice) e confira com `strlen`.
3. Escreva `void copia_n(int *dst, const int *src, size_t n)` com **dois ponteiros** que andam juntos.
4. Sem executar, diga o resultado de `int v[] = {1,2,3}; int *p = v; int a = *p++ + *p++;` e explique por que **não se deve escrever isso** (ordem de avaliação, [[Operadores em C]]). Depois responda: e `*p++` sozinho?
5. Troque `int` por `double` num programa seu e imprima as distâncias em bytes entre elementos consecutivos. O que mudou?

---

# ✅ Resumo do Conceito

- **Somar `n` a um ponteiro avança `n` elementos** = `n × sizeof(*p)` bytes (`int *` → +4, `char *` → +1, `double *` → +8)
- Permitido: `p ± n`, `p++`/`p--`/`p += n`, **`p2 - p1`** (distância em **elementos**, tipo `ptrdiff_t`) e **comparações**; **não** existe `p + q`, `p * n` etc. (erro de compilação)
- Para obter a distância em **bytes**, converta para `char *` antes de subtrair
- **Só vale dentro do mesmo array**, podendo chegar a **um além do fim** (formar e comparar, nunca desreferenciar); além disso é **UB**
- Dois padrões úteis: **laço com ponteiro de fim** (`it < v + n`) e **dois cursores** (inverter um array)
- **`*p++`** lê e **anda o ponteiro**; **`(*p)++`** altera o **dado**; **`*++p`** anda e lê; **`++*p`** altera o dado e lê; busca com `(ini, fim)` devolve ponteiro ou `NULL` (posição = `r - v`); `while ((*d++ = *s++))` copia textos
- `void *` não tem aritmética no padrão; use `char *` para deslocar em bytes

---

# 🔗 Conexões com outras anotações

- [[Ponteiros em C]] — `&`, `*`, `NULL`, `const` e o tipo do ponteiro
- [[Arrays em C]] — memória contígua e limites dos índices
- [[Arrays e Ponteiros em C]] — `v[i]` equivale a `*(v + i)`
- [[Tipos de Dados em C]] — `ptrdiff_t`, `size_t` e `sizeof`
- [[Operadores em C]] — precedência de `*p++` e `++` em ponteiros
- [[Controle de Fluxo em C]] — laços e o limite *off-by-one*
- [[Como C Funciona]] — comportamento indefinido e regiões de memória
- [[Ponteiros]] — o equivalente em Go (sem aritmética)
