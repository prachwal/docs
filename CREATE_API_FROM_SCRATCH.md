# Tworzenie API od zera — techniczny przewodnik

To krótki, praktyczny przewodnik (checklista + przykłady) jak dodać prostą część API do czystego projektu Vite (Preact) takiego jak `alpha-vite-app`.

Zawiera: co doinstalować, jak wygląda podstawowy routing, jak zbudować projekt, prosty endpoint zwracający "hello" oraz jak testować lokalnie i zdalnie.

## 1) Co zainstalować

W katalogu projektu uruchom (najprościej npm):

```bash
# runtime
npm install express compression sirv cors

# opcjonalnie (lokalny dev):
npm install -D vite

# pomocne narzędzia (opcjonalne):
npm install -D cross-env
```

Jeśli używasz TypeScript, zainstaluj także typy:

```bash
npm install -D @types/express @types/node
```

## 2) Struktura plików (proponowana)

- `api/` — katalog z aplikacją Express
  - `api/index.js` — entrypoint serwera (eksportuje app / handler)
  - `api/routes.js` — router z endpointami
  - `api/vercel.js` — (opcjonalnie) wrapper kompatybilny z Vercel
- `src/` — aplikacja klienta (Vite)
- `dist/client` — wynik builda klienta

## 3) Przykładowy minimalny serwer — plik `api/index.js`

Plik implementuje Express i montuje router pod `/api`. W dev użyj Vite w middleware mode, w prod serwuj statyczne pliki z `dist/client`.

```javascript
// api/index.js
import express from 'express';
import { dirname } from 'node:path';
import { fileURLToPath } from 'node:url';
import apiRouter from './routes.js';

const __dirname = dirname(fileURLToPath(import.meta.url));
const app = express();

app.use(express.json());
app.use('/api', apiRouter);

// Fallback: serwuj index.html (client-side app)
app.use('*', (req, res) => {
  res.sendFile(new URL('../dist/client/index.html', import.meta.url));
});

export default app;

// Lokalny uruchamianie:
if (!process.env.VERCEL) {
  const port = process.env.PORT || 5173;
  app.listen(port, () => console.log(`Listening ${port}`));
}
```

## 4) Przykładowy router — plik `api/routes.js`

```javascript
// api/routes.js
import express from 'express';
const router = express.Router();

router.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

router.get('/hello', (req, res) => {
  res.json({ message: 'hello' });
});

export default router;
```

## 5) (Opcjonalnie) Wrapper dla Vercel — `api/vercel.js`

Jeżeli deployujesz na Vercel i chcesz, żeby `api/index.js` działało jako funkcja, dodaj mały wrapper, który wywoła Express app jako handler:

```javascript
// api/vercel.js
import app from './index.js';

export default async function handler(req, res) {
  return app(req, res);
}
```

W `vercel.json` możesz wskazać `api/vercel.js` jako entrypoint funkcji.

### Konfiguracja routingu dla wrappera (przykłady)

Poniżej przykładowe podejścia do mapowania ruchu za pomocą `vercel.json`. Wybierz jedno z nich zależnie od tego, czy chcesz, aby cały ruch przechodził przez funkcję (Express), czy tylko wybrane ścieżki (np. `/api`) - to wpływa na cold starts i cache'owanie statycznych zasobów.

- Mapowanie tylko `/api/*` do funkcji (zalecane):

```json
{
  "functions": {
    "api/vercel.js": { "memory": 128, "maxDuration": 10 }
  },
  "routes": [
    { "src": "/assets/(.*)", "dest": "/dist/client/assets/$1" },
    { "src": "/api/(.*)", "dest": "/api/vercel.js" },
    { "src": "/(.*)", "dest": "/dist/client/$1" }
  ]
}
```

W tym wariancie statyczne pliki (SPA) są serwowane bezpośrednio z `dist/client` (lepsze cache), a tylko żądania do `/api/*` trafiają do funkcji.

- Mapowanie całego ruchu do funkcji (wszystko przez Express):

