---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.2.5"
---
# Implementando Threads em Kernel Space

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.2.5

---

# 🔧 2.2.5 — Implementando Threads no Núcleo

Agora vamos considerar o oposto do modelo anterior: o núcleo sabe sobre as threads e as gerencia diretamente.

> 💡 **Threads no núcleo:** modelo em que o núcleo mantém uma tabela global de threads que controla todas as threads do sistema. Quando uma thread quer criar uma nova ou destruir uma existente, ela faz uma **chamada de sistema ao núcleo**, que então cria ou destrói a thread atualizando sua tabela. Não há sistema de tempo de execução no espaço do usuário — o núcleo faz tudo.
> 

---

## Como funciona — a tabela de threads do núcleo

A tabela de threads do núcleo mantém os registradores, estado e outras informações de cada thread — a mesma informação que threads de usuário armazenam na tabela de threads do espaço do usuário, mas agora mantida **dentro do núcleo**.

Essa informação é um subconjunto das informações que os núcleos tradicionais mantêm a respeito dos seus processos de thread única — ou seja, o estado de processo. Além disso, o núcleo também mantém a tabela de processos tradicional para controlar os processos.

```
┌──────────────────────────────────────────┐
│             Espaço do usuário            │
│  ┌─────────────┐      ┌─────────────┐   │
│  │  Processo A │      │  Processo B │   │
│  │  Thread 1   │      │  Thread 3   │   │
│  │  Thread 2   │      │  Thread 4   │   │
│  └──────┬──────┘      └──────┬──────┘   │
├─────────┴─────────────────────┴──────────┤
│                 Núcleo                   │
│  ┌───────────────────────────────────┐   │
│  │        Tabela de threads          │   │
│  │  T1 | T2 | T3 | T4 | ...         │   │
│  ├───────────────────────────────────┤   │
│  │        Tabela de processos        │   │
│  │  Proc A | Proc B | ...            │   │
│  └───────────────────────────────────┘   │
└──────────────────────────────────────────┘
```

---

## O que muda na criação e bloqueio de threads

**Criação e destruição via chamada de sistema:**

Todas as chamadas que poderiam bloquear uma thread são implementadas como chamadas de sistema — a um custo consideravelmente maior do que uma chamada a um procedimento do sistema de tempo de execução. Isso é a principal desvantagem em relação às threads de usuário.

**Quando uma thread bloqueia:**

Quando uma thread é bloqueada, o núcleo tem a opção de executar outra thread do **mesmo processo** (se houver uma pronta) ou de um **processo diferente**. Com threads de usuário, o sistema de tempo de execução só podia executar threads do próprio processo até o núcleo assumir a CPU.

```
Thread bloqueada (threads de núcleo):
        │
        ▼
Núcleo verifica tabela global de threads
        │
        ├──► Há thread pronta no mesmo processo? → executa ela
        │
        └──► Não há? → executa thread de outro processo
```

Com threads de usuário, quando uma thread bloqueava, o sistema de tempo de execução executava threads apenas do seu próprio processo — sem visibilidade global.

---

## Vantagens das threads no núcleo

**1. Chamadas bloqueantes não travam o processo inteiro**

Quando uma thread faz uma chamada de sistema bloqueante, o núcleo pode executar outra thread do mesmo processo enquanto espera. Não é necessário usar `select` ou wrappers — o núcleo simplesmente troca de thread.

**2. Falta de página não trava o processo inteiro**

Se uma thread causa uma falta de página, o núcleo pode conferir se o processo tem outras threads executáveis e, se assim for, executar uma delas enquanto espera que a página seja trazida do disco.

**3. O escalonador do núcleo controla as trocas**

Não é necessário que as threads cedam voluntariamente a CPU — o timer de hardware garante que o núcleo retome o controle e possa escalonar outra thread.

---

## Desvantagem principal — custo das chamadas de sistema

> ⚠️ **Custo substancial:** toda operação sobre threads (criação, término, bloqueio, desbloqueio) envolve uma chamada de sistema ao núcleo. Se as operações de thread forem frequentes, a sobrecarga será muito maior do que no modelo de espaço do usuário. Em decorrência desse custo, alguns sistemas **reciclam threads** em vez de destruí-las:
> 

