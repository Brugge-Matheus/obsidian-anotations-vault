---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.2.7"
---
# Convertendo Thread em Multithread

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.2.7

---

# 🔁 2.2.7 — Convertendo Código de uma Thread em Código Multithread

Muitos programas existentes foram escritos para processos *monothread*. Convertê-los para *multithreading* é muito mais complicado do que pode parecer em um primeiro momento. Esta seção examina apenas algumas das armadilhas mais importantes.

---

## Problema 1 — Variáveis globais compartilhadas entre threads

O código de uma thread em geral consiste em múltiplos procedimentos, exatamente como um processo. Esses procedimentos podem ter variáveis locais, variáveis globais e parâmetros.

- **Variáveis locais e parâmetros** → não causam problema algum, pois ficam na pilha de cada thread, que é privada
- **Variáveis globais para uma thread, mas não para o programa inteiro** → são o problema central

> 💡 **Variáveis globais de thread:** variáveis que são globais no sentido de que muitos procedimentos dentro da thread as usam — como qualquer variável global — mas que outras threads deveriam deixar sozinhas. Em um programa monothread, uma variável global é acessada por um procedimento de cada vez. Em multithreading, dois procedimentos de threads diferentes podem acessar a mesma variável simultaneamente, causando conflitos.
> 

### O exemplo do `errno` — Figura 2.17

O UNIX mantém uma variável global chamada `errno`. Quando um processo (ou thread) faz uma chamada de sistema que falha, o código de erro é colocado em `errno`.

> 📌 **Figura 2.17:** Conflitos entre threads sobre o uso de uma variável global.
> 

```
       Thread 1                    Thread 2
           │
           ▼
Access (errno definido = X)
           │
           │ ◄── escalonador troca para Thread 2
           │                          │
           │                          ▼
           │               Open (errno sobrescrito = Y)
           │                          │
           │ ◄── controle volta para Thread 1
           ▼
errno é inspecionado → valor Y (ERRADO!)
Thread 1 lê o errno de Thread 2
e se comporta de forma incorreta
```

**O que acontece passo a passo:**

1. Thread 1 executa `access` para descobrir se ela tem permissão para acessar um arquivo
2. O SO retorna a resposta na variável global `errno`
3. Antes que Thread 1 tenha chance de ler `errno`, o escalonador decide que ela teve CPU suficiente e troca para Thread 2
4. Thread 2 executa `open` que falha, fazendo `errno` ser **sobrescrito**
5. Quando Thread 1 retoma, ela lerá o `errno` errado — o de Thread 2 — e se comportará incorretamente

---

## Soluções para variáveis globais compartilhadas

### Solução 1 — Proibir variáveis globais completamente

Embora válida como ideal, essa solução entra em conflito com grande parte dos *softwares* existentes. Inviável na prática.

### Solução 2 — Variáveis globais privadas por thread (Figura 2.18)

> 📌 **Figura 2.18:** Threads podem ter variáveis globais privadas.
> 

```
┌─────────────────────────────────┐
│       Espaço do processo        │
│                                 │
│  ┌───────────────┐              │
│  │ Código Thread1│              │
│  ├───────────────┤              │
│  │ Código Thread2│              │
│  ├───────────────┤              │
│  │ Pilha Thread1 │◄─┐           │
│  ├───────────────┤  │ cada thread
│  │ Pilha Thread2 │◄─┘ tem sua   │
│  ├───────────────┤  própria     │
│  │ Glob. Thread1 │◄─┐ cópia     │
│  ├───────────────┤  │           │
│  │ Glob. Thread2 │◄─┘           │
└─────────────────────────────────┘
```

Cada thread tem sua própria cópia privada de `errno` e outras variáveis globais. Isso cria um novo nível de escopo — variáveis visíveis a todos os procedimentos de uma thread (mas não às outras threads), além dos níveis já existentes:

- Variáveis visíveis apenas a um procedimento (locais)
- Variáveis visíveis a todos os procedimentos de uma thread (globais de thread)
- Variáveis visíveis em toda parte no programa (globais de processo)

Acessar essas variáveis globais privadas é um pouco complicado, já que a maioria das linguagens tem formas de expressar variáveis locais e globais, mas não formas intermediárias. Para isso, novos procedimentos de biblioteca podem ser introduzidos:

```c
create_global("bufptr");
// aloca memória para o ponteiro bufptr no heap ou em área
// especial reservada para a thread que emitiu a chamada

set_global("bufptr", &buf);
// armazena o valor de um ponteiro no local de memória
// criado por create_global

bufptr = read_global("bufptr");
// retorna o endereço armazenado na variável global,
// de modo que seus dados possam ser acessados
```

Apenas a thread que emitiu a chamada `create_global` tem acesso à variável. Se outra thread criar uma variável global com o mesmo nome, ela obterá um local **diferente** da memória — sem conflito com a existente.

---

## Problema 2 — Procedimentos de biblioteca não reentrantes

> 💡 **Procedimento reentrante:** procedimento que pode ser chamado por uma segunda thread enquanto uma chamada anterior ainda não foi concluída, sem causar problemas. Um procedimento é não reentrante quando usa estado interno compartilhado — como um buffer fixo — que pode ser corrompido se uma segunda chamada ocorrer antes da primeira terminar.
> 

Muitos procedimentos de biblioteca **não são reentrantes** — não foram projetados para ter uma segunda chamada feita enquanto uma anterior ainda não foi concluída.

**Exemplo — envio de mensagem pela rede:**

