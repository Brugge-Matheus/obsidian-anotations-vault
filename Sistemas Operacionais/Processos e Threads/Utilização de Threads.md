---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seções 2.2 e 2.2.1"
---
# Utilização de Threads

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seções 2.2 e 2.2.1

---

# 🧵 2.2 — Threads

Em sistemas operacionais tradicionais, cada processo tem um espaço de endereços e uma única **thread** de controle. Na realidade, essa é quase a definição de um processo.

> 💡 **Thread:** uma linha (*thread*) de execução dentro de um processo. Cada thread tem seu próprio contador de programa, registradores e pilha — mas **compartilha o espaço de endereços e todos os dados** com as outras threads do mesmo processo. Threads são às vezes chamadas de **processos leves**.
> 

> 💡 **Multithreading:** situação em que múltiplas threads existem no mesmo processo, executando quase em paralelo dentro de um único espaço de endereços compartilhado. Algumas CPUs têm suporte de hardware direto para multithreading e permitem trocas de threads em escala de nanossegundos.
> 

## Por que threads e não apenas processos?

Já vimos o argumento antes — é o mesmo usado para justificar a existência de processos. Em vez de pensar sobre interrupções, temporizadores e trocas de contexto, podemos pensar sobre processos em paralelo. Com threads, acrescentamos um elemento novo: a **capacidade de entidades em paralelo compartilharem um espaço de endereços e todos os seus dados**.

Essa capacidade é essencial para determinadas aplicações. Ter múltiplos **processos** (com espaços de endereços separados) não funcionaria nesses casos — pois cada processo teria sua própria cópia da memória e as mudanças de um não seriam visíveis para o outro.

---

# ⚡ 2.2.1 — Utilização de Threads

Por que alguém iria querer ter um tipo de processo dentro de um processo? Há várias razões. Vamos examiná-las a partir de exemplos concretos.

---

## Razão 1 — Múltiplas atividades simultâneas no mesmo dado

Muitas aplicações têm múltiplas atividades ocorrendo simultaneamente, algumas das quais podem ser bloqueadas de tempos em tempos. Ao decompor essas aplicações em múltiplas threads sequenciais executadas quase em paralelo, o modelo de programação torna-se muito mais simples.

### Exemplo: processador de texto com 3 threads

Considere um processador de texto. O usuário está escrevendo um livro de 800 páginas e apaga uma frase na página 1. O processador precisa reformatar o livro inteiro até a página 600 para saber qual será a primeira linha daquela página — algo potencialmente demorado.

**Com uma única thread:**

```
Usuário apaga frase na página 1
          │
          ▼
Processador reformata tudo até página 600
(usuário fica travado esperando)
          │
          ▼
Página 600 é exibida
```

O usuário fica bloqueado enquanto a reformatação ocorre — experiência péssima.

**Com 3 threads no mesmo processo:**

```
┌─────────────────────────────────────────────────────────────┐
│                   Mesmo espaço de endereços                 │
│                    (mesmo documento em RAM)                 │
│                                                             │
│  Thread 1 (interativa) ──► responde teclado e mouse         │
│  Thread 2 (reformatação) ► reformata em segundo plano       │
│  Thread 3 (backup) ──────► salva no disco periodicamente    │
└─────────────────────────────────────────────────────────────┘
```

- Thread 1 continua respondendo ao usuário normalmente
- Thread 2 reformata o documento em segundo plano
- Thread 3 faz backups automáticos sem interferir nas outras

> ⚠️ **Por que 3 processos separados não funcionariam aqui?** Porque as três threads precisam operar sobre o **mesmo documento**. Três processos separados teriam três cópias distintas do documento na memória — modificações feitas por um processo não seriam visíveis para os outros. Threads resolvem isso justamente por compartilharem o mesmo espaço de endereços.
> 

---

## Razão 2 — Threads são mais leves que processos

Threads são mais leves do que processos — têm mais facilidade (ou seja, mais rapidez) para criar e destruir.

|  | Processo | Thread |
| --- | --- | --- |
| **Espaço de endereços** | Próprio, isolado | Compartilhado com o processo |
| **Custo de criação** | Alto | 10 a 100× mais rápido que um processo |
| **Custo de destruição** | Alto | Muito baixo |
| **Comunicação entre si** | Precisa de IPC (pipes, sockets…) | Direta — compartilham memória |
| **Troca de contexto** | Pesada (troca de espaço de endereços) | Leve (mesmo espaço de endereços) |

> 💡 **IPC (Inter-Process Communication):** mecanismos usados para comunicação entre processos distintos, como pipes, sockets, memória compartilhada e filas de mensagens. Entre threads do mesmo processo, a comunicação é direta via memória compartilhada — sem necessidade de IPC.
> 

