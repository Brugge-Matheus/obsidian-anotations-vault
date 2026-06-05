---
tags:
  - sistemas-operacionais
  - so/virtualização
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.7.6"
---
# Exonúcleos e Uninúcleos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.7.6

---

# 🔬 O Problema com as Máquinas Virtuais Tradicionais

Em vez de clonar a máquina real como é feito com as máquinas virtuais, outra estratégia é **dividi-la** — em outras palavras, dar a cada usuário um **subconjunto dos recursos**.

O problema das VMs tradicionais (como VM/370) é que elas precisam manter **tabelas de remapeamento** para tudo — cada máquina virtual pensa que tem seu próprio disco de 0 a algum máximo, mas o monitor da máquina virtual tem de manter tabelas para remapear os endereços de discos e todos os outros recursos. Isso adiciona overhead.

---

# 🧪 Exonúcleos — Dando Recursos Reais a Cada VM

> 💡 **O que é um exonúcleo?** Um **exonúcleo** (*exokernel*) é uma abordagem onde, em vez de virtualizar o hardware e fazer cada VM acreditar que tem uma máquina completa, o sistema **divide os recursos reais do hardware** entre as VMs e diz a cada uma exatamente quais recursos ela recebeu.
> 

Com o exonúcleo, uma máquina virtual pode obter os blocos de disco de 0 a 1.023, a próxima pode ficar com os blocos 1.024 a 2.047, e assim por diante:

```
Disco físico (4.096 blocos):
┌──────────────┬──────────────┬──────────────┬──────────────┐
│  VM 1        │  VM 2        │  VM 3        │  VM 4        │
│  blocos 0    │  blocos 1024 │  blocos 2048 │  blocos 3072 │
│  a 1023      │  a 2047      │  a 3071      │  a 4095      │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

## A Vantagem do Exonúcleo

**A vantagem do esquema do exonúcleo é que ele poupa uma camada de mapeamento.** Nos outros projetos, cada máquina virtual no nível do usuário pensa que tem seu próprio disco, com blocos sendo executados de 0 a algum máximo — de maneira que o monitor da máquina virtual tem de manter tabelas para remapear os endereços de discos (e todos os outros recursos).

Com o exonúcleo, esse remapeamento **não é necessário**. O exonúcleo precisa apenas manter o registro de para qual máquina virtual foi atribuído qual recurso.

```
VM Tradicional:           Exonúcleo:
VM pensa: bloco 0         VM sabe: bloco 1024
       ↓ VMM remapeia     (recurso real alocado)
VMM traduz para: 1024     → sem remapeamento necessário
```

## Separação de Conceitos

Esse método ainda tem a vantagem de **separar a multiprogramação** (no exonúcleo) do **código do sistema operacional do usuário** (em espaço do usuário), mas com **menos sobrecarga**, tendo em vista que tudo que o exonúcleo precisa fazer é manter as máquinas virtuais distantes umas das outras.

---

# 📦 LibOS — Sistema Operacional de Biblioteca

As funções do sistema operacional eram vinculadas com os aplicativos na máquina virtual na forma de um **LibOS** (*library operating system* — sistema operacional de biblioteca).

> 💡 **O que é um LibOS?** Um LibOS é um sistema operacional que provê funcionalidade apenas para o(s) aplicativo(s) executado(s) na máquina virtual em nível de usuário. Em vez de ser um SO completo compartilhado por todos, é uma biblioteca vinculada diretamente ao aplicativo.
> 

```
Exonúcleo:

┌─────────────────────────────────────────────────┐
│                  MODO USUÁRIO                   │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │  Aplicativo + LibOS                     │   │
│   │  (SO de biblioteca vinculado ao app)    │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │  Aplicativo + LibOS                     │   │
│   │  (diferente SO de biblioteca)           │   │
│   └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│                  MODO NÚCLEO                    │
│              Exonúcleo                          │
│  (aloca recursos reais, garante isolamento)     │
└─────────────────────────────────────────────────┘
```

Essa ideia foi esquecida por algumas décadas para ser redescoberta nos últimos anos na forma de **uninúcleos**.

---

# 🎯 Uninúcleos — O Renascimento do LibOS

> 💡 **O que é um uninúcleo?** Um **uninúcleo** (*unikernel*) é um sistema mínimo que contém **funcionalidade suficiente apenas para suportar um único aplicativo** (como um servidor web) em uma máquina virtual. O aplicativo e o SO de biblioteca são compilados juntos em uma única imagem executável.
> 

Os uninúcleos têm o potencial de serem **altamente eficientes**, pois a proteção entre o sistema operacional (LibOS) e o aplicativo **não é necessária**: como há apenas um aplicativo na máquina virtual, todo o código pode ser executado em modo núcleo.

```
VM Tradicional:           Uninúcleo:
┌─────────────┐           ┌─────────────────────────┐
│  App        │           │  App + LibOS compilados  │
│  ──── ──   │           │  juntos em uma imagem    │
│  SO completo│           │  (tudo em modo núcleo)   │
│  (Linux/Win)│           └─────────────────────────┘
└─────────────┘
→ SO completo para       → apenas o necessário
  um app só                para aquele app
```

## Por que Uninúcleos são Eficientes?

**1. Sem proteção entre SO e aplicativo** — como há apenas um único aplicativo, não há necessidade de separação entre modo usuário e modo núcleo dentro da VM. Todo o código roda em modo núcleo, eliminando o overhead de troca de contexto.

**2. Tamanho mínimo** — o uninúcleo contém apenas as funcionalidades que o aplicativo realmente usa. Um servidor web simples não precisa de drivers de áudio, sistema de arquivos complexo ou suporte a múltiplos usuários.

**3. Boot ultrarrápido** — por serem extremamente pequenos, uninúcleos inicializam em milissegundos.

---

# ⚖️ Comparativo Geral

|  | VM Tradicional | Exonúcleo | Uninúcleo |
| --- | --- | --- | --- |
| **SO dentro da VM** | Completo (Linux, Windows) | LibOS específico | LibOS mínimo |
| **Remapeamento de recursos** | Sim — VMM mantém tabelas | Não — recursos reais alocados | Não necessário |
| **Proteção SO/App** | Sim — modos núcleo/usuário | Sim | Não — tudo em modo núcleo |
| **Tamanho** | Gigabytes | Pequeno | Minúsculo (MBs) |
| **Flexibilidade** | Alta — qualquer SO | Média | Baixa — um app por VM |
| **Performance** | Overhead do VMM | Menor overhead | Máxima eficiência |
| **Caso de uso** | Uso geral | Pesquisa | Serviços em nuvem, microserviços |

---

# ✅ Resumo do Conceito

- **Exonúcleo** — divide os recursos reais do hardware entre as VMs ao invés de clonar a máquina inteira. Cada VM sabe exatamente quais recursos físicos possui — sem remapeamento necessário. Separa a multiprogramação (exonúcleo) do código do SO (LibOS em modo usuário)
- **LibOS** (*library operating system*) — SO vinculado diretamente ao aplicativo na máquina virtual, provendo funcionalidade apenas para aquele app específico
- **Uninúcleo** — evolução moderna do LibOS. Aplicativo + SO mínimo compilados juntos em uma única imagem. Como há apenas um app, não há necessidade de proteção entre SO e aplicativo — tudo roda em modo núcleo. Altamente eficiente, tamanho mínimo, boot ultrarrápido