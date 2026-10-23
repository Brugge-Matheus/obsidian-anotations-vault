---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.3.3"
---
# Acelerando a Paginação

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.3.3

---

# ⚡ 3.3.3 — Acelerando a Paginação

## As duas questões fundamentais

Já vimos os princípios básicos da memória virtual e da paginação. Chegou o momento de entrar em mais detalhes sobre as **implementações** possíveis. Em qualquer sistema de paginação, duas questões fundamentais precisam ser abordadas:

```
1. O MAPEAMENTO do endereço virtual para o
   endereço físico PRECISA ser RÁPIDO

2. MESMO que o espaço do endereço virtual seja
   AMPLO, a tabela de páginas NÃO DEVE ser muito
   GRANDE
```

> ⚠️ **Por que a velocidade importa tanto?** O mapeamento virtual-físico deve ser feito em **cada referência de memória**. Todas as instruções devem, em última análise, vir da memória, e muitas delas referenciam operandos na memória também. Em consequência, é preciso fazer uma, duas, ou às vezes mais referências à tabela de páginas por instrução. Se a execução de uma instrução leva, digamos, 1 ns, a procura na tabela de páginas precisa ser feita em **menos de 0,2 ns** para evitar que o mapeamento se torne um gargalo significativo.

> ⚠️ **Por que o tamanho importa tanto?** Todos os computadores modernos usam endereços virtuais de pelo menos 32 bits, com 64 bits tornando-se a norma para desktops e notebooks. Mesmo que um processador moderno utilize apenas 48 dos 64 bits para endereçamento, com um tamanho de página de 4 KB, um espaço de endereço de 48 bits tem **64 bilhões de páginas**. A tabela de páginas precisaria ter 64 bilhões de entradas de 64 bits cada — centenas de gigabytes só para armazenar a tabela de páginas, **por processo** (já que cada processo tem seu próprio espaço de endereços virtuais).

Essas duas restrições motivam as técnicas vistas a seguir.

---

## 🔧 Abordagem 1 — Tabela única de registradores de hardware

```
O PROJETO MAIS SIMPLES (pelo menos
conceitualmente):

  → uma ÚNICA tabela de página consistindo em
    uma série de REGISTRADORES DE HARDWARE
    RÁPIDOS

  → com UMA ENTRADA para CADA página virtual

  → INDEXADA pelo NÚMERO da página virtual
    (igual ao visto na Figura 3.10)
```

```
QUANDO um processo é INICIALIZADO:

  → o sistema operacional CARREGA os
    registradores com a tabela de páginas do
    processo, tirada de uma CÓPIA mantida na
    memória principal

  → DURANTE a execução do processo, NÃO são
    necessárias MAIS referências de memória
    para a tabela de páginas
```

**Vantagens:**

```
→ É DIRETO

→ NÃO exige referências de memória DURANTE o
  mapeamento (os registradores já estão
  carregados)
```

**Desvantagens:**

```
→ É TERRIVELMENTE CARO se a tabela de páginas
  for GRANDE — simplesmente NÃO é prático na
  maioria das vezes

→ TER de carregar a tabela de páginas INTEIRA
  em CADA troca de contexto ARRUINARIA
  COMPLETAMENTE o desempenho
```

---

## 🔧 Abordagem 2 — Tabela inteira na memória principal

```
NO OUTRO EXTREMO, a tabela de página pode estar
INTEIRAMENTE na MEMÓRIA PRINCIPAL.

Tudo o que o HARDWARE precisa, então, é de UM
ÚNICO REGISTRADOR que APONTE para o INÍCIO da
tabela de páginas.
```

**Vantagem:**

```
→ Esse projeto PERMITE que o mapa virtual-
  físico seja MODIFICADO em uma troca de
  contexto por meio do simples CARREGAMENTO
  de UM registrador (muito mais barato do que
  recarregar uma tabela inteira de
  registradores)
```

**Desvantagem:**

```
→ Ele exige UMA OU MAIS referências de memória
  para LER as entradas na tabela de páginas
  DURANTE a execução de CADA instrução,
  tornando-a MUITO LENTA
```

> ⚠️ A necessidade de mapeamentos de página **rápidos** para espaços de endereçamento **extensos** é uma limitação muito significativa sobre como os computadores são construídos atualmente — nenhuma das duas abordagens acima, sozinha, resolve bem o problema.

