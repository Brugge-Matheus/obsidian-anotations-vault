---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.2.3"
---
# Gerenciando a Memória Livre

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.2.3

---

# 🗃️ 3.2.3 — Gerenciando a Memória Livre

## O problema geral

Quando a memória é designada **dinamicamente**, o sistema operacional deve **gerenciá-la**. Em termos gerais, há duas maneiras de se rastrear o uso de memória:

```
1. MAPAS DE BITS
2. LISTAS LIVRES (listas encadeadas)
```

---

## 🗺️ Gerenciamento de Memória com Mapas de Bits

Com um **mapa de bits** (ou *bitmap*), a memória é dividida em **unidades de alocação** tão pequenas quanto umas poucas palavras e tão grandes quanto vários **quilobytes**.

```
Correspondendo a cada unidade de alocação há um
BIT no mapa de bits:

  0 → a unidade está LIVRE
  1 → a unidade está OCUPADA
      (ou vice-versa)
```

> 📌 **Figura 3.6 — (a) Uma parte da memória com cinco processos e três espaços. As marcas indicam as unidades de alocação de memória. As regiões sombreadas (0 no mapa de bits) estão livres. (b) Mapa de bits correspondente. (c) A mesma informação como uma lista.**

```
(a) Parte da memória com processos e espaços:

┌────┬────┬──────┬────┬─────────┬────┬──────┐
│  A │////│   B  │////│    C    │////│   D  │
└────┴────┴──────┴────┴─────────┴────┴──────┘
0    8    16     24   ...

(b) Mapa de bits correspondente:

  11111000
  11111111
  11001111
  11111000

(c) Como uma lista encadeada:

P0,5 → L5,3 → P8,6 → P14,4 → L18,2 → P20,6 → P26,3 → L29,3(X)
```

### O fator crítico — o tamanho da unidade de alocação

```
O TAMANHO da unidade de alocação é um IMPORTANTE
fator de projeto.

QUANTO MENOR a unidade de alocação:
  → MAIOR o mapa de bits

MESMO com uma unidade de alocação tão pequena quanto
4 BYTES:
  → 32 bits de memória exigirão apenas 1 bit do mapa

  → uma memória de 32n bits usará um mapa de n bits

  → o mapa de bits ocupará APENAS 1/32 da memória
```

### O trade-off — unidade grande vs. unidade pequena

```
SE a unidade de alocação for definida GRANDE:
  → o mapa de bits será MENOR

  → MAS uma quantidade CONSIDERÁVEL de memória será
    DESPERDIÇADA na ÚLTIMA unidade do processo, se o
    tamanho dele não for um múltiplo EXATO da
    unidade de alocação
```

### Como o mapa de bits é usado na prática

```
Um mapa de bits proporciona uma maneira SIMPLES de
controlar as palavras na memória, em uma quantidade
FIXA dela, porque seu tamanho depende SOMENTE dos
tamanhos da memória e da unidade de alocação.

O PRINCIPAL PROBLEMA:

  Quando fica decidido carregar um processo com
  tamanho de k unidades, o gerenciador de memória
  deve PROCURAR o mapa de bits para encontrar uma
  SEQUÊNCIA de k bits 0 consecutivos.

  ⚠️ PROCURAR em um mapa de bits por uma sequência
  de um comprimento determinado é uma OPERAÇÃO
  LENTA (pois a sequência pode ULTRAPASSAR limites
  de palavras no mapa) — este é um ARGUMENTO
  CONTRÁRIO aos mapas de bits.
```

---

## 🔗 Gerenciamento de Memória com Listas Encadeadas

Outra maneira de controlar o uso da memória é manter uma **lista encadeada** de segmentos de memória livres e alocados, na qual um **segmento** contém um processo ou é um espaço vazio entre dois processos.

### A estrutura da lista

```
A memória da Figura 3.6(a) é representada na
Figura 3.6(c) como uma lista encadeada de segmentos.

Cada entrada na lista especifica:

  → se é um espaço LIVRE (L) ou ALOCADO a um
    processo (P)

  → o ENDEREÇO no qual se inicia esse segmento

  → o COMPRIMENTO

  → um PONTEIRO para o item SEGUINTE
```

### Vantagem de manter ordenada por endereço

```
Nesse exemplo, a lista de segmentos é mantida
ORDENADA pelos endereços.

Essa ordenação tem a VANTAGEM de que, quando um
processo é TERMINADO ou TRANSFERIDO, atualizar a
lista é algo SIMPLES de se fazer.
```

### As 4 combinações de vizinhos ao terminar um processo

Um processo que termina a sua execução tem **dois vizinhos** (exceto quando ele está no início ou no fim da memória). Eles podem ser tanto **processos** quanto **espaços livres**, levando a **quatro combinações**:

> 📌 **Figura 3.7 — Quatro combinações de vizinhos para o processo que termina, X**

