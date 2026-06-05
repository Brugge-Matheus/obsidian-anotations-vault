---
tags:
  - redes
  - redes/internet
---

# Termos usados Internet/Web

### **1. Internet x Web**

### **Internet**

A Internet é uma **infraestrutura global** composta por:

- cabos submarinos,
- fibras ópticas,
- roteadores,
- satélites,
- provedores,
- protocolos de rede (principalmente TCP/IP).

É literalmente uma **rede de redes**, interconectando milhões de dispositivos ao redor do mundo. Ela fornece vários serviços além da Web, como:

- e-mail (SMTP),
- transferência de arquivos (FTP),
- SSH,
- VoIP (chamadas pela internet),
- streaming.

### **Web (World Wide Web)**

A Web é **um serviço** que roda sobre a Internet, criado por Tim Berners-Lee em 1989. A Web funciona através de:

- páginas escritas em HTML,
- estilo via CSS,
- interatividade com JavaScript,
- hiperlinks conectando essas páginas.

**Resumo:**
Internet = infraestrutura.
Web = projeto que usa a internet para exibir páginas e informações.

---

### **2. Site, Página, Domínio, URL**

### **Site**

Conjunto organizado de páginas, arquivos, imagens, scripts e funcionalidades que formam uma aplicação acessível pela Web.

### **Página Web**

Documento individual acessado no navegador. Cada página é carregada a partir de uma URL específica.

### **Domínio**

Nome único que identifica um site, usando letras para facilitar o acesso (pois IP é difícil de lembrar). Os domínios têm níveis:

- **TLD (Top-Level Domain):** `.com`, `.org`, `.edu`, `.br`
- **SLD (Second-Level Domain):** nome escolhido (ex: `google`)
- **Subdomínio:** prefixo opcional (ex: `maps.google.com`)

### **URL**

É o endereço completo do recurso. Exemplo detalhado:
`https://www.site.com.br/pasta/artigo?id=20#comentarios`

Componentes:

- `https` → protocolo
- `www` → subdomínio
- `site.com.br` → domínio
- `/pasta/artigo` → caminho
- `?id=20` → parâmetros (query string)
- `#comentarios` → âncora (ponto específico da página)

---

### **3. HTTP, HTTPS, Protocolo**

### **Protocolo**

Conjunto de regras padronizadas que define como a comunicação ocorre. Sem protocolos, cada fabricante faria sua própria "língua", impossibilitando comunicação.

### **HTTP**

Utiliza o modelo **Request-Response**:

1. Navegador envia uma requisição (GET, POST...).
2. Servidor responde com dados (HTML, JSON, imagens etc.).

Camada: **Aplicação** (Modelo OSI 7).

### **HTTPS**

Versão segura do HTTP que utiliza **TLS/SSL** para:

- criptografar dados,
- garantir autenticidade do servidor,
- evitar interceptação (ataques MITM).

O cadeado no navegador indica conexão HTTPS.

---

### **4. Navegador, Servidor, Cliente**

### **Navegador**

Software que interpreta códigos web e renderiza interface visual. Possui:

- motor de layout (Blink, Gecko, WebKit),
- JavaScript engine (V8, SpiderMonkey),
- gerenciamento de abas,
- cache,
- armazenamento local.

### **Cliente**

O lado que inicia a comunicação. Pode ser um navegador, aplicativo mobile, IoT, etc.

### **Servidor**

Máquina física ou virtual que processa requisições. Um servidor web pode usar:

- Apache,
- Nginx,
- IIS,
- Node.js,
- etc.

### **Modelo Cliente–Servidor**

Base da comunicação moderna:

- cliente pede,
- servidor envia,
- cliente interpreta e exibe.

---

### **5. DNS, IP, Porta**

### **IP**

Endereço lógico que identifica dispositivos na Rede. Tipos:

- **IPv4:** 32 bits (ex: `192.168.0.1`)
- **IPv6:** 128 bits (ex: `2001:db8::1`)

Funções:

- identificar a máquina,
- permitir roteamento entre redes.

### **DNS**

O **Domain Name System** traduz domínios em IPs. Ele funciona de forma distribuída e hierárquica, com:

- servidores root,
- TLD Servers,
- Authoritative Servers,
- DNS Recursivo (normalmente fornecido pelo provedor).

Sem DNS, você teria que acessar sites digitando IPs.

### **Porta**

Identifica **qual serviço** será acessado dentro de um mesmo IP.
Exemplos:

- 80 → HTTP
- 443 → HTTPS
- 21 → FTP
- 53 → DNS
- 3306 → MySQL

Cada porta é como uma "porta" de um apartamento dentro de um mesmo prédio (IP).

---

### **6. Download, Upload, Largura de Banda, Latência**

### **Download**

Receber arquivos/dados da internet para sua máquina. Eventos comuns:

- carregar páginas,
- baixar apps,
- assistir vídeos.

### **Upload**

Enviar dados para servidores ou outros dispositivos. Ex:

- mandar fotos,
- enviar formulários,
- lives.

### **Largura de Banda**

É o **tamanho máximo** do "canal" por onde trafega a internet. Medida: Mbps ou Gbps.

