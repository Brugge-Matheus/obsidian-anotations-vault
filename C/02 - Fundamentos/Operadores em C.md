---
tags:
  - c
  - c/fundamentos
source: "Padrão ISO C (C17/C23: expressões, operadores, precedência, ordem de avaliação); experimentos com gcc 13.3, clang 18 e UBSan em Linux x86-64"
---
# Operadores em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3, `clang` 18 e `-fsanitize=undefined` | Anterior: [[Tipos de Dados em C]] | Próxima: [[Controle de Fluxo em C]] | Compare com [[Operações Aritméticas]] e [[Operadores de Comparação e Lógicos]] (Go)

---

# 🧮 O que é um operador

> 💡 **Operador:** símbolo que executa uma operação sobre **operandos** (os valores sobre os quais ele age). **Expressão** é qualquer combinação de operandos e operadores que **produz um valor** (veja [[Sintaxe, Blocos e Escopo em C]]).

Os operadores se classificam pela **aridade** (quantos operandos recebem):

| Aridade | Operandos | Exemplo |
|---|---|---|
| **Unário** | 1 | `-x`, `!ok`, `i++`, `sizeof x` |
| **Binário** | 2 | `a + b`, `x == y`, `p && q` |
| **Ternário** | 3 | `cond ? a : b` (o único) |

Duas ideias aparecem o tempo todo neste assunto:

- **Operando de tipos diferentes:** C converte os operandos **automaticamente** antes de operar (tipagem fraca, [[O que é C]]). As regras completas estão em [[Conversões de Tipo em C]]; aqui só citamos o efeito.
- ***lvalue*:** uma expressão que **designa um objeto na memória**, como `x` ou `v[2]`. Operadores que **modificam** (`=`, `++`, `+=`) exigem um *lvalue* à esquerda: `5 = x` ou `(a + b)++` não compilam.

---

# ➕ Operadores aritméticos

| Operador | Operação | Exemplo | Resultado |
|---|---|---|---|
| `+` `-` | Soma, subtração (e `+`/`-` unários) | `7 + 2` | `9` |
| `*` | Multiplicação | `7 * 2` | `14` |
| `/` | Divisão | `7 / 2` | **`3`** |
| `%` | **Resto** da divisão (só inteiros) | `7 % 3` | `1` |

## Divisão inteira e resto com negativos

Se **os dois operandos são inteiros**, `/` é divisão **inteira** (descarta a parte fracionária). Desde o C99 o resultado é **truncado em direção a zero**, e o `%` **tem o sinal do dividendo**. Resultados reais:

```
 7 /  2 =  3     -7 /  2 = -3     7 / -2 = -3
 7 %  3 =  1     -7 %  3 = -1     7 % -3 =  1
```

A identidade que sempre vale é `(a / b) * b + a % b == a`: `(-7/2)*2 + (-7%2)` dá `-7`. Para dividir como número real, ao menos um operando precisa ser ponto flutuante (`7.0 / 2` = `3.5`, visto em [[Tipos de Dados em C]]). Para o resto de *floats* use `fmod` de `<math.h>`.

## Divisão por zero e outros limites

| Caso | Resultado |
|---|---|
| Inteiro `/ 0` ou `% 0` | **UB**: no x86 o processador gera uma exceção e o programa morre com *Floating point exception* (código de saída 136); o UBSan diz `division by zero` |
| `INT_MIN / -1` | **UB**: o resultado (+2 147 483 648) não cabe em `int` (UBSan: `cannot be represented in type 'int'`) |
| *Float* `/ 0.0` | **Definido (IEEE 754)**: `1.0/0` dá `inf`, `-1.0/0` dá `-inf` e `0.0/0` dá `nan` (impresso como `-nan` neste x86) |
| `signed` que estoura (`INT_MAX + 1`) | **UB**; o `unsigned` dá a volta (visto em [[Tipos de Dados em C]]) |

---

# 📝 Atribuição e atribuição composta

