---
tags:
  - nginx
  - nginx/cgi
---

# CGI e FastCGI

### O que é CGI?

**CGI** significa **Common Gateway Interface**.

Não é exatamente uma linguagem nem um servidor.
É uma **interface padrão** que define como um servidor web pode executar um programa externo para gerar conteúdo dinâmico.

Em outras palavras:

- o navegador faz uma requisição;
- o servidor web recebe essa requisição;
- em vez de devolver um arquivo estático pronto, ele chama um programa externo;
- esse programa processa a lógica;
- o programa devolve uma saída;
- o servidor web entrega essa saída ao cliente como resposta HTTP.

---

### Contexto histórico: por que o CGI foi criado?

No começo da web, os sites eram basicamente compostos por **arquivos estáticos**:

- HTML
- imagens
- arquivos simples de texto

Ou seja, o servidor apenas entregava arquivos prontos.
Não havia lógica de negócio, autenticação sofisticada, geração dinâmica de páginas ou interação complexa com banco de dados.

Com o crescimento da web, surgiu a necessidade de:

- processar formulários;
- gerar páginas personalizadas;
- consultar bancos de dados;
- criar sistemas com login;
- responder com conteúdo diferente para cada usuário.

Os servidores web da época não tinham um mecanismo padronizado para executar programas externos e integrar isso à resposta HTTP.

O CGI surgiu para resolver exatamente isso:

> **criar um padrão universal para ligar o servidor web a programas externos capazes de gerar conteúdo dinâmico**

---

### Qual problema o CGI resolvia?

Antes do CGI, não havia uma forma padronizada e portável de fazer um servidor web "delegar" processamento para um programa externo.

O CGI resolveu problemas como:

- permitir páginas dinâmicas;
- receber dados de formulários;
- integrar a web com scripts e aplicações;
- padronizar a comunicação entre servidor web e programa;
- tornar possível usar diferentes linguagens para gerar conteúdo web.

Com CGI, o servidor podia executar programas escritos em:

- C
- Perl
- Shell
- Python
- e depois outras linguagens também

Isso foi um grande salto para a web.

---

### Como o CGI funciona?

O modelo clássico funciona assim:

1. o cliente envia uma requisição HTTP;
2. o servidor web identifica que aquele recurso deve ser tratado por CGI;
3. o servidor cria um **novo processo** no sistema operacional;
4. esse processo executa o script ou programa CGI;
5. o servidor passa informações da requisição para esse programa, geralmente por:
    - variáveis de ambiente;
    - entrada padrão (`stdin`);
6. o programa processa os dados;
7. o programa escreve a resposta na saída padrão (`stdout`);
8. o servidor lê essa saída e envia a resposta HTTP ao cliente.

---

### Como o servidor passa os dados para o CGI?

O CGI usa principalmente:

### Variáveis de ambiente

O servidor preenche variáveis com informações da requisição, por exemplo:

- método HTTP
- query string
- content type
- content length
- IP do cliente
- path solicitado

Exemplos clássicos:

- `REQUEST_METHOD`
- `QUERY_STRING`
- `CONTENT_LENGTH`
- `CONTENT_TYPE`
- `SCRIPT_NAME`
- `REMOTE_ADDR`

### Entrada padrão

Se a requisição tiver corpo, como em um `POST`, os dados podem ser enviados ao programa pela entrada padrão.

### Saída padrão

O programa devolve algo como:

- cabeçalhos HTTP
- linha em branco
- corpo da resposta

Exemplo conceitual:

- `Content-Type: text/html`
- corpo HTML gerado dinamicamente

---

### Por que o CGI foi tão importante?

Porque ele permitiu que a web saísse do modelo puramente estático.

Sem CGI, não existiria facilmente, na época:

- formulários funcionais;
- páginas dinâmicas;
- aplicações web iniciais;
- integração entre navegador e backend.

Ele foi um marco da transição da **web de documentos** para a **web de aplicações**.

---

### Qual era o grande problema do CGI?

O principal problema do CGI clássico era a **performance**.

Cada requisição normalmente gerava:

- um novo processo;
- inicialização do interpretador ou binário;
- carregamento de dependências;
- execução da lógica;
- finalização do processo.

Isso era muito caro em termos de:

