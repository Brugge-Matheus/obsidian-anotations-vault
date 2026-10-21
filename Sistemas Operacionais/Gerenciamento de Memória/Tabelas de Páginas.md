---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.3.2"
---
# Tabelas de Páginas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.3.2

---

# 🗂️ 3.3.2 — Tabelas de Páginas

## Revisão rápida: como o endereço virtual é dividido

Em uma implementação simples, o mapeamento de endereços virtuais em endereços físicos pode ser resumido assim:

```
O endereço VIRTUAL é dividido em DOIS campos:

  1. NÚMERO DA PÁGINA (bits mais significativos)
  2. DESLOCAMENTO (bits menos significativos)

EXEMPLO — endereço de 16 bits, páginas de 4 KB:

  → 4 bits SUPERIORES  = número da página
    (especificam 1 das 16 páginas virtuais)

  → 12 bits INFERIORES = deslocamento
    (especificam 1 dos 4.096 bytes dentro
     da página selecionada)
```

> 💡 Essa divisão **não é fixa**: uma divisão com 3 ou 5 (ou outro número qualquer) de bits para a página também é possível — divisões diferentes implicam **tamanhos de página diferentes**.

```
COMO O NÚMERO DA PÁGINA é usado:

  1. O número da PÁGINA VIRTUAL serve como um
     ÍNDICE dentro da tabela de páginas, a fim
     de ENCONTRAR a entrada correspondente a
     essa página virtual

  2. A partir dessa entrada, chega-se ao NÚMERO
     DO QUADRO (se ele existir)

  3. O número do quadro de página é ANEXADO aos
     bits MAIS SIGNIFICATIVOS do deslocamento,
     SUBSTITUINDO o número de página virtual

  4. Isso forma um ENDEREÇO FÍSICO, que pode ser
     enviado para a memória
```

> 💡 **O propósito da tabela de páginas, em uma frase:** mapear páginas virtuais em quadros de página. Matematicamente falando, a tabela de páginas é uma **função**, com o número da página virtual como **argumento** e o número do quadro físico como **resultado**. Usando o resultado dessa função, um campo de página virtual em um endereço virtual pode ser substituído por um campo de quadro de página, formando assim um endereço de memória física.

---

## 📌 Figura 3.11 — Estrutura de uma entrada da tabela de páginas

O leiaute exato de uma entrada na tabela de páginas depende bastante da máquina, mas o **tipo de informação presente** é mais ou menos o mesmo de uma máquina para outra. O tamanho varia de um computador para outro, mas **64 bits** é um tamanho comum nos computadores de uso geral atuais.

```
┌──────────────────────────────┬───┬───┬───┬───┬───┬───┐
│   Número do quadro de página   │ M │ P │Sup│ C │ R │P/A│
└──────────────────────────────┴───┴───┴───┴───┴───┴───┘
                                  │   │   │   │   │   │
                                  │   │   │   │   │   └─ Presente/ausente
                                  │   │   │   │   └───── Referenciada
                                  │   │   │   └───────── Caching desabilitado
                                  │   │   └───────────── Supervisor
                                  │   └───────────────── Proteção
                                  └───────────────────── Modificada
```

### Campo mais importante — número do quadro de página

```
O OBJETIVO do mapeamento de páginas é justamente
LOCALIZAR esse valor.

SE o tamanho da página for de 4 KB (isto é, 2¹²
bytes):
  → só precisamos dos 52 bits MAIS
    SIGNIFICATIVOS (em um endereço de 64 bits)
    para o número do quadro de página

  → isso deixa 12 bits para CODIFICAR outras
    informações sobre a página
```

### Bit presente/ausente

```
INDICA se a entrada é VÁLIDA e usada.

SE o bit for 0:
  → a página virtual à qual a entrada pertence
    NÃO está atualmente na memória

  → ACESSAR uma entrada da tabela de páginas
    com esse bit em 0 CAUSA uma FALTA DE PÁGINA
```

### Bits de proteção

```
Dizem quais TIPOS de acesso são permitidos.

FORMA MAIS SIMPLES:
  → 1 bit, com 0 = ler/escrever e 1 = ler
    somente

ARRANJO mais SOFISTICADO:
  → 3 bits, para habilitar LEITURA, ESCRITA e
    EXECUÇÃO da página separadamente
```

