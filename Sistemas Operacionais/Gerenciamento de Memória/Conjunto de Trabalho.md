---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.4.8"
---
# Conjunto de Trabalho

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.4.8

---

# 🎯 3.4.8 — O Algoritmo de Substituição de Páginas do Conjunto de Trabalho

## Paginação por demanda

Na forma **mais pura** de paginação, os processos são inicializados **sem nenhuma de suas páginas na memória**.

```
1. A CPU tenta buscar a PRIMEIRA instrução
   → falta de página → o SO traz a página com a
     primeira instrução

2. Outras faltas (variáveis globais, pilha)
   geralmente ocorrem logo em seguida

3. Após um tempo, o processo tem a MAIOR PARTE das
   páginas de que precisa e executa com poucas
   faltas
```

> 💡 Essa estratégia se chama **paginação por demanda**: as páginas são carregadas **apenas sob demanda**, não antecipadamente.

## Localidade de referência

É fácil escrever um programa que leia sistematicamente todas as páginas de um grande espaço de endereçamento, causando tantas faltas que não haveria memória para conter todas. **Felizmente, a maioria dos processos não funciona assim.**

```
LOCALIDADE DE REFERÊNCIA:

  Durante QUALQUER FASE de execução, o processo
  referencia apenas uma FRAÇÃO relativamente
  PEQUENA das suas páginas.

  Exemplo: cada passo de um compilador de múltiplos
  passos referencia apenas uma fração das páginas, e
  a cada passo essa fração é DIFERENTE.
```

## Conjunto de trabalho e ultrapaginação

```
CONJUNTO DE TRABALHO:
  as páginas que um processo está ATUALMENTE usando.

SE o conjunto de trabalho inteiro ESTÁ na memória:
  → o processo executa SEM causar muitas faltas até
    passar para outra fase (p. ex. o próximo passo
    do compilador)

SE a memória disponível é PEQUENA DEMAIS para o
conjunto de trabalho:
  → o processo causará MUITAS faltas de página e será
    executado LENTAMENTE
  → executar uma instrução leva alguns nanossegundos;
    ler uma página do disco leva cerca de 10 ms
  → a um ritmo de uma ou duas instruções por 10 ms,
    seria necessária uma ETERNIDADE para terminar
```

> ⚠️ Um programa que causa faltas de página a todo momento está **ultrapaginando** (***thrashing***) (Denning, 1968b).

## O modelo do conjunto de trabalho e a pré-paginação

Em multiprogramação, processos são movidos para o disco (todas as suas páginas removidas) para dar vez a outros. Quando um processo **volta**, tecnicamente nada precisa ser feito: ele causa faltas até carregar seu conjunto de trabalho. O problema:

```
Ter INÚMERAS faltas de página toda vez que um
processo é carregado é LENTO e DESPERDIÇA tempo de
CPU, pois o SO leva alguns MILISSEGUNDOS de CPU para
processar cada falta.
```

Por isso muitos sistemas tentam **controlar o conjunto de trabalho** de cada processo e garantir que ele está na memória **antes** de deixar o processo executar:

- Essa abordagem é o **modelo do conjunto de trabalho** (Denning, 1970), projetado para reduzir substancialmente a taxa de faltas
- Carregar as páginas **antes** de deixar o processo executar também é chamado de **pré-paginação**
- O conjunto de trabalho **muda com o passar do tempo**

## Definição formal: w(k, t)

```
Programas raramente referenciam seu espaço de
endereçamento de modo UNIFORME: as referências
tendem a se AGRUPAR em um pequeno número de páginas.

Em qualquer instante t, existe um conjunto formado
por TODAS as páginas usadas pelas k referências de
memória MAIS RECENTES.

  → esse conjunto é  w(k, t)  = o CONJUNTO DE TRABALHO

  → w(k, t) é uma função monotonicamente NÃO
    DECRESCENTE de k (aumentar k nunca diminui o
    conjunto)

  → à medida que k cresce, o limite de w(k, t) é
    FINITO, pois o programa não pode referenciar mais
    páginas do que seu espaço de endereçamento contém
```

### 📌 Figura 3.18 — Tamanho do conjunto de trabalho em função de k

