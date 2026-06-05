---
tags:
  - nginx
  - nginx/cgi
---

# FastCGI no Nginx

### Implementação do FastCGI no Nginx: Arquitetura e Fluxo

Diferente de servidores legados, o Nginx não possui um interpretador de linguagens (como o PHP) embutido em seu código. Ele trata o processamento dinâmico como um serviço externo, utilizando o protocolo **FastCGI** para a comunicação.

### 1. O Modelo de Processos Persistentes (Anti-Fork)

Ao contrário do CGI antigo, que criava um processo do zero para cada clique do usuário, o Nginx se conecta a um **Pool de Processos Persistentes** (geralmente gerenciados pelo **PHP-FPM**).

- **Workers Pré-alocados:** Os processos "trabalhadores" do PHP já estão carregados na memória, com o interpretador e extensões prontos.
- **Loop de Escuta:** Esses processos ficam em um loop infinito aguardando conexões. Quando uma tarefa termina, o processo não morre; ele volta para a fila de espera.
- **Ganho de Performance:** Elimina-se o custo de `fork()` e `exec()` do sistema operacional em cada requisição.

### 2. O Fluxo da Requisição (Passo a Passo)

1. **Triagem:** O Nginx recebe a requisição HTTP. Se a extensão for `.php`, ele aciona o módulo `ngx_http_fastcgi_module`.
2. **Tradução de Protocolo:** O Nginx traduz os dados do HTTP (texto plano) para o formato binário do **FastCGI**.
3. **Encaminhamento (The Pass):** O pacote é enviado para o PHP-FPM através de um **Socket** (Unix ou TCP).
4. **Execução Isolada:** Um Worker livre do PHP-FPM assume a tarefa, executa o código e devolve a resposta.
5. **Entrega:** O Nginx recebe a resposta binária, converte de volta para HTTP e entrega ao navegador do cliente.

---

### O Papel do PHP-FPM (FastCGI Process Manager)

O PHP-FPM é o software que realmente gerencia os processos PHP. O Nginx fala com o FPM, e o FPM fala com os Workers.

- **Paralelismo via Multi-processos:** Como o PHP é *single-threaded* (um processo faz uma coisa por vez), o paralelismo é alcançado tendo **vários Workers** rodando simultaneamente.
- **Gestão de Memória:** O FPM pode "reciclar" um Worker após ele atender X requisições para evitar vazamentos de memória (*memory leaks*).
- **Isolamento:** Se um script PHP travar ou der erro fatal, apenas aquele Worker específico é afetado. O Nginx e os outros Workers continuam operando.

---

### Canais de Comunicação: Unix vs TCP

| Tipo | Descrição | Quando usar? |
| --- | --- | --- |
| **Unix Socket** | Comunicação via arquivo no sistema (`.sock`) | Nginx e PHP no **mesmo servidor**. É mais rápido (menos overhead). |
| **TCP Socket** | Comunicação via rede (`IP:Porta`) | Nginx e PHP em **servidores diferentes**. Permite escalabilidade horizontal. |

---

### Exemplo de Bloco de Configuração

```
location ~ \\.php$ {
    # Endereço do PHP-FPM (Socket Unix para performance local)
    fastcgi_pass unix:/var/run/php/php-fpm.sock;

    # Arquivo padrão de entrada
    fastcgi_index index.php;

    # Variáveis essenciais do protocolo
    include fastcgi_params;

    # Informa ao PHP onde o arquivo físico está no disco
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

---

### Resumo para Revisão Rápida

- **CGI:** 1 Requisição = 1 Processo novo (Lento).
- **FastCGI:** 1 Requisição = 1 Worker persistente (Rápido).
- **PHP-FPM:** O gerente que controla os Workers.
- **Nginx:** O intermediário que traduz HTTP para FastCGI.
- **Servidor Embutido:** Quando a aplicação aprende a falar HTTP (Node/Go), mas ainda usa Nginx como escudo (Proxy).

---

## Conexão com Sistemas Operacionais

- **Unix Domain Socket vs TCP** — `fastcgi_pass unix:/run/php/php8.2-fpm.sock` conecta via Unix Domain Socket: não há camada IP, os dados são copiados diretamente entre buffers do kernel, sem TCP handshake. É mais rápido que TCP loopback (127.0.0.1:9000), que percorre toda a pilha TCP dentro do kernel mesmo sendo local → [[System Calls]], [[Arquivos]] (o socket file é um inode especial no sistema de arquivos)

- **TCP loopback** — `fastcgi_pass 127.0.0.1:9000` usa TCP completo: o pacote passa pela camada de socket, pelo TCP state machine do kernel, segmentação e reassembly. Ligeiramente mais lento localmente, mas funciona entre máquinas distintas → [[System Calls]]

- **fastcgi_buffer_size e fastcgi_buffers** — o Nginx aloca memória no espaço do kernel para armazenar a resposta FastCGI antes de enviá-la ao cliente. Esse buffering evita que o worker PHP fique bloqueado aguardando um cliente lento; o Nginx drena o buffer para o cliente enquanto o PHP já pode atender a próxima requisição → [[Memória]]

- **Formato de registro FastCGI** — cada mensagem FastCGI tem um cabeçalho de 8 bytes com: version (1 byte), type (1 byte), requestId (2 bytes), contentLength (2 bytes), paddingLength (1 byte), reserved (1 byte), seguido do corpo. Esse protocolo binário é eficiente de parsear comparado a texto → [[Bits e Bytes]]

---

## Conexão com Go

- **Go como alternativa ao FastCGI** — em vez de configurar Nginx + PHP-FPM via FastCGI, uma aplicação Go usa `http.ListenAndServe` e se comunica com o Nginx via proxy reverso HTTP normal. Não há protocolo FastCGI, não há pool manager separado — o próprio runtime de Go gerencia goroutines por requisição → [[HTTP (net-http)]], [[Goroutines]]