### Bit supervisor

```
INDICA se a página é acessível APENAS ao código
PRIVILEGIADO (o sistema operacional / supervisor)
ou TAMBÉM aos programas do USUÁRIO.

QUALQUER tentativa de um programa de usuário de
acessar uma página de SUPERVISOR resulta em uma
FALHA.
```

### Bits modificada e referenciada

```
BIT MODIFICADO (também chamado de bit SUJO
/ dirty bit):

  → é CONFIGURADO AUTOMATICAMENTE pelo hardware
    sempre que se ESCREVE na página

  → é IMPORTANTE quando o SO decide RECUPERAR um
    quadro de página:

      SE a página foi MODIFICADA (está "suja"):
        → ela DEVE ser escrita de volta no
          armazenamento não volátil antes de
          ser reaproveitada

      SE ela NÃO foi modificada (está "limpa"):
        → pode ser simplesmente ABANDONADA, já
          que a cópia em disco/SSD ainda é
          válida

BIT REFERENCIADA:

  → é MARCADO sempre que uma página é
    REFERENCIADA, seja para LEITURA ou para
    ESCRITA

  → seu valor AJUDA o SO a ESCOLHER uma página
    para ser SUBSTITUÍDA quando houver uma falta
    de página

  → páginas que NÃO estão sendo usadas são
    candidatas MUITO MELHORES do que as que
    ESTÃO sendo usadas — esse bit tem papel
    importante em vários ALGORITMOS DE
    SUBSTITUIÇÃO de páginas (vistos mais adiante
    neste capítulo)
```

### Bit de caching desabilitado

```
PERMITE que o CACHING seja DESABILITADO para a
página.

IMPORTANTE para páginas que são mapeadas em
REGISTRADORES DE DISPOSITIVOS, em vez de na
memória comum.

SE o SO está PARADO em um laço estreito esperando
que algum dispositivo de E/S RESPONDA a um
comando:

  → é FUNDAMENTAL que o hardware continue
    BUSCANDO a palavra do dispositivo

  → e NÃO use uma CÓPIA ANTIGA guardada em cache

COM esse bit, o caching pode ser DESABILITADO.

Máquinas com espaços de E/S SEPARADOS, que não
usam E/S mapeada em memória, NÃO precisam desse
bit.
```

### Por que o endereço de disco NÃO está na tabela de páginas

```
O endereço de DISCO (o endereço do bloco no disco
ou SSD) usado para ARMAZENAR uma página quando ela
NÃO está na memória NÃO faz parte da tabela de
páginas.

A RAZÃO é SIMPLES:

  A tabela de páginas armazena apenas as
  informações que o HARDWARE precisa para
  TRADUZIR um endereço virtual para um
  endereço físico.

  As informações que o SISTEMA OPERACIONAL
  precisa para LIDAR com faltas de página são
  mantidas em TABELAS DE SOFTWARE dentro do
  próprio SO.

  O HARDWARE não precisa dessas informações.
```

---

## 🧠 O que a memória virtual realmente representa

Antes de entrar em mais detalhes de implementação, vale reforçar o que a memória virtual **faz**, no fundo:

```
A memória virtual basicamente CRIA uma NOVA
ABSTRAÇÃO — o ESPAÇO DE ENDEREÇAMENTO —, que é
uma abstração da memória FÍSICA.

Isso é EXATAMENTE da mesma maneira que um
PROCESSO é uma abstração do PROCESSADOR FÍSICO
(CPU).
```

```
A memória virtual PODE ser implementada:

  DIVIDINDO-SE o espaço do endereço virtual em
  PÁGINAS

  e MAPEANDO cada uma delas em algum quadro de
  página da memória física — ou NÃO as mapeando
  (temporariamente)

Portanto, ela diz respeito BASICAMENTE à
ABSTRAÇÃO criada pelo sistema operacional e a
como essa abstração é GERENCIADA.
```