---

## 🚀 A solução: Translation Lookaside Buffer (TLB)

A observação-chave que leva à solução: a **maioria** dos programas tende a fazer um **grande número de referências** a um **pequeno número de páginas**, e não o contrário. Ou seja, apenas uma **pequena fração** das entradas da tabela de páginas é intensamente lida; o restante mal é usado.

```
A SOLUÇÃO concebida:

  Equipar os computadores com um PEQUENO
  DISPOSITIVO DE HARDWARE para mapear
  endereços virtuais em endereços físicos SEM
  ter de PASSAR pela tabela de páginas.
```

> 💡 **TLB (translation lookaside buffer)**, também chamado de **memória associativa**: dispositivo que normalmente fica **dentro da MMU** e consiste em um **pequeno número de entradas** — raramente mais do que 256, frequentemente apenas 64.

### 📌 Figura 3.12 — Um TLB para acelerar a paginação

```
┌────────┬────────────────┬────────────┬───────────┬────────────────┐
│ Válida │ Página virtual │ Modificada │ Proteção  │ Quadro de página│
├────────┼────────────────┼────────────┼───────────┼────────────────┤
│   1    │       140       │     1      │    RW     │       31        │
│   1    │       20        │     0      │    RX     │       38        │
│   1    │       130       │     1      │    RW     │       29        │
│   1    │       129       │     1      │    RW     │       62        │
│   1    │       19        │     0      │    RX     │       50        │
│   1    │       21        │     0      │    RX     │       45        │
│   1    │       860       │     1      │    RW     │       14        │
│   1    │       861       │     1      │    RW     │       75        │
└────────┴────────────────┴────────────┴───────────┴────────────────┘
```

```
CADA ENTRADA do TLB contém informações sobre
UMA página, incluindo:

  → o NÚMERO DA PÁGINA VIRTUAL

  → um bit configurado quando a página é
    MODIFICADA

  → o CÓDIGO DE PROTEÇÃO (permissões de
    ler/escrever/executar)

  → o QUADRO DE PÁGINA FÍSICA em que a página
    está localizada

Esses campos têm uma CORRESPONDÊNCIA de um
para um com os campos na TABELA DE PÁGINAS,
EXCETO pelo número da página virtual, que NÃO
é necessário na tabela de páginas (lá, a
posição JÁ identifica a página).

Outro bit indica se a entrada é VÁLIDA (em uso)
ou não.
```

> 💡 **Exemplo de como esse TLB poderia surgir:** um processo em um laço que inclua as páginas virtuais 19, 20 e 21 — essas entradas teriam códigos de proteção para leitura e execução. Os dados principais usados (um arranjo sendo processado) estão nas páginas 129 e 130. A página 140 contém os índices usados nos cálculos. Por fim, a pilha se encontra nas páginas 860 e 861.

### Como o TLB funciona

```
Quando um endereço VIRTUAL é apresentado à MMU
para TRADUÇÃO:

  1. O HARDWARE primeiro verifica se o seu
     número de página virtual está presente no
     TLB, comparando-o com TODAS as entradas
     SIMULTANEAMENTE (isto é, EM PARALELO —
     exige hardware especial, mas toda MMU com
     TLB o possui)

  2. SE uma correspondência VÁLIDA é encontrada
     e o acesso NÃO viola os bits de proteção:
       → o QUADRO DE PÁGINA é tirado
         DIRETAMENTE do TLB, SEM ir à tabela de
         páginas na memória

  3. SE o número da página virtual ESTIVER
     presente no TLB, mas a instrução estiver
     tentando ESCREVER em uma página SOMENTE
     DE LEITURA:
       → uma FALTA DE PROTEÇÃO é gerada
```

### O que acontece em uma ausência no TLB

```
QUANDO o número da página virtual NÃO está no
TLB:

  1. A MMU DETECTA a ausência e realiza uma
     BUSCA na tabela de páginas COMUM (na
     memória)

  2. Ela então DESTITUI uma das entradas do TLB
     e a SUBSTITUI pela entrada de tabela de
     páginas que acabou de ser buscada

  3. SE a MESMA página é usada novamente em
     seguida, da SEGUNDA vez ela resultará em
     uma PRESENÇA no TLB, em vez de uma
     AUSÊNCIA

QUANDO uma entrada é RETIRADA do TLB:

  → o bit MODIFICADO é COPIADO DE VOLTA na
    entrada correspondente da tabela de
    páginas NA MEMÓRIA

  → os OUTROS valores já estão ali, EXCETO o
    bit de REFERÊNCIA

  → quando o TLB é CARREGADO da tabela de
    páginas, TODOS os campos são trazidos da
    memória
```

