---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seções 3.4.6 e 3.4.7"
---
# LRU e Envelhecimento

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seções 3.4.6 e 3.4.7

> 💡 O **LRU** é a ideia teórica (boa aproximação do ótimo, mas caro). O **envelhecimento** é a forma prática de **simulá-lo em software**, passando antes pelo **NFU**.

---

# 🧠 3.4.6 — Páginas Usadas Menos Recentemente (LRU)

## A observação que o motiva

```
Boa aproximação do algoritmo ÓTIMO se baseia em:

  Páginas usadas INTENSAMENTE nas últimas instruções
  → provavelmente serão usadas DE NOVO em breve

  Páginas que NÃO foram usadas há TEMPOS
  → provavelmente seguirão sem uso por um longo tempo
```

Isso sugere um algoritmo realizável: **na falta de página, jogue fora a página que não tem sido usada há mais tempo**. Essa estratégia é a paginação **LRU** (*least recently used*).

> 💡 Compare: o ótimo olha para o **futuro** (irrealizável); o LRU olha para o **passado recente** como previsão do futuro.

## Por que é caro

Na teoria é realizável, mas **não é nem um pouco barato**:

```
Para implementar o LRU POR COMPLETO, é necessário
manter uma LISTA ENCADEADA de TODAS as páginas na
memória:

  FRENTE  ← página usada MAIS recentemente
  TRÁS    ← página usada MENOS recentemente

A DIFICULDADE: a lista precisa ser ATUALIZADA A
CADA REFERÊNCIA DE MEMÓRIA.

  → encontrar a página na lista
  → removê-la
  → movê-la para a FRENTE

é uma operação que demanda MUITO tempo, mesmo em
hardware (supondo que um hardware assim possa ser
construído).
```

## Implementação com hardware especial (contador)

```
Equipar o hardware com um CONTADOR C de 64 bits,
automaticamente INCREMENTADO após CADA instrução.

Cada ENTRADA da tabela de páginas também deve ter um
CAMPO grande o bastante para conter esse contador.

APÓS CADA referência de memória:
  → o valor atual de C é ARMAZENADO na entrada da
    tabela de páginas da página recém-referenciada

QUANDO ocorre uma falta de página:
  → o SO examina TODOS os contadores na tabela de
    páginas para encontrar o MAIS BAIXO
  → essa página é a usada MENOS recentemente
```

---

# 💻 3.4.7 — Simulação do LRU em Software

## Por que simular?

O LRU anterior é realizável em princípio, mas **poucas máquinas** (se é que existe alguma) têm o hardware necessário. É preciso uma solução implementável em **software**.

## Primeira tentativa: NFU (Not Frequently Used)

```
NFU = "não usadas frequentemente"

IMPLEMENTAÇÃO:
  → um CONTADOR DE SOFTWARE associado a cada página,
    inicialmente ZERO

  → a CADA interrupção de relógio, o SO percorre
    TODAS as páginas na memória
  → para cada página, o bit R (0 ou 1) é SOMADO ao
    contador

  → os contadores controlam MAIS OU MENOS quão
    frequentemente cada página foi referenciada

  → em uma falta de página, a de contador MAIS BAIXO
    é escolhida para substituição
```

### O defeito do NFU: "lembra um elefante"

```
O NFU JAMAIS ESQUECE NADA.

EXEMPLO: um compilador de MÚLTIPLOS PASSOS
  → as páginas intensamente usadas no PASSO 1 podem
    ainda ter um contador ALTO nos passos seguintes

  → se o passo 1 tiver o tempo de execução mais
    longo, as páginas dos passos seguintes terão
    contadores MENORES que os do passo 1

  → CONSEQUÊNCIA: o SO removerá as páginas ÚTEIS
    (do passo atual) em vez das que não estão mais
    sendo usadas (do passo 1)
```

## A correção: algoritmo de envelhecimento (aging)

Uma pequena modificação no NFU dá uma **boa simulação do LRU**, em duas partes:

```
1. Os contadores são DESLOCADOS 1 bit à DIREITA
   antes que o bit R seja acrescentado

2. O bit R é adicionado ao bit MAIS À ESQUERDA
   (em vez do mais à direita)
```

> 💡 **Intuição:** o bit mais à esquerda é o **mais significativo**, então uma referência **recente** pesa mais que uma antiga. A cada tique, as referências antigas "envelhecem" andando para a direita até saírem do contador.

### 📌 Figura 3.17 — Exemplo (6 páginas, 5 interrupções de relógio)

Cada coluna mostra os bits R lidos no tique e os contadores resultantes (8 bits):

```
            (a)       (b)       (c)       (d)       (e)
Bits R →   101011    110010    110101    100010    011000
(págs 0-5)

Pág 0    10000000  11000000  11100000  11110000  01111000
Pág 1    00000000  10000000  11000000  01100000  10110000
Pág 2    10000000  01000000  00100000  00010000  10001000
Pág 3    00000000  00000000  10000000  01000000  00100000
Pág 4    10000000  11000000  01100000  10110000  01011000
Pág 5    10000000  01000000  10100000  01010000  00101000
```