`=` **atribui** e, por ser **expressão**, tem o valor atribuído: `a = b = c = 5` funciona da **direita para a esquerda** (e `if (x = 5)` é o bug clássico, já tratado em [[Sintaxe, Blocos e Escopo em C]]).

A **atribuição composta** `a op= b` é uma abreviação de `a = a op b` em que **`a` é avaliado uma única vez**:

| Forma | Equivale a | | Forma | Equivale a |
|---|---|---|---|---|
| `x += 5` | `x = x + 5` | | `x <<= 2` | `x = x << 2` |
| `x -= 3` | `x = x - 3` | | `x \|= m` | `x = x \| m` |
| `x *= 2` | `x = x * 2` | | `x &= m` | `x = x & m` |
| `x /= 4` `x %= 4` | divisão e resto | | `x ^= m` | `x = x ^ m` |

A diferença aparece com efeito colateral: em `v[i++] += 5` o `i++` roda **uma vez** (testado: `w[0]=5`, `i=1`); expandido à mão (`v[i++] = v[i++] + 5`) seria um erro.

---

# ⏫ Incremento e decremento: `++` e `--`

Somam ou subtraem 1 de um *lvalue*. A **posição** muda o valor da expressão:

| Forma | Faz | Valor da expressão | Exemplo com `i = 5` |
|---|---|---|---|
| `++i` (**pré**) | incrementa **e depois** usa | valor **novo** | `int r = ++i;` → `r = 6`, `i = 6` |
| `i++` (**pós**) | usa **e depois** incrementa | valor **antigo** | `int r = i++;` → `r = 5`, `i = 6` |

Sozinhos (`i++;`) são equivalentes. Funcionam também em ponteiros (*Aritmética de Ponteiros*, nota futura).

> ⚠️ **Não modifique a mesma variável duas vezes na mesma expressão.** `i = i++ + ++i;` é **comportamento indefinido** (veja *ordem de avaliação* abaixo). O `gcc` avisa (`-Wsequence-point`), o `clang` também (`-Wunsequenced`) e ambos imprimiram `4`, mas o padrão **não garante nada**: outro compilador (ou o mesmo com outra flag) pode dar outro número.

---

# ⚖️ Relacionais e de igualdade

`<`  `>`  `<=`  `>=`  `==`  `!=` comparam e devolvem **`int`**: `1` (verdadeiro) ou `0` (falso), como visto em [[Tipos de Dados em C]].

- **`==` compara, `=` atribui.** Trocar um pelo outro compila, mas não faz o que se quer.
- **Não encadeie comparações.** `1 < n < 3` **não** significa "n entre 1 e 3": é `(1 < n) < 3`, ou seja, `0` ou `1` comparado com `3`, **sempre verdadeiro**. Testado com `n = 5`: resultado `1` (o `gcc` avisa com `-Wparentheses`). O correto é `1 < n && n < 3`.
- **Cuidado com `signed` × `unsigned`:** `-1 < 1u` é **falso**, pois o `-1` é convertido para `unsigned` ([[Conversões de Tipo em C]]).
- **Float:** não use `==` (veja [[Tipos de Dados em C]]).

---

# 🔗 Operadores lógicos: `&&`, `||` e `!`

| Operador | Significado | Observação |
|---|---|---|
| `&&` | E | Verdadeiro se **ambos** forem ≠ 0 |
| `\|\|` | OU | Verdadeiro se **ao menos um** for ≠ 0 |
| `!` | NÃO | `!0` = `1`; `!` de qualquer valor ≠ 0 = `0` |

Aceitam **qualquer valor escalar** (0 é falso, o resto é verdadeiro) e **sempre devolvem `0` ou `1`**. Daí o truque `!!x`, que **normaliza** qualquer valor para `0`/`1` (`!!5` = `1`).

## Curto-circuito (*short-circuit*)

`&&` e `||` avaliam **da esquerda para a direita** e **param assim que o resultado é conhecido**:

- `a && b`: se `a` é falso, `b` **nem é avaliado**;
- `a || b`: se `a` é verdadeiro, `b` **nem é avaliado**.

