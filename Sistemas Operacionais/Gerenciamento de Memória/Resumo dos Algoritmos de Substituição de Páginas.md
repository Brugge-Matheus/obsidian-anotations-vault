---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.4.10"
---
# Resumo dos Algoritmos de Substituição de Páginas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.4.10

---

# 📋 3.4.10 — Resumo dos Algoritmos de Substituição de Página

## 📌 Figura 3.21 — Algoritmos discutidos

| Algoritmo | Comentário |
|---|---|
| **Ótimo** | Não implementável, mas útil como um padrão de desempenho |
| **NRU** (não usada recentemente) | Aproximação muito rudimentar do LRU |
| **FIFO** (primeiro a entrar, primeiro a sair) | Pode descartar páginas importantes |
| **Segunda chance** | Algoritmo FIFO bastante melhorado |
| **Relógio** | Realista |
| **LRU** (usada menos recentemente) | Excelente algoritmo, porém difícil de ser implementado de maneira exata |
| **NFU** (não frequentemente usada) | Aproximação bastante rudimentar do LRU |
| **Envelhecimento** (*aging*) | Algoritmo eficiente que se aproxima bem do LRU |
| **Conjunto de trabalho** | Implementação um tanto cara |
| **WSClock** | Algoritmo bom e eficiente |

## Algoritmo por algoritmo

```
ÓTIMO
  → remove a página que será referenciada por ÚLTIMO
  → não há como saber qual será → NÃO pode ser usado
  → útil como MEDIDA-PADRÃO para avaliar os outros

NRU
  → divide as páginas em 4 CLASSES (bits R e M)
  → escolhe uma página ALEATÓRIA da classe mais baixa
  → fácil de implementar, mas MUITO RUDIMENTAR
  → há outros melhores

FIFO
  → controla a ordem de CARREGAMENTO em uma lista
  → remove a mais ANTIGA (trivial), mas ela pode
    ainda estar em USO → MÁ ESCOLHA

SEGUNDA CHANCE
  → FIFO + confere se a página está sendo USADA antes
    de removê-la; se estiver, é poupada
  → melhora MUITO o desempenho

RELÓGIO
  → simplesmente uma IMPLEMENTAÇÃO DIFERENTE da
    segunda chance
  → MESMAS propriedades de desempenho, mas leva um
    pouco MENOS TEMPO para executar

LRU
  → algoritmo EXCELENTE
  → mas NÃO pode ser implementado sem HARDWARE
    ESPECIAL; se ele não está disponível, não pode
    ser usado

NFU
  → tentativa RUDIMENTAR, não muito boa, de
    aproximar o LRU

ENVELHECIMENTO
  → aproximação MUITO MELHOR do LRU
  → implementável de maneira EFICIENTE
  → BOA ESCOLHA

CONJUNTO DE TRABALHO
  → desempenho RAZOÁVEL, mas de certa maneira CARO
    de implementar

WSCLOCK
  → variante que NÃO APENAS dá bom desempenho, como
    também é EFICIENTE de implementar
```

## Os dois melhores algoritmos "puros"

```
Como um todo, os dois melhores algoritmos "puros" são:

  → ENVELHECIMENTO  (baseado no LRU)
  → WSCLOCK         (baseado no conjunto de trabalho)

Os dois proporcionam um BOM DESEMPENHO de paginação
e podem ser implementados EFICIENTEMENTE.
```

> 💡 **A "família" de cada um:**
>
> - Linha **LRU** (passado recente): NRU → NFU → **envelhecimento**
> - Linha **FIFO/relógio**: FIFO → segunda chance → relógio
> - Linha **conjunto de trabalho**: conjunto de trabalho → **WSClock** (que também herda o relógio)

## Na prática: sistemas reais usam variantes

```
Na prática, os sistemas operacionais podem
implementar suas PRÓPRIAS VARIANTES de algoritmos de
substituição de página.

WINDOWS:
  → combina elementos de DIFERENTES políticas
    (relógio/LRU e conjunto de trabalho)
  → usa estratégias diferentes para:
      • substituição LOCAL  (remoção de páginas
        apenas DESTE processo)
      • substituição GLOBAL (remoção de páginas de
        QUALQUER lugar)
  → e ainda varia a substituição com base no
    HARDWARE subjacente

LINUX (em 2008):
  → adotou uma solução de substituição LRU DIVIDIDA
  → mantém LISTAS LRU SEPARADAS para:
      • páginas com conteúdo de ARQUIVO
      • páginas contendo dados "ANÔNIMOS" (isto é,
        não referentes a arquivos)
  → MOTIVO: essas páginas geralmente têm PADRÕES DE
    USO DIFERENTES, e a probabilidade de páginas
    anônimas serem REUTILIZADAS é MUITO MAIOR
```

> 📝 A distinção **local vs. global** (se a vítima pode ou não ser de outro processo) foi levantada na introdução de [[Substituição de Páginas e Algoritmo Ótimo]] e é retomada pelo livro na Seção 3.5.1.

---

# ✅ Resumo do Conceito

- **Ótimo:** padrão de referência, irrealizável. **NRU, NFU:** aproximações rudimentares do LRU. **FIFO:** pode descartar páginas importantes. **Segunda chance:** FIFO bastante melhorado; **relógio:** mesma coisa, implementação mais rápida
- **LRU:** excelente, mas exige hardware especial. **Envelhecimento:** boa aproximação do LRU, eficiente, boa escolha. **Conjunto de trabalho:** razoável, mas caro. **WSClock:** bom e eficiente
- Os **dois melhores "puros"** são o **envelhecimento** (família LRU) e o **WSClock** (família conjunto de trabalho)
- SOs reais usam **variantes e combinações**: o **Windows** combina relógio/LRU e conjunto de trabalho, com substituição local e global; o **Linux** (2008) separa listas LRU para páginas de **arquivo** e **anônimas**, porque as anônimas têm mais chance de ser reutilizadas

---

## 🔗 Notas Relacionadas

- [[Substituição de Páginas e Algoritmo Ótimo]] — o problema geral e o algoritmo ideal
- [[Não Usada Recentemente (NRU)]] — 3.4.2
- [[FIFO, Segunda Chance e Relógio]] — 3.4.3 a 3.4.5
- [[LRU e Envelhecimento]] — 3.4.6 e 3.4.7
- [[Conjunto de Trabalho]] — 3.4.8
- [[WSClock]] — 3.4.9
