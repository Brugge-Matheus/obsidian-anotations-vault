---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.4"
---
# Tamanho de Página

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.4

---

# 📏 3.5.4 — Tamanho de Página

## O que está em jogo

O **tamanho de página** é um parâmetro que **pode ser escolhido pelo sistema operacional**. Mesmo que o hardware tenha sido projetado com páginas de **4.096 bytes**, o SO pode tratar, por exemplo, os pares de página 0 e 1, 2 e 3, 4 e 5 e assim por diante **como páginas de 8 KB**, sempre alocando **dois quadros de 4.096 bytes consecutivos** para elas.

```
Hardware: páginas de 4 KB
   ┌────┬────┬────┬────┬────┬────┐
   │ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ ...
   └────┴────┴────┴────┴────┴────┘

O SO "agrupa" de dois em dois:
   ┌─────────┬─────────┬─────────┐
   │ 0 e 1   │ 2 e 3   │ 4 e 5   │ ...   ← "páginas" de 8 KB
   └─────────┴─────────┴─────────┘
```

> ⚠️ **Não existe tamanho ótimo geral.** Determinar o melhor tamanho exige **equilibrar vários fatores que competem entre si**. Alguns pedem páginas **pequenas**, outros pedem páginas **grandes**. O restante da nota mostra os dois lados e, no final, uma **fórmula** que acha o ponto de equilíbrio.

---

# 🔹 Argumentos a favor de páginas PEQUENAS

## 1. Fragmentação interna

```
Um segmento de código, dados ou pilha escolhido AO
ACASO NÃO ocupará um número INTEIRO de páginas.

  → NA MÉDIA, METADE da página FINAL estará VAZIA

  → esse espaço extra é DESPERDIÇADO

  → o desperdício se chama FRAGMENTAÇÃO INTERNA
    (o espaço está DENTRO de uma página já alocada,
     mas não é usado por ninguém)
```

**Exemplo concreto:** um segmento de dados com 10.000 bytes:

| Tamanho da página | Páginas usadas | Espaço alocado | Desperdício na última página |
|---|---|---|---|
| **512 B** | 20 (10.240 B) | 10.240 B | **240 B** |
| **4 KB** | 3 (12.288 B) | 12.288 B | **2.288 B** |
| **64 KB** | 1 (65.536 B) | 65.536 B | **55.536 B** |

```
FÓRMULA DO LIVRO:

  Com n segmentos na memória e páginas de p bytes:

        desperdício médio ≈ n × p / 2  bytes
```

> 💡 **Conclusão:** quanto **maior** a página, **mais** espaço se perde na última página de cada segmento. Isso defende um tamanho de página **pequeno**.

(É o mesmo dilema do tamanho da unidade de alocação visto em [[Gerenciando a Memória Livre]]: unidade grande desperdiça espaço na última unidade.)

## 2. Programas com fases

```
EXEMPLO DO LIVRO: um programa com OITO fases
SEQUENCIAIS de 4 KB cada (usa só uma fase por vez).

  Páginas de 32 KB:
    → o programa demanda 32 KB o tempo INTEIRO
      de execução

  Páginas de 16 KB:
    → precisa de apenas 16 KB

  Páginas de 4 KB (ou menores):
    → exigirá apenas 4 KB a qualquer instante
```

Com páginas pequenas, a memória guarda **só o que realmente está em uso**; com páginas grandes, ela guarda também "vizinhança" inútil. **Em geral, um tamanho de página grande causará mais desperdício de espaço na memória.**

---

# 🔸 Argumentos a favor de páginas GRANDES

## 1. Tabela de páginas menor

```
Páginas PEQUENAS → programas precisarão de MUITAS
páginas → e, portanto, uma GRANDE tabela de páginas.

EXEMPLO: um programa de 32 KB precisa de
  → apenas 4 páginas de 8 KB
  → mas 64 páginas de 512 bytes
```

## 2. Transferências de e para o disco/SSD

