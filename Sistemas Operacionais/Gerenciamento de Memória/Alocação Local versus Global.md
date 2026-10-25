---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seções 3.5 e 3.5.1"
---
# Alocação Local versus Global

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seções 3.5 e 3.5.1

---

# ♟️ 3.5 — Questões de Projeto para Sistemas de Paginação

Nas seções anteriores vimos **como a paginação funciona** ([[Paginação]]) e alguns **algoritmos básicos de substituição de página** ([[Resumo dos Algoritmos de Substituição de Páginas]]). Mas conhecer os mecanismos básicos **não é o suficiente**.

> 💡 **A analogia do livro:** é a diferença entre saber como mover a torre, o cavalo, o bispo e as outras peças do xadrez e **ser um bom jogador**. Para projetar um sistema de paginação e fazê-lo **funcionar bem**, é preciso saber muito mais.

As seções seguintes examinam **outras questões** que os projetistas de SO precisam considerar cuidadosamente para obter um **bom desempenho**. A primeira delas é a que foi "varrida para baixo do tapete" até agora: **de quem é a página que será removida?**

---

# 🏘️ 3.5.1 — Políticas de Alocação Local versus Global

## A pergunta que ficou em aberto

Nos algoritmos de escolha da página a ser substituída, havia uma questão importante associada que foi deixada de lado (já levantada em [[Substituição de Páginas e Algoritmo Ótimo]]):

```
COMO A MEMÓRIA DEVE SER ALOCADA ENTRE OS PROCESSOS
CONCORRENTES EM EXECUÇÃO?
```

Em outras palavras: quando ocorre uma falta de página, o algoritmo de substituição procura a vítima **apenas entre as páginas do processo que faltou**, ou entre **todas as páginas na memória**?

## 📌 Figura 3.22 — O exemplo do livro

Três processos (**A**, **B** e **C**) compõem o conjunto de processos executáveis. Cada página tem uma **idade** (quanto menor, menos recentemente foi usada, como nos contadores de [[LRU e Envelhecimento]]).

```
(a) CONFIGURAÇÃO ORIGINAL

   Página   Idade         Página   Idade
   A0        10           B4        5
   A1         7           B5        6
   A2         5           B6       12
   A3         4           C1        3
   A4         6           C2        5
   A5         3           C3        6
   B0         9
   B1         4
   B2         6
   B3         2
```

> ⚠️ Suponha que **o processo A tenha uma falta de página** (precisa de uma página nova, que vamos chamar de **A6**).

### (b) Substituição LOCAL

```
O algoritmo considera APENAS as 6 páginas
atualmente alocadas para A (A0 a A5).

  → a de MENOR valor de idade é A5 (idade 3)
  → A5 é removida e A6 entra no lugar dela

  A0  A1  A2  A3  A4  [A6]      ← A5 saiu
  B0 ... B6  (intactas)
  C1  C2  C3 (intactas)
```

### (c) Substituição GLOBAL

```
O algoritmo considera TODAS as páginas na memória,
sem levar em conta a quem pertencem.

  → a de MENOR valor de idade é B3 (idade 2)
  → B3 é removida (do processo B!) e A6 entra
    no lugar dela

  A0  A1  A2  A3  A4  A5
  B0  B1  B2  [A6]  B4  B5  B6   ← B3 saiu
  C1  C2  C3
```

## As duas definições

| | **Algoritmo LOCAL** (Figura 3.22(b)) | **Algoritmo GLOBAL** (Figura 3.22(c)) |
|---|---|---|
| **Quem pode perder a página?** | só o **próprio processo** que faltou | **qualquer processo** |
| **Ideia de alocação** | a cada processo, uma **fração FIXA** da memória | quadros de página são **alocados dinamicamente** entre os processos executáveis |
| **Nº de quadros de cada processo** | **constante** | **varia com o tempo** |

## Qual funciona melhor?

> ✅ Em geral, **algoritmos globais funcionam melhor**, principalmente quando o tamanho do conjunto de trabalho ([[Conjunto de Trabalho]]) pode **variar muito** ao longo da vida do processo.

