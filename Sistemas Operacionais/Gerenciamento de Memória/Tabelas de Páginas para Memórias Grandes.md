---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.3.4"
---
# Tabelas de Páginas para Memórias Grandes

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.3.4

---

# 🗜️ 3.3.4 — Tabelas de Páginas para Memórias Grandes

## O problema que resta

O TLB (visto em [[Acelerando a Paginação]]) resolve a **velocidade** da tradução. Mas ele não resolve o outro problema: o **tamanho** da tabela de páginas quando o espaço de endereçamento virtual é enorme.

```
Lembrete (3.3.1): com endereços de 64 bits e
páginas de 4 KB, uma tabela de páginas LINEAR
teria 2⁵² entradas — e cada processo precisa
da sua PRÓPRIA tabela.
```

O livro apresenta **duas maneiras** de lidar com isso:

```
1. TABELAS DE PÁGINAS MULTINÍVEL
2. TABELAS DE PÁGINAS INVERTIDAS
```

---

# 🌳 Tabelas de Páginas Multinível

## A ideia central

> 💡 **O segredo:** evitar manter **todas** as tabelas de páginas na memória o tempo inteiro. Em especial, as que **não são necessárias** não devem ser mantidas.

Em vez de uma tabela gigante e linear, usamos uma **hierarquia** de tabelas menores — e só criamos as partes da hierarquia que o processo realmente usa.

## 📌 Figura 3.13(a) — Endereço de 32 bits dividido em três campos

```
Bits:    10        10          12
      ┌────────┬────────┬──────────────┐
      │  PT1   │  PT2   │ Deslocamento  │
      └────────┴────────┴──────────────┘
```

```
PT1          → 10 bits: índice na tabela de
               NÍVEL SUPERIOR
               (2¹⁰ = 1.024 entradas)

PT2          → 10 bits: índice em uma tabela de
               SEGUNDO NÍVEL
               (2¹⁰ = 1.024 entradas)

DESLOCAMENTO → 12 bits: posição do byte dentro da
               página (2¹² = 4 KB por página)

Total de páginas: 2¹⁰ × 2¹⁰ = 2²⁰ (mais de 1 milhão)
```

> 💡 Se você não tem certeza de como "bits viram quantidade", releia a lógica: **N bits = 2ᴺ combinações**. Aqui, o endereço de 32 bits é fatiado em três pedaços, cada um servindo de índice em um nível diferente.

## 📌 Figura 3.13(b) — Tabela de páginas de dois níveis

```
 Tabela de nível          Tabelas de segundo nível
    SUPERIOR                  (1.024 entradas cada)
  (1.024 entradas)

 ┌──────────┐
 │  1023    │ ───────────►  [ tabela p/ os 4 MB do
 ├──────────┤                 TOPO da memória → PILHA ]
 │  ...     │
 │ (não     │  ← entradas sombreadas:
 │  usadas) │    NÃO usadas (bit presente/ausente = 0)
 │  ...     │
 ├──────────┤
 │    1     │ ───────────►  [ tabela p/ os 4 MB
 ├──────────┤                 seguintes → DADOS ]
 │    0     │ ───────────►  [ tabela p/ os primeiros
 └──────────┘                 4 MB → CÓDIGO ]
                                    │
                                    ▼
                         cada entrada aponta para um
                         QUADRO DE PÁGINA (a página real)
```

## Como funciona a tradução, passo a passo

```
1. A MMU recebe o endereço virtual de 32 bits

2. Extrai o campo PT1 e usa como ÍNDICE na
   tabela de nível SUPERIOR

   → cada uma das 1.024 entradas representa
     4 MB (2²² bytes), pois o espaço total de
     4 GB (2³²) foi dividido em 1.024 pedaços

3. A entrada encontrada APONTA para (contém o
   número do quadro de) uma tabela de SEGUNDO
   NÍVEL

4. O campo PT2 é usado como índice nessa tabela
   de segundo nível, para encontrar o NÚMERO DO
   QUADRO DE PÁGINA da página em si

5. O número do quadro é combinado com o
   DESLOCAMENTO → ENDEREÇO FÍSICO
```

## Exemplo numérico do livro

Endereço virtual de 32 bits `0x00403004` (4.206.596 em decimal):

```
4.206.596 − 4 MB (4.194.304) = 12.292 bytes
dentro da área de DADOS

Decomposição:
  PT1         = 1   → entrada 1 do nível superior
                      (cobre de 4 MB a 8 MB − 1: DADOS)
  PT2         = 3   → entrada 3 da tabela de segundo
                      nível
                      (cobre os bytes 12.228 a 16.383
                       dentro daquele pedaço de 4 MB)
  DESLOCAMENTO = 4  → byte 4 dentro da página

SE a página NÃO estiver na memória:
  → bit presente/ausente = 0 → FALTA DE PÁGINA

SE estiver presente:
  → o número do quadro (da tabela de 2º nível)
    é combinado com o deslocamento (4) para
    formar o endereço físico, que vai ao
    barramento de memória
```

## 💰 Por que isso economiza memória

