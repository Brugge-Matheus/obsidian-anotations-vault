---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.2"
---
# Controle de Carga

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.2

---

# ⚖️ 3.5.2 — Controle de Carga

## O problema: thrashing mesmo com tudo "certo"

Mesmo com o **melhor algoritmo de substituição de páginas** ([[Resumo dos Algoritmos de Substituição de Páginas]]) e uma **ótima alocação global** de quadros entre os processos ([[Alocação de Quadros e PFF]]), ainda pode ocorrer **thrashing** (ultrapaginação, veja [[Conjunto de Trabalho]]).

```
SEMPRE que os conjuntos de trabalho COMBINADOS de
TODOS os processos EXCEDEM a capacidade da memória,
o THRASHING pode ser esperado.
```

### Um sintoma clássico (no PFF)

```
O algoritmo PFF indica que:

  → ALGUNS processos precisam de MAIS memória
  → mas NENHUM processo precisa de MENOS memória

Nesse caso:
  → NÃO há maneira de dar mais memória àqueles que
    precisam dela SEM PREJUDICAR alguns outros

  → a ÚNICA SOLUÇÃO REAL é LIVRAR-SE TEMPORARIAMENTE
    de alguns processos
```

> 💡 Em outras palavras: o problema deixou de ser **como dividir** a memória; agora **simplesmente não há memória suficiente** para todos os processos ao mesmo tempo. Resta **reduzir a carga** (o número de processos competindo pela memória). Daí o nome: **controle de carga**.

---

## 🔪 Opção 1 — Eliminar processos (OOM killer)

```
A SOLUÇÃO MAIS SIMPLES É DIRETA: ELIMINAR alguns
processos.
```

Os sistemas operacionais muitas vezes têm um processo especial chamado **OOM** (*out of memory killer* — exterminador de falta de memória).

```
O OOM é ATIVADO quando o sistema está com POUCA
MEMÓRIA.

O que ele faz:

  1. REVISA todos os processos em execução

  2. ATRIBUI a cada um uma PONTUAÇÃO que indica o
     quão "RUIM" ele é

       → usar MUITA memória AUMENTA a pontuação
         de um processo ruim

       → processos IMPORTANTES (como processos-raiz
         e do sistema) obtêm pontuações BAIXAS

  3. Tenta MINIMIZAR o número de processos a serem
     ENCERRADOS (enquanto ainda libera memória
     suficiente)

  4. Depois de considerar todos os processos,
     EXTERMINA o processo (ou processos) com
     PONTUAÇÃO MAIS ALTA, liberando seus recursos
     para manter o sistema funcionando
```

> ⚠️ É uma solução **drástica**: o processo eliminado **perde todo o seu trabalho**. Por isso a pontuação procura sacrificar quem consome muita memória e protege processos essenciais.

---

## 💾 Opção 2 — Levar processos para o armazenamento (swapping)

Uma maneira **muito mais amigável** de reduzir o número de processos competindo pela memória:

```
LEVAR alguns deles para o DISCO ou SSD e LIBERAR
todas as páginas que eles estão segurando.

EXEMPLO:
  → um processo é levado para o disco/SSD
  → seus quadros de página são DIVIDIDOS entre
    outros processos que estão sofrendo thrashing

SE o thrashing PARAR:
  → o sistema pode executar dessa maneira por um
    tempo

SE NÃO PARAR:
  → OUTRO processo tem de ser retirado, e assim por
    diante, até o thrashing CESSAR
```

> ✅ **Conclusão do livro:** mesmo com a paginação, a **troca de processos** ([[Troca de Processos (Swapping)]]) talvez ainda seja necessária. Mas agora ela é usada para **reduzir a demanda potencial por memória**, em vez de para reivindicar páginas. Assim, **paginação e troca de processos não são mutuamente contraditórias**.

### Relação com o escalonamento de dois níveis

```
A ideia de trocar processos para o armazenamento
não volátil, a fim de aliviar a carga sobre a
memória, é REMINISCENTE do ESCALONAMENTO DE DOIS
NÍVEIS:

  → alguns processos são colocados em armazenamento
    não volátil
  → um ESCALONADOR DE CURTO PRAZO é usado para
    escalonar os processos RESTANTES
```

Claramente, as duas ideias podem ser **combinadas**: remove-se **apenas um número suficiente** de processos do disco/SSD para tornar **aceitável** a frequência de faltas de páginas. **Periodicamente**, alguns processos são trazidos do armazenamento para a memória e outros são levados para ele. (Veja [[Introdução ao Escalonamento]].)