- CPU
- memória
- tempo de resposta
- escalabilidade

### Exemplo do problema

Imagine 100 requisições simultâneas para um script em Perl ou PHP via CGI clássico.

O servidor precisaria criar 100 processos separados.
Isso aumenta muito o custo do sistema operacional e degrada o desempenho rapidamente.

---

### Resumo do problema do CGI

O CGI resolveu o problema da **dinamicidade**, mas criou ou evidenciou um novo problema:

> **cada requisição custava caro demais**

Ou seja, ele foi revolucionário funcionalmente, mas ineficiente operacionalmente para ambientes maiores.

---

### O que é FastCGI?

**FastCGI** é uma evolução do CGI.

Ele mantém a ideia de separar o servidor web da aplicação dinâmica, mas muda radicalmente a forma como essa comunicação acontece.

Em vez de criar um novo processo para cada requisição, o FastCGI usa **processos persistentes**.

Esses processos ficam rodando continuamente, esperando requisições do servidor web.

---

### Qual problema o FastCGI resolveu?

O FastCGI foi criado para resolver principalmente:

- o alto custo de criar um processo por requisição;
- a baixa escalabilidade do CGI clássico;
- a lentidão causada por inicialização repetida da aplicação;
- a dificuldade de atender alto volume de tráfego com conteúdo dinâmico.

Então, a ideia foi:

> **manter processos da aplicação já ativos, prontos para receber múltiplas requisições ao longo do tempo**

Isso reduz drasticamente o overhead.

---

### Como o FastCGI funciona?

O fluxo conceitual é assim:

1. o cliente faz uma requisição;
2. o servidor web recebe a requisição;
3. o servidor identifica que aquele recurso deve ser encaminhado a um processo FastCGI;
4. em vez de criar um novo processo, ele envia a requisição para um processo FastCGI já existente;
5. esse processo executa a lógica e devolve a resposta;
6. o servidor web responde ao cliente.

Ou seja:

- CGI clássico → processo novo por requisição
- FastCGI → processo persistente reaproveitado

---

### Diferença central entre CGI e FastCGI

### CGI clássico

- comunicação simples por execução isolada;
- um processo novo para cada requisição;
- mais lento;
- menos escalável.

### FastCGI

- comunicação entre servidor e aplicação por protocolo próprio;
- processos persistentes;
- mais rápido;
- mais eficiente;
- melhor para ambientes de produção.

---

### Por que "Fast" CGI?

Porque ele foi projetado para ser uma versão mais eficiente do CGI, reduzindo justamente o custo operacional que tornava o CGI clássico lento.

O nome destaca essa ideia de aceleração por:

- reutilização de processos;
- menor overhead por requisição;
- melhor uso de recursos.

---

### Como o FastCGI se encaixa na arquitetura moderna?

O FastCGI foi muito importante na consolidação de uma arquitetura em que:

- o **servidor web** cuida de:
    - conexões;
    - sockets;
    - arquivos estáticos;
    - TLS/SSL;
    - proxying;
    - buffering;
- a **aplicação** cuida de:
    - lógica de negócio;
    - processamento dinâmico;
    - integração com banco;
    - sessões;
    - autenticação;
    - renderização dinâmica.

Essa separação tornou a arquitetura mais limpa e escalável.

---

### CGI e FastCGI no ecossistema do PHP

O PHP passou por várias fases de integração com servidores web.

### 1. CGI

No começo, PHP podia rodar via CGI clássico.

Isso funcionava, mas tinha os problemas de performance já citados.

### 2. mod_php

Depois, especialmente com Apache, o PHP passou a ser embutido diretamente no servidor com `mod_php`.

Isso melhorava performance, porque o interpretador ficava carregado dentro do processo do servidor web.

Mas isso também tinha desvantagens:

- acoplamento forte com Apache;
- consumo maior de memória;
- menos isolamento entre servidor web e aplicação;
- menor flexibilidade arquitetural.

### 3. FastCGI / PHP-FPM

Depois, o modelo mais moderno ganhou força: PHP rodando como processo separado, geralmente via **FastCGI**, usando o **PHP-FPM**.

Esse modelo se tornou o padrão moderno para PHP com Nginx e também muito comum com Apache.

