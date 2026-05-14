# exame-video-poc-v2

POC v2 de vídeo social para o app Exame Plus. Camada web intermediária que recebe dados de um story de vídeo via payload estruturado, renderiza o player (YouTube ou Vimeo) em experiência social fake-fullscreen e se comunica com o app via eventos postMessage.

Evolução da [exame-video-poc](../exame-video-poc) (v1), que usava uma lista hardcoded de vídeos. A v2 é orientada a dados: toda a experiência é configurada pelo payload recebido do app.

---

## O que esta POC valida

| Validação | Status |
|-----------|--------|
| Receber payload via `injectJavaScript` (base64url) | ✅ |
| Receber payload via `?payload=<base64url>` (browser) | ✅ |
| Validar e normalizar payload com defaults | ✅ |
| Renderizar player YouTube (vídeo e live) | ✅ |
| Renderizar player Vimeo | ✅ |
| Cover real (sem barras pretas) via `VideoCover.resize()` | ✅ |
| Overlays com dados do payload (title, subtitle, badge, CTA) | ✅ |
| Badge AO VIVO com animação pulse | ✅ |
| Botão CTA enviando evento `analytics` ao app | ✅ |
| Fake fullscreen sem `requestFullscreen()` | ✅ |
| Eventos estruturados web → app | ✅ |
| Evento `ready` no init | ✅ |
| Eventos `player:ready`, `player:play`, `player:pause`, `player:error` | ✅ |
| Eventos `fullscreen:enter`, `fullscreen:exit`, `close` | ✅ |
| Overlay de erro para payload inválido | ✅ |
| Fallback de autoplay bloqueado | ✅ |
| Reactions animados (apenas em lives) | ✅ |
| Safe areas (notch / Dynamic Island) | ✅ |
| `100dvh` para evitar bug Safari iOS | ✅ |
| Frame desktop (390×844) para testar no browser | ✅ |
| `body.in-webview` detectado automaticamente | ✅ |

---

## Como rodar localmente

```bash
cd exame-video-poc-v2

# Opção 1 — Python (sem dependências)
python3 -m http.server 8080
# acesse http://localhost:8080

# Opção 2 — Live Server (VS Code extension)
# Cmd+Shift+P → "Live Server: Open with Live Server"
```

> **Nota:** o YouTube IFrame API requer `http://` ou `https://`. Abrir via `file://` faz o postMessage da API falhar e o player não inicializa. Use sempre um servidor local.

---

## Como testar no browser (sem React Native)

Passe o payload como query param `?payload=<base64url>`:

### Gerar URL no console do browser

```javascript
const payload = {
  version: 1,
  story: {
    id: '1',
    slug: 'big-buck-bunny',
    title: 'Big Buck Bunny',
    author: 'Blender Foundation',
  },
  media: {
    provider: 'youtube',
    videoId: 'aqz-KE-bpKQ',
    aspectRatio: '16:9',
    autoplay: true,
    muted: true,
  },
  overlay: {
    title: 'Big Buck Bunny',
    subtitle: '@Blender Foundation',
    description: 'Curta-metragem de animação 3D de domínio público.',
    showLiveBadge: false,
    ctaLabel: 'Saiba Mais',
    ctaUrl: 'https://example.com',
  },
  behavior: { showDebug: true },
};

const bytes = new TextEncoder().encode(JSON.stringify(payload));
let binary = '';
bytes.forEach(b => { binary += String.fromCharCode(b); });
const encoded = btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
console.log(location.origin + location.pathname + '?payload=' + encoded);
```

Cole a URL gerada no browser. Com `behavior.showDebug: true`, os logs aparecem no console do DevTools.

### Payloads de exemplo rápidos

**YouTube normal (16:9):**
```
http://localhost:8080/?payload=eyJ2ZXJzaW9uIjoxLCJzdG9yeSI6eyJpZCI6IjEiLCJzbHVnIjoiYmlnLWJ1Y2stYnVubnkiLCJ0aXRsZSI6IkJpZyBCdWNrIEJ1bm55In0sIm1lZGlhIjp7InByb3ZpZGVyIjoieW91dHViZSIsInZpZGVvSWQiOiJhcXotS0UtYnBLUSIsImFzcGVjdFJhdGlvIjoiMTY6OSJ9LCJvdmVybGF5Ijp7InRpdGxlIjoiQmlnIEJ1Y2sgQnVubnkiLCJzdWJ0aXRsZSI6IkBCbGVuZGVyIn0sImJlaGF2aW9yIjp7InNob3dEZWJ1ZyI6dHJ1ZX19
```

**YouTube Live (badge AO VIVO + reactions):**
```
http://localhost:8080/?payload=eyJ2ZXJzaW9uIjoxLCJzdG9yeSI6eyJpZCI6IjIiLCJzbHVnIjoibG9maS1saXZlIiwidGl0bGUiOiJsb2ZpIGhpcCBob3AgcmFkaW8ifSwibWVkaWEiOnsicHJvdmlkZXIiOiJ5b3V0dWJlX2xpdmUiLCJ2aWRlb0lkIjoiamZLZlBmeUpSZGsiLCJhc3BlY3RSYXRpbyI6IjE2OjkifSwib3ZlcmxheSI6eyJ0aXRsZSI6ImxvZmkgaGlwIGhvcCByYWRpbyIsInN1YnRpdGxlIjoiQExvZmlHaXJsIOKAoiBBTyBWSVZPIiwic2hvd0xpdmVCYWRnZSI6dHJ1ZX0sImJlaGF2aW9yIjp7InNob3dEZWJ1ZyI6dHJ1ZX19
```