```
Antes de X terminar         Após X terminar

(a)  A  │  X  │  B          A  │/////│  B
     torna-se

(b)  A  │  X  │/////        A  │///////////│
     torna-se

(c)  /////│  X  │  B        /////////│  B
     torna-se

(d)  /////│  X  │/////      /////////////////
     torna-se
```

```
(a) Ambos vizinhos são PROCESSOS:
  → a atualização exige SUBSTITUIR um P por um L
    (uma entrada de processo vira uma de espaço
     livre — nenhuma fusão possível)

(b) e (c) Um vizinho é PROCESSO, outro é ESPAÇO
    LIVRE:
  → DUAS entradas são FUNDIDAS em UMA
  → a lista fica UMA ENTRADA mais CURTA

(d) AMBOS vizinhos são ESPAÇOS LIVRES:
  → TRÊS entradas são FUNDIDAS em uma
  → DOIS itens são REMOVIDOS da lista
```

### A tabela de processos como origem da vaga

```
Como a VAGA da tabela de processos para o que está
sendo CONCLUÍDO geralmente APONTA para a entrada da
LISTA do próprio processo:

  → talvez seja mais CONVENIENTE ter a lista como
    uma lista DUPLAMENTE ENCADEADA, em vez daquela
    com encadeamento SIMPLES da Figura 3.6(c)

  → essa estrutura torna mais FÁCIL encontrar a
    ENTRADA ANTERIOR e ver se a FUSÃO é possível
```

---

## 🔍 Algoritmos de Alocação — Como Escolher um Espaço Livre

Quando processos e espaços livres são mantidos em uma lista **ordenada por endereço**, vários algoritmos podem ser usados para alocar memória a um processo criado (ou um existente em disco/SSD sendo trocado para a memória). Presumimos que o gerenciador de memória sabe quanto memória alocar.

### 1 — First Fit (Primeiro Encaixe)

> 💡 **First fit (primeiro encaixe):** o gerenciador de memória examina uma lista de segmentos até encontrar um espaço livre que tenha tamanho suficiente. O espaço livre é então dividido em duas partes: uma para o processo e outra para a memória não utilizada (exceto no caso estatisticamente improvável de um encaixe exato).

```
First fit é um algoritmo RÁPIDO, pois ele procura
fazer a MENOR busca possível.
```

### 2 — Next Fit (Encaixe Seguinte)

> 💡 **Next fit:** uma pequena variação do first fit. Funciona da mesma maneira que o first fit, exceto por **memorizar a posição** em que se encontra um espaço livre adequado sempre que o encontra. Da vez seguinte que for chamado para encontrar um espaço livre, ele começa procurando na lista do ponto onde havia parado, em vez de sempre do princípio, como faz o first fit.

```
⚠️ Simulações realizadas por Bays (1977) mostram que
o next fit tem um desempenho LIGEIRAMENTE PIOR do
que o first fit.
```

### 3 — Best Fit (Melhor Encaixe)

> 💡 **Best fit:** faz uma busca em TODA a lista, do início ao fim, e escolhe o MENOR espaço livre que seja adequado. Em vez de escolher um espaço livre grande demais que talvez seja necessário mais tarde, o best fit tenta encontrar um que seja de um tamanho PRÓXIMO do tamanho real necessário, para casar da melhor maneira possível a solicitação com os segmentos disponíveis.

**Comparando first fit e best fit com a Figura 3.6:**

```
SE um bloco de tamanho 2 for necessário:

  first fit ALOCARÁ o espaço livre em 5

  best fit  ALOCARÁ em 18
```

**As desvantagens do best fit:**

```
O best fit é MAIS LENTO do que o first fit, pois
ele tem de procurar na lista INTEIRA toda vez que
é chamado.

⚠️ De uma maneira um tanto SURPREENDENTE, ele
TAMBÉM resulta em um DESPERDÍCIO MAIOR de memória
do que o first fit ou o next fit, pois tende a
preencher a memória com segmentos MINÚSCULOS e
INÚTEIS.

O first fit gera espaços livres MAIORES em média.
```

### 4 — Worst Fit (Pior Encaixe)

> 💡 **Worst fit:** para contornar o problema de quebrar um espaço livre em um processo e um trecho livre minúsculo, a solução seria SEMPRE escolher o MAIOR espaço livre disponível, de maneira que o novo segmento livre gerado seja grande o bastante para ser útil.

```
⚠️ NO ENTANTO, simulações demonstraram que o worst
fit TAMPOUCO é uma ótima ideia.
```

### 5 — Quick Fit

> 💡 **Quick fit:** mantém listas em SEPARADO para alguns dos tamanhos mais COMUNS solicitados.

```
EXEMPLO — uma tabela com n entradas:

  1ª entrada: um ponteiro para o início de uma
              lista de espaços livres de 4 KB

  2ª entrada: um ponteiro para uma lista de espaços
              livres de 8 KB

  e assim por diante...

Espaços livres de, digamos, 21 KB:
  → poderiam ser colocados na lista de espaços
    livres de 20 KB
  → ou em uma lista de espaços livres de tamanhos
    ESPECIAIS
```

**Vantagem e desvantagem do quick fit:**