```
Transferências geralmente acontecem com UMA PÁGINA de
cada vez.

SE o armazenamento NÃO for um SSD, mas um DISCO
MAGNÉTICO:
  → a maior parte do tempo é gasta no TEMPO DE BUSCA
    e na ROTAÇÃO
  → transferir uma página pequena leva praticamente
    o MESMO tempo que uma página grande

EXEMPLO DO LIVRO:
  → carregar 64 páginas de 512 B: 64 × 10 ms = 640 ms
  → carregar 4 páginas de 8 KB:    4 × 12 ms =  48 ms

  (mesma quantidade de dados: 32 KB. Mais de 13×
   mais lento com páginas pequenas.)
```

## 3. Entradas do TLB

Talvez o argumento **mais importante**: páginas pequenas ocupam **muito espaço no TLB** ([[Acelerando a Paginação]]).

```
EXEMPLO: seu programa usa 1 MB de memória, com um
CONJUNTO DE TRABALHO de 64 KB.

  Páginas de 4 KB:
    → ocupa pelo menos 16 ENTRADAS no TLB
      (64 KB ÷ 4 KB = 16)

  Páginas de 2 MB:
    → UMA ÚNICA entrada de TLB seria suficiente
      (na teoria — talvez você queira separar dados
       de instruções)
```

> ✅ **Entradas de TLB são ESCASSAS e CRÍTICAS** para o desempenho (poucas dezenas), então **vale a pena usar páginas grandes sempre que possível**.

### Como os sistemas equilibram isso na prática

```
Sistemas operacionais às vezes usam TAMANHOS
DIFERENTES de páginas para PARTES DIFERENTES do
sistema.

  EXEMPLO: páginas GRANDES para o NÚCLEO e menores
  para os PROCESSOS DO USUÁRIO.

Alguns SOs se esforçam MUITO para usar páginas
grandes: até MOVEM a memória de um processo para
encontrar ou criar intervalos CONTÍGUOS de memória
adequados para suportar uma página grande.

  → esse recurso às vezes é chamado de
    PÁGINAS GIGANTES TRANSPARENTES
    (transparent huge pages)
```

## 4. Máquinas que carregam a tabela em registradores

```
EM ALGUMAS MÁQUINAS, a tabela de páginas deve ser
CARREGADA (pelo SO) em REGISTRADORES DE HARDWARE toda
vez que a CPU TROCAR de um processo para outro.

NESSAS MÁQUINAS:
  → tamanho de página PEQUENO = o tempo para carregar
    os registradores de página fica MAIS LONGO
  → e o ESPAÇO ocupado pela tabela de páginas AUMENTA
    à medida que o tamanho da página DIMINUI
```

---

# 🧮 A matemática do equilíbrio

O último ponto pode ser analisado **matematicamente**. Vamos montar a conta passo a passo.

## Passo 1 — Definir as variáveis

| Símbolo | Significado | Exemplo do livro |
|---|---|---|
| **s** | tamanho médio de um processo (bytes) | 1 MB |
| **p** | tamanho da página (bytes) | (o que queremos descobrir) |
| **e** | bytes ocupados por cada entrada da tabela de páginas | 8 bytes |

## Passo 2 — Custo da tabela de páginas

```
Nº aproximado de páginas por processo = s / p

Cada página tem uma entrada de e bytes na tabela:

  espaço da tabela de páginas = (s / p) × e = s·e / p
```

> 💡 Quanto **menor** o **p**, **maior** esse custo (mais páginas → tabela maior).

## Passo 3 — Custo da fragmentação interna

```
A memória desperdiçada na ÚLTIMA página do processo,
em média, é METADE da página:

  desperdício = p / 2
```

> 💡 Quanto **maior** o **p**, **maior** esse custo.

## Passo 4 — Somar os dois custos

```
custo adicional total  =  s·e / p   +   p / 2
                         └───┬───┘     └─┬─┘
                      tabela de        fragmentação
                      páginas          interna
                      (cai se p sobe)  (sobe se p sobe)
```

Os dois termos **puxam em direções opostas**:

- **1º termo** (tabela): **grande** quando a página é **pequena**
- **2º termo** (fragmentação): **grande** quando a página é **grande**

> ✅ O **valor ótimo** precisa estar em **algum ponto intermediário**.

## Passo 5 — Achar o mínimo