Teste real com uma função `f` que imprime ao ser chamada:

```
f(0) && f(1)  →  r=0, f chamada 1 vez   (só f(0); f(1) foi pulada)
f(1) || f(2)  →  r=1, f chamada 1 vez   (só f(1); f(2) foi pulada)
f(0) &  f(1)  →  r=0, f chamada 2 vezes  (o & bit a bit NÃO faz curto-circuito)
```

Isso é uma ferramenta de **segurança**: `i < n && v[i] > 0` só lê `v[i]` se `i` for um índice válido. Não ponha **efeitos colaterais importantes** do lado direito (`if (ok && contar++ > 3)`): ele pode não rodar.

> ⚠️ **`&` não é `&&`.** `5 & 3` = `1` (bit a bit), `5 && 3` = `1` (lógico) coincidem por acaso aqui, mas `4 & 2` = `0` enquanto `4 && 2` = `1`.

---

# 🔢 Operadores bit a bit

Operam **bit por bit** sobre inteiros (a representação binária está em [[Bits e Bytes]]). Os operandos sofrem as **promoções inteiras** ([[Conversões de Tipo em C]]).

| Operador | Nome | Regra | `12 op 10` (`1100` e `1010`) |
|---|---|---|---|
| `&` | E | 1 só se **ambos** 1 | `1000` = **8** |
| `\|` | OU | 1 se **algum** 1 | `1110` = **14** |
| `^` | OU exclusivo (XOR) | 1 se forem **diferentes** | `0110` = **6** |
| `~` | NÃO (inverte todos os bits) | `0↔1` | `~12` (em 8 bits) = `11110011` = **243** |
| `<<` | Deslocamento à esquerda | empurra bits, entra `0` | `1 << 3` = **8** |
| `>>` | Deslocamento à direita | empurra bits para baixo | `0xF0 >> 4` = **15** |

`x << n` equivale a **multiplicar por 2ⁿ**, e `x >> n` a **dividir por 2ⁿ** (em valores sem sinal).

## As quatro operações com máscaras

Uma **máscara** é um valor com `1` apenas nos bits de interesse. Com `n` sendo o número do bit (0 = menos significativo):

| Operação | Código | Teste real (`reg` começa em 0) |
|---|---|---|
| **Ligar** (*set*) o bit `n` | `reg \|= (1u << n);` | bits 3 e 0 → `0x09` |
| **Desligar** (*clear*) | `reg &= ~(1u << n);` | bit 3 → `0x01` |
| **Inverter** (*toggle*) | `reg ^= (1u << n);` | bit 5 → `0x21` |
| **Testar** | `(reg >> n) & 1u` | bit 5 → `1`; bit 3 → `0` |

É a base da programação de **registradores de hardware** e de protocolos ([[MOC - Protocolos de Hardware]]; um registrador de 8 bits é um `uint8_t`, como visto em [[Tipos de Dados em C]]). A nota *Campos de Bits e Máscaras* aprofunda isso.

## Deslocamentos: onde mora o perigo

| Caso | Resultado | Teste real (UBSan) |
|---|---|---|
| Quantidade **negativa** ou **≥ largura** do tipo | **UB** | `1 << 32` em `int`: `shift exponent 32 is too large for 32-bit type 'int'` |
| `<<` de valor **negativo** com sinal | **UB** | `-1 << 1`: `left shift of negative value -1` |
| `<<` que não cabe no tipo com sinal | **UB** | `1 << 31` em `int`: `cannot be represented in type 'int'` |
| `>>` de valor **negativo** com sinal | **Definido pela implementação** | `-8 >> 1` = `-4` no `gcc` (preserva o sinal) |

> ✅ **Regra prática:** faça manipulação de bits sempre em **tipos sem sinal** e com literal `u` (`1u << 31` é válido). Veja [[Tipos de Dados em C]] sobre o comportamento do `signed`.

---

# ❓ Operador ternário e operador vírgula

## Ternário `cond ? a : b`

