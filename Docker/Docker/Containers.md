---
tags:
  - docker
  - docker/docker
---

# Containers, Imagens e Registros

## 1. O que são Containers?

- **Containers** são como "caixas" que carregam todo o ambiente necessário para executar um software.
- Eles incluem **código, bibliotecas, dependências e configurações** específicas. Dessa forma, o software roda de forma **consistente em qualquer máquina** que suporte containers.
- Containers são **leves e rápidos**, pois compartilham o kernel do sistema operacional com a máquina onde estão rodando (máquina host), diferente das máquinas virtuais que criam sistemas operacionais completos.
- Um container é criado a partir de uma **imagem** e, ao ser executado, ele cria um ambiente isolado que só interage com o exterior de forma controlada.

**Exemplo**: Imagine que você tem um site que usa uma versão específica do Python e algumas bibliotecas específicas. Colocar esse site em um container significa que ele terá tudo o que precisa para rodar corretamente, mesmo que a máquina onde o container está seja diferente.

### Principais Benefícios dos Containers:

- **Consistência**: O software roda igual, independente da máquina.
- **Portabilidade**: Pode ser levado para qualquer ambiente (desenvolvimento, testes ou produção).
- **Escalabilidade**: Facilita o gerenciamento e criação de várias instâncias do mesmo software.

## 2. O que são Imagens?

- **Imagens** são a "base" dos containers. Elas contêm o "esqueleto" de tudo o que o container precisa para ser criado e executado.
- **Imutáveis**: Após criada, uma imagem não pode ser alterada. Qualquer mudança gera uma nova imagem.
- As imagens são criadas em **camadas**: cada etapa (ou comando) adiciona uma nova camada. Esse sistema de camadas permite que a imagem seja **mais leve e eficiente**, pois várias imagens podem compartilhar camadas comuns.
- Uma imagem sozinha não é "executável" – ela é apenas o molde. Quando você inicia um container, ele é uma instância ativa dessa imagem.

**Exemplo**: Se você tem uma imagem com o seu site e deseja atualizá-lo, você cria uma nova imagem com as atualizações e, ao rodar o novo container, ele usará essa nova versão.

### Como uma Imagem é Criada:

1. Você define um arquivo chamado **Dockerfile** que descreve passo a passo o ambiente.
2. O Docker lê o Dockerfile e cria a imagem, adicionando camadas conforme as instruções (como instalar o sistema, bibliotecas e o código do aplicativo).

## 3. O que são Registries (Registros)?

- **Registries** são locais onde as imagens de containers são armazenadas, organizadas e distribuídas.
- Eles permitem que você armazene e compartilhe suas imagens com outras pessoas ou com servidores.
- Um registry pode ser **público** (como o Docker Hub) ou **privado** (como uma instalação privada na sua própria infraestrutura).

**Exemplos de Registries**:

- **Docker Hub**: Registry público popular onde você encontra várias imagens já prontas para uso.
- **GitHub Container Registry**: Registry integrado ao GitHub, onde você pode manter imagens junto com o código.
- **Google Container Registry (GCR)** e **Amazon Elastic Container Registry (ECR)**: Oferecidos por provedores de nuvem para facilitar o uso de containers em seus ecossistemas.

**Exemplo**: Você cria uma imagem com seu software, envia essa imagem para o Docker Hub, e qualquer pessoa com acesso ao Docker Hub (ou ao seu repositório privado) pode baixar e rodar o container com seu software.

---

## Fluxo de Trabalho: Do Código ao Container Rodando

1. **Criar uma Imagem**:
    - Desenvolva o software e defina o ambiente no Dockerfile.
    - Crie a imagem com o Docker, que se baseará no Dockerfile.
2. **Enviar para um Registro (Registry)**:
    - Faça o **upload** da imagem para um registry (como Docker Hub) para compartilhamento ou para uso em diferentes ambientes.
3. **Executar o Container**:
    - Baixe a imagem de um registry e inicie um container com ela em uma máquina que tenha suporte a containers.
    - O container inicia a partir da imagem e executa seu software.

Esse fluxo é muito útil para garantir que, independentemente do ambiente, o software vai rodar da mesma forma.

---

## Resumo

- **Container**: Instância de um ambiente isolado que executa o software. Baseado em uma imagem.
- **Imagem**: Conjunto de instruções que define como criar um container. Imutável e dividida em camadas.
- **Registry (Registro)**: Local onde as imagens são armazenadas e compartilhadas.

---

## Conexões com Sistemas Operacionais

**Container = processo Linux com namespaces + cgroups** — Um container não é uma VM, não é um processo especial: é um processo Linux comum ao qual o kernel aplicou restrições via namespaces (isolamento de visibilidade) e cgroups (isolamento de recursos). O `nginx` dentro de um container é literalmente um processo `nginx` no host, só enxerga seu próprio PID namespace. Ver [[O Modelo de Processos]], [[Processos]].

**Ciclo de vida do container = ciclo de vida de um processo** — Os estados `created → running → paused → stopped → removed` mapeiam diretamente para os estados de processo do kernel: criado (fork sem exec), rodando (escalonado), suspenso (cgroup freezer = SIGSTOP), terminado (exit), removido (estruturas de dados limpas). Ver [[Estados de Processos]].

**Docker Hub como gerenciador de pacotes distribuído** — O Docker Hub é um repositório centralizado de imagens, análogo a um repositório `apt` ou `yum`. A diferença é que em vez de instalar binários no SO do host, cada imagem traz seu próprio userland isolado. A distribuição e o versionamento seguem o mesmo princípio. Ver [[Processos]] (ambientes de execução e distribuição de software).

**Imutabilidade = camadas read-only no OverlayFS** — As camadas da imagem são montadas pelo kernel como read-only no LowerDir do OverlayFS. Qualquer tentativa de escrita é redirecionada para o UpperDir (camada do container). Essa imutabilidade é imposta pelas permissões de filesystem do kernel. Ver [[Arquivos]].