Maior largura de banda = mais dados por segundo.

### **Latência**

É o **tempo** que o dado leva para ir e voltar até o servidor.
Importante em:

- jogos online,
- chamadas de vídeo,
- aplicações em tempo real.

Baixa latência = mais responsividade.

---

### **7. Cookie, Cache, Sessão**

### **Cookie**

Arquivo pequeno que armazena dados do usuário, como:

- preferências,
- autenticação,
- carrinhos de compras,
- tokens.

Pode ter:

- data de expiração,
- escopo por domínio,
- regras de segurança (HttpOnly, Secure).

### **Cache**

Armazena localmente arquivos para acelerar carregamentos. Há diferentes caches:

- cache do navegador,
- cache de CDN,
- cache de servidor,
- cache DNS.

Reduz a necessidade de baixar tudo de novo.

### **Sessão**

Identifica um usuário enquanto ele navega em um site autenticado. É normalmente vinculada a:

- cookies de sessão,
- tokens JWT,
- IDs armazenados no servidor.

---

### **8. Frontend, Backend, Fullstack**

### **Frontend**

Parte visual e interativa acessada pelo usuário. Envolve:

- HTML (estrutura),
- CSS (estilo),
- JavaScript (lógica e interatividade),
- Frameworks (React, Vue, Angular),
- ferramentas (Webpack, Vite).

### **Backend**

Parte que roda no servidor, responsável por:

- regras de negócio,
- autenticação,
- persistência de dados,
- geração de conteúdo dinâmico,
- APIs.

Tecnologias comuns:

- Node.js,
- Python (Django, Flask),
- Java (Spring),
- PHP (Laravel),
- C# (.NET).

### **Fullstack**

Domina frontend + backend. Muitas vezes também mexe com:

- banco de dados,
- deploy,
- arquitetura,
- CI/CD.

---

### **9. API, REST, JSON**

### **API**

Interface que permite comunicação entre sistemas. Pode ser:

- Web API,
- API nativa,
- API de hardware.

### **REST**

Representational State Transfer — um conjunto de princípios arquiteturais:

- URLs como identificação de recursos,
- uso de métodos HTTP (GET, POST, PUT, DELETE),
- comunicação sem estado (stateless),
- respostas geralmente em JSON.

### **JSON**

Formato leve para transporte de dados.
Características:

- baseado em chave → valor,
- compatível com quase todas as linguagens,
- fácil de ler e escrever.

---

### **10. ISP, Hosting, CDN**

### **ISP**

Provedor de Internet. Fornece:

- IP público,
- roteamento,
- infraestrutura de conexão.

Exemplos: Vivo, Claro, TIM, etc.

### **Hosting**

Serviço que aloca:

- servidores virtuais ou dedicados,
- bancos de dados,
- armazenamento,
- emails corporativos.

Exemplos: AWS, Azure, Hostinger.

### **CDN**

Content Delivery Network — rede distribuída de servidores que entrega conteúdo perto do usuário.
Benefícios:

- menor latência,
- menores custos de banda,
- disponibilidade global.

Usada para imagens, vídeos, scripts e sites inteiros.

---

## Conexão com Sistemas Operacionais

**HTTP request/response:** Cada requisição e resposta é um fluxo de texto transmitido sobre um socket TCP. O kernel recebe os bytes da rede e os deposita num buffer de recepção do socket. O processo userspace (navegador ou servidor) lê esses bytes com a syscall `recv()` → [[System Calls]], [[Memória]].

**HTML/CSS/JS parsing:** A análise e renderização do HTML, CSS e JavaScript acontecem inteiramente no processo do navegador (espaço de usuário). O motor de layout (Blink, Gecko) é um processo normal do sistema operacional → [[Processos]].

**TLS handshake:** Durante o handshake TLS, o processador executa operações criptográficas intensas (troca de chaves, geração de MACs). Processadores modernos possuem extensões de hardware como **AES-NI** (aceleração de AES) e extensões SHA, que tornam essas operações muito mais rápidas do que implementações puramente em software → [[Processadores]], [[Bits e Bytes]].

**Cookies:** São enviados como texto nos cabeçalhos HTTP (`Cookie:` na requisição, `Set-Cookie:` na resposta). O navegador os armazena em arquivos no sistema de arquivos do usuário (ex: `~/.config/google-chrome/Default/Cookies`, que é um banco SQLite) → [[Arquivos]].

**CDN:** Servidores geograficamente distribuídos que armazenam cópias de conteúdo estático. O roteamento *anycast* faz com que a mesma faixa de IPs seja anunciada em múltiplos pontos geográficos — o roteador mais próximo do cliente responde. Da perspectiva do SO, acessar um CDN é idêntico a acessar qualquer outro servidor: socket → connect → send/recv → [[Dispositivos de IO]].

---

## Conexão com Go

- `http.Get("https://exemplo.com")` — cliente HTTP que abre socket TCP, realiza handshake TLS e retorna o body da resposta → [[HTTP (net-http)]]
- `json.Unmarshal(data, &struct{})` — deserialização de JSON recebido em respostas de API → [[JSON (marshal e unmarshal)]]
