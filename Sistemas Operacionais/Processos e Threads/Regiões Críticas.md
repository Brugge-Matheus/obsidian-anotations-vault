---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
  - so/sincronizacao
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.4.2"
---
# Regiões Críticas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.4.2

---

# 🔒 2.4.2 — Regiões Críticas

Na seção anterior ([[Race Condition]]), vimos que race conditions surgem quando dois ou mais processos acessam dados compartilhados de forma não coordenada. A pergunta natural é: **como evitar isso?**

A chave está em **proibir que mais de um processo leia e escreva os dados compartilhados ao mesmo tempo**. Em outras palavras, precisamos de **exclusão mútua** — alguma forma de garantir que se um processo está usando uma variável ou arquivo compartilhado, os outros serão impedidos de fazer a mesma coisa.

> 💡 **Exclusão mútua (*mutual exclusion*):** propriedade que garante que apenas um processo por vez possa estar acessando um recurso compartilhado ou executando uma determinada seção de código. É o mecanismo fundamental para prevenir race conditions.

O problema do spool ocorreu porque o processo B começou a usar uma das variáveis compartilhadas antes de o processo A ter terminado de usá-la. A escolha das operações adequadas para alcançar a exclusão mútua é uma **questão de projeto fundamental** em qualquer sistema operacional.

---

## 🗺️ O conceito de Região Crítica

O problema de evitar condições de corrida pode ser formulado de maneira abstrata.

Durante parte do tempo, um processo está ocupado realizando computações internas e outras coisas que **não levam a condições de corrida** — esse código é inofensivo. No entanto, às vezes um processo tem de acessar uma memória compartilhada ou arquivos, ou realizar outras tarefas críticas que **podem levar a corridas**. Essa parte do programa onde a memória compartilhada é acessada se chama:

> 💡 **Região crítica** (ou **seção crítica**): parte do código de um processo que acessa recursos compartilhados — memória, arquivos, dispositivos — e que, portanto, **não pode ser executada por dois processos ao mesmo tempo**. É a "zona de perigo" onde race conditions podem ocorrer.

Se conseguíssemos arranjar as coisas de modo que **jamais dois processos estivessem em suas regiões críticas ao mesmo tempo**, poderíamos evitar as corridas.

---

## 📋 As 4 condições para uma boa solução

Embora evitar que dois processos estejam simultaneamente em suas regiões críticas seja necessário, isso **não é suficiente** para garantir que processos em paralelo cooperem de modo correto e eficiente usando dados compartilhados. Precisamos que quatro condições se mantenham para chegar a uma boa solução:

```
┌─────────────────────────────────────────────────────────────────────┐
│              4 Condições para Exclusão Mútua Correta                │
├─────┬───────────────────────────────────────────────────────────────┤
│  1  │ Dois processos JAMAIS podem estar simultaneamente dentro      │
│     │ de suas regiões críticas.                                     │
├─────┼───────────────────────────────────────────────────────────────┤
│  2  │ Nenhuma suposição pode ser feita a respeito de velocidades    │
│     │ ou do número de CPUs.                                         │
├─────┼───────────────────────────────────────────────────────────────┤
│  3  │ Nenhum processo executando FORA de sua região crítica pode    │
│     │ bloquear qualquer outro processo.                             │
├─────┼───────────────────────────────────────────────────────────────┤
│  4  │ Nenhum processo deve ser obrigado a esperar ETERNAMENTE       │
│     │ para entrar em sua região crítica.                            │
└─────┴───────────────────────────────────────────────────────────────┘
```

**Analisando cada condição:**

**Condição 1 — Exclusão mútua estrita:** é a condição fundamental. Se violada, race conditions são possíveis. Dois processos na região crítica ao mesmo tempo = desastre.

**Condição 2 — Independência de velocidade e número de CPUs:** a solução não pode depender de "o processo A é sempre mais rápido que B" ou de "há exatamente 2 CPUs". Deve funcionar em qualquer hardware.

**Condição 3 — Não bloqueio por processo fora da região crítica:** um processo lento ou bloqueado *fora* de sua região crítica não deve impedir outros de entrar nas suas. Se o processo A trava em sua região não crítica, o processo B ainda deve poder usar a sua própria região crítica normalmente.

**Condição 4 — Sem espera eterna:** um processo que quer entrar em sua região crítica deve conseguir fazê-lo em tempo finito. Isso previne **starvation** — situação em que um processo nunca consegue acesso ao recurso que precisa.

---

## 📊 Visualizando o comportamento correto — Figura 2.22

> 📌 **Figura 2.22 — Exclusão mútua usando regiões críticas**

```
                 A entra na          A deixa a
                 região crítica      região crítica
                      │                   │
Processo A  ──────────┼───────────────────┼──────────────────────────►
                      │                   │
                      │         B tenta   B entra na   B deixa a
                      │         entrar    reg. crítica reg. crítica
                      │           │           │              │
Processo B  ──────────┼───────────┼───────────┼──────────────┼─────►
                      │           │           │              │
                     T₁          T₂          T₃             T₄

                               ◄──────────►
                               B bloqueado
                                (espera)

                          Tempo ──────────────────────────────►
```

**O que acontece:**

- Em **T₁**: o processo A entra em sua região crítica
- Em **T₂**: o processo B *tenta* entrar em sua região crítica, mas não consegue — outro processo já está em sua região crítica. B é **temporariamente suspenso**
- Em **T₃**: A deixa sua região crítica. B é liberado e **entra de imediato**
- Em **T₄**: B sai de sua região crítica — voltamos à situação original, sem nenhum processo em suas regiões críticas

Este comportamento satisfaz todas as 4 condições: nunca há dois processos simultaneamente na região crítica, B não espera eternamente, e A não é bloqueado por B quando B está fora de sua região crítica.

---

# ✅ Resumo do Conceito

- Race conditions ocorrem quando processos acessam dados compartilhados sem coordenação — a solução é a **exclusão mútua**
- A **região crítica** (ou seção crítica) é a parte do código que acessa recursos compartilhados e que deve ser protegida
- Uma boa solução de exclusão mútua deve satisfazer **4 condições**: (1) nunca dois processos simultâneos na região crítica, (2) independência de velocidade/CPU, (3) processos fora da região crítica não bloqueiam outros, (4) nenhuma espera eterna
- A pergunta que fica é: **como implementar** essa exclusão mútua na prática? Isso é o tema de [[Exclusão mútua com espera ocupada]]

---

## 🔗 Notas Relacionadas

- [[Race Condition]] — o problema que as regiões críticas resolvem
- [[Exclusão mútua com espera ocupada]] — implementações práticas: desabilitar interrupções, variáveis de trava, alternância, Peterson, TSL/XCHG