```
 w(k,t)
   │          ___________________________
   │       .-'
   │     .'
   │   .'
   │  /
   │ /
   │/
   └───────────────────────────────────── k
```

```
POR QUE A CURVA TEM ESSE FORMATO:

  → CRESCIMENTO RÁPIDO no início: poucas referências
    já cobrem as páginas realmente usadas

  → CRESCIMENTO MUITO MAIS LENTO para k maior: o
    conjunto de páginas usadas muda lentamente

EXEMPLO: um programa em um laço ocupando 2 páginas e
acessando dados de 4 páginas pode referenciar as 6
páginas a cada 1.000 instruções, mas a referência
mais recente a alguma OUTRA página pode ter sido há
um milhão de instruções, na fase de inicialização.

  → o conteúdo do conjunto de trabalho NÃO É SENSÍVEL
    ao valor de k escolhido: existe uma gama de
    valores de k para os quais o conjunto não muda
```

> 💡 Como o conjunto de trabalho varia lentamente, é possível estimar quais páginas serão necessárias quando o programa for reiniciado, com base no conjunto de trabalho de quando foi parado pela última vez. **A pré-paginação consiste em carregar essas páginas antes de retomar o processo.**

## Do modelo ao algoritmo de substituição

Ter o conjunto de trabalho leva imediatamente a um algoritmo possível:

```
Em uma FALTA DE PÁGINA:
  → encontrar uma página que NÃO esteja no conjunto
    de trabalho
  → removê-la
```

Para implementar, é preciso determinar com precisão **quais páginas estão no conjunto de trabalho**. Por definição, é o conjunto de páginas usadas nas **k referências mais recentes** (o valor de k é escolhido antecipadamente).

### Por que não se faz "do jeito exato"

```
Seria possível imaginar um REGISTRADOR DE
DESLOCAMENTO de comprimento k:

  → a cada referência de memória, desloca 1 posição
    à esquerda e insere à direita o número da página
    referenciada mais recentemente
  → o conjunto dos k números de página no registrador
    seria o conjunto de trabalho

MAS manter e processar esse registrador a cada falta
tem CUSTO PROIBITIVO → essa técnica NUNCA é usada.
```

### Aproximação: usar tempo de execução em vez de contar referências

```
Em vez de "as páginas usadas nas últimas 10 milhões
de referências", definir:

  conjunto de trabalho = as páginas usadas durante os
  últimos 100 ms do TEMPO DE EXECUÇÃO

Na prática, é tão bom quanto e muito mais fácil de
usar.

Para cada processo, só importa seu PRÓPRIO tempo de
execução:
  Se um processo começa no tempo T e teve 40 ms de
  CPU no tempo real T + 100 ms, para fins de
  conjunto de trabalho seu tempo é 40 ms.

  → a quantidade de tempo de CPU que um processo
    realmente usou desde que foi inicializado é seu
    TEMPO VIRTUAL ATUAL

Com essa aproximação, o conjunto de trabalho de um
processo é o conjunto de páginas que ele referenciou
durante os últimos τ SEGUNDOS de TEMPO VIRTUAL.
```

## 📌 O algoritmo (Figura 3.19)

A ideia básica: **encontrar uma página que não esteja no conjunto de trabalho e removê-la**. Só as páginas **na memória** são candidatas; as ausentes são ignoradas.

**Tabela de páginas (trecho):** cada entrada contém ao menos **(1)** o tempo (aproximado) em que a página foi usada pela última vez (*Instante do último uso*) e **(2)** o bit **R**. Os outros campos (número do quadro, proteção, M) não são necessários para este algoritmo.

```
Tempo virtual atual: 2204

 Instante do último uso │ R
 ───────────────────────┼───
         2084           │ 1
         2003           │ 1
         1980           │ 1
         1213           │ 0   ← não referenciada nesta
         2014           │ 1     interrupção de relógio
         2020           │ 1
         2032           │ 1
         1620           │ 0   ← não referenciada nesta
                                interrupção de relógio

 Varrer TODAS as páginas examinando o bit R:
   se (R == 1)
       → estabelecer Instante do último uso para o
         tempo virtual atual
   se (R == 0 e idade > τ)
       → REMOVER esta página
   se (R == 0 e idade ≤ τ)
       → lembrar o MENOR tempo
```

