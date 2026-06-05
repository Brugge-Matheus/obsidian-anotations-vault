---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.1.4"
---
# Hierarquia de Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.1.4

---

# 🌳 2.1.4 — Hierarquia de Processos

Em alguns sistemas, quando um processo cria outro, o **processo-pai** e o **processo-filho** continuam a estar associados de certas maneiras. O processo-filho pode por si só criar mais processos, formando uma **hierarquia de processos**.

> 💡 **Processo-pai (*parent process*):** o processo que executou a chamada de sistema de criação (`fork` no UNIX) e originou outro processo.
> 

> 💡 **Processo-filho (*child process*):** o processo recém-criado pela chamada de `fork`. Pode, por sua vez, criar seus próprios filhos, formando uma árvore.
> 

> ⚠️ **Diferentemente das plantas e dos animais** que usam a reprodução sexuada, um processo tem apenas **um pai** (mas zero, um, dois ou mais filhos). Então um processo lembra mais uma hidra do que, digamos, uma vaca.
> 

---

## Hierarquia no UNIX — Grupos de Processos e `init`

No UNIX, um processo e todos os seus filhos e demais descendentes formam, juntos, um **grupo de processos**.

> 💡 **Grupo de processos:** conjunto formado por um processo UNIX e todos os seus descendentes (filhos, netos etc.). Sinais enviados ao grupo chegam a todos os seus membros simultaneamente.
> 

### Sinais e o grupo de processos

Quando um usuário envia um sinal do teclado (p. ex., pressionando **CTRL-C**), o sinal é entregue a **todos os membros do grupo de processos** associados ao teclado no momento — em geral, todos os processos ativos que foram criados na janela atual. Individualmente, cada processo pode:

- **Capturar** o sinal e tratá-lo com um handler próprio
- **Ignorá-lo** completamente
- **Assumir a ação predefinida**, que é ser encerrado pelo sinal

### O processo `init` e a árvore de inicialização

Como exemplo de onde a hierarquia de processos tem um papel fundamental, veja como o UNIX se inicializa logo após o computador ser ligado:

> 💡 **`init`:** processo especial presente na imagem de inicialização (*boot*) do sistema UNIX. É o **ancestral de todos os processos** — todos os processos do sistema inteiro pertencem a uma única árvore com `init` em sua raiz.
> 

```
Ligou o computador
        │
        ▼
   ┌─────────┐
   │  init   │  ← processo especial criado na inicialização (boot)
   └────┬────┘    lê arquivo dizendo quantos terminais existem
        │         cria um processo de login para cada terminal
        │
   ┌────┴──────────────────────────┐
   │                               │
   ▼                               ▼
┌──────────┐                 ┌──────────┐
│ login    │  ...            │ login    │   ← um por terminal
│(terminal1)                 │(terminalN)
└────┬─────┘                 └──────────┘
     │  (conexão bem-sucedida)
     ▼
┌──────────┐
│  shell   │  ← processo de shell para aceitar comandos
└────┬─────┘
     │  (usuário digita comando, ex: ls -l *.c)
     ▼
┌──────────┐  ┌──────────┐
│   ls     │  │  grep    │  ← processos-filhos criados pelo shell
└──────────┘  └──────────┘    para executar os comandos
```

O fluxo completo:

1. `init` lê um arquivo dizendo **quantos terminais existem** e cria um novo processo de conexão para cada terminal
2. Esses processos **esperam que alguém se conecte**
3. Se uma conexão é bem-sucedida, o processo de conexão executa um **shell** para aceitar os comandos
4. O shell pode **iniciar mais processos** (para executar comandos digitados) e assim por diante
5. Desse modo, **todos os processos** no sistema inteiro pertencem a uma única árvore, com `init` em sua raiz

---

## Hierarquia no Windows — Sem hierarquia formal

O Windows **não tem conceito de uma hierarquia de processos**. Todos os processos são iguais.

> 💡 **Handle de processo (Windows):** identificador especial retornado ao pai quando um processo é criado via `CreateProcess`. O pai pode usar esse *handle* para controlar o filho — mas é **livre para passar esse identificador a qualquer outro processo**, invalidando assim qualquer noção de hierarquia.
> 

| Aspecto | UNIX | Windows |
| --- | --- | --- |
| **Hierarquia formal** | Sim — árvore de processos com `init` na raiz | Não — todos os processos são iguais |
| **Relação pai-filho** | Permanente e estrutural | Temporária — baseada em *handle* transferível |
| **Grupos de processos** | Sim — descendentes formam um grupo | Não há equivalente direto |
| **Sinais para o grupo** | Sim — CTRL-C atinge todos do grupo | Não se aplica da mesma forma |
| **Ancestral comum** | `init` (ou `systemd`) | Não há |
| **Encerramento em cascata** | Não automático | Não automático |

> ⚠️ No UNIX, processos **não podem deserdar seus filhos** — a hierarquia é estrutural e permanente. No Windows, como o *handle* pode ser transferido, qualquer processo pode assumir o "controle" de outro, tornando a hierarquia essencialmente inexistente.
> 

---

# ✅ Resumo do Conceito

- **Hierarquia de processos** — estrutura em árvore formada quando processos criam filhos que por sua vez criam mais filhos; cada processo tem exatamente **um pai**, mas pode ter zero ou mais filhos
- **Grupo de processos (UNIX)** — conjunto de um processo e todos os seus descendentes; sinais enviados ao grupo (ex: CTRL-C) chegam a todos os membros simultaneamente
- **`init`** — ancestral de todos os processos no UNIX; criado no *boot*, lê a configuração de terminais, cria processos de login e shells; toda a árvore de processos do sistema tem `init` na raiz
- **Capturar / ignorar / ação padrão** — as três respostas possíveis de um processo ao receber um sinal no UNIX
- **Windows sem hierarquia** — todos os processos são iguais; o *handle* retornado ao pai pode ser transferido para qualquer outro processo, eliminando qualquer hierarquia formal
- **Sem cascata em ambos** — nem UNIX nem Windows encerram filhos automaticamente ao encerrar o pai