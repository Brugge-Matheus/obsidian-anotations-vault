---
tags:
  - sistemas-operacionais
  - so/conceitos
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.5.5"
---
# Proteção

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.5.5

---

# 🔐 Por que Proteção é Necessária?

Computadores contêm grandes quantidades de informações que os usuários muitas vezes querem proteger e manter confidenciais — e-mails, planos de negócios, declarações fiscais e muito mais. Cabe ao sistema operacional **gerenciar a segurança** de forma que os arquivos, por exemplo, sejam acessíveis somente por usuários autorizados.

Além dos arquivos, há muitos outros recursos que precisam de proteção — processos, memória, dispositivos de I/O. Um processo não deve conseguir ler a memória de outro. Um usuário comum não deve conseguir desligar o computador de outros usuários em um servidor. Um programa malicioso não deve conseguir acessar o disco diretamente.

---

# 🗂️ Proteção de Arquivos no UNIX — Bits rwx

O UNIX usa um modelo de proteção baseado em **bits de permissão**. Cada arquivo recebe um código de proteção binário de **9 bits**, organizados em três campos de 3 bits. Os campos definem permissões para três grupos distintos de usuários:

> 💡 **Terminologia importante:** os três campos não são "owner, grupo e usuário" — todos os três são compostos de usuários do sistema. Os nomes corretos são:
> 

> - **Owner** — o proprietário do arquivo
> 

> - **Group** — membros do grupo do proprietário
> 

> - **Others** — todos os demais usuários do sistema (também chamado de "world")
> 

> 
> 

> "Usuário" sozinho é ambíguo porque os três grupos são compostos de usuários. O termo técnico para o terceiro campo é **others**.
> 

```
rwxr-x--x

r w x  r - x  - - x
│ │ │  │   │      │
│ │ │  │   │      └── outros usuários: pode executar (x), não pode ler nem escrever
│ │ │  │   └───────── grupo: pode ler (r) e executar (x), não pode escrever
│ │ └──────────────── dono: pode ler (r), escrever (w) e executar (x)
│ └────────────────── w = write (escrita)
└──────────────────── r = read (leitura)
```

Cada campo de 3 bits controla as permissões para um grupo de usuários diferente:

| Campo | Para quem se aplica |
| --- | --- |
| **Primeiro rwx** | O **proprietário** do arquivo |
| **Segundo rwx** | Os **membros do grupo** do proprietário |
| **Terceiro rwx** | **Todos os demais** usuários do sistema |

Cada campo tem três bits:

| Bit | Significado para arquivo | Significado para diretório |
| --- | --- | --- |
| **r** (read) | Pode ler o conteúdo | Pode listar os arquivos |
| **w** (write) | Pode escrever/modificar | Pode criar e remover arquivos |
| **x** (execute) | Pode executar o programa | Pode entrar no diretório (busca) |

> 💡 Para diretórios, **x indica permissão de busca** — poder navegar para dentro do diretório. Um diretório com `r` mas sem `x` permite listar os arquivos, mas não acessá-los.
> 

---

# 🔢 Notação Octal das Permissões

Como cada campo tem 3 bits, ele pode ser representado como um número de 0 a 7 em octal:

```
rwx = 111 em binário = 7 em octal
r-x = 101 em binário = 5 em octal
--- = 000 em binário = 0 em octal

Exemplos comuns:
rwxr-xr-x = 755 → dono pode tudo, grupo e outros podem ler e executar
rwxr-x--x = 751 → dono pode tudo, grupo pode ler/executar, outros só executar
rw-r--r-- = 644 → dono pode ler/escrever, grupo e outros só leem
rwx------ = 700 → só o dono tem acesso total, ninguém mais vê nada
```

---

# 👤 Proprietário, Grupo e Outros

## Proprietário

Cada arquivo tem um **proprietário** — normalmente quem o criou. O proprietário pode mudar as permissões do próprio arquivo.

## Grupos

Usuários podem ser membros de **grupos** — conjuntos definidos pelo administrador. O sistema de grupos permite compartilhar arquivos entre uma equipe sem expô-los a todos os usuários do sistema.

Por exemplo: um grupo `pesquisa` pode incluir todos os pesquisadores de um laboratório. Um arquivo com permissões `rwxrwx---` é totalmente acessível para o dono e para o grupo, mas completamente invisível para qualquer outro usuário.

## Superusuário (root)

O **superusuário** (root no UNIX / administrador no Windows) **ignora todas as regras de proteção**. Ele pode ler, escrever e executar qualquer arquivo, independente das permissões definidas.

> ⚠️ É por isso que comprometer a conta root é o objetivo número um de ataques — com root, o invasor tem controle total sobre o sistema. O Tanenbaum menciona que muitos usuários comuns — especialmente estudantes — dedicam esforço considerável tentando encontrar falhas que permitam tornar-se superusuários sem a senha.
> 

---

# ✅ Resumo do Conceito

- O SO gerencia a **segurança** para que arquivos e recursos sejam acessíveis apenas por usuários autorizados
- No UNIX, cada arquivo tem **9 bits de permissão** — três campos (dono, grupo, outros) com três bits cada (r, w, x)
- **r** = leitura, **w** = escrita, **x** = execução (para arquivos) ou busca (para diretórios)
- As permissões podem ser representadas em **notação octal** (ex: 755, 644, 700)
- O **superusuário/root** ignora todas as regras de proteção — comprometê-lo dá controle total