> ⚠️ **Mantendo consistência:** se o sistema operacional quiser alterar os bits em uma entrada da tabela de páginas (por exemplo, para tornar uma página somente-leitura gravável), ele fará isso modificando-a na memória. No entanto, para garantir que a próxima escrita nessa página seja bem-sucedida, ele **também deve atualizar a entrada correspondente no TLB** com os bits de permissão novos.

---

## 🖥️ Gerenciamento do TLB por software

Até aqui presumimos que todas as máquinas com memória virtual paginada têm tabelas de página reconhecidas pelo **hardware**, mais um TLB.

```
NESSE ESQUEMA (gerenciamento por HARDWARE):

  → o GERENCIAMENTO e o TRATAMENTO das faltas
    de TLB são feitos INTEIRAMENTE pelo
    hardware da MMU

  → desvios de controle para o SO ocorrem
    APENAS quando uma página não está NA
    MEMÓRIA
```

Esse pressuposto é verdadeiro para muitas CPUs. No entanto, muitas máquinas **RISC** — incluindo **SPARC**, **MIPS** e a (fora de uso) **HP PA** — realizam **todo** esse gerenciamento de página em **software**.

```
NESSAS MÁQUINAS:

  → as entradas de TLB são EXPLICITAMENTE
    carregadas pelo SISTEMA OPERACIONAL

  → QUANDO ocorre uma AUSÊNCIA no TLB, em vez
    de a MMU ir às tabelas de páginas para
    encontrar e buscar a referência de página
    necessária:

      → ela apenas GERA uma FALTA DE TLB e
        JOGA o problema no COLO do sistema
        operacional

  → o sistema DEVE:
      1. ENCONTRAR a página
      2. REMOVER uma entrada do TLB
      3. INSERIR uma NOVA
      4. REINICIAR a instrução que FALHOU

  → tudo isso deve ser feito em um PUNHADO de
    instruções, pois ausências do TLB ocorrem
    com MUITO MAIS frequência do que faltas de
    página
```

> ⚠️ **Por que as ausências do TLB são tão mais comuns que as faltas de página?** Geralmente existem **milhares** de páginas na memória — portanto, as faltas de página são **raras**. Mas os TLBs normalmente contêm apenas umas **64 entradas**, portanto as ausências do TLB acontecem **o tempo todo**.

```
Os FABRICANTES de hardware PODERIAM reduzir o
número de ausências do TLB AUMENTANDO o seu
tamanho — MAS isso é CARO, e a área de chip que
um TLB MAIOR ocuparia deixaria MENOS espaço
para outros recursos importantes, como CACHES.

  → o projeto do chip está repleto de DILEMAS
```

> 💡 De maneira bastante **surpreendente**, se o TLB for **moderadamente** grande (digamos, 64 entradas) para reduzir a taxa de ausências, o gerenciamento de **software** do TLB acaba sendo **aceitavelmente eficiente**. O principal ganho aqui é uma **MMU muito mais simples**, o que libera uma área considerável no chip da CPU para **caches** e outros recursos que podem melhorar o desempenho.

---

## 🔍 Tipos de ausências do TLB — nuances importantes

É essencial compreender a diferença entre os diversos tipos de ausência.

```
AUSÊNCIA LEVE (soft miss):

  → ocorre QUANDO a página referenciada NÃO
    se encontra no TLB, MAS ESTÁ na memória

  → tudo o que é necessário: ATUALIZAR o TLB

  → NÃO é necessário realizar E/S em disco
    (ou SSD)

  → em GERAL, leva de 10 a 20 instruções de
    máquina para lidar, podendo ser concluída
    em ALGUNS NANOSSEGUNDOS

AUSÊNCIA COMPLETA (hard miss):

  → ocorre QUANDO a página EM SI não está na
    memória (e, é claro, também não está no
    TLB)

  → um ACESSO a disco ou SSD é necessário para
    TRAZER a página

  → pode levar VÁRIOS MILISSEGUNDOS, dependendo
    do armazenamento não volátil usado

  → é FACILMENTE um MILHÃO de vezes mais lenta
    do que uma ausência leve
```

