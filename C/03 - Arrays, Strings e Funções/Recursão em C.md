---
tags:
  - c
  - c/funcoes
source: "Padrão ISO C (C17/C23: chamadas de função e recursão); experimentos com gcc 13.3 (-O0 e -O2) em Linux x86-64"
---
# Recursão em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 em `-O0` e `-O2` | Anterior: [[Funções em C]] | Próxima: [[Funções Variádicas em C]] | Veja também [[Big O Notation]] e [[Complexidade de Algoritmos]]

---

# 🔁 O que é recursão

> 💡 **Recursão:** uma função que **chama a si mesma**, resolvendo um problema em termos de uma **versão menor do mesmo problema**.

Toda função recursiva tem **duas partes obrigatórias**:

| Parte | Papel | No fatorial `n! = n × (n-1)!` |
|---|---|---|
| **Caso base** | a situação **mais simples**, resolvida direto, que **encerra** a recursão | `0! = 1` (e `1! = 1`) |
| **Caso recursivo** | reduz o problema e **chama a si mesma** com um valor **mais próximo do caso base** | `n! = n × fat(n - 1)` |

```c
unsigned long long fat(unsigned n) {
    if (n <= 1) return 1;            // caso base: para aqui
    return n * fat(n - 1);           // caso recursivo: n diminui a cada chamada
}
// fat(0..5) → 1 1 2 6 24 120      fat(20) → 2432902008176640000
```

**Sem caso base, ou sem avançar em direção a ele, a recursão nunca para** (veja *Estouro de pilha*). Em C não há nada de especial na sintaxe: é apenas uma função cujo corpo contém uma chamada a si mesma.

---

# 📚 Como funciona: a pilha de chamadas

Lembrando [[Funções em C]]: **cada chamada cria um novo *stack frame*** com seus próprios parâmetros e variáveis locais. Na recursão, várias chamadas da **mesma função** ficam **empilhadas ao mesmo tempo**, cada uma com seu `n`:

```
fat(4)  espera ┐
  fat(3)  espera │   as chamadas EMPILHAM até o caso base
    fat(2)  espera │
      fat(1) → 1   ← caso base: começa a DESEMPILHAR
    fat(2) = 2 × 1 = 2
  fat(3) = 3 × 2 = 6
fat(4) = 4 × 6 = 24
```

Teste real do tamanho de cada quadro: numa função recursiva com uma variável local, a diferença de endereço entre dois níveis foi de **48 bytes** (com `-O0`), sempre para um endereço **menor** (a pilha cresce para baixo). **Profundidade × tamanho do quadro = memória de pilha usada.**

## Tipos de recursão

| Tipo | Descrição | Exemplo |
|---|---|---|
| **Direta** | a função chama a si mesma | `fat`, `fib` |
| **Indireta (mútua)** | `A` chama `B`, que chama `A` | `par(n)` ↔ `impar(n)` (precisa de **protótipo** da segunda, [[Funções em C]]) |
| **De cauda** (*tail*) | a chamada recursiva é a **última coisa** feita; nada sobra para fazer depois | `mdc(a, b)` abaixo |

Exemplos testados: `mdc(48, 18)` = 6 e `mdc(17, 5)` = 1 (algoritmo de Euclides, recursão de cauda: `return b == 0 ? a : mdc(b, a % b);`); `par(10)` = 1 e `impar(7)` = 1.

---

# 💥 Estouro de pilha (*stack overflow*)

A pilha é **finita**: **8 MiB** por padrão no Linux (`ulimit -s` = 8192, [[Arrays em C]]). Recursão **profunda demais** a esgota e o programa morre com `Segmentation fault` (código 139). Teste real com `soma(n) = n + soma(n - 1)` em `-O0` (quadro de ~32 bytes, teto de ≈ 8 MiB ÷ 32 ≈ 260 mil níveis):

| `n` | Resultado em `-O0` |
|---|---|
| 10 000 | ✅ funciona |
| 200 000 | ✅ funciona |
| **300 000** | ❌ `Segmentation fault` |
| **1 000 000** | ❌ `Segmentation fault` |

Com **`-O2`**, o mesmo `soma(10 000 000)` **funcionou**, pois o `gcc` **transformou a recursão em um laço**. Isso é uma **otimização**, e **não** uma garantia da linguagem:

