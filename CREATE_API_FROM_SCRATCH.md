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

## 2) Struktura plików (proponowana) — dostosowana do tego repozytorium

- `api/` — katalog z aplikacją Express napisaną w TypeScript
  - `api/index.ts` — entrypoint serwera (eksportuje `app` i uruchamia server w trybie lokalnym)
  - `api/routes.ts` — router z endpointami
  - `api/vercel.ts` — wrapper kompatybilny z Vercel (eksportuje domyślny handler)
  - `api/tsconfig.json` — konfiguracja TypeScript dla serwera
- `src/` — aplikacja klienta (Vite)
- `dist/client` — wynik builda klienta

### 3) Rzeczywisty serwer tego repozytorium — `api/index.ts`

W tym projekcie serwer API jest napisany w TypeScript i używa `winston` do logowania.
Główne elementy:

- eksportuje domyślnie `app` (Express) — przydatne do testów i do wrappera Vercel;
- uruchamia nasłuch tylko, gdy nie jesteśmy w środowisku Vercel (`if (!process.env.VERCEL)`);
- domyślny port to 3000 (można go nadpisać przez `PORT`).

Przykładowe fragmenty (z repo):

```ts
// api/index.ts
import express from "express";
import apiRouter from "./routes.js";
import winston from "winston";

const app = express();

// request logging middleware i konfiguracja winston
// ...

app.use(express.json());
app.use("/api", apiRouter);

// error handling middleware, zwraca JSON z `api: 'api'` i `version: 'v2'`

export default app;

if (!process.env.VERCEL) {
  const port = process.env.PORT || 3000;
  app.listen(port, () => /* logger.info(`Listening: ${port}`) */ null);
}
```

## 4) Rzeczywisty router — `api/routes.ts`

W tym repo router znajduje się w `api/routes.ts` i zwraca dodatkowe pola `api` i `version`.

Przykład z repo:

```ts
// api/routes.ts
import express, { Request, Response } from "express";

const router = express.Router();

router.get("/health", (req: Request, res: Response) => {
  res.json({
    status: "ok",
    timestamp: new Date().toISOString(),
    api: "api",
    version: "v2",
  });
});

router.get("/hello", (req: Request, res: Response) => {
  res.json({ message: "hello", api: "api", version: "v2" });
});

export default router;
```

## 5) Wrapper dla Vercel — `api/vercel.ts`

Repo zawiera `api/vercel.ts` który eksportuje domyślną funkcję wywołującą Express `app`.
To pozwala użyć aplikacji jako funkcji serverless na Vercel.

Przykład z repo:

```ts
import app from "./index.js";

export default async function handler(req: any, res: any) {
  return (app as any)(req, res);
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

## 6) Skrypty npm (aktualne w tym repo)

Ten projekt ma już skonfigurowane skrypty związane z API i buildem. Najważniejsze to:

- `npm run dev` — uruchamia skrypt `scripts/dev-start.sh` (dev klient + inne ustawienia projektu)
- `npm run build` — uruchamia `scripts/build-full.sh` (buduje całość projektu)
- `npm run preview` — `vite preview` (podgląd klienta)
- `npm run api` (alias `api:start`) — `bash ./scripts/start-api.sh` (uruchamia lokalny serwer API)
- `npm run api:kill` — zatrzymuje serwer (`scripts/kill-api.sh`)
- `npm run api:logs` — śledzi logi: `tail -n 200 -f logs/api-server.log`
- `npm run api:v2:build` — kompiluje TypeScript w `api/` (używa `tsc -p api/tsconfig.json`)
- `npm run api:v2:start` — uruchamia skompilowany serwer: `PORT=3000 node api/dist/index.js`

Przykładowe użycie:

```bash
# build api TypeScript -> api/dist
npm run api:v2:build

# uruchom skompilowany serwer (port 3000 domyślnie)
npm run api:v2:start
```

## 7) Budowanie i uruchamianie lokalne — jak to zrobić tutaj

1) Skompiluj API (TypeScript -> `api/dist`):

```bash
npm run api:v2:build
```

2) Uruchom skompilowany serwer API (domyślnie na porcie 3000):

```bash
npm run api:v2:start
```

3) Alternatywnie uruchom lokalny skrypt (jeżeli wolisz dev script który może uruchamiać klienta i serwer):

```bash
npm run api
# lub
npm run dev
```

## 8) Prostey endpoint "hello" — test lokalny

Po uruchomieniu serwera (np. `npm run dev` lub `npm run preview`) sprawdź:

```bash
curl -s http://localhost:3000/api/hello | jq
# expected: { "message": "hello", "api": "api", "version": "v2" }
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
# zależności do testów
npm install -D supertest vitest

# przykładowy test uruchamiający app z api/index.ts (bez startowania listenera)
# i uruchomienie testów:
npx vitest
```

## 11) Dobre praktyki i uwagi

- Separuj logikę biznesową od routerów (`services/`, `utils/`).
- Waliduj wejścia i obsługuj błędy jednoznacznie.
- Nie trzymać wrażliwych sekretów w kodzie — używaj env vars.
- Jeżeli aplikacja ma dużo ruchu, rozważ oddzielne funkcje serverless tylko dla API (bez serwowania full SPA przez funkcję).

---

Plik ten możesz skopiować do `docs/` projektu i dostosować do swoich wymagań.
