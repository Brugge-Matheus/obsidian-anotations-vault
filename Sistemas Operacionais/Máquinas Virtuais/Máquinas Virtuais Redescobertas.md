---
tags:
  - sistemas-operacionais
  - so/virtualização
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.7.5 (continuação)"
---
# Máquinas Virtuais Redescobertas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.7.5 (continuação)

---

# 🔄 Por que as VMs Foram Redescobertas?

Embora a IBM tenha um produto de máquina virtual disponível há quatro décadas, a ideia da virtualização foi em grande parte **ignorada no mundo dos PCs** até há pouco tempo. Nos últimos anos, porém, novas necessidades, novo software e novas tecnologias combinaram-se para torná-la um tópico de grande interesse.

---

# 🏢 Hipervisores — Tipo 1 e Tipo 2

> 💡 **O que é um hipervisor?** Um hipervisor (*hypervisor*) — também chamado de **VMM** (*Virtual Machine Monitor*) — é o software que cria e gerencia máquinas virtuais, controlando o acesso ao hardware físico. Existem dois tipos fundamentais.
> 

## Hipervisor Tipo 1 — Bare Metal

O hipervisor roda **diretamente no hardware**, sem SO hospedeiro abaixo. Ele mesmo gerencia o hardware e oferece máquinas virtuais para os SOs convidados:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   SO hóspede │  │   SO hóspede │  │   SO hóspede │
│   (Linux)    │  │  (Windows)   │  │   (outro)    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       └──────────────────┴──────────────────┘
                          │
            ┌─────────────▼─────────────┐
            │    Hipervisor Tipo 1      │
            │  (roda direto no hardware)│
            └─────────────┬─────────────┘
                          │
            ┌─────────────▼─────────────┐
            │       Hardware físico     │
            └───────────────────────────┘
```

**Exemplos:** VMware ESXi, Microsoft Hyper-V, Xen.

## Hipervisor Tipo 2 — Hosted

O hipervisor roda **sobre um SO hospedeiro** existente, como um processo comum. Ele usa o SO hospedeiro para criar processos, armazenar arquivos e realizar outras funções:

```
┌──────────────┐  ┌──────────────┐
│   SO hóspede │  │   SO hóspede │
│   (Linux)    │  │  (Windows)   │
└──────┬───────┘  └──────┬───────┘
       └──────────┬──────┘
                  │
     ┌────────────▼────────────┐
     │   Hipervisor Tipo 2     │
     │  (processo do usuário)  │
     └────────────┬────────────┘
                  │
     ┌────────────▼────────────┐
     │  SO Hospedeiro          │
     │  (Windows, Linux, macOS)│
     └────────────┬────────────┘
                  │
     ┌────────────▼────────────┐
     │     Hardware físico     │
     └─────────────────────────┘
```

**Exemplos:** VirtualBox, VMware Workstation, QEMU.

## A Diferença Prática

Na prática, a distinção real entre os dois tipos é que o tipo 2 usa um **sistema operacional hospedeiro** e o seu sistema de arquivos para criar processos, armazenar arquivos e assim por diante. Um hipervisor tipo 1 não tem suporte subjacente e precisa realizar todas essas funções sozinho.

Após um hipervisor tipo 2 ser inicializado, ele lê o arquivo de imagem da instalação para o **sistema operacional hóspede** escolhido e instala o OS hóspede em um **disco virtual** — que é apenas um grande arquivo no sistema de arquivos do SO hospedeiro. Quando o OS hóspede é inicializado, ele faz o mesmo que no hardware real — em geral iniciando alguns processos de segundo plano e então uma GUI. Para o usuário, o OS hóspede comporta-se como quando está sendo executado diretamente no hardware, embora esse não seja o caso.

---

# ✅ Resumo do Conceito

- **Hipervisor Tipo 1** (bare metal) — roda diretamente no hardware, sem SO abaixo. Mais eficiente e usado em data centers (VMware ESXi, Hyper-V, Xen)
- **Hipervisor Tipo 2** (hosted) — roda sobre um SO hospedeiro como um processo comum. Mais fácil de instalar e usar (VirtualBox, VMware Workstation)