> ⚠️ **C não garante a eliminação da recursão de cauda** (*tail call optimization*), diferentemente de linguagens funcionais. A versão com **acumulador** `soma_acc(n, acc)` (recursão de cauda) também **estourou com `-O0`** para `n = 10 000 000`. Um programa correto **não pode depender** de uma flag de otimização para não travar. Para profundidade **grande ou desconhecida**, escreva um **laço**.

---

# 🐢 O custo da recursão ingênua

Recursão é elegante, mas pode ser **muito cara** se o mesmo subproblema é recalculado. O exemplo clássico é Fibonacci:

```c
unsigned long long fib(unsigned n) {
    return n < 2 ? n : fib(n - 1) + fib(n - 2);     // duas chamadas por nível
}
```

Contando as chamadas (testado):

| `n` | `fib(n)` | Chamadas à função | Versão com laço |
|---|---|---|---|
| 10 | 55 | **177** | `n` iterações |
| 20 | 6 765 | **21 891** | 20 |
| 30 | 832 040 | **2 692 537** | 30 |

O número de chamadas **cresce exponencialmente** (≈ O(2ⁿ), [[Big O Notation]]) porque `fib(n-2)` é recalculado dentro de `fib(n-1)`, enquanto o laço é **O(n)**. Medido em `-O0`, `fib(40)` recursivo levou **0,70 s** e a versão iterativa, **0,001 s**. (O `-O2` reduz o tempo do recursivo, mas **não muda a complexidade**.)

Duas saídas: **trocar por um laço** ou guardar os resultados já calculados (**memoização**, técnica de *programação dinâmica*).

---

# ⚖️ Recursão × iteração e cuidados com os tipos

Tudo que se faz com recursão se faz com **laço** (e o contrário). Escolha por clareza e risco:

| Prefira **recursão** quando... | Prefira **laço** quando... |
|---|---|
| o problema é **naturalmente recursivo**: árvores, diretórios, *divide e conquista* (ordenações, busca binária) | a profundidade pode ser **grande** ou desconhecida |
| a solução recursiva é **muito mais clara** | o **desempenho** importa (sem custo de chamada e de pilha) |
| a profundidade é **pequena e limitada** (ex.: log₂ n) | existe versão **iterativa simples** (fatorial, Fibonacci, soma) |

Cada chamada tem **custo**: empilhar o quadro, passar argumentos e retornar ([[Funções em C]]).

Por fim, lembre dos **limites dos tipos** ([[Tipos de Dados em C]]): `fat(20)` cabe em `unsigned long long`, mas **`fat(21)` estoura** (deu `14197454024290336768`, resultado errado por volta do módulo 2⁶⁴). Em recursão numérica o **overflow** aparece rápido; com `int` (com sinal) seria **UB**.

---

# ✅ Resumo do Conceito

- **Recursão** = função que chama a si mesma; precisa de **caso base** (para) e **caso recursivo** (avança em direção ao base)
- Cada chamada empilha um ***stack frame***; as chamadas ficam **empilhadas** até o caso base e **desempilham** na volta (quadro de 32–48 bytes nos testes)
- **Tipos:** direta, **indireta/mútua** (exige protótipo) e **de cauda** (chamada recursiva é a última ação)
- **Estouro de pilha:** pilha de **8 MiB**; `soma` recursiva em `-O0` quebrou entre 200 mil e 300 mil níveis (`Segmentation fault`)
- **C não garante** otimização de recursão de cauda: o `-O2` **pode** virar laço, mas não conte com isso
- **Custo:** `fib` ingênuo faz **O(2ⁿ)** chamadas (`fib(30)` = 2,7 milhões); a versão com laço é O(n) — 0,70 s × 0,001 s para `fib(40)` em `-O0`
- Prefira recursão para problemas naturalmente recursivos e de profundidade limitada; **laço** quando a profundidade é grande ou o desempenho importa
- Atenção ao **overflow** do tipo (`21!` estoura 64 bits)

---

# 🔗 Conexões com outras anotações

- [[Funções em C]] — *stack frame*, chamada e retorno, protótipos
- [[Arrays em C]] — o limite de 8 MiB da pilha (também estoura com arrays locais grandes)
- [[Tipos de Dados em C]] — limites de `unsigned long long` e overflow
- [[Controle de Fluxo em C]] — o laço como alternativa à recursão
- [[O Compilador]] — `-O2` e a transformação de recursão em laço
- [[Como C Funciona]] — regiões de memória e a pilha do processo
- [[Big O Notation]] e [[Complexidade de Algoritmos]] — por que O(2ⁿ) × O(n) importa
