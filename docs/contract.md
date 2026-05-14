# Contrato da POC v2 — Video Story

Documentação do protocolo de comunicação entre o app Exame Plus e a página `index.html` desta POC.

## Visão geral

```
App Exame Plus (React Native)          POC v2 (HTML/JS na WebView)
─────────────────────────────          ──────────────────────────────
                                       DOMContentLoaded → init()
                          ←── { type: "ready" }
injectJavaScript(
  '__pocV2LoadStory("base64url")'
)                         ───►  __pocV2LoadStory(encoded)
                                  → PayloadCodec.decode()
                                  → PayloadValidator.validate()
                                  → PlayerFactory.create()
                                  → OverlayRenderer.render()
                          ←── { type: "player:ready" }
                          ←── { type: "player:play" }
                               ...
(usuário toca fullscreen) ←── { type: "fullscreen:enter" }
app trava orientação em landscape
(usuário toca fechar)     ←── { type: "fullscreen:exit" }
                          ←── { type: "close" }
app desbloqueia orientação, fecha WebView
```

---

## Canal de comunicação

| Direção | Mecanismo | Notas |
|---------|-----------|-------|
| Web → App | `window.ReactNativeWebView.postMessage(JSON.stringify(msg))` | Sempre JSON estruturado |
| App → Web | `webViewRef.injectJavaScript('window.__pocV2LoadStory("…"); true;')` | Payload base64url |

O canal app→web usa `injectJavaScript` (chamada de função direta), **não** `window.postMessage`. Isso evita conflito com os eventos postMessage do player Vimeo, que usam `window.addEventListener('message', ...)` internamente.

---

## Payload: `WebStoryVideoPayload`

```typescript
type WebStoryVideoPayload = {
  version: 1;

  story: {
    id: string;          // obrigatório
    slug: string;
    title: string;
    description?: string;
    sourceLabel?: string;
    author?: string;
    publishedAt?: string;
    category?: string;
  };

  media: {
    provider: 'youtube' | 'youtube_live' | 'vimeo';  // obrigatório
    videoId: string;                                   // obrigatório
    sourceUrl?: string;    // URL original (para abrir externamente)
    posterUrl?: string;    // thumbnail de fallback
    aspectRatio?: string;  // '16:9' | '9:16' | '1:1' — default: '16:9'
    autoplay?: boolean;    // default: true
    muted?: boolean;       // default: true (obrigatório para autoplay no mobile)
    loop?: boolean;        // default: false
  };

  overlay?: {
    eyebrow?: string;       // label pequeno acima do título (ex: "EXAME")
    title?: string;         // título principal do vídeo
    subtitle?: string;      // autoria, canal ou descrição curta
    description?: string;   // texto de apoio (máx. 3 linhas)
    ctaLabel?: string;      // label do botão CTA
    ctaUrl?: string;        // URL do CTA (app decide se abre)
    badge?: string;         // badge customizado (não usada na v2 — reservado)
    showLiveBadge?: boolean; // exibe badge "AO VIVO" com pulse
  };

  behavior?: {
    startFullscreen?: boolean;       // girar para landscape ao abrir
    allowFullscreenToggle?: boolean; // exibe botão de fullscreen — default: true
    showDebug?: boolean;             // habilita console.log — default: false
  };

  theme?: {
    mode?: 'dark' | 'light';  // default: 'dark' (light reservado para futuro)
    accentColor?: string;      // cor do CTA — default: '#6366f1'
  };
};
```

### Campos obrigatórios

| Campo | Motivo |
|-------|--------|
| `version` | Deve ser exatamente `1` |
| `story.id` | Identificador do story para analytics |
| `media.provider` | Determina qual player carregar |
| `media.videoId` | ID do vídeo no provider |

---

## Encoding do payload

O payload é serializado como JSON e encodado em **base64url** antes de ser enviado.

### Por que base64url?

- Evita problemas com aspas, barras, quebras de linha e caracteres Unicode em `injectJavaScript`.
- Strings base64url contêm apenas `[A-Za-z0-9\-_]` — seguro para interpolação em JS.
- Mesma estratégia usada em JWT tokens.

### Base64url vs base64 padrão

| Símbolo | Base64 padrão | Base64url |
|---------|--------------|-----------|
| `+`     | `+`          | `-`       |
| `/`     | `/`          | `_`       |
| `=`     | padding      | omitido   |

### Encoding no app (TypeScript / React Native)

```typescript
function toBase64Url(payload: WebStoryVideoPayload): string {
  const json  = JSON.stringify(payload);
  // Codifica Unicode corretamente antes do btoa
  const bytes = new TextEncoder().encode(json);
  let binary  = '';
  bytes.forEach(b => { binary += String.fromCharCode(b); });
  return btoa(binary)
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');
}
```

### Decoding na página (JavaScript)