Exemplo do livro: um processo que precisa de **12 MB** — 4 MB na base (código), 4 MB seguintes (dados) e 4 MB no topo (pilha). Entre o topo dos dados e a base da pilha existe um **espaço gigante não usado**.

```
Espaço de endereçamento: mais de 1 milhão de
páginas possíveis.

MAS só são necessárias QUATRO tabelas de páginas:

  1 × tabela de nível SUPERIOR
  3 × tabelas de segundo nível
        (código, dados, pilha)

As 1.021 entradas restantes do nível superior
têm bit presente/ausente = 0.
```

```
SE o processo tentar acessar essa região vazia:
  → falta de página
  → o SO percebe que o processo referencia uma
    memória que NÃO deveria
  → toma as medidas cabíveis: enviar um sinal
    ou MATAR o processo
```

> ✅ A economia vem de **não criar** tabelas de segundo nível para regiões do espaço virtual que o processo nunca usa. Uma tabela linear teria que ter entrada para **todas** as páginas, usadas ou não.

> 📝 No exemplo, o livro escolheu números "redondos" e fez PT1 = PT2 (10 e 10 bits), mas na prática outros valores também são possíveis.

## Mais de dois níveis — a evolução na Intel/AMD

O esquema de dois níveis pode ser expandido para **três, quatro ou mais níveis**. Níveis adicionais dão mais flexibilidade.

```
INTEL 80386 (1985) — 32 bits, até 4 GB:
  → tabela de dois níveis:
      DIRETÓRIO DE PÁGINAS → TABELAS DE PÁGINAS
      → quadros de página de 4 KB
  → ambos com 1.024 entradas:
      2¹⁰ × 2¹⁰ × 2¹² = 2³² bytes endereçáveis ✔

PENTIUM PRO (≈10 anos depois):
  → adicionou OUTRO nível:
      TABELA DE PONTEIROS DE DIRETÓRIOS DE PÁGINA
      (page directory pointer table)
  → ampliou cada entrada de 32 para 64 bits,
    para endereçar memória acima de 4 GB
  → mas só tinha 4 entradas na tabela de
    ponteiros, 512 por diretório e 512 por
    tabela de páginas:
      limite total AINDA de 4 GB

x86 de 64 bits (originalmente pela AMD):
  → um nível adicional → "MAPA DE PÁGINA NÍVEL 4"
    (page map level 4)
  → usa TODAS as 512 entradas em TODAS as tabelas:
      2⁹ × 2⁹ × 2⁹ × 2⁹ × 2¹² = 2⁴⁸ bytes
      (256 TB endereçáveis)

PROCESSADORES MAIS NOVOS:
  → suporte a um QUINTO nível
  → endereços de 57 bits → até 128 petabytes
  → permite mapear arquivos enormes
```

> ⚠️ **O preço de tantos níveis:** as travessias das tabelas de páginas (*page table walks*) se tornam **ainda mais demoradas**, pois cada nível adicional exige mais um acesso à memória em caso de ausência no TLB. (Os projetistas chegaram a achar que 4 níveis bastariam — "eles estavam errados".)

---

# 🔄 Tabelas de Páginas Invertidas

## A ideia: inverter a perspectiva

Uma alternativa à quantidade cada vez maior de níveis é a **tabela de páginas invertida**. Foi usada pela primeira vez em processadores como **PowerPC**, **UltraSPARC** e **Itanium**.

```
TABELA TRADICIONAL:
  → UMA entrada por PÁGINA DE ESPAÇO DE
    ENDEREÇO VIRTUAL
  → cresce com o espaço virtual (2⁵² entradas!)

TABELA INVERTIDA:
  → UMA entrada por QUADRO DE PÁGINA na
    MEMÓRIA REAL (física)
  → cresce com a RAM, não com o espaço virtual
```

```
EXEMPLO (do livro):
  Endereços virtuais de 64 bits
  Páginas de 4 KB
  16 GB de RAM

  → a tabela invertida exige apenas
    4.194.304 entradas (2²²)

  Cada entrada registra QUAL
  (processo, página virtual) está localizado
  naquele quadro de página.
```

## 📌 Figura 3.14 — Tradicional vs. invertida

```
Tabela TRADICIONAL        Memória física      Tabela INVERTIDA
(1 entrada por cada       de 1 GB =           (tabela de espalhamento/hash)
 uma das 2⁵² páginas)     2¹⁸ quadros de 4 KB

┌─────────┐ 2⁵²−1        ┌─────────┐ 2¹⁸−1    ┌─────────┐ 2¹⁸−1 ─► [ ]─►[ ]─►[ ]
│         │              │         │          │         │
│  ...    │              │  ...    │          │  ...    │ ─► [ ]─►[ ]
│         │              │         │          │         │
└─────────┘ 0            └─────────┘ 0        └─────────┘ 0 ─► [ ]─►[ ]
Indexada pela            (1 entrada por        Indexada pelo ESPALHAMENTO (hash)
PÁGINA VIRTUAL            quadro físico)        da página virtual
                                               cada nó = (página virtual, quadro)
```

## A grande desvantagem: a tradução fica difícil

