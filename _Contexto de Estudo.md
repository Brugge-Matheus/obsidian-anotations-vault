---
tags:
  - sistemas-operacionais
---
# Contexto para Novo Chat

> Copie e cole este texto ao iniciar um novo chat para continuar os estudos do ponto onde parou.
> 

---

Olá! Estou estudando o livro **Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição** e usamos uma dinâmica específica que preciso que você siga:

## O que estamos fazendo

Leio o livro capítulo por capítulo e tiro dúvidas conceituais com você. Após cada tópico, você faz anotações completas e didáticas no meu Notion, usando seus conhecimentos combinados com as informações do livro que envio via print.

## Dinâmicas que seguimos

1. **Anotações no Notion** — você tem acesso ao meu Notion via MCP. A página principal se chama "Sistemas Operacionais" (ID: `31e095dd-55dc-80ac-a344-fc3447877c2a`). Toda anotação vai para a página correspondente ao tópico — nunca crie páginas novas, apenas preencha as já existentes
2. **Formato das anotações** — completas, didáticas, com diagramas em ASCII quando o livro tiver figuras, callouts para definições de termos novos (sempre que um termo técnico aparecer pela primeira vez, defina-o no lugar), tabelas comparativas e resumo ao final de cada página
3. **Figuras do livro** — sempre que o Tanenbaum mostrar uma figura, reproduza um diagrama equivalente em ASCII nas anotações
4. **Dúvidas conceituais** — respondo suas dúvidas na conversa, e se forem relevantes para o conteúdo, você as salva como notas complementares na página correspondente do Notion
5. **Escopo por mensagem** — sempre indico qual seção/tópico quero anotar, e você faz apenas aquele tópico — nunca avance para o próximo sem minha instrução
6. **Definições inline** — sempre que um termo técnico aparecer pela primeira vez numa anotação, defina-o ali mesmo em um callout, sem precisar de consulta externa

## O que já estudamos — Capítulo 1 (Introdução) ✅

O Capítulo 1 foi concluído integralmente. Os principais tópicos cobertos foram:

**Hardware e revisão:**

- Processadores (pipeline, superescalar, multinúcleo, hyperthreading)
- Memória (hierarquia, cache, RAM, memória virtual)
- Armazenamento não volátil (HD, SSD)
- Dispositivos de I/O (controladores, drivers, DMA, interrupções)
- Barramentos (PCIe, DDR, DMI, USB, SATA)
- Inicialização do computador (BIOS, UEFI, MBR, GPT, bootloader)

**Conceitos de SO:**

- Processos (espaço de endereçamento, tabela de processos, árvore de processos, pilha, heap, COW)
- Espaços de endereçamento (memória virtual, paginação, MMU, page fault, swap, thrashing)
- Arquivos (sistema de arquivos, diretórios, montagem, arquivos especiais, pipes)
- Proteção (bits rwx, owner/group/others, superusuário, shell, GUI)
- Hardware de proteção (modo núcleo/usuário, registradores base/limite, MMU)

**Chamadas de sistema:**

- System calls (mecanismo, 11 passos do read(), POSIX, assembly, rotinas de biblioteca)
- Syscalls para gerenciamento de processos (fork, execve, waitpid, exit, COW, árvore de processos, init/systemd)
- Syscalls para gerenciamento de arquivos (open, close, read, write, lseek, stat, descritores)
- Syscalls para gerenciamento de diretórios (mkdir, rmdir, link, unlink, mount, umount, i-nodes)
- Syscalls diversas (chdir, chmod, kill/sinais, time/Y2K38)
- API do Windows (modelo event-driven, WinAPI, CreateProcess, ausência de hierarquia, COW diferente)

**Estrutura de sistemas operacionais:**

- Sistemas Monolíticos (procedimentos, DLLs, bibliotecas compartilhadas)
- Sistemas de Camadas (THE, MULTICS, anéis x86)
- Micronúcleos (MINIX 3, POLA, servidor de reencarnação, LibOS)
- Modelo Cliente-Servidor
- Máquinas Virtuais (VM/370, hipervisor tipo 1 e 2, paravirtualização, JVM, contêineres)
- Exonúcleos e Uninúcleos

**O mundo de acordo com a linguagem C:**

- A linguagem C (ponteiros, ausência de GC)
- Arquivos de cabeçalho e modelo de execução (macros, #include, make, linker, segmentos)

## Próximo tópico

**Capítulo 2 — Processos e Threads**, começando pela seção **2.1 — Processos**