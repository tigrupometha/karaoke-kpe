# 🎤 Karaokê KPE – Confra 2025

Aplicação web de karaokê em página única (SPA) feita para a confraternização da KPE Engenharia, com suporte a:

- **Modo Palco (Host)**: notebook/TV que reproduz os vídeos e gerencia a fila
- **Modo Controle (Remote)**: celulares que pesquisam músicas e adicionam à fila em tempo real

Toda a interface usa a identidade visual da KPE (logo oficial, amarelo PANTONE 6 C e roxo PANTONE 2592 C).

---

## 🧱 Arquitetura Geral

- **Tipo de aplicação**: SPA estática em um único arquivo `index.html`
- **Frontend**: HTML5, CSS3 (Tailwind via CDN + estilos customizados) e JavaScript puro (ES6+)
- **Comunicação em tempo real**: [PeerJS](https://peerjs.com/) (WebRTC P2P)
- **Reprodução de vídeo**: YouTube IFrame Player API
- **Busca de vídeos**: APIs públicas compatíveis com YouTube (Piped + fallback para Invidious)
- **Persistência local**: `localStorage` (fila de músicas no Palco)

Não há backend próprio; qualquer servidor estático (Netlify, Vercel, GitHub Pages, etc.) é suficiente.

---

## 🎯 Objetivo do Projeto

Fornecer um **karaokê corporativo** para a KPE Engenharia, permitindo:

- Palco (notebook/TV) tocar os vídeos de karaokê
- Vários celulares conectarem como controle remoto
- Participantes pesquisarem músicas e adicionarem à fila em tempo real
- Fila automática que pula para a próxima música quando a atual termina

---

## 🧩 Funcionalidades Principais

### 1. Seleção de Modo

Na tela inicial, o usuário escolhe:

- **💻 PALCO**
  - Gera um código de sala de 6 caracteres
  - Reproduz os vídeos de karaokê
  - Gerencia e exibe a fila de músicas
  - Sincroniza o estado com todos os controles conectados

- **📱 CONTROLE**
  - Conecta a um palco existente informando o código da sala
  - Permite pesquisar músicas
  - Adiciona músicas à fila do palco
  - Exibe a fila em tempo real (somente leitura)

### 2. Busca de Músicas (YouTube)

- Campo de busca que aceita nome da música ou artista
- Se o termo não contiver "karaoke", o sistema adiciona o prefixo `"karaoke "`
- Uso de múltiplos endpoints públicos (Piped) com fallback:
  - `https://pipedapi.kavin.rocks/search?q=`
  - `https://pipedapi.in.projectsegfau.lt/search?q=`
  - `https://api.piped.yt/search?q=`
  - `https://pipedapi.adminforge.de/search?q=`
- Fallback adicional utilizando Invidious:
  - `https://inv.nadeko.net/api/v1/search?q=...&type=video`
- Filtro apenas para vídeos (`filter=videos` / `type=video`)

### 3. Exibição dos Resultados

Cada resultado de busca inclui:

- Thumbnail do vídeo
- Duração formatada (MM:SS)
- Título e canal
- Número de visualizações normalizado (K/M)
- Botão **➕** para adicionar à fila
- No modo Palco, clicar na thumbnail pode **adicionar e tocar imediatamente**

### 4. Paginação Incremental (Carregar Mais)

- Inicialmente são exibidos **15 resultados**
- Todos os resultados são guardados em `allSearchResults`
- Botão **"⬇️ Carregar mais X músicas"** aparece ao final da lista
- Cada clique exibe **+5 resultados**, até mostrar todos

### 5. Player de Vídeo (Palco)

- Player YouTube incorporado via **IFrame Player API**
- Autoplay ao selecionar uma música
- Estado monitorado por eventos:
  - `onStateChange` → detecta `ENDED` para avançar a fila
  - `onError` → pula automaticamente para a próxima música
- Caixa "Tocando agora" exibe título atual e botão **⏭️** para pular manualmente

### 6. Fila de Músicas (Playlist)

#### Palco (Host)

- Estrutura interna: array `playlist = [{ videoId, title }, ...]`
- Dados persistidos em `localStorage` com chave `kpeKaraokePlaylist`
- Destaques na UI:
  - **Tocando agora** (fundo especial + ícone ▶️)
  - **Próxima** (marcada em verde)
  - Demais itens numerados
- Ações disponíveis:
  - Adicionar música (via botão ➕ ou reprodução imediata)
  - Remover item individual
  - Limpar fila inteira (com `confirm()`)

#### Comportamento Automático

- Quando uma música termina (`ENDED`):
  1. Remove o item atual da `playlist`
  2. Busca o próximo item (primeiro da fila)
  3. Chama `playVideo(videoId, title)` para tocar
  4. Se a fila estiver vazia, exibe placeholder "Fila concluída" e esconde "Tocando agora"

### 7. Sincronização Palco ⇄ Controles (PeerJS)

- Cada sala é identificada por um ID PeerJS: `kpe-karaoke-<ROOMCODE>`
- **Host (Palco):**
  - Cria um Peer fixo com o ID baseado no código da sala
  - Aceita múltiplas conexões (`peer.on('connection')`)
  - Mantém uma lista de `connections[]`
  - Envia atualizações de fila via mensagens:
    - `{ type: 'playlist-sync', playlist, currentVideo }`
  - Recebe requisições de controle:
    - `{ type: 'add-to-queue', videoId, title }`

- **Remote (Controle):**
  - Cria um Peer anônimo
  - Conecta ao host via `peer.connect('kpe-karaoke-' + roomCode)`
  - Ao conectar, passa a enviar:
    - `{ type: 'add-to-queue', ... }`
  - Recebe do host:
    - `{ type: 'playlist-sync', ... }` e atualiza a fila visual

- Status de conexão exibido na UI (com bolinha colorida):
  - `connected`
  - `waiting`
  - `disconnected`

### 8. Layout Específico para Modo Controle (Mobile)

- Classe `remote-mode` aplicada ao `<body>` quando o controle conecta
- Ajustes principais:
  - Oculta a coluna do player
  - Coluna de resultados usa largura total
  - Fila atual (`remoteQueueView`) fixa no rodapé, tipo **painel deslizante**
  - Espaçamento extra inferior para não esconder o conteúdo principal
  - Botões, textos e thumbnails reduzidos para caber bem em telas pequenas

### 9. Notificações (Toasts)

- Função `showToast(message, type)`
- Tipos:
  - `success` → roxo
  - `warning` → amarelo (texto roxo escuro)
  - `info` → roxo
- Desaparecem automaticamente após 2,5 segundos

---

## 🧪 Fluxo de Uso

### Modo Palco (Host)

1. Abrir o site no notebook/PC conectado à TV/som
2. Selecionar **"Entrar como Palco"**
3. Aguardar geração do código da sala (ex: `KHWJZD`)
4. Compartilhar este código com os participantes
5. Fila é carregada do `localStorage` (se existir)
6. Pesquisar músicas e adicionar à fila **ou** aguardar envios dos controles

### Modo Controle (Remote)

1. Abrir o site no celular
2. Selecionar **"Entrar como Controle"**
3. Digitar o código informado pelo palco
4. Ao conectar:
   - Campo de busca habilitado
   - Fila atual visível em painel inferior
5. Pesquisar e usar o botão **➕** para enviar músicas ao palco

---

## 🛠️ Tecnologias Utilizadas

- **Tailwind CSS via CDN**
  - `<script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>`
  - Utilizado principalmente via classes utilitárias + CSS customizado em `<style>`

- **PeerJS**
  - `<script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>`
  - Facilidade de uso sobre WebRTC para comunicação P2P

- **YouTube IFrame Player API**
  - `<script src="https://www.youtube.com/iframe_api"></script>`
  - Player criado dinamicamente via `new YT.Player('ytPlayer', { ... })`

- **APIs de Busca (compatíveis com YouTube)**
  - Piped: endpoints múltiplos para redundância
  - Invidious: fallback quando Piped falha

- **LocalStorage**
  - `localStorage.setItem('kpeKaraokePlaylist', JSON.stringify(playlist))`

---

## 📂 Estrutura do Projeto

Como o projeto é uma SPA em um único arquivo, a estrutura é simples:

```text
/
├── index.html   # Aplicação completa (HTML, CSS, JS)
└── README.md    # Este documento
```

Dentro do `index.html`:

- `<head>`
  - Metadados, título, fontes e inclusão de Tailwind + PeerJS + YouTube API
  - `<style>` com todo o CSS customizado (identidade KPE + responsividade + animações)

- `<body>`
  - Header com logo da KPE e badge "CONFRA 2025"
  - Seção de seleção de modo (Palco / Controle)
  - Tela de entrada de código (para modo Controle)
  - App principal com:
    - Player (somente Palco)
    - Busca + resultados
    - Fila do palco
    - Fila do controle (modo remoto)
  - Overlays de loading e toasts

- `<script>`
  - Estado global (`appMode`, `playlist`, `peer`, etc.)
  - Funções de inicialização (host e remote)
  - Lógica de PeerJS
  - Lógica de busca e paginação
  - Lógica de player e fila
  - Helpers (formatadores, toasts, loading)

---

## 🚀 Execução em Ambiente Local

Como o projeto é estático, existem duas formas simples de rodar localmente:

### 1. Abrir Diretamente no Navegador (sem servidor)

1. Baixar/clonar o projeto
2. Abrir o arquivo `index.html` diretamente no navegador

> ⚠️ Observação: algumas funcionalidades P2P (PeerJS/WebRTC) e a API do YouTube podem requerer contexto seguro (HTTPS ou `localhost`). Para testes mais realistas, use um servidor local.

### 2. Usar um Servidor HTTP Local

Qualquer servidor estático serve. Exemplos:

**Usando Node + `serve`:**

```bash
npm install -g serve
serve .
```

**Usando Python 3:**

```bash
python -m http.server 8080
```

Depois, acessar em:

```text
http://localhost:8080/
```

---

## 🌐 Deploy / Hospedagem

Como é um site estático, pode ser hospedado em qualquer serviço de static hosting:

- **Netlify** (recomendado pela simplicidade)
- **Vercel**
- **GitHub Pages**
- **Azure Static Web Apps / S3 / etc.**

### Passos Gerais

1. Enviar o `index.html` para o serviço de hospedagem
2. Garantir que o site esteja acessível via **HTTPS**
3. No dia da confraternização, abrir a URL do site no notebook/TV (modo Palco)
4. Participantes acessam a mesma URL em seus celulares (modo Controle)

---

## 🔐 Considerações de Segurança e Limitações

- O uso de PeerJS/WebRTC requer contexto seguro (HTTPS) para funcionar em todos os navegadores
- A aplicação depende de serviços de terceiros (Piped/Invidious/YouTube), que podem sofrer instabilidades
- Não há autenticação nem autorização; o controle é baseado apenas no conhecimento do código da sala
- A fila é salva apenas no navegador do **Palco** (não é compartilhada entre dispositivos se o palco mudar de máquina)

---

## 🧹 Manutenção e Extensões Futuras

Possíveis evoluções técnicas:

- Múltiplas salas em paralelo com um painel administrativo para o organizador
- Limitar número de músicas adicionadas por usuário (rate limiting por sessão)
- Identificação opcional do participante (nome) ao adicionar à fila
- Filtros adicionais na busca (idioma, duração)
- Estatísticas de uso (músicas mais tocadas na festa)

---

## 👨‍💻 Contato Técnico

Este projeto foi estruturado para ser **auto-contido** e de fácil manutenção. Qualquer desenvolvedor web com conhecimentos em HTML/CSS/JS conseguirá:

- Ler o `index.html`
- Ajustar estilos e textos
- Alterar ou adicionar endpoints de busca
- Expandir a lógica do player e da fila

Para evoluções mais avançadas (ex.: backend próprio, analytics, autenticação), recomenda-se criar uma API dedicada e modularizar a lógica atualmente toda embutida em `index.html`.