Quando o número de threads necessárias muda dinâmica e rapidamente, é muito útil poder criá-las e destruí-las com baixo custo.

---

## Razão 3 — Ganho de desempenho com I/O e CPU sobrepostos

O uso de threads não resulta em ganho de desempenho quando todas elas são vinculadas à CPU. Mas quando há uma **computação substancial E também I/O substancial**, usar threads permite que essas atividades se **sobreponham**, acelerando a aplicação.

### Exemplo: servidor web multithreaded

Um servidor web recebe milhares de requisições de páginas. Sem threads, cada requisição precisaria ser concluída antes da próxima ser atendida — enquanto o disco é lido, a CPU fica ociosa.

> 📌 **Figura 2.8:** Um servidor web com múltiplas threads (*multithreaded*).
> 

```
       Processo do servidor web
┌──────────────────────────────────────────┐
│         Espaço do usuário                │
│                                          │
│  ┌──────────────┐    ┌───────────────┐   │
│  │   Thread     │    │    Thread     │   │
│  │ Despachante  │───►│   Operária 1  │   │
│  │              │    ├───────────────┤   │
│  │ (lê pedidos  │───►│   Operária 2  │   │
│  │  da rede e   │    ├───────────────┤   │
│  │  distribui)  │───►│   Operária 3  │   │
│  └──────────────┘    └───────────────┘   │
│          │             Cache de páginas  │
│          │          (acessado por todas) │
├──────────────────────────────────────────┤
│              Espaço do núcleo            │
│                  Núcleo                  │
└──────────┬───────────────────┬───────────┘
           │                   │
      Conexão de rede        Disco
```

**Como funciona:**

- A **thread despachante** fica em loop lendo requisições da rede (`get_next_request`) e entregando trabalho a uma operária disponível (`handoff_work`)
- Cada **thread operária** aguarda trabalho (`wait_for_work`), verifica se a página está no cache (`look_for_page_in_cache`) e, se não estiver, lê do disco (`read_page_from_disk`) e retorna ao cliente

**Por que não usar um único processo de thread única?**

Enquanto o servidor esperasse o disco, estaria completamente bloqueado e não receberia nenhum novo pedido — muito menos solicitações seriam processadas por segundo.

> 📌 **Figura 2.9 — Esboço do código:**
> 

```
// Thread despachante                // Thread operária
while (TRUE) {                       while (TRUE) {
    get_next_request(&buf);              wait_for_work(&buf);
    handoff_work(&buf);                  look_for_page_in_cache(&buf, &page);
}                                        if (page_not_in_cache(&page))
                                             read_page_from_disk(&buf, &page);
                                         return_page(&page);
                                     }
```

Quando uma thread operária é bloqueada na operação de disco, outra thread é escolhida para ser executada — possivelmente a despachante, para adquirir mais trabalho, ou outra operária que já tenha sua página no cache.

---

## Razão 4 — Úteis em sistemas com múltiplas CPUs

Threads são úteis em sistemas com **múltiplas CPUs**, nos quais o paralelismo real é possível. Nesse caso, threads de um mesmo processo podem realmente executar simultaneamente em CPUs diferentes — o que não seria possível com um único processo de thread única.

---

## Processos leves vs. Processos pesados

| Termo | Significado |
| --- | --- |
| **Processo pesado** | Processo tradicional — espaço de endereços isolado, criação custosa |
| **Processo leve** | Sinônimo de thread — executa dentro de um processo, compartilha memória |
| **Multithreading** | Ter múltiplas threads no mesmo processo executando quase em paralelo |

---

# ✅ Resumo do Conceito

- **Thread** — linha de execução dentro de um processo; tem PC, registradores e pilha próprios, mas **compartilha o espaço de endereços** com todas as outras threads do mesmo processo
- **Por que threads existem:** quando múltiplas atividades de um programa precisam operar sobre os **mesmos dados simultaneamente** — algo impossível com processos separados
- **Mais leves que processos:** criar/destruir uma thread é 10 a 100× mais rápido que um processo; comunicação entre threads é direta via memória, sem IPC
- **Sobreposição de CPU e E/S:** threads permitem que computação e operações de E/S ocorram ao mesmo tempo dentro do mesmo programa, aumentando o desempenho
- **Exemplo do processador de texto:** 3 threads (interativa, reformatação, backup) operam sobre o mesmo documento — impossível com 3 processos separados
- **Exemplo do servidor web:** thread despachante distribui trabalho para threads operárias; quando uma operária bloqueia no disco, outra assume a CPU — muito mais requisições por segundo
- **Processos leves** — sinônimo de thread; termo que enfatiza o menor custo em relação a processos tradicionais (pesados)