> 💡 Procurar o mapeamento na hierarquia da tabela de páginas (durante uma ausência do TLB) é conhecido como um **passeio na tabela de páginas** (*page table walk*).

### A questão é ainda mais complicada

Uma ausência não é apenas "leve" ou "completa" de forma binária — algumas ausências são ligeiramente **mais leves** (ou mais completas) do que outras:

```
POSSIBILIDADE 1 — Falta de página MENOR
(minor page fault):

  → o PASSEIO não encontra a página na tabela
    de páginas DO PROCESSO, e o programa
    incorre, portanto, em uma falta de página

  → MAS a página PODE estar, na REALIDADE, na
    memória — por exemplo, se ela foi trazida
    do armazenamento não volátil por OUTRO
    processo

  → NESSE CASO, NÃO precisamos acessar o
    armazenamento não volátil NOVAMENTE —
    basta MAPEAR a página de maneira
    apropriada nas tabelas de páginas

POSSIBILIDADE 2 — Falta de página MAIOR
(major page fault):

  → ocorre SE a página PRECISAR ser trazida do
    ARMAZENAMENTO NÃO VOLÁTIL (disco/SSD) de
    fato

POSSIBILIDADE 3 — Falta de SEGMENTAÇÃO:

  → é POSSÍVEL que o programa apenas tenha
    ACESSADO um endereço INVÁLIDO, e NENHUM
    mapeamento precise ser acrescentado ao TLB

  → NESSE caso, o sistema operacional
    NORMALMENTE MATA o programa com uma FALTA
    DE SEGMENTAÇÃO (segmentation fault)

  → essa é a ÚNICA situação em que o PROGRAMA
    de fato fez algo ERRADO
```

> ✅ Todos os **outros** casos (ausência leve, falta de página menor, falta de página maior) são corrigidos **automaticamente** pelo hardware e/ou pelo sistema operacional — ao custo de um pouco de desempenho. Apenas a falta de segmentação indica um erro genuíno do programa.

---

# ✅ Resumo do Conceito

- Dois problemas fundamentais motivam as otimizações de paginação: o mapeamento virtual-físico precisa ser **rápido** (feito em toda referência de memória) e a tabela de páginas não pode ser **grande demais** mesmo para espaços de endereçamento enormes
- **Tabela de registradores de hardware** (uma entrada por página virtual): rápida durante a execução, mas **inviável** para tabelas grandes e **arruína o desempenho** em trocas de contexto, por exigir o recarregamento de toda a tabela
- **Tabela inteira na memória principal**, com um único registrador apontando para seu início: troca de contexto barata, mas **lenta** por exigir referências extras de memória a cada instrução
- **TLB (translation lookaside buffer)**, ou memória associativa: um pequeno cache de hardware (tipicamente ~64 entradas) dentro da MMU que guarda as traduções **mais usadas**, explorando o fato de que a maioria dos programas concentra acessos em poucas páginas. Em uma **presença** no TLB, o quadro de página é obtido diretamente, sem tocar a tabela de páginas na memória
- Em uma **ausência** no TLB, o hardware (ou, em arquiteturas RISC como SPARC/MIPS/HP PA, o **software** do SO) busca a tradução na tabela de páginas, substitui uma entrada do TLB e reinicia a instrução — processo chamado de **passeio na tabela de páginas**
- Ausências de TLB são **muito mais frequentes** que faltas de página (milhares de páginas na memória vs. ~64 entradas no TLB), o que torna viável o gerenciamento por software quando o TLB é moderadamente grande — trocando uma MMU mais simples por mais espaço de chip para caches
- As ausências se dividem em **ausência leve** (página está na memória, só falta no TLB — rápida, nanossegundos), **falta de página menor** (página está na memória mas não na tabela do processo), **falta de página maior** (página precisa ser buscada em disco/SSD — milissegundos) e **falta de segmentação** (acesso a endereço inválido — único caso em que o programa errou de fato, e o SO o encerra)

---

## 🔗 Notas Relacionadas

- [[Tabelas de Páginas]] — a estrutura (3.3.2) que o TLB existe para acelerar
- [[Paginação]] — o mecanismo básico de tradução de endereços (3.3.1) que esta seção otimiza
- [[Memória Virtual]] — a abstração conceitual introduzida no Cap. 1
