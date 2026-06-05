---
tags:
  - sistemas-operacionais
  - so/estrutura
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.7.4"
---
# O Modelo Cliente Servidor

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.7.4

---

# 🔄 O que é o Modelo Cliente-Servidor?

Uma ligeira variação da ideia do micronúcleo é distinguir duas classes de processos:

- **Servidores** — processos que prestam algum serviço
- **Clientes** — processos que usam esses serviços

Esse modelo é conhecido como **modelo cliente-servidor**. A essência encontra-se na presença de **processos-clientes** e **processos-servidores**.

> 💡 **O que é um servidor nesse contexto?** Não é necessariamente uma máquina física separada — é um **processo** que oferece um serviço. Da mesma forma, um cliente é um **processo** que solicita esse serviço. Ambos podem rodar na mesma máquina ou em máquinas diferentes conectadas por uma rede.
> 

---

# 📨 Comunicação por Troca de Mensagens

A comunicação entre clientes e servidores é realizada muitas vezes pela **troca de mensagens**:

```
Para obter um serviço:

Processo-cliente
        ↓ constrói uma mensagem dizendo o que quer
        ↓ envia ao serviço apropriado
Processo-servidor
        ↓ realiza o trabalho
        ↓ envia de volta uma resposta
Processo-cliente
        ↓ recebe a resposta e continua
```

Por exemplo, um processo que precisa fazer um `read` envia uma mensagem para um servidor de arquivos dizendo a ele o que ler. O servidor faz a leitura e envia de volta os dados.

---

# 💻 Mesma Máquina ou Rede — A Abstração é a Mesma

Uma propriedade elegante desse modelo é que **funciona tanto para uma única máquina quanto para uma rede de máquinas**:

```
Caso 1 — Mesma máquina:
┌─────────────────────────────────────┐
│           Máquina 1                 │
│  ┌──────────┐     ┌──────────────┐  │
│  │ Cliente  │────▶│   Servidor   │  │
│  └──────────┘     └──────────────┘  │
│         Micronúcleo                 │
└─────────────────────────────────────┘
→ mensagens entregues localmente
  (determinadas otimizações são possíveis)

Caso 2 — Máquinas diferentes em rede:
┌──────────┐    rede    ┌──────────────────┐
│ Máquina 1│◄──────────▶│    Máquina 2     │
│ Cliente  │            │ Servidor arq.    │
│ Núcleo   │            │ Núcleo           │
└──────────┘            └──────────────────┘
→ mesmas mensagens, entregues pela rede
```

Se o cliente e o servidor estiverem na mesma máquina, determinadas otimizações são possíveis — mas **conceitualmente ainda estamos falando da troca de mensagens**. O cliente não precisa saber se o servidor está local ou remoto — a interface é a mesma.

---

# 🌐 Generalização para Redes — Figura 1.27

Uma generalização dessa ideia é ter clientes e servidores sendo executados em **computadores diferentes**, conectados por uma rede local ou remota.

> 📌 **Figura 1.27 — O modelo cliente-servidor em uma rede**
> 

```
Máquina 1      Máquina 2          Máquina 3           Máquina 4
┌─────────┐  ┌──────────────┐  ┌────────────────┐  ┌───────────────────┐
│ Cliente │  │Servidor de   │  │Servidor de     │  │Servidor de        │
│         │  │arquivos      │  │processos       │  │terminais          │
│ Núcleo  │  │Núcleo        │  │Núcleo          │  │Núcleo             │
└────┬────┘  └──────────────┘  └────────────────┘  └───────────────────┘
     │                    Rede
     └─────────────────── Mensagem do cliente ao servidor ──────────────▶
```

Tendo em vista que os clientes se comunicam com os servidores enviando mensagens, os clientes não precisam saber se as mensagens são entregues localmente em suas próprias máquinas, ou se são enviadas através de uma rede para servidores em uma máquina remota. **Do ponto de vista do cliente, a mesma coisa acontece em ambos os casos: pedidos são enviados e as respostas retornadas.**

Cada vez mais, muitos sistemas envolvem usuários em seus PCs domésticos como clientes e grandes máquinas em outra parte operando como servidores. Na realidade, **grande parte da web opera dessa maneira** — um PC solicita uma página web para um servidor e ele a entrega. Esse é o uso típico do modelo cliente-servidor em uma rede.

---

# ⚖️ Comparação com o Micronúcleo

O modelo cliente-servidor é uma **variação** e **complemento** do micronúcleo — não uma alternativa completamente diferente. A relação entre os dois é:

|  | Micronúcleo | Modelo Cliente-Servidor |
| --- | --- | --- |
| **Foco** | O que fica no núcleo vs. modo usuário | Como processos se comunicam |
| **Mecanismo** | Troca de mensagens entre processos | Troca de mensagens entre cliente e servidor |
| **Escopo** | Dentro de uma máquina | Pode se estender para múltiplas máquinas |
| **Ideia central** | Minimizar o núcleo | Separar quem fornece serviços de quem usa |

> 💡 O modelo cliente-servidor pode ser visto como uma forma de **organizar** os processos de usuário que rodam acima de um micronúcleo — os servidores do MINIX 3 (servidor de arquivos, servidor de processos) são exatamente processos-servidores nesse modelo.
> 

---

# ✅ Resumo do Conceito

- O **modelo cliente-servidor** distingue dois papéis: **servidores** (prestam serviços) e **clientes** (usam serviços) — ambos são **processos**, não necessariamente máquinas separadas
- A comunicação é feita por **troca de mensagens** — o cliente envia um pedido, o servidor processa e responde
- A abstração é **uniforme** — funciona igual para processos na mesma máquina ou em máquinas diferentes conectadas por uma rede
- É uma **generalização** do micronúcleo — os servidores que rodam em modo usuário no micronúcleo são exatamente servidores nesse modelo
- Grande parte da **web** opera no modelo cliente-servidor — navegador (cliente) solicita página para servidor web, que entrega a resposta