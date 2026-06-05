---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.1.3"
---
# Término de Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.1.3

---

# ⏹️ 2.1.3 — Término de Processos

Após um processo ter sido criado, ele começa a ser executado e realiza qualquer que seja o seu trabalho. No entanto, nada dura para sempre, nem mesmo os processos. Cedo ou tarde, o novo processo terminará, normalmente devido a uma das quatro condições abaixo.

## As quatro condições de término

| # | Condição | Tipo | Descrição |
| --- | --- | --- | --- |
| 1 | **Saída normal** | Voluntária | O processo concluiu seu trabalho e chama `exit` (UNIX) ou `ExitProcess` (Windows) |
| 2 | **Saída por erro** | Voluntária | O processo descobre um erro fatal e decide encerrar por conta própria |
| 3 | **Erro fatal** | Involuntária | O processo causa um erro que o SO detecta e força o encerramento |
| 4 | **Encerrado por outro processo** | Involuntária | Outro processo executa uma chamada de sistema para matar o processo |

---

## 1. Saída normal — voluntária

A maioria dos processos termina por ter realizado o seu trabalho. Quando um compilador termina de traduzir o programa dado a ele, por exemplo, executa uma chamada de sistema para dizer ao SO que terminou.

> 💡 **`exit` (UNIX) / `ExitProcess` (Windows):** chamada de sistema usada por um processo para informar ao SO que concluiu sua execução e deve ser encerrado de forma limpa. Libera todos os recursos alocados (memória, descritores de arquivo etc.).
> 

Programas baseados em tela também dão suporte ao término voluntário. Processadores de texto, visualizadores da Internet e programas similares sempre têm um ícone ou item no *menu* em que o usuário pode clicar para dizer ao processo para remover quaisquer arquivos temporários que ele tenha aberto e então concluí-lo.

---

## 2. Saída por erro — voluntária

A segunda razão para o término é que o processo descobre um **erro fatal** por conta própria e decide encerrar. Por exemplo, se um usuário digita o comando:

```
cc algo.c
```

para compilar o programa `algo.c` e esse arquivo não existe, o compilador simplesmente anuncia esse fato e termina a execução.

Processos interativos baseados em tela geralmente **não** terminam nesses casos — em vez disso, abrem uma caixa de diálogo e pedem ao usuário para tentar de novo. A saída por erro é mais comum em processos de lote e ferramentas de linha de comando.

---

## 3. Erro fatal — involuntário

A terceira razão é um **erro causado pelo processo**, muitas vezes decorrente de um erro de programa (*bug*). Exemplos incluem:

- Executar uma **instrução ilegal**
- Referenciar uma **memória inexistente**
- **Dividir um número por zero**

> ⚠️ **Tratamento de sinais no UNIX:** em alguns sistemas (p. ex., UNIX), um processo pode dizer ao SO que ele gostaria de lidar sozinho com determinados erros. Nesse caso, o processo é **sinalizado** (interrompido) em vez de terminado quando ocorrer um dos erros — dando a ele a chance de se recuperar ou encerrar de forma controlada.
> 

---

## 4. Encerrado por outro processo — involuntário

A quarta razão ocorre quando um processo executa uma chamada de sistema dizendo ao SO para **terminar outro processo**.

> 💡 **`kill` (UNIX):** chamada de sistema que envia um sinal a outro processo, podendo encerrá-lo. O processo que envia o sinal precisa ter a **autorização necessária** para fazê-lo — normalmente precisa ser o mesmo usuário dono do processo alvo, ou o superusuário.
> 

> 💡 **`TerminateProcess` (Windows):** função Win32 equivalente ao `kill` do UNIX para encerrar outro processo forçadamente.
> 

### Encerramento em cascata

Em alguns sistemas, quando um processo é finalizado — seja voluntariamente ou de outra maneira — **todos os processos que ele criou também são encerrados de imediato**.

> ⚠️ Nem o UNIX, tampouco o Windows, funcionam dessa maneira. No UNIX e no Windows, processos-filhos continuam existindo após o término do pai — eles não são encerrados automaticamente em cascata.
> 

---

# ✅ Resumo do Conceito

- **Quatro condições encerram processos:** saída normal (voluntária), saída por erro (voluntária), erro fatal (involuntária) e encerramento por outro processo (involuntária)
- **`exit` / `ExitProcess`** — chamadas usadas pelo próprio processo para encerrar voluntariamente e liberar recursos
- **Saída por erro voluntária** — o processo detecta um problema (ex: arquivo inexistente) e encerra por conta própria, sem intervenção do SO
- **Erro fatal involuntário** — instrução ilegal, acesso a memória inválida ou divisão por zero fazem o SO encerrar o processo; no UNIX, o processo pode optar por receber um sinal em vez de ser terminado
- **`kill` / `TerminateProcess`** — chamadas que permitem a um processo encerrar outro, desde que tenha autorização
- **Sem cascata no UNIX e Windows** — o término do pai não encerra automaticamente os filhos; eles continuam em execução de forma independente