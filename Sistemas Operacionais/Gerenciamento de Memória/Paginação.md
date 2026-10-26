---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seções 3.3 e 3.3.1"
---
,# Memória Virtual e Paginação

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seções 3.3 e 3.3.1

---

# 🧠 3.3 — Memória Virtual

## O problema que a motiva

Os esquemas vistos até agora (base/limite, swapping) presumem que todo o espaço de endereçamento de um processo **cabe na memória física**. Mas isso deixou de ser garantido por dois motivos que foram piorando com o tempo:

```
1. PROGRAMAS cada vez MAIORES

   Software comercial é frequentemente projetado
   para rodar na MAIOR memória disponível no
   momento do lançamento — se a memória cresce,
   o software cresce para ocupá-la (fenômeno
   informalmente chamado de "inchaço do software"
   / bloatware).

2. EXECUTAR VÁRIOS programas AO MESMO TEMPO

   Mesmo que CADA programa individualmente caiba
   na memória, a SOMA de vários programas rodando
   simultaneamente pode ULTRAPASSAR a memória
   física disponível.
```

> ⚠️ **Por que não resolver isso só com swapping?** Trocar processos inteiros entre disco e memória (visto em 3.2.2) é uma solução **lenta**: mesmo com SSDs modernos, as taxas de transferência são muito inferiores à velocidade da RAM, e o problema piora ainda mais se o processo que precisa ser carregado for **maior** que toda a memória física disponível — nesse caso, swapping simplesmente não resolve.

## A ideia antiga: sobreposições (overlays)

Antes de 1961, programas maiores que a memória física eram divididos **manualmente** pelo próprio programador em pedaços chamados **sobreposições (overlays)**.

```
FUNCIONAMENTO:

  Overlay 0 iniciava a execução e, ao terminar,
  chamava outro overlay, que sobrescrevia o
  overlay 0 na memória (ou parte dele) e
  assim por diante.

PROBLEMA:

  O trabalho de DIVIDIR o programa em pedaços
  era do PROGRAMADOR — tedioso, demorado e
  sujeito a erros.

  Ninguém nunca automatizou esse trabalho antes
  que alguém tivesse uma ideia melhor.
```

## A ideia nova: memória virtual (Fotheringham, 1961)

```
IDEIA CENTRAL:

  Cada programa tem seu PRÓPRIO espaço de
  endereçamento, dividido em pedaços chamados
  PÁGINAS.

  Cada página é uma faixa CONTÍGUA de endereços.

  Essas páginas são mapeadas para a memória
  física, mas NEM TODAS as páginas precisam
  estar na memória física ao mesmo tempo para
  o programa poder ser executado.
```

Quando o programa referencia uma parte do espaço de endereçamento que **está** na memória física, o hardware realiza o mapeamento necessário instantaneamente. Quando o programa referencia uma parte do espaço de endereçamento que **não está** na memória física, o sistema operacional é avisado para buscar a parte faltante e executar novamente a instrução que falhou.

> 💡 Em outras palavras, essa é exatamente a mesma ideia das **sobreposições**, exceto que agora ela é feita **automaticamente**, pelo SO e pelo hardware, em vez de pelo programador. Essa técnica é chamada **memória virtual**.

## ⚙️ Figura 3.8 — A posição e função da MMU

A **MMU (Memory Management Unit)** é o componente de hardware responsável por traduzir endereços virtuais em endereços físicos.

```
┌──────────────────────────────────┐
│               CPU                │
│                                  │
│   gera ENDEREÇOS VIRTUAIS        │
│          │                       │
│          ▼                       │
│   ┌───────────────┐              │
│   │      MMU       │  ←── parte do chip da CPU
│   └───────────────┘      (nos processadores atuais)
│          │                       │
└──────────┼───────────────────────┘
           │ ENDEREÇOS FÍSICOS
           ▼
   ┌─────────────────────┐
   │  Barramento de       │
   │  memória              │
   └───────────┬──────────┘
               ▼
       ┌───────────────┐
       │  Memória       │
       │  Física (RAM)  │
       └───────────────┘
```

```
A CPU envia endereços VIRTUAIS para a MMU.

A MMU compara esses endereços virtuais com os
endereços físicos correspondentes e os envia
para o BARRAMENTO de memória.

HISTORICAMENTE a MMU era um chip SEPARADO; hoje
ela está INTEGRADA ao chip da CPU, mas
CONCEITUALMENTE continua separada e é tratada
dessa forma pelos diagramas (e por este resumo).
```

---

# 🧩 3.3.1 — Paginação

## Páginas e quadros de página (page frames)

A maioria dos sistemas de memória virtual usa uma técnica chamada **paginação**.

