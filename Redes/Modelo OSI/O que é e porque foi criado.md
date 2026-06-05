---
tags:
  - redes
  - redes/osi
---

# O que é e porque foi criado?

**O Modelo OSI** não é um software ou um hardware, mas sim um **mapa conceitual**.

---

## 1. O que é o Modelo OSI?

O **Modelo OSI** (*Open Systems Interconnection*, ou Interconexão de Sistemas Abertos) é um modelo de referência de **7 camadas** que descreve como a informação de um aplicativo em um computador se move através de uma rede física até um aplicativo em outro computador.

Ele foi desenvolvido pela **ISO** (*International Organization for Standardization*) em **1984**.

---

## 2. Por que ele foi criado? (O Problema do "Caos")

Antes da década de 80, o mundo das redes era um verdadeiro "Velho Oeste". Cada grande fabricante de tecnologia (como IBM, Apple, DEC) criava seus próprios protocolos e regras de comunicação.

### 2.1. A Falta de Interoperabilidade

Se uma empresa comprasse computadores da IBM, ela só poderia usar equipamentos de rede da IBM. Um computador da marca "A" não conseguia "falar" com um computador da marca "B" porque eles usavam linguagens e cabos completamente diferentes. As redes eram **fechadas e proprietárias**.

### 2.2. A Necessidade de Padronização

Para que a internet e as redes globais pudessem crescer, era preciso que qualquer fabricante pudesse criar um produto que funcionasse com o de outro. A ISO criou o Modelo OSI para:

- **Criar uma linguagem comum:** Para que engenheiros e desenvolvedores pudessem discutir redes usando os mesmos termos.
- **Dividir para conquistar:** Ao quebrar o processo complexo de comunicação em 7 partes menores (camadas), ficou mais fácil desenvolver novas tecnologias. Se alguém inventa um novo tipo de fibra óptica (Camada 1), isso não obriga o desenvolvedor do navegador (Camada 7) a mudar o código dele.

---

## 3. Os 3 Objetivos Principais do Modelo OSI

1. **Interoperabilidade:** Garantir que sistemas diferentes se comuniquem de forma transparente.
2. **Modularidade:** Permitir que uma camada seja alterada sem afetar as outras (independência tecnológica).
3. **Facilitação de Ensino e Diagnóstico:** Ajudar técnicos a identificar onde está um problema. Se o "ping" não funciona, o problema pode ser na Camada 3 (Rede); se o cabo está cortado, o problema é na Camada 1 (Física).

---

## 4. Analogia

> **Analogia:** Imagine o Modelo OSI como as regras de uma **Copa do Mundo**. Não importa se um jogador é brasileiro, japonês ou alemão; todos seguem as mesmas regras do jogo (tamanho do campo, tempo de partida, faltas). O Modelo OSI é o "livro de regras" que permite que computadores de "nacionalidades" (fabricantes) diferentes joguem no mesmo campo (a rede).

---

**Dica de estudo:** Embora hoje usemos na prática o modelo **TCP/IP** (que é mais simples), o **Modelo OSI** continua sendo o padrão usado no mundo inteiro para **ensino, certificações e diagnóstico de problemas**.

---

## Conexão com Sistemas Operacionais

O OSI é um modelo de **camadas de abstração** — cada camada só se comunica com as camadas adjacentes (a de cima e a de baixo). Esse princípio é idêntico ao modelo de camadas do sistema operacional: **hardware → kernel → userspace**. O kernel não "pula" camadas para falar com o usuário, assim como a camada de rede não fala diretamente com a camada física sem passar pela de enlace.

O conceito de **encapsulamento** (cada camada adiciona seu cabeçalho ao dado recebido da camada acima) é análogo ao funcionamento dos **stack frames** em chamadas de função: cada chamada empilha seu próprio frame com endereço de retorno, variáveis locais e parâmetros — sem afetar os frames das outras funções. O processador mantém o registrador SP (Stack Pointer) para saber onde está o topo da pilha, exatamente como cada camada OSI sabe onde começa o seu cabeçalho dentro do PDU → [[Processadores]].

Na prática, quando um processo em userspace quer enviar dados pela rede, ele faz uma chamada de sistema (`write()`, `send()`, `sendto()`) que passa o controle para o kernel. O kernel percorre as camadas do modelo de rede (TCP/IP) de cima para baixo, encapsulando o dado em cada etapa. Isso é exatamente a interface entre userspace e kernel → [[System Calls]], [[Processos]].
