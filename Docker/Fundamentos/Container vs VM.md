---
tags:
  - docker
  - docker/fundamentos
---

# Container vs VM

Para entender a diferença, precisamos olhar para onde a "linha de corte" da virtualização acontece.

## 1. Máquinas Virtuais (Virtualização de Hardware)

Uma VM é uma tentativa de simular um computador físico inteiro via software.

- **Hypervisor:** É a camada de software (como VMware, VirtualBox ou KVM) que fica entre o hardware real e a VM. Ele engana a VM fazendo-a acreditar que tem sua própria CPU, memória e placa de rede.
- **Guest OS (SO Convidado):** Como a VM "pensa" que é um computador real, ela precisa carregar um **Kernel inteiro** próprio.
- **O Custo:** Quando você sobe uma VM Linux de 50MB de aplicação, você acaba carregando 500MB+ de Kernel e bibliotecas do sistema só para gerenciar o hardware virtual. É pesado, lento para dar boot e consome muitos recursos.

## 2. Containers (Virtualização de Processos / SO)

Um container não simula hardware. Ele é apenas um **processo isolado** rodando em cima do Kernel da máquina hospedeira (Host).

- **Compartilhamento de Kernel:** Todos os containers em uma máquina compartilham o mesmo Kernel do Linux. Se você tem 100 containers, todos usam o mesmo "cérebro" do sistema operacional do Host.
- **Isolamento Lógico:** O container usa recursos do próprio Kernel (Namespaces e Cgroups) para criar uma "bolha". Ele acha que está sozinho, mas é apenas uma ilusão criada pelo SO.
- **O Ganho:** Como não há um "Guest OS" para carregar, o boot é instantâneo (é o tempo de iniciar um processo) e o consumo de memória é apenas o que a aplicação realmente usa.

---

## 3. Tabela Comparativa

| Característica | Máquina Virtual (VM) | Container |
| --- | --- | --- |
| **Abstração** | Nível de Hardware (Simula uma máquina) | Nível de Sistema Operacional (Abstrai o SO) |
| **Sistema Operacional** | Possui seu próprio Kernel (Guest OS) | Compartilha o Kernel do Host |
| **Tamanho** | Gigabytes (GB) | Megabytes (MB) |
| **Velocidade de Boot** | Minutos / Segundos longos | Milissegundos |
| **Isolamento** | Forte (Isolamento total via hardware) | Menos forte (Compartilha o Kernel) |
| **Portabilidade** | Depende do formato da imagem (OVA, VDI) | Altamente portável (Padrão OCI) |

---

## 4. Como Funciona por Dentro (Under the Hood)

Se você der um comando `top` no seu computador (Host) enquanto um container está rodando, você verá o processo da aplicação do container listado ali, como qualquer outro programa (Chrome, Spotify, etc).

Se você fizer o mesmo em uma máquina rodando uma VM, você verá apenas o processo do **Hypervisor** consumindo muita memória, mas não conseguirá ver os processos que estão rodando "dentro" da VM, porque eles estão em um kernel isolado.

### Tipos de Hypervisor

Existem dois tipos de hypervisor relevantes para entender o modelo de virtualização:

- **Tipo 1 (Bare-metal):** Roda diretamente no hardware, sem SO intermediário. Exemplos: KVM (integrado ao Kernel Linux), VMware ESXi. É o modelo usado em servidores de nuvem.
- **Tipo 2 (Hosted):** Roda como uma aplicação em cima de um SO existente. Exemplos: VirtualBox, VMware Workstation. Tem camada extra de overhead.

### Por que o container não precisa de boot?

Uma VM precisa passar por todo o processo de inicialização do Kernel (carregar drivers, montar filesystems, iniciar serviços). Isso leva segundos ou minutos.

Um container é apenas um `fork()` + `exec()` com Namespaces e Cgroups configurados. O Kernel já está rodando. O "boot" do container é o tempo que leva para o processo da aplicação entrar em estado `RUNNING`. Isso é da ordem de milissegundos.

---

## 5. Resumo

> "Enquanto a VM isola através de uma barreira de hardware virtual, os containers isolam através de políticas e restrições dentro do próprio Kernel Linux. A VM é uma casa independente; o container é um apartamento em um prédio que compartilha a mesma infraestrutura (Kernel), mas tem sua própria porta trancada (Isolamento)."

---

## Conexão com Sistemas Operacionais

- **VM = hypervisor cria hardware virtual → níveis de privilégio de CPU (rings) → [[Processadores]], [[A Origem das Máquinas Virtuais]]**
  - O hypervisor explora os modos de proteção do processador (ring 0 para o hypervisor, ring 1/3 para o Guest OS) para interceptar instruções privilegiadas do kernel convidado.
  
- **Container = namespaces + cgroups sobre o mesmo kernel → [[Processos]]**
  - Containers são processos normais do Linux. A diferença está nos atributos de namespace e nos limites de cgroup anexados ao processo pelo Kernel.

- **`top` mostra processos do container diretamente → eles são processos Linux normais → [[O Modelo de Processos]]**
  - No modelo de processo Linux, cada container é apenas mais um nó na árvore de processos global. O isolamento é uma camada de filtragem, não uma barreira física.

- **VM tem Guest OS (kernel completo) → overhead de recursos (context switches, etc.) → [[Processos]]**
  - Toda chamada de sistema feita pela aplicação dentro da VM passa por: app → kernel convidado → hypervisor → kernel do host. São dois níveis de context switch, dobrando o overhead.

- **Container compartilha o kernel → menos isolamento mas boot zero → [[Implementação de Processos]]**
  - A implementação de um processo container no Kernel é idêntica à de qualquer outro processo. O Kernel não tem um tipo especial "container process"; ele apenas aplica políticas diferentes baseadas no namespace e cgroup do processo.

- **Hypervisor (Tipo 1: bare-metal como KVM; Tipo 2: hosted como VirtualBox) → [[Máquinas Virtuais Redescobertas]]**
  - O estudo da evolução das máquinas virtuais (de CP/CMS no IBM System/370 até KVM e containers) é o contexto histórico que explica por que containers são considerados o próximo passo na evolução da virtualização.
