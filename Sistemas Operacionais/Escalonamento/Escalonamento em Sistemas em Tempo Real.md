---
tags:
  - sistemas-operacionais
  - so/escalonamento
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.5.4"
---
# Escalonamento em Sistemas de Tempo Real

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.5.4

---

# ⏱️ 2.5.4 — Escalonamento em Sistemas de Tempo Real

## O que define um sistema de tempo real

> 💡 **Sistema de tempo real:** aquele em que o **tempo** tem um papel essencial. Em geral, um ou mais dispositivos físicos externos ao computador geram estímulos, e o computador tem de reagir em conformidade dentro de um **período de tempo fixo**.

```
Exemplos concretos de estímulos externos:

  Aparelho de CD:
    → recebe os bits à medida que eles saem do drive
    → deve convertê-los em música dentro de um intervalo
      muito estrito
    → se o cálculo levar tempo demais, a música soará
      estranha

  UTI hospitalar:
    → monitoramento de pacientes

  Avião:
    → piloto automático

  Fábrica automatizada:
    → controle de robôs
```

> ⚠️ **A regra de ouro dos sistemas de tempo real:** em todos esses casos, **ter a resposta certa, mas tê-la tarde demais, é muitas vezes tão ruim quanto não a ter**. Não basta o sistema produzir o resultado correto — ele precisa produzi-lo **dentro do prazo**.

---

## 🔴 Tempo Real Crítico vs. Tempo Real Não Crítico

Sistemas em tempo real são geralmente categorizados em duas classes:

> 💡 **Tempo real crítico (*hard real-time*):** há prazos absolutos que **devem** ser cumpridos — se não, o sistema falha. Não há tolerância.

> 💡 **Tempo real não crítico (*soft real-time*):** descumprir um prazo ocasional é **indesejável**, mas mesmo assim **tolerável**. O sistema continua funcionando, apenas com qualidade degradada.

```
Em AMBOS os casos, o comportamento em tempo real é
conseguido dividindo o programa em uma SÉRIE DE
PROCESSOS, cada um dos quais tem comportamento
PREVISÍVEL e CONHECIDO antecipadamente.

Esses processos em geral têm VIDA CURTA e podem ser
concluídos em bem menos de UM SEGUNDO.
```

---

## 🔔 Eventos Periódicos e Aperiódicos

Quando um evento externo é detectado, cabe ao escalonador programar os processos de uma maneira que **todos os prazos sejam atendidos**.

```
Os eventos a que um sistema de tempo real talvez
tenha de responder podem ser categorizados como:

PERIÓDICOS:
  → ocorrem em INTERVALOS REGULARES

APERIÓDICOS:
  → ocorrem de maneira IMPREVISÍVEL
```

Um sistema pode ter de responder a **múltiplos fluxos** de eventos periódicos. Dependendo de quanto tempo cada evento exige para o processamento, tratar todos talvez **não seja possível**.

---

## 📐 A Escalonabilidade — o teste matemático

### A fórmula

Se há **m** eventos periódicos e o evento **i** ocorre com o período **Pᵢ** e exige **Cᵢ** segundos de tempo da CPU para lidar com cada evento, então a carga só pode ser tratada se:

```
        m   Cᵢ
        Σ  ──  ≤ 1
       i=1  Pᵢ
```

> 💡 **Escalonável:** diz-se de um sistema de tempo real que atende a esse critério. Isso significa que ele **realmente pode ser implementado**.

```
Um processo que fracassa em atender esse teste NÃO
pode ser escalonado, pois o montante total de tempo
de CPU que os processos querem COLETIVAMENTE é MAIOR
do que a CPU pode proporcionar.
```

### Exemplo numérico

```
Sistema de tempo real NÃO CRÍTICO com TRÊS eventos
periódicos:

  Períodos:  100ms, 200ms, 500ms
  Exigências: 50ms, 30ms, 100ms (de tempo de CPU)

Verificando a fórmula:

  50/100 + 30/200 + 100/500
  = 0,5 + 0,15 + 0,2
  = 0,85  ≤ 1  ✅

O sistema é ESCALONÁVEL.
```

```
Adicionando um QUARTO evento:

  Período: 1 segundo (1.000ms)

O sistema PERMANECERÁ escalonável DESDE QUE esse
evento não precise de mais de 150ms de tempo de CPU
por evento:

  0,85 + X/1000 ≤ 1
  X ≤ 150ms
```

> ⚠️ **Pressuposto implícito nesse cálculo:** o **overhead de troca de contexto** é tão pequeno que pode ser **ignorado**. Na prática, trocas de contexto sempre têm algum custo — mas para fins deste cálculo teórico, considera-se desprezível.

---

## 🧮 Estático vs. Dinâmico

Algoritmos de escalonamento de tempo real podem ser **estáticos** ou **dinâmicos**:

```
ESTÁTICOS:
  → tomam suas decisões de escalonamento ANTES de o
    sistema começar a ser executado

  → funciona APENAS quando há uma informação PERFEITA
    disponível ANTECIPADAMENTE sobre:
      - o trabalho a ser feito
      - os prazos que precisam ser cumpridos

DINÂMICOS:
  → tomam suas decisões no TEMPO DE EXECUÇÃO, após o
    sistema já ter começado

  → NÃO têm essas restrições — mais flexíveis, adaptam-se
    a mudanças imprevisíveis
```

---

# ✅ Resumo do Conceito

- Um sistema de **tempo real** é aquele em que o tempo tem papel essencial — estímulos externos exigem resposta dentro de um prazo fixo. Uma resposta correta, mas tardia, é tão ruim quanto nenhuma resposta
- **Tempo real crítico (hard):** prazos absolutos, sem tolerância a atraso. **Tempo real não crítico (soft):** atrasos ocasionais são toleráveis, ainda que indesejáveis
- O comportamento de tempo real é alcançado dividindo o programa em processos de **vida curta** e comportamento previsível
- Eventos podem ser **periódicos** (intervalos regulares) ou **aperiódicos** (imprevisíveis)
- Um sistema é **escalonável** se a soma das frações Cᵢ/Pᵢ de todos os eventos periódicos for **≤ 1** — ou seja, se a CPU consegue atender a todas as exigências coletivas dos eventos dentro dos seus períodos
- Algoritmos podem ser **estáticos** (decisões antes da execução, exigem informação perfeita antecipada) ou **dinâmicos** (decisões em tempo de execução, mais flexíveis)

---

## 🔗 Notas Relacionadas

- [[Introdução ao Escalonamento]] — a categoria "tempo real" já introduzida ali, com suas metas de cumprimento de prazos e previsibilidade
- [[Escalonamento em Sistemas Interativos]] — comparação com sistemas de propósito geral, que podem executar programas arbitrários e não cooperativos
- [[Escalonamento em Sistemas em Lote]] — contraste: sistemas em lote não têm restrição de prazo, apenas otimizam vazão e tempo de retorno médio