```json
{
  "functions": {
    "api/vercel.js": { "memory": 256, "maxDuration": 30 }
  },
  "routes": [
    { "src": "/assets/(.*)", "dest": "/dist/client/assets/$1" },
    { "src": "(.*)", "dest": "/api/vercel.js" }
  ]
}
```

To podejście umożliwia pełną kontrolę nad routingiem i fallbackami w Express, ale oznacza, że każda prosba przechodzi przez funkcję (podlega cold-startom i limitom funkcji).

### Dodatkowe uwagi

- Jeżeli używasz wrappera `api/vercel.js`, upewnij się, że plik eksportuje domyślną funkcję przyjmującą `(req,res)` i wywołującą `app(req,res)`.
- Testuj reguły `vercel.json` lokalnie przy pomocy `vercel dev` — symuluje routing i funkcje.
- Możesz również mapować pojedyncze endpointy (np. `/auth/verify`) do osobnych funkcji (np. `api/auth/verify.js`) zamiast jednej dużej aplikacji Express — to zazwyczaj poprawia skalowalność.

## 6) Skrypty npm (package.json)

Dodaj do `package.json` przynajmniej:

```json
{
  "scripts": {
    "dev": "node api/index.js",
    "build:client": "vite build --outDir dist/client",
    "build": "npm run build:client",
    "preview": "cross-env NODE_ENV=production node api/index.js"
  }
}
```

Jeśli planujesz SSR, dodaj `build:server` zgodnie z projektem.

## 7) Budowanie i uruchamianie lokalne

1. Build klienta:

```bash
npm run build:client
```

2. Uruchom serwer w trybie preview (production):

```bash
npm run preview
# lub bez cross-env (Linux): NODE_ENV=production node api/index.js
```

3. Development (z Vite middleware):

```bash
npm run dev
# W tym trybie możesz skonfigurować index.js aby tworzył Vite dev server w middlewareMode
```

## 8) Prostey endpoint "hello" — test lokalny

Po uruchomieniu serwera (np. `npm run dev` lub `npm run preview`) sprawdź:

```bash
curl -s http://localhost:5173/api/hello | jq
# expected: { "message": "hello" }
```

Jeśli nie masz `jq`, rezultat zobaczysz surowo.

## 9) Testowanie zdalne (deploy)

- Deploy na Vercel (najprostszy):

```bash
npm run build
vercel --prod
```

- Upewnij się, że w `vercel.json` masz wyraźnie zdefiniowane funkcje i routes; jeżeli używasz wrappera `api/vercel.js`, zadeklaruj go jako entrypoint funkcji.

- Po deployu, wykonaj:

```bash
curl -s https://<your-deployment>.vercel.app/api/hello | jq
```

Alternatywa (test zdalny bez deployu): uruchom `vercel dev` lub użyj `ngrok` aby wystawić lokalny port.

## 10) Testy automatyczne (lokalnie)

- Dodaj prosty test (np. z jest/vitest) który uruchamia serwer w pamięci i wysyła żądania do `/api/hello`.

Przykład prostego testu z `node` + `supertest`:

```javascript
// test/api.hello.test.js (przykład)
import request from 'supertest';
import app from '../api/index.js';

test('GET /api/hello', async () => {
  const res = await request(app).get('/api/hello');
  expect(res.statusCode).toBe(200);
  expect(res.body.message).toBe('hello');
});
```

Instalacja testowych zależności:

```bash
npm install -D supertest vitest
```

Uruchom testy: `npx vitest`.

## 11) Dobre praktyki i uwagi

- Separuj logikę biznesową od routerów (`services/`, `utils/`).
- Waliduj wejścia i obsługuj błędy jednoznacznie.
- Nie trzymać wrażliwych sekretów w kodzie — używaj env vars.
- Jeżeli aplikacja ma dużo ruchu, rozważ oddzielne funkcje serverless tylko dla API (bez serwowania full SPA przez funkcję).

---

Plik ten możesz skopiować do `docs/` projektu i dostosować do swoich wymagań.
