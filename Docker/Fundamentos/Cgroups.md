---
tags:
  - docker
  - docker/fundamentos
---

# Cgroups

Se os **Namespaces** servem para isolar a **visão** (o que o container enxerga), os **Cgroups (Control Groups)** servem para isolar e limitar os **recursos** (o quanto o container pode consumir).

Sem Cgroups, um único container com um bug (curva de memória infinita, por exemplo) poderia derrubar o seu computador inteiro ou "roubar" toda a CPU de outros containers.

---

## 1. O que são Cgroups?

O **Cgroups** é um recurso do Kernel Linux que permite organizar processos em grupos hierárquicos e distribuir recursos de hardware entre esses grupos.

Imagine que o Kernel é o "síndico" do prédio (seu computador). Ele usa os Cgroups para dizer:

> *"Apartamento 101 (Container A), você só pode usar 10% da eletricidade do prédio e no máximo 512MB de água por dia. Se tentar usar mais, eu corto a torneira."*

---

## 2. As 4 Funções Principais do Cgroup

### 2.1. Limitação de Recursos (Resource Limiting)

É o uso mais comum. Você define um teto máximo.

- **Memória:** Impede que um processo ultrapasse X megabytes.
- **CPU:** Define que o processo só pode usar uma fração dos núcleos da CPU.

### 2.2. Priorização (Prioritization)

Se o sistema estiver sobrecarregado (CPU a 100%), o Cgroup decide quem tem preferência.

- Você pode configurar para que o Container de Banco de Dados tenha mais "peso" de CPU do que o Container de Logs, por exemplo.

### 2.3. Contabilização (Accounting)

O Cgroup monitora e gera relatórios detalhados de quanto recurso cada grupo está usando.

- É assim que o comando `docker stats` consegue te mostrar em tempo real o uso de memória e CPU de cada container.

### 2.4. Controle (Control)

Permite congelar (**freeze**) ou pausar todos os processos de um grupo de uma vez só.

- Quando você dá um `docker pause`, o Docker usa o Cgroup para suspender a execução dos processos naquele grupo.

---

## 3. Onde eles vivem? (Under the Hood)

Diferente de outros recursos, os Cgroups no Linux são controlados através de um **Virtual Filesystem**. Tudo é baseado em arquivos e pastas.

Se você estiver em um Linux real (ou dentro do WSL2), você pode navegar até:
`/sys/fs/cgroup/`

Lá dentro, você verá pastas como:

- `cpu`
- `memory`
- `pids` (limita o número máximo de processos que um container pode criar — evita "fork bombs")
- `blkio` (limita a velocidade de leitura/escrita no disco)

Quando você inicia um container Docker com o comando:

```bash
docker run -m 512m --cpus 0.5 my-app
```

O Docker faz o seguinte por debaixo dos panos:

1. Cria uma pasta dentro de `/sys/fs/cgroup/memory/docker/<container_id>/`.
2. Escreve o valor `512MB` dentro de um arquivo chamado `memory.limit_in_bytes`.
3. Adiciona o PID do processo da sua aplicação dentro do arquivo `tasks` dessa pasta.

**Pronto!** A partir desse milissegundo, o Kernel Linux vai monitorar aquele processo e garantir que ele não passe de 512MB.

---

## 4. O que acontece se o limite for atingido?

Aqui está um detalhe técnico importante:

1. **CPU:** Se o container atingir o limite de CPU, o Kernel apenas "atrasa" o processo (throttling). Ele fica mais lento, mas continua vivo.
2. **Memória:** Se o container tentar alocar mais memória do que o permitido pelo Cgroup, o Kernel ativa o **OOM Killer (Out of Memory Killer)**. Ele simplesmente "mata" o processo na hora.
   - É por isso que às vezes seu container morre com o status `OOMKilled`.

---

## 5. Cgroups v1 vs v2

O Kernel Linux tem duas gerações de Cgroups:

**Cgroups v1 (legado):**
- Hierarquias separadas para cada controlador (memória, CPU, pids, etc.)
- Um processo podia pertencer a diferentes hierarquias de forma inconsistente
- Cada subsistema montava seu próprio ponto em `/sys/fs/cgroup/<subsystem>/`

**Cgroups v2 (moderno, padrão nas distribuições atuais):**
- Hierarquia unificada: uma única árvore em `/sys/fs/cgroup/`
- Um processo só pode pertencer a um único grupo na árvore
- Controle de recursos mais consistente e previsível
- Requerido para algumas funcionalidades do Kubernetes moderno