É uma **expressão**: avalia `cond`; se ≠ 0 produz `a`, senão `b` (só **um** dos dois é avaliado).

```c
int m = (3 > 2) ? 10 : 20;        // m = 10
printf("%s\n", n == 1 ? "item" : "itens");
```

Use para escolhas **simples**; aninhar vários (`a ? b : c ? d : e`) prejudica a leitura. Os tipos de `a` e `b` se **unificam** pelas conversões usuais ([[Conversões de Tipo em C]]).

## Vírgula `,`

Avalia o operando da esquerda, **descarta** o valor e produz o da direita: `int v = (1, 2, 3);` vale `3` (o `gcc` avisa `left-hand operand of comma expression has no effect`). Seu uso útil é no `for` com duas variáveis ([[Controle de Fluxo em C]]). **Atenção:** as vírgulas que separam **argumentos** (`f(a, b)`) e **declarações** (`int a, b;`) **não** são o operador vírgula.

## Outros operadores que você já viu (ou verá)

| Operador | Função | Onde |
|---|---|---|
| `sizeof` | tamanho em bytes | [[Tipos de Dados em C]] |
| `(tipo)` | conversão explícita (*cast*) | [[Conversões de Tipo em C]] |
| `&` `*` (unários) | endereço de / valor apontado | *Ponteiros* (nota futura) |
| `[]` `.` `->` `()` | índice, campo, campo via ponteiro, chamada | *Arrays*, *Structs*, *Funções* |

Note que `&` e `*` têm **papel duplo**: `a & b` é E bit a bit (binário), `&a` é endereço (unário); `a * b` multiplica, `*p` acessa o apontado.

---

# 🥇 Precedência e associatividade

Quando há vários operadores na mesma expressão, a **precedência** diz quais ligam **primeiro** (como na matemática: `*` antes de `+`) e a **associatividade** desempata operadores de **mesma** precedência (da esquerda para a direita ou o contrário). Da **mais alta** para a **mais baixa**:

| Nível | Operadores | Associatividade |
|---|---|---|
| 1 | `()` `[]` `.` `->` `x++` `x--` (pós-fixos) | esquerda → direita |
| 2 | `++x` `--x` `+` `-` (unários) `!` `~` `*` `&` `sizeof` `(tipo)` | **direita → esquerda** |
| 3 | `*` `/` `%` | esquerda → direita |
| 4 | `+` `-` | esquerda → direita |
| 5 | `<<` `>>` | esquerda → direita |
| 6 | `<` `<=` `>` `>=` | esquerda → direita |
| 7 | `==` `!=` | esquerda → direita |
| 8 | `&` | esquerda → direita |
| 9 | `^` | esquerda → direita |
| 10 | `\|` | esquerda → direita |
| 11 | `&&` | esquerda → direita |
| 12 | `\|\|` | esquerda → direita |
| 13 | `?:` | **direita → esquerda** |
| 14 | `=` `+=` `-=` `*=` `/=` `%=` `<<=` `>>=` `&=` `^=` `\|=` | **direita → esquerda** |
| 15 | `,` | esquerda → direita |

Não decore a tabela: **memorize o essencial e use parênteses** no resto. As armadilhas reais:

| Expressão | Como é lida | Resultado real |
|---|---|---|
| `flags & 1 == 0` (`flags = 6`) | `flags & (1 == 0)` — `==` liga **mais forte** que `&` | **`0`** (o programador queria `1`) |
| `(flags & 1) == 0` | com parênteses, como se pretendia | **`1`** |
| `1 + 2 << 3` | `(1 + 2) << 3` — `+` liga mais forte que `<<` | **`24`** (e não `17`) |
| `*p++` | `*(p++)`: lê o apontado **e depois** avança `p` | com `p` em `10`: vale `10` e `p` passa a apontar `20` |
| `(*p)++` | incrementa o **valor apontado** | `arr[0]` foi de `10` para `11` |