```javascript
function fromBase64Url(encoded) {
  const base64 = encoded.replace(/-/g, '+').replace(/_/g, '/');
  const padded  = base64 + '='.repeat((4 - base64.length % 4) % 4);
  const binary  = atob(padded);
  const bytes   = new Uint8Array(binary.length);
  for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i);
  return JSON.parse(new TextDecoder('utf-8').decode(bytes));
}
```

---

## Payload via query param (modo browser/standalone)

Para testes no browser sem React Native, o payload pode ser passado via URL:

```
index.html?payload=<base64url>
```

A página detecta o query param em `init()` e chama `__pocV2LoadStory` diretamente, sem esperar o app.

### Gerar URL de teste no console do browser

```javascript
const payload = {
  version: 1,
  story:   { id: '1', slug: 'test', title: 'Vídeo de Teste' },
  media:   { provider: 'youtube', videoId: 'aqz-KE-bpKQ', aspectRatio: '16:9' },
  overlay: { title: 'Vídeo de Teste', subtitle: '@Blender', showLiveBadge: false },
  behavior: { showDebug: true },
};

const bytes = new TextEncoder().encode(JSON.stringify(payload));
let binary  = '';
bytes.forEach(b => { binary += String.fromCharCode(b); });
const encoded = btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');

console.log('URL:', location.origin + location.pathname + '?payload=' + encoded);
```

---

## Eventos: Web → App

Todos os eventos são enviados como JSON via `window.ReactNativeWebView.postMessage`.

| `type` | `payload` | Quando |
|--------|-----------|--------|
| `ready` | — | Página carregou, `__pocV2LoadStory` registrado |
| `player:ready` | — | Player inicializado e pronto para reprodução |
| `player:play` | — | Reprodução iniciada ou retomada |
| `player:pause` | — | Reprodução pausada |
| `player:error` | `{ code: string, message: string }` | Erro no player ou payload inválido |
| `fullscreen:enter` | — | Usuário ativou modo paisagem |
| `fullscreen:exit` | — | Usuário saiu do modo paisagem |
| `close` | — | Usuário fechou o player |
| `analytics` | `{ event: string, data: object }` | Ação rastreável do usuário |

### Exemplos de payload de analytics

```json
{ "type": "analytics", "payload": { "event": "cta_clicked", "data": { "url": "https://..." } } }
{ "type": "analytics", "payload": { "event": "story_like",  "data": { "storyId": "42", "storySlug": "test" } } }
{ "type": "analytics", "payload": { "event": "story_share", "data": { "storyId": "42" } } }
```

### Códigos de erro (`player:error`)

| `code` | Descrição |
|--------|-----------|
| `DECODE_ERROR` | Falha ao decodificar o base64url |
| `INVALID_PAYLOAD` | Campo obrigatório ausente ou inválido |
| `2` | YouTube: parâmetro inválido |
| `5` | YouTube: erro no player HTML5 |
| `100` | YouTube: vídeo não encontrado |
| `101` / `150` | YouTube: embedding desabilitado pelo owner |

---

## Eventos: App → Web

O único canal app→web é via `injectJavaScript`. Não há `postMessage` do app para a página.

| Chamada injetada | Quando |
|------------------|--------|
| `window.__pocV2LoadStory("base64url"); true;` | Após receber `{ type: "ready" }` |

---

## Comportamento de fullscreen

A POC usa **fake fullscreen** — sem `requestFullscreen()` do browser. A tela cheia é simulada pelo app via rotação da orientação do dispositivo, usando `expo-screen-orientation`.

```
Página envia:  { type: "fullscreen:enter" }
App executa:   ScreenOrientation.lockAsync(LANDSCAPE_RIGHT)

Página envia:  { type: "fullscreen:exit" }
App executa:   ScreenOrientation.unlockAsync()

Página envia:  { type: "close" }
App executa:   ScreenOrientation.unlockAsync() → fecha a WebView
```

### Por que não `requestFullscreen()`?

- iOS WKWebView ignora `requestFullscreen()` chamado via JS.
- O YouTube IFrame controla seu próprio fullscreen nativo (que abre fora da WebView no iOS).
- Com `fs: 0` e `playsinline: 1`, o YouTube fica confinado ao iframe, sem fullscreen nativo.

---

## Validação de segurança

A página aplica as seguintes regras de segurança:

1. **Apenas providers conhecidos** são aceitos: `youtube`, `youtube_live`, `vimeo`.
2. **`textContent` exclusivamente** para todos os textos do overlay — nunca `innerHTML`.
3. **URLs de CTA** não são abertas pela página — o evento `analytics` é enviado ao app, que decide.
4. **`videoId`** é usado apenas como ID no player, não como URL direta.
5. **Iframes de player** têm `pointer-events: none` — cliques nos controles são interceptados pelos overlays nativos da página.

---

## Navegação

- [Voltar ao README](../README.md)