```
COM quick fit:
  → encontrar um espaço livre do tamanho EXIGIDO é
    algo EXTREMAMENTE RÁPIDO

  MAS ele tem as MESMAS desvantagens de TODOS os
  esquemas que ordenam por tamanho do espaço livre:

  → quando um processo TERMINA sua execução ou é
    TRANSFERIDO da memória, descobrir seus
    VIZINHOS para ver se uma FUSÃO é possível é algo
    bastante DISPENDIOSO

  → SE a fusão NÃO for feita:
    → a memória logo se FRAGMENTARÁ em um grande
      número de pequenos segmentos livres nos quais
      NENHUM processo se encaixará
```

---

## ⚡ Otimizações — Listas Separadas para Processos e Espaços

Todos os **quatro algoritmos** (first fit, next fit, best fit, worst fit) podem ser **acelerados** mantendo-se **listas em separado** para os processos e os espaços livres.

```
Dessa maneira, TODOS eles dedicam TODA a sua
ENERGIA para inspecionar espaços LIVRES, NÃO
processos.

⚠️ O PREÇO INEVITÁVEL que é pago por essa
ACELERAÇÃO na alocação é a COMPLEXIDADE e a LENTIDÃO
ADICIONAIS ao DESALOCAR a memória, já que um
segmento LIBERADO precisa ser REMOVIDO da lista de
processos e INSERIDO na lista de espaços livres.
```

### Otimização adicional — ordenar a lista de espaços livres por TAMANHO

```
SE listas DISTINTAS são mantidas para processos e
espaços livres:

  A lista de espaços livres PODE ser mantida
  ORDENADA POR TAMANHO, a fim de tornar o BEST FIT
  MAIS RÁPIDO.

Quando o best fit PROCURA em uma lista de segmentos
de memória livre do MENOR para o MAIOR:

  → tão logo ENCONTRA um que se encaixe, ele SABE
    que esse segmento é o MENOR que funcionará

  → NÃO são necessárias mais BUSCAS, como ocorre
    com o esquema de uma lista ÚNICA

Com uma lista de espaços livres ordenada por
tamanho:
  → o FIRST FIT e o BEST FIT são igualmente rápidos
  → e o NEXT FIT fica SEM SENTIDO
```

### Otimização — armazenar dados diretamente nos espaços livres

```
Quando os espaços livres são mantidos em listas
SEPARADAS dos processos, uma PEQUENA otimização é
possível.

Em vez de ter um conjunto SEPARADO de estruturas de
dados para manter a lista de espaços livres:

  → a INFORMAÇÃO pode ser ARMAZENADA nos PRÓPRIOS
    espaços livres

  → a PRIMEIRA palavra de cada espaço livre pode
    ser seu TAMANHO

  → a SEGUNDA palavra um PONTEIRO para a entrada
    seguinte

  → os NÓS da lista da Figura 3.6(c), que exigem
    TRÊS palavras e um bit (P/L), NÃO são mais
    necessários
```

---

# ✅ Resumo do Conceito

- Há duas formas principais de o SO rastrear memória livre: **mapas de bits** e **listas encadeadas**
- **Mapa de bits:** cada unidade de alocação corresponde a um bit (0=livre, 1=ocupado). Quanto **menor** a unidade, **maior** o mapa; quanto **maior** a unidade, **mais desperdício** na última unidade de um processo. O problema principal é que **buscar** uma sequência de k bits livres consecutivos é uma operação **lenta**
- **Listas encadeadas:** cada nó representa um segmento (processo P ou livre L), com endereço, comprimento e ponteiro. Ao terminar um processo, há **4 combinações de vizinhos possíveis** (Figura 3.7), que podem exigir fusão de 2 ou 3 entradas em uma — uma lista **duplamente encadeada** facilita encontrar a entrada anterior
- **Algoritmos de alocação:** **first fit** (rápido, primeira que serve), **next fit** (continua de onde parou — ligeiramente pior que first fit), **best fit** (busca o menor espaço adequado — mais lento E desperdiça mais memória em fragmentos minúsculos), **worst fit** (sempre escolhe o maior — também não é ótimo), **quick fit** (listas separadas por tamanhos comuns — busca extremamente rápida, mas fusão de vizinhos ao liberar memória é cara, levando a fragmentação)
- **Otimização geral:** manter listas **separadas** para processos e espaços livres acelera a busca por alocação, ao custo de mais complexidade na desalocação. Ordenar a lista de livres **por tamanho** torna first fit e best fit igualmente rápidos, e torna next fit obsoleto
- **Otimização de armazenamento:** os próprios espaços livres podem guardar seu tamanho e o ponteiro seguinte, eliminando a necessidade de estruturas de dados externas

---

## 🔗 Notas Relacionadas

- [[Troca de Processos (Swapping)]] — o contexto em que esses algoritmos de alocação são usados, ao trazer processos de volta do disco
- [[Espaços de Endereçamento e Registradores-Base e Limite]] — os registradores usados para acessar corretamente a memória alocada por esses algoritmos
