---
tags:
  - c
  - c/memoria
source: "Padrão ISO C (C17/C23: aritmética e comparação de ponteiros, ptrdiff_t); experimentos com gcc 13.3 e AddressSanitizer em Linux x86-64"
---
# Aritmética de Ponteiros em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 e `-fsanitize=address` | Anterior: [[Ponteiros em C]] | Próxima: [[Arrays e Ponteiros em C]] | Compare com [[Ponteiros]] (Go, que **não** tem aritmética de ponteiros)

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

> ⚠️ Os mesmos perigos de [[Arrays em C]] valem aqui: C **não verifica** nada. Ultrapassar os limites com ponteiros é a origem do *buffer overflow* (*Erros Clássicos de Memória*).

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

# 📐 Deslocamento em bytes e `void *`

Para andar **byte a byte** num objeto qualquer, converta para `unsigned char *` (cada passo = 1 byte):

```c
unsigned x = 0x04030201;
*((unsigned char *)&x + 2)      // 03  (o 3º byte; ordem little-endian de Ponteiros em C)
```

Já o **`void *`** não tem tamanho de elemento definido, então a aritmética **não faz parte do padrão**. O `gcc` aceita como extensão (tratando-o como `char *`), mas avisa com `-Wpedantic`/`-Wpointer-arith`: *pointer of type 'void \*' used in arithmetic*. Para ser portável, converta antes para `char *`.

---

# ✅ Resumo do Conceito

- **Somar `n` a um ponteiro avança `n` elementos** = `n × sizeof(*p)` bytes (`int *` → +4, `char *` → +1, `double *` → +8)
- Permitido: `p ± n`, `p++`/`p--`/`p += n`, **`p2 - p1`** (distância em **elementos**, tipo `ptrdiff_t`) e **comparações**; **não** existe `p + q`, `p * n` etc. (erro de compilação)
- Para obter a distância em **bytes**, converta para `char *` antes de subtrair
- **Só vale dentro do mesmo array**, podendo chegar a **um além do fim** (formar e comparar, nunca desreferenciar); além disso é **UB**
- Dois padrões úteis: **laço com ponteiro de fim** (`it < v + n`) e **dois cursores** (inverter um array)
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
