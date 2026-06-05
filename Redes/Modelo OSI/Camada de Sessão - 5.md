---
tags:
  - redes
  - redes/osi
---

# Camada de Sessão - 5

A **Camada 5 (Sessão)** é o "Gerente de Diálogo" ou o "Secretário" do Modelo OSI. Enquanto a Camada 6 se preocupa com o *formato* do dado, a Camada 5 se preocupa em manter a **conversa viva e organizada**.

---

### 📞 Camada 5: Sessão (Session Layer)

A Camada de Sessão é responsável por estabelecer, gerenciar e encerrar as comunicações (sessões) entre dois dispositivos. Ela decide quem fala, quando fala e por quanto tempo.

### 1. O que ela faz?

Ela realiza três funções essenciais para que a comunicação não seja uma bagunça:

- **Controle de Diálogo:** Define se a comunicação será nos dois sentidos ao mesmo tempo (Full-Duplex) ou um de cada vez (Half-Duplex).
- **Agrupamento:** Garante que os dados relacionados sejam tratados como uma única unidade.
- **Pontos de Restauração (Checkpoints):** Se você estiver baixando um arquivo de 100MB e a conexão cair no 50MB, a Camada 5 permite que o download continue de onde parou, em vez de começar do zero.

### 2. Exemplo da Vida Real: A Secretária e a Chamada Telefônica

Imagine que você é um executivo e quer falar com o diretor de outra empresa:

- **Camada 7 (Aplicação):** Você decide: *"Quero fechar o contrato"*.
- **Camada 6 (Apresentação):** Você prepara o contrato em PDF e criptografa.
- **Camada 5 (Sessão):** Sua **secretária** liga para a secretária do diretor.
    1. **Estabelecimento:** Elas confirmam: *"O Sr. Matheus pode falar agora? Sim, o Diretor está livre"*. (A sessão começou).
    2. **Gerenciamento:** Se a linha ficar muda por um segundo, a secretária diz: *"Alô? Ainda está aí? Parei na cláusula 5"*. (Ponto de restauração).
    3. **Encerramento:** Quando vocês terminam, as secretárias se despedem e desligam o telefone. (A sessão fechou).

### 3. Protocolos e Tecnologias Comuns da Camada 5

Estes são exemplos que atuam nesta camada:

- **NetBIOS:** Usado antigamente para identificar computadores em redes Windows.
- **RPC (Remote Procedure Call):** Permite que um programa execute um código em outro computador como se fosse local.
- **SQL:** Algumas partes do gerenciamento de consultas a bancos de dados.
- **NFS (Network File System):** Protocolo para compartilhamento de arquivos em rede.

### 4. Resumo (Callout):

> 💡 **Ponto Chave:** A Camada 5 é a "camada do gerenciamento". Ela abre a porta, mantém a porta aberta enquanto for necessário e fecha a porta quando a conversa acaba.
> 

---

### 🛠️ Exemplo Prático no Navegador:

Quando você faz login no seu banco ou no Facebook:

1. Uma **sessão** é aberta entre o seu navegador e o servidor do site.
2. Mesmo que você mude de página dentro do site, você continua logado porque a **Camada 5** mantém essa sessão ativa.
3. Se você ficar muito tempo sem mexer, a sessão "expira" por segurança e a Camada 5 encerra a conexão.

---

### 🔗 Conexões com SO e Go

- **Camada de Sessão invisível no TCP/IP:** no modelo real, suas responsabilidades estão absorvidas pelo próprio TCP (controle de conexão) e pela camada de aplicação — não existe um protocolo de sessão explícito no dia a dia.
- **TLS opera parcialmente aqui:** o handshake TLS estabelece a sessão (troca de chaves, autenticação do certificado) e a retomada de sessão (*session tickets*) evita repetir o handshake completo → [[Bits e Bytes]] (criptografia, troca de chaves)
- **Session token (cookie HTTP):** é um identificador aleatório que o servidor associa ao estado do usuário armazenado em memória ou banco de dados; do ponto de vista do kernel, não passa de um cabeçalho de texto trafegando em um socket → [[Memória]], [[Arquivos]]
- **RPC (Remote Procedure Call):** gerencia sessões entre sistemas distribuídos, permitindo que um processo chame funções em outro host como se fossem locais — analogia com IPC entre processos no mesmo sistema → [[Processos]]
