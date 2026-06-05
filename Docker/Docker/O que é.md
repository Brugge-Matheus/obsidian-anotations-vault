---
tags:
  - docker
  - docker/docker
---

# O que é o Docker?

### Introdução ao Docker

**Docker** é uma plataforma de software que permite criar, gerenciar e executar containers de forma fácil e eficiente. Com o Docker, desenvolvedores e equipes de TI conseguem empacotar aplicativos com todas as suas dependências, garantindo que o software funcione da mesma maneira em qualquer ambiente, seja em um laptop, em um servidor de produção ou na nuvem.

### Por que o Docker é Importante?

Tradicionalmente, configurar um aplicativo para rodar em diferentes máquinas envolvia muitos ajustes, porque as versões de bibliotecas e dependências poderiam ser diferentes entre os ambientes. Com o Docker, todos os elementos necessários para rodar o software ficam dentro do container, resolvendo essa questão de compatibilidade.

### Componentes Principais do Docker

1. **Imagens (Images)**: Uma imagem é como uma "foto" que contém todo o necessário para rodar um aplicativo: o sistema operacional básico, bibliotecas e o código do aplicativo.
    - As imagens são imutáveis e podem ser armazenadas em registries (repositórios de imagens) como o Docker Hub.
2. **Containers**: Um container é uma instância em execução de uma imagem. Ele é um ambiente isolado que executa o aplicativo e só interage com o sistema externo de maneira controlada.
    - Cada container é criado a partir de uma imagem e pode ser iniciado, parado ou removido quando não for mais necessário.
3. **Dockerfile**: É um arquivo de texto com uma lista de instruções que o Docker usa para criar uma imagem. Cada comando no Dockerfile cria uma camada da imagem, o que permite otimizar o armazenamento e melhorar a reutilização de componentes.
4. **Docker Hub (ou Registry)**: O Docker Hub é o registro público mais usado, onde as imagens podem ser armazenadas e compartilhadas. Você pode baixar imagens prontas ou enviar suas próprias imagens para reutilizá-las em diferentes ambientes.

### Como o Docker Funciona

- O Docker utiliza o kernel do sistema operacional da máquina host, mas cria um ambiente isolado para cada container.
- Isso permite que os containers compartilhem recursos sem necessidade de virtualizar um sistema operacional completo, tornando-os mais leves e rápidos que máquinas virtuais tradicionais.

### Vantagens do Docker

- **Portabilidade**: Containers Docker podem ser executados em qualquer ambiente com suporte a Docker.
- **Consistência**: Uma vez configurada, uma imagem sempre vai rodar do mesmo jeito.
- **Escalabilidade**: Docker facilita a criação de múltiplas instâncias de um aplicativo, o que ajuda no balanceamento de carga e alta disponibilidade.
- **Isolamento**: Cada container é um ambiente isolado, o que minimiza conflitos entre aplicativos e suas dependências.

### Exemplo Prático de Uso do Docker

Imagine que você tem um aplicativo web em Python e precisa garantir que ele funcione tanto no seu ambiente de desenvolvimento quanto em produção. Com o Docker:

1. Você cria um Dockerfile definindo o ambiente Python e as bibliotecas necessárias.
2. Usa o Docker para construir uma imagem a partir desse Dockerfile.
3. Sobe a imagem para o Docker Hub.
4. Em produção, basta baixar essa imagem e rodar o container para garantir que o aplicativo funcione da mesma forma.

### Docker no Ciclo de Desenvolvimento

- **Desenvolvimento**: Criação de imagens e containers para testar o aplicativo em ambientes controlados.
- **Teste**: Facilita testes automatizados em ambientes consistentes.
- **Produção**: Deploy de containers escaláveis e gerenciáveis, aproveitando a portabilidade e o isolamento.

### Resumo

O Docker é uma plataforma que simplifica a criação e execução de containers, possibilitando a consistência e a portabilidade dos aplicativos em diferentes ambientes. É uma ferramenta essencial para equipes que precisam de flexibilidade, escalabilidade e garantia de que o software funcione da mesma forma em qualquer lugar.

---

## Conexões com Sistemas Operacionais

**Docker como "empacotamento de processos"** — Cada container é, na essência, um processo Linux com isolamento. Não é uma VM; é um processo comum com restrições aplicadas pelo kernel. Ver [[Processos]] e [[O Modelo de Processos]].

**Isolamento via chamadas de sistema** — O isolamento que o Docker provê (namespaces, cgroups) é implementado via syscalls do kernel Linux. O próprio Docker CLI comunica-se com o daemon por meio de chamadas de sistema de socket. Ver [[System Calls]].

**Docker Hub como repositório de software** — O Docker Hub funciona como um gerenciador de pacotes distribuído (análogo ao `apt` ou `yum`), mas em vez de pacotes binários, distribui imagens de container com todo o ambiente de execução embutido. A diferença fundamental: enquanto um pacote `apt` assume dependências do SO do host, uma imagem Docker carrega seu próprio userland. Ver [[Processos]] (distribuição de software e ambientes de execução).

**Deployment tradicional vs. Docker** — No modelo tradicional, um processo Python depende das bibliotecas instaladas no SO host — qualquer diferença de versão quebra a execução. Docker resolve isso empacotando o processo junto com seu espaço de usuário (bibliotecas, runtime, configurações), isolado do host via namespaces. O kernel ainda é compartilhado, mas o userland é privado. Ver [[Processos]], [[System Calls]].