Verificar qual versão está ativa:
```bash
stat -fc %T /sys/fs/cgroup/
# tmpfs = cgroups v1
# cgroup2fs = cgroups v2
```

---

## 6. Resumo

> "Cgroups são o 'Gerenciador de Tarefas' turbinado do Kernel. Enquanto os Namespaces garantem que os processos não se vejam, os Cgroups garantem que eles não se destruam disputando hardware. Sem Cgroups, não existiria computação em nuvem (Cloud Computing) multitenant, pois não haveria como garantir performance isolada."

---

## Conexão com Sistemas Operacionais

- **Cgroups implementados como filesystem virtual em `/sys/fs/cgroup/` → [[Arquivos]] (VFS, filesystems virtuais)**
  - O Linux expõe os Cgroups via VFS (Virtual Filesystem Switch). Escrever em um arquivo como `memory.limit_in_bytes` é uma operação de arquivo que internamente aciona os subsistemas do Kernel responsáveis pela limitação. Isso é o mesmo padrão do `/proc` e do `/dev`.

- **Limite de memória: o OOM Killer do Kernel mata o processo quando o limite do cgroup é excedido → [[Memória Virtual]], [[Espaços de Endereçamento]]**
  - Quando um processo tenta alocar mais páginas além do limite do cgroup, o alocador de páginas do Kernel falha e aciona o OOM Killer. O OOM Killer seleciona a vítima (normalmente o processo que mais consumiu) e envia `SIGKILL`. O espaço de endereçamento do processo é então liberado.

- **Throttling de CPU: o Kernel usa CFS (Completely Fair Scheduler) + CPU shares do cgroup → [[Processos]] (escalonamento)**
  - O CFS (Completely Fair Scheduler) é o escalonador padrão do Linux. Ele usa o conceito de "virtual runtime" para distribuir CPU de forma justa. Os cgroups adicionam a capacidade de limitar a fração de tempo de CPU que um grupo pode consumir (por exemplo, `cpu.cfs_quota_us` / `cpu.cfs_period_us`).

- **Cgroup `pids` previne fork bomb (limita contagem máxima de PIDs) → [[Criação de Processos]], [[Hierarquia de Processos]]**
  - Uma fork bomb é um processo que se reproduz recursivamente até esgotar a tabela de PIDs do sistema. O cgroup `pids` impede isso definindo `pids.max`, um limite máximo de processos simultâneos dentro do grupo. Novas chamadas `fork()` ou `clone()` falham com `EAGAIN` quando o limite é atingido.

- **Cgroup `blkio` limita I/O de disco → [[Armazenamento não Volátil]], [[Dispositivos de IO]]**
  - O subsistema `blkio` (ou `io` no v2) permite definir limites de throughput (bytes/s) e operações por segundo (IOPS) para cada dispositivo de bloco. O Kernel intercepta as requisições de I/O na camada de bloco e aplica throttling antes de enviá-las ao driver do dispositivo.

- **`docker stats` lê dos arquivos de contabilização do cgroup → [[Implementação de Processos]]**
  - O Docker lê arquivos como `memory.usage_in_bytes`, `cpuacct.usage`, `blkio.throttle.io_service_bytes` para construir a saída do `docker stats`. Esses valores são mantidos pelo Kernel em tempo real, sem custo adicional de instrumentação.

- **`docker pause` = SIGSTOP para todos os processos do cgroup (subsistema freezer) → [[Estados de Processos]]**
  - O subsistema `freezer` do cgroup permite "congelar" todos os processos de um grupo atomicamente. Internamente, o Kernel coloca todos os processos do grupo no estado `TASK_STOPPED` (equivalente ao efeito do SIGSTOP), sem que nenhum processo consiga escapar individualmente.

- **cgroups v1 vs v2: v2 é hierarquia unificada, árvore única → padrão moderno do Linux → [[Processos]]**
  - A migração para v2 reflete uma evolução na forma como o Kernel gerencia grupos de processos: de múltiplas hierarquias independentes para uma única árvore de controle, alinhando melhor o modelo de Cgroups com a árvore de processos do SO.

- **Escrever em `/sys/fs/cgroup/memory/docker/<id>/memory.limit_in_bytes` é como o Docker define limites → [[System Calls]]**
  - A operação de escrever nesse arquivo é uma syscall `write()` comum. O VFS encaminha a escrita para o handler do Cgroup, que valida o valor e configura o limite no subsistema de memória do Kernel. Não há syscall especial para Cgroups — tudo passa pelo filesystem virtual.