```
SE um algoritmo LOCAL é usado E o conjunto de
trabalho CRESCE:
  → resulta em THRASHING (ultrapaginação), MESMO se
    houver um número SUFICIENTE de quadros de página
    disponíveis em outros lugares da memória
    (porque o processo não pode "pegar emprestado")

SE o conjunto de trabalho DIMINUI:
  → os algoritmos locais DESPERDIÇAM memória
    (o processo segura quadros de que não precisa)

SE um algoritmo GLOBAL é usado:
  → o sistema tem de DECIDIR CONTINUAMENTE quantos
    quadros de página designar para cada processo
    (veja [[Alocação de Quadros e PFF]])
```

## Algumas combinações são livres, outras não

### Onde a escolha é independente do algoritmo

```
Alguns algoritmos de substituição funcionam TANTO
com política LOCAL quanto GLOBAL:

  FIFO:
    → global: substitui a página MAIS ANTIGA em TODA a
      memória
    → local:  substitui a página mais antiga POSSUÍDA
      PELO PROCESSO ATUAL

  LRU (ou uma aproximação dele):
    → global: substitui a página MENOS USADA
      RECENTEMENTE em toda a memória
    → local:  a menos usada recentemente que
      PERTENCE ao processo atual

Nesses casos, a escolha de local versus global é
INDEPENDENTE do algoritmo.
```

### Onde só a estratégia local faz sentido

```
Para OUTROS algoritmos, apenas uma estratégia LOCAL
faz sentido.

EM PARTICULAR: o CONJUNTO DE TRABALHO e o WSCLOCK.

  → eles se referem a um PROCESSO ESPECÍFICO e devem
    ser aplicados nesse contexto

  → NÃO HÁ um conjunto de trabalho para a máquina
    como um todo

  → tentar usar a UNIÃO de todos os conjuntos de
    trabalho PERDERIA a propriedade de LOCALIDADE e
    NÃO FUNCIONARIA bem
```

| Algoritmo | Local | Global |
|---|---|---|
| FIFO | ✅ | ✅ |
| LRU / envelhecimento | ✅ | ✅ |
| [[Conjunto de Trabalho]] | ✅ | ❌ (não faz sentido) |
| [[WSClock]] | ✅ | ❌ (não faz sentido) |

> 📝 Como mencionado no resumo, o **Windows** combina estratégias de substituição **local** e **global**, enquanto o **Linux** (2008) separa listas LRU de arquivo e anônimas — veja [[Resumo dos Algoritmos de Substituição de Páginas]].

---

# ✅ Resumo do Conceito

- A **3.5** parte da ideia de que conhecer os mecanismos básicos não basta (xadrez: saber mover as peças ≠ jogar bem); as próximas seções tratam de questões práticas de projeto
- A questão da **3.5.1:** ao substituir uma página, a vítima vem **só do processo que faltou** (**local**) ou de **todos** (**global**)?
- **Figura 3.22:** com A faltando, o **local** remove A5 (menor idade *entre as páginas de A*); o **global** remove B3 (menor idade *de toda a memória*), prejudicando o processo B
- **Local** = fração fixa de memória por processo. **Global** = quadros distribuídos dinamicamente, o número por processo varia com o tempo
- **Global geralmente é melhor**, em especial quando o conjunto de trabalho varia muito: o local sofre **thrashing** se o conjunto cresce (mesmo havendo memória livre) e **desperdiça memória** se ele encolhe
- Algoritmos como **FIFO e LRU** aceitam as duas políticas; o **conjunto de trabalho e o WSClock** só fazem sentido como **locais**, pois o conjunto de trabalho é uma propriedade de cada processo

---

## 🔗 Notas Relacionadas

- [[Alocação de Quadros e PFF]] — como o sistema decide quantos quadros dar a cada processo (continuação da 3.5.1)
- [[Substituição de Páginas e Algoritmo Ótimo]] — onde a pergunta "do próprio processo ou de outro?" foi levantada
- [[Resumo dos Algoritmos de Substituição de Páginas]] — como Windows e Linux combinam local e global
- [[Conjunto de Trabalho]] — a noção de conjunto de trabalho por processo e o thrashing
- [[WSClock]] — algoritmo que só opera de forma local
- [[LRU e Envelhecimento]] — de onde vêm as "idades" da Figura 3.22
- [[Paginação]] — a base do mecanismo