```
O ESPAÇO DE ENDEREÇAMENTO VIRTUAL é dividido em
unidades chamadas PÁGINAS.

As unidades correspondentes na MEMÓRIA FÍSICA são
chamadas de QUADROS DE PÁGINA (page frames).

PÁGINAS e QUADROS DE PÁGINA têm SEMPRE o MESMO
TAMANHO.
```

> 💡 **Página:** unidade de divisão do espaço de endereçamento **virtual**.
> 💡 **Quadro de página (page frame):** unidade de divisão correspondente na memória **física**, do mesmo tamanho de uma página.

## Exemplo numérico — 64 KB virtuais, 32 KB físicos

Considere um computador com endereços virtuais de **16 bits** (espaço de endereçamento virtual de 0 a 64 KB) e apenas **32 KB** de memória física.

```
Com páginas de 4 KB:

  Espaço de endereçamento VIRTUAL:
    64 KB ÷ 4 KB  = 16 PÁGINAS virtuais (0 a 15)

  Memória FÍSICA:
    32 KB ÷ 4 KB  = 8 QUADROS DE PÁGINA (0 a 7)
```

Como há **16 páginas virtuais** mas apenas **8 quadros de página físicos**, nem todas as páginas virtuais podem estar mapeadas na memória física simultaneamente — algumas estarão presentes, outras não.

> ⚠️ **Tamanhos reais de página:** embora o exemplo use 4 KB por simplicidade, sistemas reais utilizam páginas que variam de **512 bytes a 1 GB**, dependendo da arquitetura. Processadores **x86-64**, por exemplo, suportam **misturar** tamanhos de página diferentes (4 KB, 2 MB e 1 GB) simultaneamente, usando páginas maiores para reduzir overhead em regiões de memória que exigem tradução intensiva.

## A tabela de páginas e o bit presente/ausente

O mapeamento entre páginas virtuais e quadros de página físicos é mantido em uma estrutura chamada **tabela de páginas**.

```
Para CADA página virtual, a tabela de páginas
guarda:

  → o QUADRO DE PÁGINA físico correspondente
    (SE a página estiver presente na memória)

  → um bit chamado PRESENTE/AUSENTE
    (present/absent bit), que indica se aquela
    página VIRTUAL está atualmente mapeada para
    um quadro físico REAL ou não
```

```
SE o bit PRESENTE/AUSENTE estiver em 1:
  → a página está na memória física
  → o endereço pode ser traduzido diretamente
    pelo hardware (pela MMU)

SE o bit PRESENTE/AUSENTE estiver em 0:
  → a página NÃO está na memória física
  → tentar acessá-la gera uma FALTA DE PÁGINA
    (page fault), que é tratada pelo SO
```

## 📌 Figura 3.9 — Exemplos de tradução de endereços

Usando o exemplo de 16 páginas virtuais mapeadas para 8 quadros físicos:

```
MOV REG, 0
  → endereço virtual 0 está na página virtual 0
  → página virtual 0 está mapeada no quadro
    físico 2
  → endereço FÍSICO resultante = 2 × 4096 = 8192

MOV REG, 8192
  → endereço virtual 8192 está na página virtual 2
  → página virtual 2 está mapeada no quadro
    físico 6
  → endereço FÍSICO resultante = 6 × 4096 = 24576

Endereço virtual 20500
  → página virtual 5 (20480 a 24575),
    deslocamento (offset) 20 dentro da página
    (20500 − 20480 = 20)
  → página virtual 5 está mapeada no quadro
    físico 3
  → endereço FÍSICO resultante = 3 × 4096 + 20
                                = 12288 + 20
                                = 12308
```

### O caso da falta de página (page fault)

```
Endereço virtual 32780 (página virtual 8,
deslocamento 12)

  → a página virtual 8 NÃO possui um quadro
    físico mapeado (bit presente/ausente = 0)

  → tentar acessá-la causa uma ARMADILHA
    (trap) para o sistema operacional —
    a FALTA DE PÁGINA (page fault)
```

```
O QUE O SO FAZ ao tratar essa falta de página:

  1. ESCOLHE um quadro de página pouco usado
     e GRAVA seu conteúdo de volta no disco
     (se ele tiver sido modificado)

  2. BUSCA a página que acabou de ser
     referenciada para o quadro recém-liberado

  3. ALTERA o mapa na tabela de páginas

  4. EXECUTA NOVAMENTE a instrução que
     causou a falta
```

> 💡 Note o padrão: a falta de página é tratada de forma **transparente** para o processo — do ponto de vista do programa, a instrução simplesmente "demora mais" para executar, mas ele nunca sabe que uma página precisou ser buscada do disco.

## 🔬 Figura 3.10 — Funcionamento interno da MMU (nível de bits)

Olhando com mais detalhe o mecanismo que a MMU usa internamente, usando o mesmo exemplo de endereços virtuais de **16 bits** e páginas de **4 KB**:

