---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.4.2"
---
# Não Usada Recentemente (NRU)

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.4.2

---

# 📊 3.4.2 — O Algoritmo de Substituição de Páginas Não Usadas Recentemente

## Os bits de status R e M

Para o SO coletar estatísticas úteis sobre o uso das páginas, a maioria dos computadores com memória virtual tem **dois bits de status** associados a cada página (já vistos na Figura 3.11, em [[Tabelas de Páginas]]):

```
R (referenciada):
  → definido SEMPRE que a página é REFERENCIADA
    (lida ou escrita)

M (modificada):
  → definido quando a página é ESCRITA
    (isto é, modificada)

Ambos ficam em cada ENTRADA da tabela de páginas.
```

> ⚠️ Esses bits precisam ser atualizados em **cada referência de memória**, então é essencial que sejam definidos pelo **hardware**. Uma vez que um bit vai a 1, ele **fica em 1** até que o SO o passe para 0.

### E se o hardware não tem esses bits?

Podem ser **simulados** com os mecanismos de falta de página e interrupção de relógio do SO:

```
1. Quando um processo é INICIALIZADO, TODAS as
   entradas da tabela de páginas são marcadas como
   NÃO PRESENTES na memória

2. Tão logo qualquer página é REFERENCIADA, ocorre
   uma FALTA DE PÁGINA

3. O SO então:
   → coloca R = 1 (em suas tabelas internas)
   → muda a entrada da tabela de páginas para
     apontar para a página correta, em modo
     SOMENTE LEITURA
   → REINICIA a instrução

4. SE a página for posteriormente MODIFICADA:
   → ocorre OUTRA falta de página (escrita em
     página somente-leitura)
   → o SO define M = 1 e muda a página para modo
     LEITURA/ESCRITA
```

## O algoritmo

Os bits R e M permitem construir um algoritmo simples:

```
1. Quando um processo é INICIALIZADO, os bits R e M
   de TODAS as suas páginas são definidos como 0

2. PERIODICAMENTE (p. ex. a cada interrupção de
   relógio), o bit R é LIMPO (volta a 0)
   → serve para DISTINGUIR páginas referenciadas
     recentemente das que não foram

3. Em uma FALTA DE PÁGINA, o SO inspeciona TODAS as
   páginas e as divide em QUATRO CLASSES, conforme
   os valores atuais de R e M
```

| Classe | R (referenciada) | M (modificada) | Significado |
|---|---|---|---|
| **0** | não | não | não referenciada, não modificada |
| **1** | não | **sim** | não referenciada, **modificada** |
| **2** | **sim** | não | referenciada, não modificada |
| **3** | **sim** | **sim** | referenciada, modificada |

> 💡 **"Classe 1 parece impossível?"** À primeira vista sim (como uma página modificada pode não ter sido referenciada?), mas ocorre quando uma página de **classe 3** tem seu **bit R limpo** por uma interrupção de relógio. As interrupções de relógio **não limpam o bit M**, porque essa informação é necessária para saber se a página precisa ser **reescrita no disco**. Limpar R mas não M leva uma página de classe 3 à classe 1. Em outras palavras: uma página de classe 1 é aquela que **foi modificada há muito tempo** e ainda não foi solicitada.

## A regra de escolha

```
O NRU (not recently used) remove uma página AO
ACASO da classe de ordem MAIS BAIXA que NÃO esteja
vazia.

Ordem de preferência para remoção:
  classe 0  →  1  →  2  →  3
```

**Ideia implícita:** é melhor remover uma página **modificada, mas não referenciada pelo menos por um tique do relógio** (em geral cerca de **20 ms**) do que uma página **não modificada que está sendo muito utilizada**.

## Prós e contras

```
PRINCIPAL ATRAÇÃO:
  → fácil de compreender
  → moderadamente eficiente de implementar
  → proporciona um desempenho que, embora NÃO ÓTIMO,
    pode ser ADEQUADO
```

---

# ✅ Resumo do Conceito

- Cada página tem os bits **R** (referenciada) e **M** (modificada), atualizados pelo **hardware** a cada acesso; se o hardware não os tem, o SO os simula usando faltas de página
- Periodicamente o SO **limpa R** (não M) para diferenciar páginas usadas recentemente das antigas
- Em uma falta de página, as páginas são classificadas em **4 classes** (0: nem R nem M; 1: só M; 2: só R; 3: R e M)
- O **NRU** remove uma página **aleatória da menor classe não vazia**: prefere sacrificar uma página modificada e parada a uma limpa e muito usada
- Simples, razoavelmente eficiente, desempenho não ótimo mas adequado

---

## 🔗 Notas Relacionadas

- [[Substituição de Páginas e Algoritmo Ótimo]] — o problema geral e o algoritmo ideal usado como referência
- [[FIFO, Segunda Chance e Relógio]] — algoritmos que também usam o bit R, mas olham para a ordem de chegada
- [[LRU e Envelhecimento]] — aproximações mais finas do "uso recente"
- [[Tabelas de Páginas]] — estrutura da entrada com os bits R e M (Figura 3.11)