O envio de uma mensagem por uma rede pode ser programado com a montagem da mensagem em um *buffer* fixo dentro da biblioteca, seguida de uma captura para o núcleo a fim de enviá-la:

```
Thread 1 monta mensagem no buffer fixo
    │
    │ ◄── interrupção de relógio: troca para Thread 2
    │
Thread 2 monta SUA mensagem no mesmo buffer fixo
(sobrescreve a mensagem de Thread 1)
    │
    │ ◄── troca de volta para Thread 1
    │
Thread 1 envia o buffer → envia a mensagem ERRADA
```

**Exemplo — `malloc` do UNIX:**

`malloc` mantém tabelas cruciais sobre o uso de memória — uma lista encadeada de pedaços de memória disponíveis. Enquanto `malloc` está atualizando essas listas, elas podem estar temporariamente em um estado inconsistente, com ponteiros que apontam para lugar nenhum. Se uma troca de threads ocorrer enquanto as tabelas estiverem inconsistentes e uma nova chamada de `malloc` chegar de uma thread diferente, um ponteiro inválido poderá ser usado — levando à falha do programa.

---

## Soluções para procedimentos não reentrantes

### Solução 1 — Reescrever a biblioteca inteira

Consertar todos esses problemas significa reescrever toda a biblioteca para torná-la reentrante. Essa atividade não é trivial e tem a possibilidade real de introduzir erros sutis.

### Solução 2 — Bit de proteção por procedimento

Fornecer a cada procedimento um *bit* de proteção que indica que a biblioteca está sendo usada. Qualquer tentativa de outra thread usar um procedimento de biblioteca enquanto a chamada anterior ainda não tiver sido concluída é bloqueada.

> ⚠️ Embora essa abordagem possa ser colocada em funcionamento, ela **elimina grande parte do paralelismo em potencial** — as threads ficam serializadas esperando umas pelas outras para usar a biblioteca.
> 

---

## Problema 3 — Sinais com multithreading

Sinais apresentam complexidade adicional em ambientes multithreaded.

Alguns sinais são **logicamente específicos a threads** — por exemplo, quando uma thread chama `alarm`, faz sentido para o sinal resultante ir até a thread que fez a chamada.

Porém, quando threads são implementadas inteiramente no espaço do usuário, o núcleo não tem ideia a respeito das threads e dificilmente poderia dirigir o sinal para a thread certa.

**Complicações adicionais com sinais:**

| Situação | Problema |
| --- | --- |
| Um processo tem apenas um alarme pendente | Várias threads chamam `alarm` independentemente — conflito |
| Sinais de teclado (ex: CTRL-C) | Não são específicos a threads — qual thread recebe? |
| Thread muda tratadores de sinal | Afeta ou não afeta as outras threads? |
| Thread quer pegar um sinal para encerrar o processo | Outra thread pode não querer isso |

> ⚠️ Em geral, sinais já são difíceis de gerenciar mesmo em um ambiente de thread única. Ir para um ambiente de múltiplas threads não torna a situação mais fácil de lidar — na prática, torna ainda mais complexa.
> 

---

## Problema 4 — Gerenciamento de pilha

Em muitos sistemas, quando a pilha de um processo estoura, o núcleo simplesmente fornece mais pilha àquele processo automaticamente.

Quando um processo tem múltiplas threads, ele também tem **múltiplas pilhas**. Se o núcleo não está ciente de todas essas pilhas:

- Ele não pode fazê-las crescer automaticamente por causa de uma falta de pilha
- Ele pode nem se dar conta de que uma falta de memória está relacionada ao crescimento da pilha de alguma thread

> ⚠️ Esses problemas certamente não são insuperáveis, mas mostram que apenas introduzir threads em um sistema existente sem uma alteração bastante substancial do sistema realmente não vai funcionar. No mínimo, as semânticas das chamadas de sistema talvez precisem ser redefinidas e as bibliotecas reescritas — e todas essas ações devem ser feitas de maneira a permanecerem compatíveis com programas já existentes para o caso limitante de um processo com apenas uma thread.
> 

---

# ✅ Resumo do Conceito

- **Conversão monothread → multithread é complexa** — vai muito além de simplesmente criar threads; exige revisão profunda de variáveis globais, bibliotecas e sinais
- **Variáveis globais compartilhadas** — o problema do `errno`: thread 1 define `errno`, é interrompida, thread 2 sobrescreve `errno`, thread 1 lê o valor errado ao retomar
- **Solução — variáveis globais privadas por thread:** cada thread tem sua própria cópia de variáveis que são "globais dentro da thread"; acessadas via `create_global`, `set_global` e `read_global`
- **Procedimentos não reentrantes** — bibliotecas com estado interno compartilhado (buffer fixo, tabelas do `malloc`) corrompem dados quando uma segunda thread chama o procedimento antes da primeira terminar
- **Solução por bit de proteção** — bloqueia a entrada de uma segunda thread no procedimento enquanto ele está em uso; funciona mas serializa as threads, eliminando o paralelismo
- **Sinais em multithreading** — sinais UNIX são direcionados ao processo, não à thread; conflitos surgem com `alarm`, CTRL-C e mudança de tratadores; muito mais difíceis de gerenciar do que em monothread
- **Múltiplas pilhas** — o núcleo pode não saber de todas as pilhas e não consegue crescê-las automaticamente em caso de estouro
- **Conclusão do Tanenbaum:** introduzir threads sem alterações substanciais no sistema e nas bibliotecas não funciona — as semânticas das chamadas de sistema precisam ser redefinidas e as bibliotecas reescritas, mantendo compatibilidade com código monothread existente