---

### O que é PHP-FPM nesse contexto?

**PHP-FPM** significa **PHP FastCGI Process Manager**.

Ele é um gerenciador de processos do PHP que implementa o modelo FastCGI de forma otimizada.

O PHP-FPM:

- mantém pools de processos PHP ativos;
- gerencia quantos processos devem existir;
- cria e recicla workers;
- controla limites;
- melhora observabilidade e tuning.

Em outras palavras:

> hoje, quando se fala de PHP com Nginx, normalmente estamos falando de **Nginx + PHP-FPM via FastCGI**

---

### CGI ainda é usado hoje?

### CGI clássico

Hoje em dia, o **CGI clássico é raro** em sistemas modernos de alto desempenho.

Ele ainda pode aparecer em:

- sistemas legados;
- ambientes simples;
- equipamentos antigos;
- scripts administrativos muito específicos;
- cenários educacionais.

Mas, no geral, ele **não é a escolha moderna** para aplicações web relevantes.

### FastCGI

Já o **FastCGI** ainda é extremamente relevante, especialmente no mundo do PHP.

Então a resposta é:

- **CGI clássico:** quase obsoleto para produção moderna
- **FastCGI:** ainda muito utilizado em vários contextos

---

### O FastCGI ainda é usado hoje em dia?

Sim, muito.

Especialmente em:

- PHP com Nginx
- PHP com Apache
- alguns ambientes de aplicações desacopladas
- cenários em que o servidor web precisa conversar com um processador externo

No entanto, o ecossistema web atual também evoluiu para outros modelos além de FastCGI.

---

### Evolução posterior: além do FastCGI

Com o tempo, surgiram outros padrões e arquiteturas para aplicações web dinâmicas.

### 1. Servidores de aplicação próprios

Muitas linguagens passaram a usar servidores próprios, por exemplo:

- Node.js com servidor HTTP embutido
- Gunicorn / uWSGI para Python
- Puma para Ruby
- servidores Go nativos
- Java com Tomcat / Jetty / Undertow

Nesse modelo, o servidor web como Nginx frequentemente atua como:

- proxy reverso;
- terminador TLS;
- balanceador de carga;
- cache.

A aplicação fica ouvindo em outra porta e o Nginx só encaminha.

### 2. Protocolos e gateways específicos

Surgiram interfaces específicas por ecossistema, como:

- **WSGI** para Python
- **ASGI** para Python assíncrono
- **Rack** para Ruby
- **Servlet API** no Java

Ou seja, muitos ecossistemas evoluíram além do CGI/FastCGI com padrões próprios.

### 3. APIs e microsserviços

Hoje muitas aplicações nem geram HTML no backend tradicional.
Elas expõem APIs e se comunicam com frontend separado.

Nesse contexto:

- Nginx atua como proxy/gateway;
- a aplicação roda em processos ou containers independentes;
- CGI não entra em cena;
- FastCGI aparece principalmente em stacks PHP.

---

### Então CGI/FastCGI ainda representam o padrão atual?

Depende da stack.

### Para PHP

Sim, especialmente o **FastCGI**, através de **PHP-FPM**, continua sendo um padrão amplamente usado e atual.

### Para outras linguagens modernas

Em muitos casos, não.

Hoje é mais comum ver:

- Nginx + Node.js
- Nginx + Gunicorn/Uvicorn
- Nginx + uWSGI
- Nginx + aplicação Go
- Nginx + microsserviços em containers

Nesses casos, o Nginx atua mais como proxy reverso e menos como "invocador" no estilo CGI.

---

### Comparação evolutiva

| Etapa | Característica principal | Problema resolvido | Limitação |
| --- | --- | --- | --- |
| **Web estática** | entrega de arquivos | publicação simples de páginas | sem lógica dinâmica |
| **CGI** | executa programa externo por requisição | introduz conteúdo dinâmico | alto custo por processo |
| **FastCGI** | processos persistentes | melhora performance e escalabilidade | ainda depende de arquitetura específica |
| **App servers modernos** | aplicações com servidores próprios | mais flexibilidade por linguagem | maior diversidade de padrões |

---

### Papel do servidor web nesse processo

Historicamente, o servidor web precisava de um mecanismo para lidar com conteúdo dinâmico.

