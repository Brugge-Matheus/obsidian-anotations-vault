---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.2.6"
---
# Implementações Híbridas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.2.6

---

# 🔀 2.2.6 — Implementações Híbridas

Vimos que threads de usuário são rápidas mas têm problemas sérios com chamadas bloqueantes, e que threads de núcleo resolvem esses problemas mas têm custo alto de syscall. A ideia natural é: **combinar os dois modelos** para aproveitar o melhor de cada um.

> 💡 **Implementação híbrida de threads:** abordagem que usa threads de núcleo como base e permite multiplexar múltiplas threads de usuário em cima de cada thread de núcleo. O programador pode determinar quantas threads de núcleo usar e quantas threads de usuário multiplexar em cada uma. Esse modelo proporciona o máximo em flexibilidade.
> 

---

## Figura 2.16 — Multiplexando threads de usuário em threads de núcleo

> 📌 **Figura 2.16:** Multiplexando threads de usuário em threads de núcleo.
> 

```
┌──────────────────────────────────────────────────────────┐
│                    Espaço do usuário                     │
│                                                          │
│   Múltiplas threads de usuário em uma thread de núcleo   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Processo                            │   │
│  │   Thread U1  Thread U2  Thread U3  Thread U4    │   │
│  │      (λ)        (λ)        (λ)        (λ)        │   │
│  │       │          │          │          │         │   │
│  │       └────┬─────┘          └────┬─────┘         │   │
│  │            ▼                     ▼               │   │
│  │     Thread de núcleo 1    Thread de núcleo 2     │   │
│  └──────────────────────────────────────────────────┘   │
├──────────────────────────────────────────────────────────┤
│                      Núcleo                              │
│          Thread K1              Thread K2                │
└──────────────────────────────────────────────────────────┘
```

Com essa abordagem:

- O núcleo está ciente **apenas** das threads de núcleo e as escalona
- Cada thread de núcleo pode ter, em cima dela, **múltiplas threads de usuário** que se revezam para usá-la
- As threads de usuário são criadas, destruídas e escalonadas exatamente como threads de usuário em um processo executado em um SO sem suporte a múltiplas threads — ou seja, pelo sistema de tempo de execução no espaço do usuário
- Cada thread de núcleo tem **algum conjunto de threads de usuário** que se revezam para usá-la

---

## Como o modelo híbrido combina as vantagens

| Aspecto | Threads de usuário puras | Threads de núcleo puras | Híbrido |
| --- | --- | --- | --- |
| **Custo de criação/destruição** | Baixo (sem syscall) | Alto (syscall) | Baixo para threads de usuário |
| **Chamada bloqueante** | Trava processo inteiro | Só trava a thread | Só trava a thread de núcleo — outras threads de usuário da mesma thread de núcleo ficam paradas, mas threads de núcleo diferentes continuam |
| **Escalonamento customizável** | Sim, por processo | Não | Sim — threads de usuário têm escalonador próprio dentro de cada thread de núcleo |
| **Paralelismo real (multicore)** | Não | Sim | Sim — múltiplas threads de núcleo rodam em CPUs diferentes |
| **Flexibilidade** | Baixa | Baixa | **Máxima** — programador controla quantas threads de núcleo e quantas de usuário por núcleo |

> ⚠️ **A ressalva do bloqueio no modelo híbrido:** se uma thread de usuário dentro de uma thread de núcleo fizer uma chamada bloqueante, a thread de núcleo inteira bloqueia — levando consigo todas as outras threads de usuário que estavam multiplexadas nela. As threads de usuário das **outras** threads de núcleo continuam rodando normalmente. Isso é melhor que o modelo puro de usuário (que trava o processo inteiro), mas pior que o modelo puro de núcleo (que isola completamente cada thread).
> 

---

## O modelo na prática — goroutines do Go

O exemplo mais conhecido hoje de implementação híbrida são as **goroutines do Go**:

```
Goroutines (threads de usuário — milhares possíveis)
    U1  U2  U3  U4  U5  U6  U7  U8 ...
     │   │   │    │   │   │   │   │
     └───┴───┘    └───┴───┘   └───┴──┘
         │            │           │
    Thread OS 1  Thread OS 2  Thread OS 3
         │            │           │
    ─────┴────────────┴───────────┴─────
                   Núcleo
```

O runtime do Go gerencia o mapeamento de goroutines para threads do SO automaticamente — o programador só cria goroutines e o runtime decide quantas threads do SO usar (geralmente uma por núcleo físico).

---

# ✅ Resumo do Conceito

- **Implementação híbrida** — combina threads de núcleo (que o SO conhece e escalona) com threads de usuário multiplexadas em cima delas; o programador controla quantas de cada tipo usar
- **Núcleo gerencia threads de núcleo** — escalona e conhece apenas as threads de núcleo; as threads de usuário dentro de cada thread de núcleo são invisíveis ao SO
- **Sistema de tempo de execução gerencia threads de usuário** — dentro de cada thread de núcleo, o sistema de tempo de execução escalona as threads de usuário exatamente como no modelo puro de espaço do usuário
- **Máxima flexibilidade** — é possível ter desde um processo com uma thread de núcleo e muitas threads de usuário (comportamento similar ao modelo puro de usuário) até um processo com uma thread de núcleo por thread de usuário (equivalente ao modelo puro de núcleo)
- **Bloqueio parcial** — uma chamada bloqueante paralisa a thread de núcleo e todas as threads de usuário nela multiplexadas, mas não afeta outras threads de núcleo do mesmo processo
- **Exemplo moderno** — goroutines do Go: milhares de goroutines (threads de usuário) mapeadas automaticamente pelo runtime em N threads do SO (uma por núcleo físico)