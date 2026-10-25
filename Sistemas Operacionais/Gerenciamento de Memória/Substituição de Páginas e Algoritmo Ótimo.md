---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seções 3.4 e 3.4.1"
---
# Substituição de Páginas e Algoritmo Ótimo

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seções 3.4 e 3.4.1

---

# 🔁 3.4 — Algoritmos de Substituição de Páginas

## O problema

Quando ocorre uma **falta de página** ([[Acelerando a Paginação]]), a memória física pode estar cheia. O SO precisa **escolher uma página para remover** e abrir espaço para a que está chegando.

```
SE a página escolhida FOI MODIFICADA (está "suja"):
  → precisa ser REESCRITA no armazenamento não
    volátil (disco/SSD) para atualizar a cópia lá

SE a página escolhida NÃO foi modificada ("limpa"),
  p. ex. uma página de código do programa:
  → a cópia em disco/SSD JÁ está atualizada
  → NÃO precisa reescrever
  → a página nova simplesmente SOBRESCREVE a
    que está sendo removida
```

> 💡 É por isso que o **bit modificada (M)** da entrada da tabela de páginas ([[Tabelas de Páginas]]) importa: ele diz se remover a página custa uma escrita em disco ou não.

## Por que a escolha importa

```
É possível escolher uma página AO ACASO para ser
descartada a cada falta.

MAS o desempenho é MUITO MELHOR se for escolhida
uma página que NÃO é muito utilizada.

SE for removida uma página MUITO utilizada:
  → ela provavelmente terá de ser TRAZIDA DE
    VOLTA logo
  → resultando em CUSTO EXTRA (outra falta de
    página, possivelmente outra escrita em disco)
```

## O mesmo problema em outras áreas

O problema da "substituição" **não é exclusivo de memória virtual**. Aparece sempre que há um espaço pequeno e rápido servindo de cache para um grande e lento:

| Onde | O que é substituído | Escala de tempo |
|---|---|---|
| **Memória virtual** | páginas na RAM | milissegundos (vem de disco/SSD) |
| **Caches de memória da CPU** | blocos de 32 ou 64 bytes | **nanossegundos** (vem da memória principal) |
| **Servidor web** (cache de páginas web) | páginas web mantidas em memória | depende do servidor |

```
Cache de CPU:
  → mesmo problema da substituição de páginas,
    mas em escala de tempo MUITO MENOR (alguns
    nanossegundos, não dezenas de microssegundos
    ou milissegundos)
  → razão: as ausências na cache são atendidas
    pela MEMÓRIA PRINCIPAL, muito mais rápida que
    um disco magnético ou SSD

Servidor web:
  → mantém certo número de páginas web intensamente
    usadas em memória cache
  → cache cheia + nova página referenciada → decidir
    qual remover (considerações semelhantes)
  → DIFERENÇA: páginas web NUNCA são modificadas na
    cache, então SEMPRE há uma cópia atualizada
    "no armazenamento não volátil"

Em memória virtual:
  → páginas na memória principal podem estar
    LIMPAS ou SUJAS
```

## Uma questão em aberto: escopo da escolha

Em todos os algoritmos que serão vistos, surge a pergunta:

```
Quando uma página for removida, ela deve ser:

  (a) uma das páginas do PRÓPRIO PROCESSO que
      causou a falta?
      → cada processo fica limitado a um número
        FIXO de páginas

  (b) ou pode pertencer a OUTRO processo?
      → sem essa limitação

Ambas são possibilidades.
O livro volta a este ponto na Seção 3.5.1.
```

---

# 🏆 3.4.1 — O Algoritmo Ótimo de Substituição de Página

## A ideia (fácil de descrever)

```
No momento em que ocorre uma falta de página, há
um determinado conjunto de páginas na memória.

  → UMA delas será referenciada na PRÓXIMA instrução
    (a página que contém essa instrução)

  → outras talvez só sejam referenciadas daqui a
    10, 100 ou 1.000 instruções — ou talvez NUNCA
    (p. ex. uma fase de inicialização já concluída)

Cada página pode ser ROTULADA com o NÚMERO DE
INSTRUÇÕES que serão executadas antes de ela ser
referenciada pela primeira vez.
```

> ✅ **Regra do algoritmo ótimo:** remover a página com o **MAIOR rótulo**.

```
EXEMPLO:
  Página X não será usada por 8 milhões de instruções
  Página Y não será usada por 6 milhões de instruções

  → remover X adia a próxima falta de página ao
    máximo

Computadores, assim como pessoas, tentam ADIAR AO
MÁXIMO a ocorrência de eventos desagradáveis.
```

## O problema: é irrealizável

```
No momento da falta de página, o SO NÃO TEM COMO
SABER quando cada página será referenciada em
seguida.

(Situação parecida com o escalonamento "tarefa mais
curta primeiro": como o sistema pode dizer qual
tarefa é a mais curta? — veja
[[Escalonamento em Sistemas em Lote]])
```

O algoritmo ótimo precisa do **futuro**, que o SO não conhece.

## Então para que serve?

Mesmo irrealizável em um sistema real, dá para usá-lo como **régua de comparação**:

```
1. Execute o programa em um SIMULADOR e registre
   TODAS as referências a páginas (1ª execução)

2. Na 2ª execução, use essas informações coletadas
   para implementar o algoritmo ótimo

3. Compare o desempenho dos algoritmos REALIZÁVEIS
   com o desempenho do MELHOR POSSÍVEL
```

> 💡 **Exemplo do livro:** se um SO atinge desempenho apenas **1% pior** que o ótimo, o esforço de procurar um algoritmo melhor resultará em uma melhoria de **no máximo 1%**.

⚠️ **Ressalva importante:** o registro de referências vale somente para **aquele programa** e **aqueles dados de entrada** medidos. O algoritmo derivado dele é específico a eles. Serve para **avaliar** outros algoritmos, mas **não tem utilidade prática** em sistemas reais. Os algoritmos das próximas notas **são** realmente úteis em sistemas reais.

---

# ✅ Resumo do Conceito

- Em uma **falta de página** com memória cheia, o SO escolhe uma página a remover. Se ela estiver **suja (M=1)**, precisa ser gravada em disco antes; se estiver **limpa**, é só sobrescrita
- Remover uma página **muito usada** gera custo extra, pois ela provavelmente será trazida de volta logo
- O mesmo problema ocorre em **caches de CPU** (escala de nanossegundos) e **caches de servidores web** (páginas web nunca ficam sujas)
- O **algoritmo ótimo** remove a página que **demorará mais para ser usada novamente**. É **irrealizável** porque exige conhecer o futuro, mas serve de **referência de comparação** em simuladores
- Questão em aberto: remover só de dentro do processo que causou a falta (escopo local) ou de qualquer processo (escopo global) → Seção 3.5.1

---

## 🔗 Notas Relacionadas

- [[Não Usada Recentemente (NRU)]] — primeiro algoritmo realizável, baseado nos bits R e M
- [[FIFO, Segunda Chance e Relógio]] — família de algoritmos baseados na ordem de chegada
- [[LRU e Envelhecimento]] — aproximação realizável do algoritmo ótimo
- [[Conjunto de Trabalho]] — algoritmo baseado no conceito de localidade
- [[Tabelas de Páginas]] — onde estão os bits R e M usados por esses algoritmos
- [[Acelerando a Paginação]] — o tratamento de faltas de página que dispara a substituição
