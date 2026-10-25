---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seções 3.4.3, 3.4.4 e 3.4.5"
---
# FIFO, Segunda Chance e Relógio

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seções 3.4.3, 3.4.4 e 3.4.5

> 💡 Estes três algoritmos formam uma **família**: o FIFO é a base, a **segunda chance** corrige seu defeito usando o bit R, e o **relógio** é a mesma ideia da segunda chance implementada de forma mais eficiente.

---

# 🚪 3.4.3 — FIFO (First In, First Out)

Outro algoritmo de paginação de **baixo custo**: **primeiro a entrar, primeiro a sair**.

## A analogia do supermercado

```
Um supermercado tem prateleiras para exibir
exatamente k produtos diferentes. Um dia, uma
empresa lança um novo alimento (um iogurte orgânico,
seco e congelado, de reconstituição instantânea em
forno de micro-ondas) e faz sucesso imediato.

O supermercado precisa se LIVRAR de um produto
antigo para estocá-lo.

Possibilidade: descobrir qual produto está há MAIS
TEMPO no estoque (algo que começou a vender há 120
anos) e descartá-lo, supondo que ninguém mais tem
interesse nele.

Na prática, o supermercado mantém uma LISTA
ENCADEADA de todos os produtos, na ORDEM em que
foram introduzidos:
  → produto novo vai para o FIM da lista
  → o que está em PRIMEIRO na lista é REMOVIDO
```

## Aplicado a páginas

```
O SO mantém uma LISTA de todas as páginas
atualmente na memória:

  FRENTE  ← página mais ANTIGA
  FIM     ← página mais RECENTE

Em uma FALTA DE PÁGINA:
  → a página da FRENTE é REMOVIDA
  → a nova página é acrescentada ao FIM da lista
```

## O defeito do FIFO

```
Aplicado a LOJAS, o FIFO pode remover a cera para
bigodes... mas também poderia remover a farinha, o
sal ou a manteiga.

Nos COMPUTADORES o problema é o mesmo: a página
mais ANTIGA ainda pode ser MUITO ÚTIL.

Por essa razão, o FIFO em sua forma MAIS PURA
raramente é usado.
```

---

# 🎟️ 3.4.4 — Segunda Chance

## A modificação simples

Uma pequena alteração no FIFO que **evita jogar fora uma página intensamente usada**: inspecionar o **bit R** da página mais antiga.

```
Olhar a página mais ANTIGA (frente da lista):

  SE R = 0:
    → a página é VELHA e pouco utilizada
    → é SUBSTITUÍDA imediatamente

  SE R = 1:
    → o bit R é LIMPO (volta a 0)
    → a página é colocada no FIM da lista
    → seu TEMPO DE CARREGAMENTO é atualizado, como
      se ela tivesse acabado de chegar à memória
    → a pesquisa CONTINUA (próxima página mais antiga)
```

> 💡 Daí o nome: uma página antiga que foi usada recentemente ganha uma **segunda chance** de permanecer.

## 📌 Figura 3.15 — Exemplo

```
(a) Páginas na ordem FIFO
    (os números são seus TEMPOS DE CARREGAMENTO)

 Carregada primeiro                    Carregada mais
       │                               recentemente
       ▼                                     ▼
     ┌───┬───┬───┬───┬────┬────┬────┬────┐
     │ A │ B │ C │ D │ E  │ F  │ G  │ H  │
     └───┴───┴───┴───┴────┴────┴────┴────┘
       0   3   7   8   12   14   15   18

(b) Falta de página no TEMPO 20, com R de A = 1

     ┌───┬───┬───┬────┬────┬────┬────┬────┐
     │ B │ C │ D │ E  │ F  │ G  │ H  │ A  │
     └───┴───┴───┴────┴────┴────┴────┴────┘
       3   7   8   12   14   15   18   20
                                        ▲
                     A é tratada como uma página
                     RECÉM-CARREGADA (tempo 20)
```

```
Suponha uma falta de página no instante 20.

A página mais antiga é A (chegou no instante 0).

  SE R de A = 0:
    → A é removida da memória (gravada em disco se
      estiver SUJA, ou simplesmente ABANDONADA se
      estiver LIMPA)

  SE R de A = 1:
    → A vai para o FINAL da lista
    → seu "tempo de carregamento" vira o momento
      atual (20)
    → o bit R também é ZERADO
    → a busca continua com B
```

## Termina sempre?

```
O que o algoritmo faz: procurar uma página ANTIGA
que NÃO tenha sido referenciada no intervalo de
relógio mais recente.

SE TODAS as páginas tiverem sido referenciadas:
  → a segunda chance se transforma em um FIFO PURO

  Imagine todas as páginas da Figura 3.15(a) com
  R = 1. Uma a uma, o SO as move para o final da
  lista, zerando R a cada vez. Por fim, a lista
  volta à página A, que agora tem R = 0 → A é
  removida.

Assim, o algoritmo SEMPRE TERMINA.
```

---

# 🕐 3.4.5 — O Algoritmo do Relógio

## O problema da segunda chance

Embora razoável, a segunda chance é **desnecessariamente ineficiente**: está sempre **movendo páginas** ao redor de sua lista.

## A solução: lista circular

```
Manter TODOS os quadros de página em uma LISTA
CIRCULAR, na forma de um RELÓGIO.

Um PONTEIRO aponta para a página MAIS ANTIGA.

              A
          L       B
        K           C  ◄── ponteiro
        J           D
          I       E
            H G F
```

> 📌 **Figura 3.16 — O algoritmo do relógio.** Quando ocorre uma falta de página, a página indicada pelo ponteiro é inspecionada. R = 0: remover a página. R = 1: zerar R e avançar o ponteiro.

```
Em uma FALTA DE PÁGINA, a página apontada é
inspecionada:

  SE R = 0:
    → a página é REMOVIDA
    → a nova página é inserida no relógio NO LUGAR
      dela
    → o ponteiro AVANÇA uma posição

  SE R = 1:
    → R é ZERADO
    → o ponteiro avança para a PRÓXIMA página

Esse processo se repete até ser encontrada uma
página com R = 0.
```

Sem muita surpresa, esse algoritmo é chamado de **relógio**.

## Segunda chance vs. relógio

| | Segunda chance | Relógio |
|---|---|---|
| **Estrutura** | lista encadeada linear | lista **circular** |
| **Quando a página tem R = 1** | é **movida** para o fim da lista | só o **ponteiro** avança |
| **Comportamento** | mesma lógica | mesma lógica |
| **Eficiência** | move páginas sem necessidade | **sem mover nada** |

---

# ✅ Resumo do Conceito

- **FIFO:** lista de páginas por ordem de chegada; em uma falta, remove a **mais antiga**. Barato, mas pode remover páginas ainda muito úteis, por isso raramente é usado puro
- **Segunda chance:** FIFO + verificação do **bit R** da página mais antiga. Se R = 0 → remove. Se R = 1 → zera R, move a página ao fim da lista (como se acabasse de chegar) e continua. Se todas têm R = 1, vira um FIFO puro, mas **sempre termina**
- **Relógio:** a mesma lógica da segunda chance, mas com os quadros em uma **lista circular** e um ponteiro. Evita ficar movendo páginas pela lista; basta avançar o ponteiro

---

## 🔗 Notas Relacionadas

- [[Substituição de Páginas e Algoritmo Ótimo]] — o problema geral
- [[Não Usada Recentemente (NRU)]] — também usa o bit R, mas classifica por R e M
- [[LRU e Envelhecimento]] — olha para o **uso**, não para a ordem de chegada
- [[Tabelas de Páginas]] — onde fica o bit R