O CGI e o FastCGI foram formas de fazer essa ponte.

Então o papel do servidor web passou a ser:

- receber conexões HTTP;
- interpretar a requisição;
- decidir se o conteúdo é estático ou dinâmico;
- se dinâmico, encaminhar para um processador apropriado;
- devolver a resposta ao cliente.

No CGI/FastCGI, o servidor web funciona como mediador entre cliente e aplicação.

---

### Visão conceitual do funcionamento no servidor web

### Com CGI

- servidor recebe requisição;
- cria processo;
- passa dados da requisição;
- lê resposta do programa;
- encerra processo.

### Com FastCGI

- servidor recebe requisição;
- envia para processo persistente;
- processo responde;
- conexão lógica termina, mas o processo continua vivo.

Essa persistência é o coração do ganho de desempenho.

---

### Impacto arquitetural do FastCGI

O FastCGI ajudou a consolidar algumas ideias muito importantes:

- separação entre servidor web e runtime da aplicação;
- melhor uso de recursos;
- escalabilidade horizontal;
- isolamento maior entre camadas;
- tuning independente de cada componente.

Isso abriu caminho para arquiteturas mais modernas e modulares.

---

### Situação atual

- CGI clássico é mais legado/histórico;
- FastCGI continua atual no PHP;
- outras stacks modernas usam servidores e interfaces próprias;
- Nginx hoje muitas vezes atua como proxy reverso para aplicações independentes.

---

## Conexão com Sistemas Operacionais

- **CGI: fork() + exec() por requisição** — para cada requisição HTTP, o servidor web chama `fork()` para duplicar seu processo e depois `exec()` para substituir o filho pelo interpretador do script. O custo do fork inclui duplicação das page tables (mesmo com Copy-on-Write), criação de novo `task_struct`, alocação de kernel stack e cópia da tabela de file descriptors → [[Criação de Processos]], [[Memória Virtual]]

- **Passagem de variáveis de ambiente via execve()** — as variáveis `REQUEST_METHOD`, `QUERY_STRING` etc. são passadas ao processo CGI pelo argumento `envp` da chamada `execve()`. Esse é o mecanismo padrão do kernel para transferir o ambiente para um novo processo → [[Criação de Processos]]

- **Comunicação via pipe** — o processo CGI escreve a resposta em `stdout`; o servidor web lê via pipe. Um pipe é um mecanismo IPC do kernel: `pipe()` cria dois file descriptors (leitura e escrita), os dados fluem pelo buffer do kernel com `read()` e `write()` → [[System Calls]]

- **Custo real do fork por requisição** — o kernel precisa duplicar as page tables (mesmo com CoW as entradas precisam ser marcadas como somente leitura), criar um novo `task_struct`, alocar kernel stack e configurar a cópia da tabela de file descriptors. Esse custo é proibitivo em alta carga → [[Criação de Processos]], [[Memória Virtual]]

- **FastCGI: processos de longa duração** — em vez de fork por requisição, os workers FastCGI são processos persistentes que ficam em loop aguardando trabalho. O kernel mantém esses processos em estado "dormindo" (aguardando I/O no socket) até chegarem novas requisições → [[Processos]], [[Estados de Processos]]

- **Protocolo FastCGI: multiplexação sobre socket** — o FastCGI usa framing de registros (headers de 8 bytes + corpo) sobre uma única conexão socket, permitindo múltiplas requisições simultâneas sem criar novos processos. O transporte é via Unix socket ou TCP socket → [[System Calls]]

- **PHP-FPM como pool manager** — o PHP-FPM é um processo gerenciador que faz fork dos workers antecipadamente e roteia requisições para eles. Essa hierarquia (master → pool manager → workers) é um exemplo clássico do modelo de processos do Unix → [[O Modelo de Processos]], [[Hierarquia de Processos]]

---

## Conexão com Go

- **Go como equivalente moderno** — um servidor HTTP em Go é um processo persistente (como FastCGI) que trata cada requisição em uma goroutine separada, sem nunca precisar de `fork()`. A criação de uma goroutine custa ~300ns vs ~100μs de um fork, tornando Go ~300x mais eficiente que CGI → [[Goroutines]]
