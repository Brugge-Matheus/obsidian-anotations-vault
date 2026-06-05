---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.2.3"
---
# Threads POSIX

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.2.3

---

# 📋 2.2.3 — Threads POSIX

Para possibilitar que se escrevam programas com threads **portáteis** entre diferentes sistemas operacionais, o IEEE definiu um padrão para threads no padrão **IEEE 1003.1c**.

> 💡 **Pthreads (POSIX Threads):** o pacote de threads definido pelo padrão IEEE 1003.1c. A maioria dos sistemas UNIX dá suporte a ele. O padrão define mais de 60 chamadas de função — aqui serão cobertas apenas as principais para dar uma ideia de como funcionam.
> 

> 💡 **Portabilidade:** a motivação central do Pthreads. Um programa escrito usando a API Pthreads pode ser compilado e executado em qualquer sistema que implemente o padrão — Linux, macOS, Solaris etc. — sem modificação.
> 

---

## Figura 2.13 — Principais chamadas de função do Pthreads

> 📌 **Figura 2.13:** Algumas das chamadas de função do Pthreads.
> 

```
┌─────────────────────┬──────────────────────────────────────────────┐
│  Chamada de thread  │                  Descrição                   │
├─────────────────────┼──────────────────────────────────────────────┤
│ pthread_create      │ Cria uma nova thread                         │
│ pthread_exit        │ Termina a thread que a chamou                │
│ pthread_join        │ Espera que uma thread específica termine     │
│ pthread_yield       │ Libera a CPU para que outra thread seja      │
│                     │ executada                                    │
│ pthread_attr_init   │ Cria e inicializa a estrutura de atributos   │
│                     │ da thread                                    │
│ pthread_attr_destroy│ Remove uma estrutura de atributos da thread  │
└─────────────────────┴──────────────────────────────────────────────┘
```

---

## As chamadas em detalhe

### Propriedades de cada thread Pthreads

Todas as threads Pthreads têm determinadas propriedades. Cada uma tem:

- Um **identificador** único
- Um **conjunto de registradores** (incluindo o contador de programa)
- Um **conjunto de atributos** — armazenados em uma estrutura de atributos, incluindo tamanho da pilha, parâmetros de escalonamento e outros itens necessários para o uso da thread

---

> 💡 **`pthread_create`:** cria uma nova thread. O identificador da thread recém-criada é retornado como valor da função. É intencionalmente parecida com `fork` do UNIX (exceto pelos parâmetros) — o identificador de thread desempenha o papel do PID, em especial para identificar threads referenciadas em outras chamadas.
> 

> 💡 **`pthread_exit`:** chamada quando uma thread terminou o trabalho para o qual foi designada. Encerra a thread e libera sua pilha.
> 

> 💡 **`pthread_join`:** usada quando uma thread precisa esperar outra terminar seu trabalho e sair antes de continuar. A thread que está esperando chama `pthread_join` passando o identificador da thread alvo como parâmetro. A chamadora fica bloqueada até que a thread alvo conclua.
> 

> 💡 **`pthread_yield`:** permite que uma thread que não está logicamente bloqueada, mas sente que já foi executada por tempo suficiente, ceda voluntariamente a CPU para que outra thread tenha chance de ser executada. Essa chamada **não existe para processos** — o pressuposto com processos é que eles são competitivos e cada um quer o máximo de CPU. Já threads de um mesmo processo são escritas pelo mesmo programador e podem cooperar, cedendo a CPU quando apropriado.
> 

> 💡 **`pthread_attr_init`:** cria e inicializa a estrutura de atributos associada a uma thread, preenchendo-a com valores padrão. Esses valores (como prioridade) podem ser modificados manipulando campos na estrutura de atributos.
> 

> 💡 **`pthread_attr_destroy`:** remove a estrutura de atributos de uma thread, liberando sua memória. **Não afeta as threads que a usam** — elas continuam a existir normalmente; apenas a estrutura de atributos é liberada.
> 

