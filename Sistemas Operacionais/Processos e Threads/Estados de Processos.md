---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.1.5"
---
# Estados de Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.1.5

---

# 🔀 2.1.5 — Estados de Processos

Embora cada processo seja uma entidade independente, com seu próprio contador de programa e estado interno, processos muitas vezes precisam **interagir entre si**. Um processo pode gerar alguma saída que outro processo usa como entrada. Considere o comando shell:

```
cat arquivo1 arquivo2 arquivo3 | grep hierarquia
```

O primeiro processo, executando `cat`, gera como saída a concatenação dos três arquivos. O segundo processo, executando `grep`, seleciona todas as linhas contendo a palavra "hierarquia". A depender das velocidades relativas dos dois processos, pode acontecer que `grep` esteja pronto para ser executado, mas não haja entrada esperando por ele — ele deve então ser **bloqueado** até que alguma entrada esteja disponível.

---

## Os três estados de um processo

> 💡 **Processo sequencial:** processo que avança uma instrução de cada vez, em sequência, como se tivesse a CPU inteira para si. É o modelo básico adotado até aqui — cada processo é uma linha única de execução (em oposição às *threads*, que veremos na seção 2.2).
> 

Quando um processo é bloqueado, ele o faz porque **não pode proceder de forma lógica** — em geral porque está esperando pela entrada que ainda não está disponível. Há também uma segunda situação: um processo que esteja conceitualmente pronto e capaz de executar pode ser bloqueado porque o SO decidiu alocar a CPU para outro processo por um tempo.

Essas duas condições são completamente diferentes:

- No **primeiro caso**, a suspensão é **inerente ao problema** — o processo não pode processar a linha de comando do usuário até que ela tenha sido digitada.
- No **segundo caso**, trata-se de um **detalhe técnico do sistema** — não há CPUs suficientes para dar a cada processo seu próprio processador privado.

Isso nos leva a três estados possíveis para qualquer processo:

| # | Estado | Descrição |
| --- | --- | --- |
| 1 | **Em execução** | Realmente usando a CPU naquele instante |
| 2 | **Pronto** | Executável, temporariamente parado para deixar outro processo ser executado |
| 3 | **Bloqueado** | Incapaz de ser executado até que algum evento externo aconteça |

> 💡 **Escalonador (*scheduler*):** componente do SO responsável por decidir *qual* processo deve ser executado, *quando* e *por quanto tempo*. É ele quem provoca as transições 2 e 3 no diagrama abaixo — retirando e colocando processos na CPU — sem que os processos sequer saibam que isso aconteceu.
> 

---

## Figura 2.2 — Diagrama de estados e transições

> 📌 **Figura 2.2:** Um processo pode estar nos estados em execução, bloqueado ou pronto. Transições entre esses estados ocorrem como mostrado.
> 

```jsx
                  ┌─────────────────┐
        ┌────1────│   Em execução   │────2────┐
        │         └─────────────────┘         │
        ▼                   ▲                 ▼
┌───────────────┐           │ 3      ┌────────────────┐
│   Bloqueado   │           └────────│     Pronto     │
└───────────────┘                    └────────────────┘
        │                                     ▲
        └─────────────────4───────────────────┘

Transições:
1 → Em execução → Bloqueado   (processo aguarda entrada — ex: pipe vazio)
2 → Em execução → Pronto      (escalonador retira da CPU — tempo esgotado)
3 → Pronto      → Em execução (escalonador concede a CPU a este processo)
4 → Bloqueado   → Pronto      (evento esperado ocorreu — entrada disponível)

NOTA: Não existe transição direta entre Bloqueado e Em execução.
      Um processo bloqueado sempre passa por Pronto antes de voltar a executar.
```

### As quatro transições em detalhe

**Transição 1 — Em execução → Bloqueado**

Ocorre quando o SO descobre que um processo não pode continuar nesse momento. Em alguns sistemas, o processo pode executar uma chamada de sistema, como `pause`, para entrar em um estado bloqueado. Em outros, incluindo UNIX, quando um processo lê de um *pipe* ou de um arquivo especial (p. ex., um terminal) e não há nenhuma entrada disponível, o processo é **automaticamente bloqueado**.

**Transições 2 e 3 — Em execução ↔ Pronto**

Causadas pelo **escalonador de processos**, sem o processo nem saber a respeito delas.

- **Transição 2:** ocorre quando o escalonador decide que o processo em andamento foi executado por tempo suficiente, e é o momento de deixar outro processo ter algum tempo de CPU.
- **Transição 3:** ocorre quando todos os outros processos tiveram sua vez e é hora do primeiro processo ter a atenção da CPU para ser executado novamente.

> ⚠️ **Os estados "Em execução" e "Pronto" são semelhantes** — em ambos o processo está disposto a ser executado, porém no segundo não há uma CPU disponível para ele temporariamente. O estado **"Bloqueado"** é fundamentalmente diferente: o processo **não pode ser executado** mesmo que a CPU esteja ociosa e não tenha nada mais a fazer.
> 

**Transição 4 — Bloqueado → Pronto**

Ocorre quando um evento externo pelo qual um processo estava esperando acontece (como a chegada de alguma entrada). Se nenhum outro processo estiver sendo executado naquele instante, a transição 3 será desencadeada imediatamente. Caso contrário, o processo talvez tenha de esperar no estado *pronto* por um intervalo curto até que a CPU esteja disponível.

---

# ✅ Resumo do Conceito

- **Três estados de processo:** *em execução* (usando a CPU agora), *pronto* (poderia executar, mas a CPU está ocupada) e *bloqueado* (aguardando evento externo — não pode executar mesmo com CPU livre)
- **Quatro transições:** 1) execução→bloqueado (entrada indisponível); 2) execução→pronto (escalonador retira da CPU); 3) pronto→execução (escalonador concede a CPU); 4) bloqueado→pronto (evento esperado ocorreu)
- **Escalonador:** componente do SO que provoca as transições 2 e 3 de forma transparente ao processo
- **Bloqueado ≠ Pronto:** pronto significa "quero executar, mas não há CPU agora"; bloqueado significa "não posso executar de jeito nenhum até um evento acontecer" — distinção fundamental
- **Sem transição direta Bloqueado→Execução:** um processo bloqueado sempre passa por Pronto antes de voltar a executar
- **Processo sequencial:** modelo em que cada processo é uma linha única de execução que avança passo a passo — base de tudo até a chegada das *threads* (seção 2.2)