> 💡 **Todos** os acessos à memória feitos por **software** utilizam endereços **virtuais** — isso vale não só para os processos de usuário, mas também para o próprio **sistema operacional**. Em outras palavras, o núcleo (kernel) também tem seus próprios mapeamentos nas tabelas de páginas.

---

## ⚡ Mapeando as tabelas do SO dentro de cada processo

```
Sempre que um processo EXECUTA uma CHAMADA DE
SISTEMA, as tabelas de páginas do SISTEMA
OPERACIONAL devem ser utilizadas.

Como uma TROCA DE CONTEXTO (que exige a troca de
tabelas de páginas) tem ALTO CUSTO, alguns
sistemas empregam um TRUQUE INTELIGENTE:
```

```
O TRUQUE:

  Simplesmente MAPEIAM as tabelas de páginas do
  SISTEMA OPERACIONAL em CADA processo do
  usuário — mas com o BIT SUPERVISOR indicando
  que essas páginas SÓ podem ser acessadas pelo
  SISTEMA.

RESULTADO:

  SE o processo de usuário TENTAR acessar tal
  página diretamente:
    → DISPARARÁ uma EXCEÇÃO

  MAS quando o processo de usuário EXECUTA uma
  CHAMADA DE SISTEMA:
    → NÃO há mais necessidade de ALTERNAR as
      tabelas de páginas

  → TODAS as tabelas de páginas do NÚCLEO *e*
    as tabelas de páginas do USUÁRIO já estão
    DISPONÍVEIS para o sistema operacional

  → isso ACELERA a chamada de sistema
```

```
POSICIONAMENTO na memória:

  Em geral, quando o SO é mapeado nos processos
  de usuário, ele é mapeado no TOPO do espaço de
  endereçamento, para NÃO interferir nos
  programas de usuário — que começam em 0 ou
  perto disso.

  Às vezes, os programas de usuário COMEÇAM em
  4K em vez de 0, para que as referências ao
  endereço 0 (que GERALMENTE é um erro) sejam
  CAPTURADAS.
```

---

# ✅ Resumo do Conceito

- A **tabela de páginas** é, matematicamente, uma **função**: recebe o **número da página virtual** como argumento e devolve o **número do quadro de página** físico correspondente, que substitui o campo de página no endereço, mantendo o deslocamento intocado
- Uma **entrada típica** de tabela de páginas (Figura 3.11) carrega, além do **número do quadro de página** (o campo mais importante), vários bits de controle: **presente/ausente** (válida ou causa falta de página), **proteção** (permissões de leitura/escrita/execução), **supervisor** (acesso restrito ao SO), **modificada/dirty** (indica se a página precisa ser regravada em disco ao ser substituída), **referenciada** (ajuda algoritmos de substituição de páginas a identificar páginas pouco usadas) e **caching desabilitado** (essencial para páginas mapeadas em registradores de E/S)
- O **endereço de disco** de uma página ausente **não** fica na tabela de páginas — o hardware só precisa do que usa para traduzir endereços; o resto é controlado por tabelas de **software** do próprio SO
- Conceitualmente, a memória virtual cria uma **nova abstração** — o espaço de endereçamento — da mesma forma que um processo abstrai a CPU; e **todo** acesso a memória por software, inclusive do próprio kernel, usa endereços **virtuais**
- Como alternar tabelas de páginas em toda troca de contexto é caro, um truque comum é **mapear as tabelas de páginas do SO dentro de cada processo de usuário**, protegidas por um bit supervisor — isso evita a troca de tabelas durante **chamadas de sistema** (embora não evite a troca entre processos diferentes), e o SO costuma ficar mapeado no **topo** do espaço de endereçamento para não colidir com o espaço do usuário

---

## 🔗 Notas Relacionadas

- [[Paginação]] — o mecanismo de tradução de endereços (3.3.1) que a tabela de páginas implementa
- [[Memória Virtual]] — a abstração conceitual de espaço de endereçamento, introduzida no Cap. 1
- [[Proteção]] — conceito relacionado aos bits de proteção e supervisor da entrada da tabela de páginas
- [[Espaços de Endereçamento e Registradores-Base e Limite]] — mecanismo mais simples de proteção/tradução de endereços, que a paginação substitui