### O outro lado: o grau de multiprogramação

```
CUIDADO: outro fator a considerar é o GRAU DE
MULTIPROGRAMAÇÃO.

Quando o número de processos na memória principal é
BAIXO DEMAIS, a CPU pode ficar OCIOSA por períodos
SUBSTANCIAIS.
(veja [[Modelando a Multiprogramação]])
```

Por isso, ao decidir **qual processo trocar**, não basta olhar para o **tamanho** dos processos e a **frequência de paginação**. Também contam suas **características**:

- se o processo é do tipo **vinculado à CPU** (*CPU-bound*) ou **à E/S** (*I/O-bound*)
- quais características os **processos restantes** têm

---

## 🗜️ Outras opções: reduzir o uso de memória

Exterminar e trocar **não são as únicas opções**. Outra solução comum é **reduzir o uso de memória por compactação e compressão**. De fato, **reduzir o consumo de memória** de um sistema está **no topo da lista de prioridades** dos projetistas de SO em qualquer lugar.

### Deduplicação (mesclagem da mesma página)

Uma técnica **inteligente e muito utilizada**, conhecida como **deduplicação**, ou **mesclagem da mesma página** (*page merging*):

```
A IDEIA É SIMPLES:

  Varrer PERIODICAMENTE a memória para ver se DUAS
  PÁGINAS (possivelmente em processos DIFERENTES)
  têm EXATAMENTE O MESMO CONTEÚDO.

SE tiverem:
  → em vez de armazenar esse conteúdo em DOIS
    quadros de página físicos
  → o SO REMOVE uma das duplicatas
  → e MODIFICA os mapeamentos da tabela de páginas
    para que haja agora DUAS páginas VIRTUAIS que
    apontam para o MESMO QUADRO
```

```
   ANTES                          DEPOIS

 Processo X                     Processo X
 pág virtual 5 ──► quadro 12    pág virtual 5 ──┐
                                                ├──► quadro 12
 Processo Y                     Processo Y      │     (compartilhado,
 pág virtual 9 ──► quadro 40    pág virtual 9 ──┘      somente leitura)

                                quadro 40 → LIVRE
```

### Cópia na escrita (copy on write)

```
O quadro compartilhado é protegido com a CÓPIA NA
ESCRITA (copy on write):

  → assim que um processo TENTA ESCREVER na página,
    UMA NOVA CÓPIA é feita

  → para que a escrita NÃO AFETE a outra página
```

> 📝 **Curiosidade de linguagem:** algumas pessoas chamam esse processo de "**desdeduplicação**". O autor observa que, embora não estejam erradas, esse tratamento **fere o uso da nossa linguagem natural**.

---

# ✅ Resumo do Conceito

- Mesmo com ótimo algoritmo de substituição e ótima alocação global, ocorre **thrashing** quando os **conjuntos de trabalho combinados** excedem a memória
- **Sintoma no PFF:** alguns processos precisam de mais memória e **nenhum precisa de menos**. A única solução real é **reduzir a carga**: livrar-se temporariamente de alguns processos
- **Eliminar processos:** o **OOM killer** pontua os processos (muito uso de memória = pontuação alta; processos-raiz e do sistema = baixa) e mata o de maior pontuação, minimizando o número de mortos
- **Trocar processos para o disco/SSD:** libera suas páginas para os outros, parando o thrashing. Usa-se o swapping para **reduzir a demanda por memória**, então paginação e swapping **não são contraditórias**. Lembra o **escalonamento de dois níveis**
- Cuidado com o **grau de multiprogramação**: processos de menos deixam a CPU ociosa. Ao escolher quem trocar, consideram-se também as características do processo (**CPU-bound ou I/O-bound**) e dos restantes
- **Outras opções:** compactação e compressão, e **deduplicação** (páginas idênticas passam a apontar para o mesmo quadro físico, protegido por **copy on write**)

---

## 🔗 Notas Relacionadas

- [[Alocação de Quadros e PFF]] — o PFF cujo sintoma ("alguns precisam de mais, nenhum de menos") indica a necessidade do controle de carga
- [[Alocação Local versus Global]] — as políticas de substituição que precedem este problema
- [[Conjunto de Trabalho]] — a soma dos conjuntos de trabalho e o thrashing
- [[Troca de Processos (Swapping)]] — o mecanismo de levar processos inteiros ao disco, agora usado para reduzir a demanda
- [[Modelando a Multiprogramação]] — grau de multiprogramação e utilização da CPU
- [[Introdução ao Escalonamento]] — escalonamento em dois níveis, processos CPU-bound e I/O-bound