```
ENDEREÇO VIRTUAL (16 bits) = 8292 (binário: 0010000001100100)

Dividido em dois campos:

┌───────────────────────┬─────────────────────────┐
│  NÚMERO DA PÁGINA      │     DESLOCAMENTO          │
│   (4 bits)             │      (12 bits)            │
│      0010              │    000001100100           │
└───────────────────────┴─────────────────────────┘
       = página 2              = offset 100 (dec)
```

```
PASSO A PASSO da tradução pela MMU:

  1. O NÚMERO DA PÁGINA (4 bits) é usado como
     um ÍNDICE dentro da tabela de páginas,
     localizando a entrada correspondente

  2. Essa entrada da tabela de páginas contém
     o NÚMERO DO QUADRO de página físico
     correspondente (ex.: quadro 6), além do
     bit PRESENTE/AUSENTE

  3. O NÚMERO DO QUADRO encontrado é copiado
     para os bits SUPERIORES de um registrador
     de saída, substituindo o número da página
     virtual original

  4. O DESLOCAMENTO (12 bits), que NUNCA é
     modificado na tradução, é copiado
     diretamente para os bits INFERIORES do
     mesmo registrador de saída

  RESULTADO: os 16 bits do registrador de saída
  agora contêm o ENDEREÇO FÍSICO correto, pronto
  para ser enviado ao barramento de memória
```

> 📌 O número da página funciona como índice para dentro da tabela de páginas e fornece o número do quadro de página de saída. O número do quadro de página é então combinado com o deslocamento para construir um endereço físico.

### Por que a tabela de páginas pode se tornar um problema

```
No exemplo de 16 bits, a tabela de páginas tem
apenas 2⁴ = 16 entradas — TRIVIAL.

MAS em sistemas REAIS:

  Endereços de 32 bits com páginas de 4 KB:
    → número da página tem 20 bits
    → a tabela de páginas precisaria de
      2²⁰ (cerca de 1 milhão) de entradas
    → GRANDE, mas ainda GERENCIÁVEL

  Endereços de 64 bits com páginas de 4 KB:
    → número da página teria 52 bits
    → a tabela de páginas precisaria de
      2⁵² entradas
    → um número ABSURDAMENTE grande
      ("um inferno de grande") — INVIÁVEL de
      se manter como uma tabela simples e
      linear
```

> ⚠️ Esse problema de **escala** da tabela de páginas em arquiteturas de 64 bits é exatamente o que motiva técnicas mais sofisticadas de organização de tabelas de páginas (tabelas multiníveis, TLBs, etc.), que serão vistas em seções posteriores.

---

# ✅ Resumo do Conceito

- **Memória virtual** resolve o problema de programas (ou do conjunto de programas rodando simultaneamente) que excedem a memória física disponível, substituindo as antigas **sobreposições manuais** (overlays) por um mecanismo **automático**, controlado pelo SO e pelo hardware (**Fotheringham, 1961**)
- A **MMU** (Memory Management Unit) é o hardware que traduz **endereços virtuais** gerados pela CPU em **endereços físicos** enviados ao barramento de memória — historicamente um chip separado, hoje integrada à CPU
- **Paginação:** o espaço de endereçamento virtual é dividido em **páginas**; a memória física é dividida em **quadros de página (page frames)** do mesmo tamanho. Tamanhos reais variam de 512 B a 1 GB (x86-64 pode misturar tamanhos)
- A **tabela de páginas** mapeia cada página virtual a um quadro físico, usando um **bit presente/ausente** para indicar se a página está atualmente carregada na memória
- Acessar uma página **ausente** causa uma **falta de página (page fault)**: o SO escolhe um quadro vítima, grava-o de volta no disco se necessário, busca a página faltante, atualiza a tabela de páginas e **reexecuta** a instrução que falhou — tudo de forma **transparente** ao processo
- No nível de bits, a MMU usa o **número da página** (bits superiores do endereço virtual) como índice na tabela de páginas para obter o **número do quadro físico**, e combina esse número com o **deslocamento** (bits inferiores, nunca alterado) para formar o endereço físico final
- Esse mecanismo de tabela simples **não escala bem**: em arquiteturas de 64 bits, uma tabela de páginas linear exigiria 2⁵² entradas — inviável, o que motiva estruturas mais sofisticadas

---

## 🔗 Notas Relacionadas

- [[Memória Virtual]] — introdução conceitual ao tema no Cap. 1, antes do mecanismo detalhado de paginação
- [[Troca de Processos (Swapping)]] — a alternativa mais simples (e mais lenta) à memória virtual, trocando processos inteiros entre disco e memória
- [[Espaços de Endereçamento e Registradores-Base e Limite]] — o mecanismo mais simples de tradução de endereços, que a paginação substitui com algo mais flexível
- [[Ausência de abstração de memória]] — o ponto de partida sem qualquer abstração, que motiva toda a evolução até a memória virtual