```
COMO LER (passo 1, coluna (a)):

  Bits R das páginas 0 a 5: 1, 0, 1, 0, 1, 1
  (páginas 0, 2, 4 e 5 foram referenciadas; 1 e 3 não)

  → cada contador é deslocado e o bit R inserido à
    ESQUERDA:
      pág 0: 10000000   pág 1: 00000000
      pág 2: 10000000   pág 3: 00000000
      pág 4: 10000000   pág 5: 10000000

Nos tiques seguintes o processo se repete:
desloca à direita + insere o novo R à esquerda.
```

```
QUANDO OCORRE UMA FALTA DE PÁGINA:

  → é removida a página cujo contador é o MAIS BAIXO

  → uma página que NÃO foi referenciada por, digamos,
    4 interrupções terá 4 ZEROS no início do seu
    contador (à esquerda) e, portanto, valor MENOR
    que uma página não referenciada por apenas 3
    interrupções
```

## As duas diferenças em relação ao LRU verdadeiro

### 1ª — Não distingue a ordem dentro de um mesmo intervalo

```
Considere as páginas 3 e 5 na Figura 3.17(e):

  Pág 3 = 00100000
  Pág 5 = 00101000

  → NENHUMA foi referenciada nas duas últimas
    interrupções de relógio
  → AMBAS foram referenciadas na interrupção
    anterior a elas

Segundo o LRU, deveríamos escolher uma das duas.
PROBLEMA: não sabemos QUAL foi referenciada por
último dentro daquele intervalo — registrando
apenas 1 bit por intervalo de tempo, PERDEMOS a
capacidade de distinguir a ordem das referências
dentro de um mesmo intervalo.

O que dá para fazer: remover a página 3, pois a
página 5 também foi referenciada duas interrupções
antes — e a 3, não.
```

### 2ª — Horizonte finito

```
Os contadores têm um número FINITO de bits (8 bits
no exemplo), o que LIMITA seu horizonte passado.

SE duas páginas têm valor 0 no contador:
  → só dá para escolher uma ao ACASO
  → provavelmente uma foi referenciada há 9
    intervalos e a outra há 1.000 intervalos, e
    NÃO temos como descobrir isso

NA PRÁTICA, 8 bits geralmente é suficiente:
  → uma interrupção de relógio dura cerca de 20 ms
  → uma página não referenciada em 160 ms
    (8 × 20 ms) provavelmente não é importante

Contadores de 16, 32 ou 64 bits dão mais histórico,
mas custam mais memória. Normalmente 8 bits
funciona muito bem.
```

## Comparação

| | **LRU verdadeiro** | **NFU** | **Envelhecimento** |
|---|---|---|---|
| **Onde roda** | exige hardware (contador de 64 bits ou lista) | software | software |
| **Como conta** | momento exato do último uso | **soma** os bits R | **desloca** e insere R à esquerda |
| **Esquece o passado?** | sim (usa o mais recente) | **não** (acumula tudo) | **sim** (bits antigos saem pela direita) |
| **Resolução temporal** | cada referência | por intervalo de relógio | por intervalo de relógio |

---

# ✅ Resumo do Conceito

- **LRU:** remove a página **não usada há mais tempo**, supondo que o passado recente prevê o futuro próximo. Boa aproximação do ótimo, mas **cara**: exige atualizar uma lista a cada referência de memória, ou um **contador de 64 bits** em hardware com campo correspondente em cada entrada da tabela de páginas
- **NFU:** simulação em software; soma o bit R de cada página a um contador a cada interrupção de relógio. **Nunca esquece nada**, então páginas muito usadas no passado distante continuam com contador alto (exemplo: compilador de múltiplos passos)
- **Envelhecimento (aging):** NFU modificado — **desloca** o contador 1 bit à direita e **insere R à esquerda**. Dá peso maior às referências recentes e deixa as antigas "saírem" do contador
- Difere do LRU em 2 pontos: **não distingue a ordem** das referências dentro de um mesmo intervalo (só 1 bit por intervalo) e tem **horizonte finito** (8 bits ≈ 160 ms de histórico com tique de 20 ms). Na prática, funciona muito bem

---

## 🔗 Notas Relacionadas

- [[Substituição de Páginas e Algoritmo Ótimo]] — o algoritmo ideal que o LRU tenta aproximar
- [[Não Usada Recentemente (NRU)]] — aproximação mais grosseira, em 4 classes
- [[FIFO, Segunda Chance e Relógio]] — usam o bit R mas não acumulam histórico
- [[Conjunto de Trabalho]] — outra forma de medir "quais páginas estão sendo usadas"
- [[Tabelas de Páginas]] — onde ficam o bit R e o campo para o contador
