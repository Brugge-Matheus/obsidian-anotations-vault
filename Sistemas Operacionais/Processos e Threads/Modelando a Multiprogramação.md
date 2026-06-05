---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.1.7"
---
# Modelando a Multiprogramação

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.1.7

---

# 📊 2.1.7 — Modelando a Multiprogramação

Quando a multiprogramação é usada, a utilização da CPU pode ser melhorada. Esta seção apresenta um **modelo probabilístico simples** para estimar quanto a CPU realmente trabalha em função do número de processos na memória.

---

## O problema: processos ficam esperando I/O

Um processo não passa o tempo todo usando a CPU. Grande parte do tempo ele está **bloqueado**, esperando por dispositivos de I/O — disco, teclado, rede. Enquanto espera, a CPU fica ociosa se não houver outro processo para executar.

A intuição imediata seria: "se cada processo usa 20% do tempo de CPU, bastam 5 processos para ocupar 100% da CPU". Isso está **errado** — porque essa lógica ignora que os processos esperam I/O em momentos diferentes e não de forma perfeitamente alternada.

---

## O modelo probabilístico

> 💡 **Fração de espera por** I/O **(p):** a fração do tempo que um processo passa bloqueado aguardando dispositivos de I/O. Por exemplo, p = 0,8 significa que o processo passa 80% do seu tempo esperando I/O e apenas 20% realmente usando a CPU.
> 

Com **n** processos na memória ao mesmo tempo, a probabilidade de que **todos** estejam esperando I/Osimultaneamente — deixando a CPU ociosa — é:

```
P(todos esperando I/O) = pⁿ
```

Portanto, a **utilização da CPU** é a probabilidade de que pelo menos um processo esteja pronto para executar:

```
Utilização da CPU = 1 − pⁿ
```

> 💡 **Grau de multiprogramação (n):** o número de processos mantidos na memória ao mesmo tempo. Quanto maior o grau, maior a chance de que sempre haja algum processo pronto — mas os ganhos diminuem progressivamente.
> 

---

## Figura 2.6 — Utilização da CPU em função do grau de multiprogramação

> 📌 **Figura 2.6:** Utilização da CPU como uma função do número de processos na memória (grau de multiprogramação), para três valores de p.
> 

```
Utilização
da CPU (%)
   100 |         ....••••••••••  ← 20% de espera de E/S (p = 0,2)
    90 |      ...
    80 |   ..         ...•••••• ← 50% de espera de E/S (p = 0,5)
    70 | ..        ...
    60 |.        ..
    50 |        .       ...•••• ← 80% de espera de E/S (p = 0,8)
    40 |       .      ..
    30 |      .      .
    20 |     .      .
    10 |    .      .
     0 +----+----+----+----+----+----+----+----+----+----+--→
       0    1    2    3    4    5    6    7    8    9   10
                        Grau de multiprogramação (n)
```

**Leitura do gráfico:**

- Processos que passam **pouco tempo esperando** I/O (p = 0,2) já atingem alta utilização com poucos processos na memória — pois quase sempre há um processo pronto
- Processos que passam **muito tempo esperando** I/O (p = 0,8) precisam de muito mais processos na memória para manter a CPU ocupada
- Em todos os casos, a curva tem **retornos decrescentes** — cada processo adicional traz menos ganho que o anterior

---

## Exemplo prático do livro

Suponha um computador com as seguintes características:

```
RAM total:        8 GB
SO + tabelas:     2 GB
Cada processo:    2 GB
─────────────────────────────────────
Processos na memória: (8 - 2) / 2 = 3 processos (grau de multiprog. = 3)
Espera de E/S média:  80% → p = 0,8
```

**Situação inicial (3 processos):**

```
Utilização = 1 - 0,8³ = 1 - 0,512 = 0,488 ≈ 49%
```

A CPU fica ociosa **51% do tempo** — um desperdício significativo.

**Adicionando 8 GB de RAM (7 processos):**

```
Processos = (16 - 2) / 2 = 7
Utilização = 1 - 0,8⁷ = 1 - 0,2097 = 0,790 ≈ 79%
Ganho: +30 pontos percentuais
```

**Adicionando mais 8 GB de RAM (11 processos):**

```
Processos = (24 - 2) / 2 = 11
Utilização = 1 - 0,8¹¹ = 1 - 0,0859 = 0,914 ≈ 91%
Ganho: +12 pontos percentuais
```

**Conclusão do exemplo:** o primeiro investimento em RAM (de 8 GB para 16 GB) aumentou a utilização em **30%** — claramente valeu a pena. O segundo investimento (de 16 GB para 24 GB) aumentou apenas **12%** — o proprietário pode concluir que a segunda adição não justificou o custo.

---

## A ressalva: o modelo é uma aproximação

> ⚠️ **O modelo probabilístico é uma simplificação.** Ele presume implicitamente que todos os n processos são **independentes** — que é bastante aceitável assumir para um sistema com cinco processos na memória, sendo três em execução e dois esperando.
> 

Mas com uma única CPU, **não podemos ter três processos sendo executados ao mesmo tempo**. Se três processos estão prontos e a CPU está ocupada com um, os outros dois têm de esperar. Portanto, os processos **não são independentes** na realidade.

Um modelo mais preciso poderia ser construído usando **teoria das filas**. Mas o ponto principal que o modelo sustenta permanece válido: **a multiprogramação deixa que os processos usem a CPU quando ela estaria em outras circunstâncias ociosa** — e isso é válido mesmo que as curvas reais sejam ligeiramente diferentes das mostradas na Figura 2.6.

---

# ✅ Resumo do Conceito

- **O problema:** processos ficam bloqueados esperando E/S a maior parte do tempo; sem multiprogramação, a CPU ficaria ociosa durante toda essa espera
- **Fração de espera (p):** proporção do tempo que um processo passa esperando E/S; quanto maior p, mais processos são necessários para manter a CPU ocupada
- **Fórmula:** `Utilização da CPU = 1 − pⁿ`, onde n é o grau de multiprogramação
- **Retornos decrescentes:** cada processo adicional na memória traz ganho de utilização, mas o ganho diminui progressivamente — os primeiros processos têm impacto muito maior
- **Aplicação prática:** o modelo permite decidir quanto de RAM adicionar; no exemplo, os primeiros 8 GB extras valeram +30%, os segundos +12% — o modelo orienta o investimento
- **Limitação:** o modelo assume processos independentes, o que não é estritamente verdade com uma única CPU; é uma aproximação útil, não um modelo exato