---

## Figura 2.14 — Exemplo de programa usando Pthreads

> 📌 **Figura 2.14:** Um exemplo de programa usando threads.
> 

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUMBER_OF_THREADS 10

void *print_hello_world(void *tid)
{
    /* Esta funcao imprime o identificador da thread e depois sai. */
    printf("Hello World. Saudacoes da thread %d\n", tid);
    pthread_exit(NULL);
}

int main(int argc, char *argv[])
{
    /* O programa principal cria 10 threads e sai. */
    pthread_t threads[NUMBER_OF_THREADS];
    int status, i;

    for (i=0; i < NUMBER_OF_THREADS; i++) {
        printf("Programa principal aqui. Criando thread %d\n", i);
        status = pthread_create(&threads[i], NULL, print_hello_world, (void *)i);

        if (status != 0) {
            printf("Oops. pthread_create retornou codigo de erro %d\n", status);
            exit(-1);
        }
    }
    exit(NULL);
}
```

**O que esse programa faz:**

1. O programa principal executa um laço `NUMBER_OF_THREADS` (10) vezes
2. A cada iteração, anuncia sua intenção e chama `pthread_create` para criar uma nova thread
3. Se a criação falhar, imprime uma mensagem de erro e termina
4. Após criar todas as threads, o programa principal termina
5. Quando uma thread é criada, ela imprime uma mensagem anunciando a si mesma e então sai via `pthread_exit`

> ⚠️ **A ordem das mensagens é indeterminada** e pode variar entre execuções consecutivas do programa. Isso porque o escalonador decide qual thread roda em qual momento — e essa decisão pode ser diferente a cada execução. Esse comportamento não determinístico é uma característica fundamental de programas multithreaded e uma das razões pelas quais eles exigem cuidado ao serem projetados.
> 

> ⚠️ **Nota de tradução do livro:** a linguagem C não permite o uso de acentuação. Por esse motivo, as palavras nos códigos são escritas sem acentuação.
> 

---

## Estrutura geral de um programa Pthreads

```
main()
  │
  ├── pthread_attr_init()       ← inicializa atributos (opcional)
  │
  ├── pthread_create(T1, ...)   ← cria thread 1
  ├── pthread_create(T2, ...)   ← cria thread 2
  ├── pthread_create(T3, ...)   ← cria thread 3
  │
  │        T1 executa → pthread_exit()
  │        T2 executa → pthread_exit()
  │        T3 executa → pthread_exit()
  │
  ├── pthread_join(T1)          ← espera T1 terminar (se necessário)
  ├── pthread_join(T2)          ← espera T2 terminar (se necessário)
  │
  └── exit()                    ← programa principal termina
```

---

# ✅ Resumo do Conceito

- **Pthreads** — padrão IEEE 1003.1c que define uma API portátil para threads; suportado pela maioria dos sistemas UNIX; permite escrever programas multithreaded que compilam e rodam em diferentes SOs sem modificação
- **`pthread_create`** — cria nova thread; retorna identificador único análogo ao PID de processos
- **`pthread_exit`** — encerra a thread chamadora e libera sua pilha
- **`pthread_join`** — bloqueia a thread chamadora até que uma thread específica termine; equivalente ao `waitpid` para processos
- **`pthread_yield`** — cede a CPU voluntariamente para outra thread; sem equivalente em processos, pois threads cooperam enquanto processos competem
- **`pthread_attr_init` / `pthread_attr_destroy`** — criam e removem a estrutura de atributos de uma thread (tamanho de pilha, prioridade etc.); destruir os atributos não afeta threads que já os usaram
- **Ordem de execução indeterminada** — em programas multithreaded, a ordem em que as threads executam e produzem saída é definida pelo escalonador e pode variar a cada execução — comportamento fundamental que exige projeto cuidadoso