O `gcc` avisa vários desses casos com `-Wparentheses` (parte do `-Wall`, [[Convenções e Estilo]]). A regra mais valiosa: **bit a bit e comparação sempre entre parênteses** (`(x & m) == 0`).

> 💡 Precedência é só **como a expressão é agrupada**, e **não** a ordem em que as partes são calculadas (próxima seção).

---

# 🧭 Ordem de avaliação e pontos de sequência

Em `f(1) + f(2)`, qual `f` roda primeiro? **O padrão não especifica**: o compilador escolhe. Testes reais (cada `f` imprime ao ser chamada):

| Expressão | `gcc` (`-O0` e `-O2`) | `clang` (`-O0`) |
|---|---|---|
| `printf("%d %d", f(1), f(2))` (argumentos) | `f(2) f(1)` | `f(1) f(2)` |
| `f(3) + f(4)` | `f(3) f(4)` | `f(3) f(4)` |

Os argumentos saíram em **ordens opostas** entre compiladores, e **ambos estão corretos**. Não escreva código que dependa disso.

**Ponto de sequência** (*sequence point*) é um ponto do programa em que todos os efeitos colaterais anteriores **já terminaram**. Há pontos de sequência no fim de cada instrução (`;`), **depois do operando esquerdo** de `&&`, `||`, `?:` e `,`, e na chamada de função (depois de avaliar os argumentos). Entre dois pontos, **modificar o mesmo objeto duas vezes** (ou modificar e também ler, fora do cálculo do novo valor) é **UB**, como em `i = i++ + ++i` e em `v[i] = i++`.

> ✅ **Regra prática:** uma expressão deve ter **no máximo um efeito colateral** (`++`, `=`, chamada que altera estado) por variável. Na dúvida, divida em duas instruções.

---

# ✅ Resumo do Conceito

- **Operador** age sobre **operandos**; **expressão** produz um valor; aridade: unário, binário, ternário; modificar exige *lvalue*
- **`/` entre inteiros** é inteira (trunca para zero) e **`%` tem o sinal do dividendo**; divisão inteira por 0 e `INT_MIN / -1` são **UB**; com *float* dá `inf`/`nan`
- **`a op= b`** avalia `a` **uma vez**; **`++i`** devolve o valor novo, **`i++`** o antigo
- Comparações devolvem **`int` 0/1**; **não encadeie** (`1 < n < 3`); `-1 < 1u` é falso
- **`&&` e `||`** devolvem 0/1 e fazem **curto-circuito**; `&` e `|` não
- **Bit a bit** (`& | ^ ~ << >>`) com **máscaras**: ligar `|= 1u<<n`, desligar `&= ~(1u<<n)`, inverter `^=`, testar `(x>>n)&1u`; faça em **tipos sem sinal**, pois deslocamentos inválidos e `<<` com negativos são **UB**
- **Precedência:** `==` liga mais forte que `&`; `+` mais que `<<`; use **parênteses** nas dúvidas
- **Ordem de avaliação** de operandos e argumentos é **não especificada**; modificar a mesma variável duas vezes sem ponto de sequência é **UB**

---

# 🔗 Conexões com outras anotações

- [[Tipos de Dados em C]] — resultado `int` das comparações, overflow, `sizeof` e tipos sem sinal
- [[Conversões de Tipo em C]] — promoções e conversões que acontecem **antes** de cada operador
- [[Sintaxe, Blocos e Escopo em C]] — expressão × instrução, atribuição como expressão
- [[Controle de Fluxo em C]] — as condições de `if`/`while` são expressões com estes operadores
- [[Convenções e Estilo]] — `-Wparentheses`, `-Wall -Wextra` e parênteses como regra de estilo
- [[Como C Funciona]] e [[O Compilador]] — comportamento indefinido, *sanitizers* e otimização
- [[Bits e Bytes]] — representação binária e complemento de dois
- [[MOC - Protocolos de Hardware]] — máscaras e deslocamentos em registradores e protocolos
- [[Operações Aritméticas]] e [[Operadores de Comparação e Lógicos]] — o equivalente em Go