```
NA TABELA TRADICIONAL:

  O processo n referencia a página virtual p
  → o hardware usa p DIRETAMENTE como ÍNDICE
  → acesso imediato

NA TABELA INVERTIDA:

  O hardware NÃO consegue mais usar p como
  índice (a tabela é indexada por QUADRO físico,
  não por página virtual)

  → ele precisa PESQUISAR a tabela inteira
    em busca de uma entrada (n, p)

  → e essa pesquisa teria que ser feita a CADA
    referência de memória, não só nas faltas de
    página

  → pesquisar uma tabela de 256 K entradas a cada
    referência NÃO é a melhor maneira de tornar
    sua máquina incrivelmente rápida
```

## A solução: TLB + tabela de espalhamento (hash)

```
1. USAR O TLB
   → SE o TLB conseguir conter todas as páginas
     intensamente usadas, a tradução acontece tão
     rápido quanto com tabelas normais

2. EM UMA AUSÊNCIA NO TLB
   → a tabela invertida precisa ser pesquisada
     em SOFTWARE

3. COMO PESQUISAR RÁPIDO: TABELA DE
   ESPALHAMENTO (hash) sobre os endereços
   virtuais

   → todas as páginas virtuais atualmente na
     memória que têm o MESMO valor de
     espalhamento são ENCADEADAS juntas
     (Figura 3.14)

   → SE a tabela de espalhamento tiver o mesmo
     número de entradas que o número de páginas
     físicas da máquina, o encadeamento médio
     será de apenas UMA entrada de comprimento
     → mapeamento muito acelerado

4. Assim que o número do quadro é encontrado,
   o novo par (virtual, físico) é INSERIDO NO TLB
```

## Quando valem a pena

```
Tabelas invertidas são COMUNS em máquinas de
64 bits, porque mesmo com um tamanho de página
muito grande, o número de entradas de uma
tabela tradicional é GIGANTESCO.

EXEMPLO: com páginas de 4 MB e endereços
virtuais de 64 bits, são necessárias 2⁴²
entradas de tabela de páginas.
```

> 📝 **Curiosidade do livro:** as tabelas invertidas foram usadas pela primeira vez em processadores como PowerPC, UltraSPARC e Itanium — e, apesar de vários desses projetos não terem sido grandes sucessos comerciais, as tabelas invertidas **sobreviveram**.

---

# 📊 Comparação: Multinível vs. Invertida

| Aspecto | Multinível | Invertida |
|---|---|---|
| **Tamanho cresce com...** | partes **usadas** do espaço virtual | tamanho da **RAM física** |
| **Entradas** | por página virtual (só nas regiões usadas) | por **quadro de página** físico |
| **Tradução** | indexação direta em cada nível | **busca** (hash) por (processo, página) |
| **Custo em ausência no TLB** | um acesso à memória **por nível** | busca em tabela hash (em software) |
| **Usada em** | x86 / x86-64 (até 5 níveis) | PowerPC, UltraSPARC, Itanium |

---

# ✅ Resumo do Conceito

- O TLB resolve a **velocidade**, mas o problema do **tamanho** da tabela de páginas em espaços de endereçamento enormes ainda precisa de outra solução — o livro apresenta duas: **multinível** e **invertida**
- **Tabela multinível:** o endereço é fatiado em campos (ex.: PT1 de 10 bits, PT2 de 10 bits, deslocamento de 12 bits). PT1 indexa a tabela de nível superior; cada entrada aponta para uma tabela de segundo nível, indexada por PT2, que devolve o quadro de página. O truque é **não criar** tabelas para regiões não usadas — um processo de 12 MB precisa de apenas **4 tabelas** (1 superior + 3 de segundo nível), em vez de mais de 1 milhão de entradas
- A hierarquia pode crescer: **80386** (2 níveis, 4 GB) → **Pentium Pro** (+page directory pointer table, entradas de 64 bits) → **x86-64** (4 níveis, 2⁴⁸ = 256 TB) → processadores recentes (**5 níveis**, 57 bits, 128 PB). O custo é que cada nível extra torna o *page table walk* mais lento
- **Tabela invertida:** uma entrada por **quadro de página físico** (não por página virtual), registrando qual (processo, página virtual) o ocupa. O tamanho depende da RAM, não do espaço virtual (16 GB de RAM e páginas de 4 KB → 4.194.304 entradas, mesmo com endereços de 64 bits)
- A desvantagem da invertida é que a **tradução exige busca** (não dá para indexar pela página virtual). A solução combina **TLB** (para o caso comum) com uma **tabela de espalhamento (hash)** pesquisada em software nas ausências, encadeando páginas com o mesmo valor de hash

---

## 🔗 Notas Relacionadas

- [[Acelerando a Paginação]] — o TLB e o page table walk, que explicam por que mais níveis significam ausências mais caras
- [[Tabelas de Páginas]] — a estrutura de entrada de tabela de páginas (3.3.2) que cada nível guarda
- [[Paginação]] — o cálculo do problema de escala (2⁵² entradas) que motiva esta seção
- [[Gerenciando a Memória Livre]] — outra estrutura que usa listas encadeadas e trade-offs de tamanho vs. velocidade