O mínimo está onde a **derivada** do custo em relação a **p** vale zero:

```
derivada de (s·e/p + p/2) em relação a p:

    −s·e / p²  +  1/2  =  0
```

Resolvendo:

```
    s·e / p²  =  1/2
    p²        =  2·s·e

    p  =  √(2·s·e)
```

> 💡 **Intuição sem cálculo:** nesse ponto ótimo, os **dois custos são IGUAIS**. Veja: se p² = 2se, então s·e/p = p/2. É como achar o ponto onde os dois "lados da gangorra" se equilibram: se o custo da tabela fosse maior que o da fragmentação, valeria aumentar p; se fosse menor, valeria diminuir.

## Passo 6 — O número do livro

```
s = 1 MB = 1.048.576 bytes
e = 8 bytes

p = √(2 × 1.048.576 × 8)
  = √16.777.216
  = 4.096 bytes  =  4 KB
```

## Conferindo com uma tabela

Com s = 1 MB e e = 8 bytes, o custo total (tabela + fragmentação) para vários tamanhos:

| Página p | Tabela (s·e/p) | Fragmentação (p/2) | **Total** |
|---|---|---|---|
| 512 B | 16.384 B | 256 B | 16.640 B |
| 1 KB | 8.192 B | 512 B | 8.704 B |
| **4 KB** | **2.048 B** | **2.048 B** | **4.096 B** ← mínimo |
| 16 KB | 512 B | 8.192 B | 8.704 B |
| 64 KB | 128 B | 32.768 B | 32.896 B |

Repare que em **4 KB** os dois custos são **iguais** (2.048 B cada), exatamente o que a intuição previa.

## Na prática

```
Computadores disponíveis comercialmente têm usado
tamanhos de páginas que variam de 512 bytes a 64 KB.

  → um valor típico costumava ser 1 KB
  → hoje, 4 KB é mais comum
```

> ⚠️ **Limite da fórmula:** ela considera **apenas** a memória desperdiçada na fragmentação e o tamanho da tabela de páginas. Fatores como o **TLB** e o **tempo de transferência de disco** (que favorecem páginas grandes) ficam de fora, o que explica por que os sistemas modernos também usam páginas muito maiores em regiões específicas.

---

# ✅ Resumo do Conceito

- O **tamanho de página** pode ser escolhido pelo SO (ex.: agrupar duas páginas de 4 KB do hardware em uma de 8 KB). **Não há tamanho ótimo geral**: há fatores em conflito
- **A favor de páginas pequenas:** menos **fragmentação interna** (em média, metade da última página fica vazia: n·p/2 para n segmentos) e a memória guarda só o que o programa usa de fato (exemplo das 8 fases de 4 KB)
- **A favor de páginas grandes:** **tabela de páginas menor**; **transferências de disco mais eficientes** (em disco magnético, o tempo é dominado por busca e rotação: 64 × 10 ms contra 4 × 12 ms); **menos entradas de TLB**, que são escassas e críticas (por isso existem as **páginas gigantes transparentes**); e, em máquinas que carregam a tabela em registradores, carga mais rápida
- **Matemática:** custo = **s·e/p + p/2** (tabela + fragmentação). O mínimo está em **p = √(2·s·e)**, ponto em que os dois custos se igualam. Com s = 1 MB e e = 8 bytes, o resultado é **4 KB**
- Tamanhos reais variam de **512 B a 64 KB**; **4 KB** é o mais comum hoje (já foi 1 KB)

---

## 🔗 Notas Relacionadas

- [[Paginação]] — páginas, quadros de página e tamanhos reais (512 B a 1 GB no x86-64)
- [[Tabelas de Páginas]] — as entradas de e bytes cujo número cresce com páginas pequenas
- [[Acelerando a Paginação]] — o TLB, cujas entradas escassas favorecem páginas grandes
- [[Tabelas de Páginas para Memórias Grandes]] — como lidar com tabelas enormes quando as páginas são pequenas
- [[Gerenciando a Memória Livre]] — o mesmo dilema de tamanho de unidade de alocação (mapas de bits)
- [[Política de Limpeza]] — outra questão de projeto do sistema de paginação