> 💡 **Reciclagem de threads:** quando uma thread é destruída, ela é marcada como não executável, mas suas estruturas de dados não são afetadas. Quando uma nova thread precisa ser criada, uma antiga é reativada, evitando parte do custo adicional. A reciclagem também é possível para threads de usuário, mas o incentivo é menor pois o custo de gerenciamento de threads de usuário é muito menor.
> 

---

## Problemas que threads de núcleo não resolvem completamente

### fork com threads — ainda complexo

O que acontece quando um processo com múltiplas threads é bifurcado via `fork`? O novo processo terá tantas threads quanto o antigo, ou apenas uma?

- Se for **chamar `exec`** logo depois: provavelmente **uma thread** é a escolha correta — de nada adianta duplicar todas as threads se o programa vai ser substituído.
- Se for **continuar executando**: reproduzir **todas as threads** talvez seja o melhor.

A resposta certa depende do que o processo está planejando fazer em seguida. Na maioria dos casos, a melhor escolha depende do contexto — e não há uma resposta universal.

### Sinais com múltiplas threads

> ⚠️ **Sinais são enviados para processos, não para threads** — pelo menos no modelo clássico UNIX. Quando um sinal chega, qual thread deve cuidar dele?
> 

As possibilidades e seus problemas:

```
Sinal chega ao processo
        │
        ├── Qualquer thread pode tratar → vencedora sorteada pelo SO
        │   (ex: Linux) — pode ser uma thread não relacionada
        │
        ├── Threads registram interesse em sinais específicos
        │   → sinal dado à thread que disse querer
        │   → e se nenhuma quiser?
        │
        └── Bloquear sinal em todas exceto uma
            → mas duas threads podem se registrar para o mesmo sinal
            → SO escolhe aleatoriamente qual trata
```

No Linux, um sinal pode ser tratado por qualquer thread — a vencedora é selecionada pelo SO. É possível bloquear o sinal em todas as threads exceto uma, mas se duas ou mais se registraram para o mesmo sinal, o SO escolhe uma aleatoriamente. A menos que o programador seja muito cuidadoso, é fácil cometer erros.

---

## Comparativo final — Threads de usuário vs. Threads de núcleo

| Aspecto | Threads de usuário | Threads de núcleo |
| --- | --- | --- |
| **Conhecimento do núcleo** | Não sabe das threads | Gerencia diretamente |
| **Troca de contexto** | Muito rápida (procedimento local) | Mais lenta (chamada de sistema) |
| **Chamada bloqueante** | Trava o processo inteiro | Só trava a thread |
| **Falta de página** | Trava o processo inteiro | Núcleo executa outra thread |
| **Monopolização da CPU** | Possível (sem timer interno) | Impossível (timer do SO age) |
| **Criação/destruição** | Barata | Cara — custo de syscall |
| **Reciclagem de threads** | Possível, mas pouco incentivo | Comum para amortizar custo |
| **fork** | Complexo | Igualmente complexo |
| **Sinais** | Problema menor (sem sinais UNIX diretos) | Complexo — qual thread trata? |
| **Escalonamento customizável** | Sim, por processo | Não — núcleo decide |

---

# ✅ Resumo do Conceito

- **Threads no núcleo** — o núcleo mantém tabela global de threads; toda criação, destruição e bloqueio envolve chamada de sistema; não há sistema de tempo de execução no espaço do usuário
- **Vantagem principal** — chamadas bloqueantes e faltas de página afetam apenas a thread causadora; o núcleo pode executar outra thread do mesmo processo ou de outro enquanto espera
- **Desvantagem principal** — toda operação de thread envolve syscall, com custo substancial; se operações forem frequentes, sobrecarga é muito maior que threads de usuário
- **Reciclagem de threads** — threads destruídas são marcadas como não executáveis mas mantidas; reativadas ao criar nova thread, evitando o custo de criação do zero
- **fork ainda é complexo** — número de threads no processo filho depende do que o programa fará a seguir; não há resposta universal
- **Sinais ainda são problemáticos** — sinais UNIX são endereçados ao processo, não à thread; definir qual thread trata cada sinal exige cuidado e a solução varia por SO
- **Caso de uso ideal** — aplicações com E/S intensiva e chamadas bloqueantes frequentes; para processamento puro de CPU, a diferença de custo das syscalls pode não compensar