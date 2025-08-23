# SvelteKit — krótkie wprowadzenie i praktyczne wskazówki

Ten dokument opisuje SvelteKit z naciskiem na koncepcje przydatne przy debugowaniu routingu i deploymentu.

## Co znajdziesz tutaj
- Czym jest SvelteKit
- Główne koncepcje routingu i plików
- Load / server-load / endpoints
- Tryby renderowania i prerender
- Adaptery i deployment
- Najczęstsze problemy z routingiem i szybkie rozwiązania

## Instalacja i uruchomienie

Poniżej znajdziesz kroki potrzebne do uruchomienia tego projektu lokalnie oraz podstawowe polecenia do budowy i deploymentu.

Wymagania wstępne:
- Node.js >= 18
- npm lub pnpm/yarn

Klonowanie i instalacja zależności:

```bash
# sklonuj repo
git clone <REPO_URL> my-app
cd my-app

# zainstaluj zależności (użyj npm lub pnpm)
npm install
```

Ustawienia środowiska:
- Utwórz plik `.env` w katalogu projektu jeśli chcesz nadpisać domyślne zmienne (np. `PORT`, `LOG_LEVEL`, `NODE_ENV`).
- Domyślny port API: 3000. Logi API trafiają do `logs/api-server.log` gdy używasz dostarczonych skryptów.

Tryb deweloperski (lokalnie):

```bash
# uruchamia serwer deweloperski (SvelteKit) + watch dla API
npm run dev

# Alternatywnie uruchom tylko SvelteKit dev
npx svelte-kit dev
```

Build produkcyjny i lokalny podgląd:

```bash
# skompiluj TypeScript API do api/dist
npm run api:v2:build

# zbuduj aplikację (powinien uruchamiać svelte-kit build)
npm run build

# lokalny preview po buildzie
npm run preview
```

Uruchamianie API w produkcji (lokalnie):

```bash
# startuje skompilowany serwer API w tle (skrypty dbają o logi)
npm run api        # wrapper start-api.sh

# lub bez wrappera (jeżeli skompilowałeś wcześniej):
PORT=3000 node api/dist/index.js

# sprawdź logi
tail -f logs/api-server.log
```

Deployment (Vercel):
- Jeśli używasz Vercel, rozważ instalację i konfigurację `@sveltejs/adapter-vercel` w `svelte.config.js` i użycie `svelte-kit build` przed deployem.
- Standardowy flow:
  1. `npm run build` (musisz wygenerować artefakty zgodne z adapterem)
  2. `vercel --prod` lub deploy przez GitHub integration

Uwagi:
- Upewnij się, że `package.json` zawiera prawidłowe skrypty (`dev`, `build`, `preview`) wywołujące `svelte-kit`, a nie tylko `vite` — to krytyczne dla routingu i SSR.
- Skrypty w `scripts/` (np. `dev-start.sh`, `start-api.sh`) upraszczają lokalny workflow — przeczytaj ich nagłówki, aby zrozumieć, gdzie trafiają logi i jak uruchamiane są procesy.

## Czym jest SvelteKit
SvelteKit to oficjalny meta-framework dla Svelte, zbudowany na Vite. Dostarcza kompletnego stacku (routing, SSR/SSG/CSR, API endpoints, adaptery do platformy) i upraszcza konfigurację Vite dla typowych aplikacji.

## Główne koncepcje
- File‑based routing: pliki w `src/routes/` mapują się na URL-e.
  - `src/routes/+page.svelte` → `/`
  - `src/routes/about/+page.svelte` → `/about`
  - dynamiczne segmenty: `src/routes/blog/[id]/+page.svelte` → `/blog/:id`
- Layouty: `+layout.svelte` daje wspólny wrapper dla poddróg.
- Rozdzielenie logiki i prezentacji:
  - `+page.ts` / `+page.server.ts` — `load` (client/server)
  - `+server.ts` — endpointy HTTP (GET/POST itd.)
  - `+page.svelte` — komponent UI
- Endpointy API: `src/routes/api/.../+server.ts` — eksportuj `GET/POST` handlers.
- Actions / form handling: w `+page.server.ts` eksportujesz `actions` (server‑side form handling).

## Load / server-load
- `+page.ts` (bez `.server`) może uruchamiać się po stronie klienta i serwera; nie ma dostępu do sekretów serwera.
- `+page.server.ts` uruchamia się tylko na serwerze i może używać sekretów lub połączeń do DB.
- `+server.ts` to endpointy HTTP; `load` zwraca dane do komponentu jako `data`.

## Rendering i prerender
- SSR (domyślnie) — serwer renderuje HTML, klient hydratuje.
- CSR — możesz wyłączyć SSR (`export const ssr = false`).
- Prerender/SSG — ustaw `prerender = true` lub użyj konfiguracji, pamiętaj, że prerender nie może korzystać z server‑only load wymagających runtime.

## Adaptery i deployment
- Adaptery konwertują output `svelte-kit build` na artefakty zgodne z platformą (Node, Vercel, static, itp.).
- Standardowy workflow:
  - `svelte-kit build` → generuje artefakty zależne od adaptera
  - deploy artefaktów na docelowej platformie
- Ważne: nie używaj `vite build` zamiast `svelte-kit build` — SvelteKit wykonuje dodatkowe kroki (routing, adapter hooks).

## Najczęstsze problemy z routingiem i naprawy
- Produkcja zwraca 404 / brak SSR:
  - Przyczyna: użyto `vite build` zamiast `svelte-kit build` lub brak adaptera.
  - Naprawa: użyj `svelte-kit build` i skonfiguruj adapter (np. `@sveltejs/adapter-vercel`).
- Dev server nie obsługuje routingu:
  - Przyczyna: uruchomiony `vite` zamiast `svelte-kit dev`.
  - Naprawa: uruchamiaj `svelte-kit dev`.
- Konflikty `/api/...`:
  - Sprawdź proxy w `vite.config` (dev) i mapowania w `vercel.json` (prod).
- Dynamiczne parametry `[id]` nie działają:
  - Sprawdź strukturę plików (folder `src/routes/blog/[id]/+page.svelte`), usuń literówki i konflikty nazw.
- FOUC przy tematach motywu lub innych data-attributes:
  - Rozwiązanie: wstępny inline script w `index.html` ustawiający `data-theme` przed bundlem.

## Przydatne polecenia
```bash
# dev (hot reload, SvelteKit dev server)
npm run dev        # lub: npx svelte-kit dev

# build (produkcja, używa adaptera)
npm run build      # powinien uruchamiać svelte-kit build

# preview lokalny po buildzie
npm run preview    # npx svelte-kit preview
```

Upewnij się, że `package.json` mapuje `dev/build/preview` na `svelte-kit`, a nie tylko na `vite`, jeśli używasz SvelteKit.

## Best practices (krótko)
- Trzymaj logikę biznesową poza komponentami (services/hooks).
- Używaj `+server.ts` dla API/server logic; `+page.server.ts` dla server‑only data; `+page.ts` dla uniwersalnego `load`.
- Konfiguruj adapter zgodnie z targetem (Vercel → `adapter-vercel`).
- Testuj `svelte-kit build` + `svelte-kit preview` przed deployem.
- Dokumentuj miejsce generowanych artefaktów (klient/serwer) i aktualizuj `vercel.json` wyłącznie jeśli to konieczne.

---

Chcesz, żebym przygotował patch do `package.json` i `scripts/*` (zamiana `vite` → `svelte-kit`) i uruchomił lokalne testy w branchu?