### Passo a passo

```
PRESSUPOSTOS:
  → o hardware inicializa os bits R e M
  → uma interrupção periódica de relógio ativa
    software que LIMPA o bit R a cada interrupção

A cada FALTA DE PÁGINA, a tabela de páginas é VARRIDA
atrás de uma página adequada para remover.

À medida que cada entrada é processada, o bit R é
examinado:

 CASO 1 — R = 1:
   → o tempo virtual atual é escrito no campo
     "Instante do último uso"
   → a página foi usada no momento da falta, está
     claramente no conjunto de trabalho e NÃO é
     candidata à remoção
   (supõe-se que τ se estenda por múltiplas
    interrupções de relógio)

 CASO 2 — R = 0 (não referenciada na interrupção atual):
   → IDADE = tempo virtual atual − Instante do último
     uso, comparada com τ

   2a. IDADE > τ:
       → a página NÃO está mais no conjunto de
         trabalho
       → ela é REMOVIDA e a nova a substitui
       → a varredura continua atualizando as
         entradas restantes

   2b. IDADE ≤ τ:
       → a página AINDA está no conjunto de trabalho
       → é temporariamente POUPADA
       → mas a página com a MAIOR idade (menor valor
         de "Instante do último uso") é MARCADA
```

### Se a varredura não encontra candidata

```
SE a tabela inteira foi varrida SEM encontrar
candidata, TODAS as páginas estão no conjunto de
trabalho. Nesse caso:

  → SE uma ou mais páginas com R = 0 foram
    encontradas: a que tiver a MAIOR IDADE é removida

  → PIOR HIPÓTESE: todas as páginas foram referenciadas
    durante a interrupção de relógio atual (todas com
    R = 1) → uma é escolhida AO ACASO para remoção,
    de preferência uma página LIMPA, se houver
```

---

# ✅ Resumo do Conceito

- **Paginação por demanda:** o processo começa sem páginas na memória e elas são carregadas só quando referenciadas (via falta de página)
- **Localidade de referência:** em qualquer fase, um processo usa só uma **fração pequena** de suas páginas
- **Conjunto de trabalho:** conjunto de páginas que o processo está usando agora. Se cabe na memória, poucas faltas; se não cabe, o processo sofre **ultrapaginação (thrashing)** — a CPU espera disco o tempo todo
- **Modelo do conjunto de trabalho:** controlar o conjunto de cada processo e garantir que ele esteja na memória antes de executá-lo (**pré-paginação**)
- **w(k, t):** páginas usadas nas k referências mais recentes no instante t. Cresce rápido no início e depois quase estabiliza; seu conteúdo é pouco sensível ao valor de k
- Na prática, usa-se **tempo virtual** (tempo de CPU do próprio processo) com um limiar **τ**, em vez de contar k referências (o registrador de deslocamento seria caro demais)
- **Algoritmo:** a cada falta, varre a tabela. R = 1 → atualiza o instante do último uso. R = 0 com idade > τ → remove. R = 0 com idade ≤ τ → poupa, mas lembra a mais velha. Se nenhuma foi removida, remove a de maior idade (ou uma aleatória, de preferência limpa, se todas têm R = 1)

---

## 🔗 Notas Relacionadas

- [[Substituição de Páginas e Algoritmo Ótimo]] — o problema geral da substituição
- [[LRU e Envelhecimento]] — outra abordagem para estimar o "uso recente" das páginas
- [[Não Usada Recentemente (NRU)]] — também depende da limpeza periódica do bit R
- [[Acelerando a Paginação]] — o custo de uma falta de página (milissegundos de CPU) que torna a ultrapaginação tão ruim
- [[Troca de Processos (Swapping)]] — processos movidos para o disco e trazidos de volta (onde a pré-paginação ajuda)
- [[Paginação]] — a base: páginas, quadros e faltas de página
- [[WSClock]] — versão melhorada deste algoritmo, baseada no relógio (3.4.9)