**Vimeo:**
```
http://localhost:8080/?payload=eyJ2ZXJzaW9uIjoxLCJzdG9yeSI6eyJpZCI6IjMiLCJzbHVnIjoidmltZW8tZGVtbyIsInRpdGxlIjoiVmltZW8gRGVtbyJ9LCJtZWRpYSI6eyJwcm92aWRlciI6InZpbWVvIiwidmlkZW9JZCI6IjgyNDgwNDIyNSIsImFzcGVjdFJhdGlvIjoiMTY6OSJ9LCJvdmVybGF5Ijp7InRpdGxlIjoiVmltZW8gRGVtbyIsInN1YnRpdGxlIjoiQFZpbWVvIFN0YWZmIFBpY2sifSwiYmVoYXZpb3IiOnsic2hvd0RlYnVnIjp0cnVlfX0
```

---

## Como testar em React Native WebView

### 1. Servidor local

No Mac:
```bash
python3 -m http.server 8080
```

### 2. URL no app

| Dispositivo | URL |
|-------------|-----|
| Emulador Android | `http://10.0.2.2:8080` |
| Dispositivo físico | `http://<IP-do-Mac>:8080` |
| Produção | URL do Cloudflare Worker |

Configure em `EXPO_PUBLIC_VIDEO_POC_V2_URL` no `.env.local` do Exame Plus.

### 3. Fluxo no app

```
1. WebViewOverlay abre a URL da POC v2
2. POC v2 envia { type: "ready" }
3. App recebe, monta WebStoryVideoPayload, codifica em base64url
4. App chama: webViewRef.injectJavaScript('window.__pocV2LoadStory("…"); true;')
5. POC v2 decodifica, valida e renderiza o player
6. POC v2 envia { type: "player:ready" }
7. Interações do usuário geram eventos (fullscreen, close, analytics)
```

### 4. Debugging remoto (Android)

```
chrome://inspect → Remote devices → WebView da POC v2
```

Permite inspecionar DOM, console, network e breakpoints dentro da WebView.

---

## Arquitetura de camadas (z-index)

```
APP LAYER      (z: 10)  — container raiz, fake fullscreen (100vw × 100dvh)
VIDEO LAYER    (z: 20)  — iframe como background, dimensionado pelo VideoCover
OVERLAY LAYER  (z: 30)  — header, sidebar, controles, footer
  gradiente-top  (z: 1 dentro do overlay)
  gradiente-bottom (z: 1 dentro do overlay)
  elementos UI   (z: 31-33)
GESTURE LAYER  (z: 40)  — pointer-events: none (reservado para gestos futuros)
ERROR/FALLBACK (z: 35-36) — sobre o vídeo, abaixo do overlay principal
TOAST          (z: 60)  — sempre no topo
```

---

## Módulos JS (arquivo único, sem bundler)

| Módulo | Responsabilidade |
|--------|-----------------|
| `Logger` | `console.log` condicional por `behavior.showDebug` |
| `AppBridge` | Envia eventos ao app via postMessage; registra `__pocV2LoadStory` |
| `PayloadCodec` | Encode/decode base64url com suporte a Unicode |
| `PayloadValidator` | Valida campos obrigatórios, normaliza defaults |
| `VideoCover` | Calcula dimensões de cover real para o wrapper do player |
| `PlayerFactory` | Cria e controla players YouTube (IFrame API) e Vimeo (iframe) |
| `OverlayRenderer` | Popula elementos HTML do overlay via `textContent` |

---

## Contrato completo

Ver [`docs/contract.md`](docs/contract.md).

---

## Limitações conhecidas

- **YouTube ToS:** overlays clicáveis sobre o iframe violam os termos. Solução: `pointer-events: none` no iframe, controles customizados via IFrame API.
- **autoplay:** exige `muted: true` em todos os browsers/WebViews modernos.
- **iOS `playsinline`:** obrigatório (`fs: 0` + `playsinline: 1`) para evitar que o iOS abra o YouTube em fullscreen nativo.
- **Vimeo background mode:** `background=1` esconde controles mas em troca os eventos `play`/`pause` podem não ser emitidos pelo player. Aceitável para a POC.
- **iPad Simulator:** pode separar a orientação visual do device frame da orientação real da app scene. Validar landscape em dispositivo físico antes de concluir que fullscreen está quebrado.
- **`screen.orientation.lock()`:** não funciona em iOS WKWebView. Por isso o fullscreen é delegado ao app via `expo-screen-orientation`.

## Próximos passos (fora do escopo desta POC)

- Substituir iframes por players nativos: `react-native-youtube-iframe` e `react-native-video`
- Overlays → componentes React Native (`position: absolute`)
- Reactions → `react-native-reanimated`
- Gestos de swipe → `react-native-gesture-handler`
- Integração com backend Exame para likes/comentários reais
