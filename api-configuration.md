# Kompleksowa dokumentacja konfiguracji API

## Wprowadzenie

Ten dokument zawiera szczegółową dokumentację konfiguracji API dla projektu opartego na Vite + Svelte z backendem Express.js. Dokumentacja obejmuje strukturę projektu, konfigurację środowisk, deployment oraz najlepsze praktyki.

## Struktura projektu

```
kapp-vite-app/
├── api/                    # Backend API (Express.js + TypeScript)
│   ├── index.ts           # Główny plik aplikacji Express
│   ├── routes.ts          # Definicje tras API
│   ├── vercel.ts          # Handler dla Vercel Functions
│   └── tsconfig.json      # Konfiguracja TypeScript dla API
├── src/                   # Frontend (Svelte)
├── docs/                  # Dokumentacja
├── scripts/               # Skrypty build/dev
└── vercel.json           # Konfiguracja deploymentu Vercel
```

## Konfiguracja API

### 1. Podstawowa konfiguracja (api/index.ts)

Główny plik aplikacji Express zawiera:

- **Logger**: Winston z konfiguracją JSON
- **Middleware**: Logowanie żądań HTTP
- **Routing**: Automatyczne przekierowanie `/api/*`
- **Obsługa błędów**: Globalny handler błędów
- **Konfiguracja portu**: Dynamiczny port z env variables

#### Konfiguracja loggera
```typescript
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [new winston.transports.Console()],
});
```

#### Middleware logowania żądań
```typescript
app.use((req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  res.on("finish", () => {
    const ms = Date.now() - start;
    logger.info("http_request", {
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration_ms: ms,
    });
  });
  next();
});
```

### 2. Routing API (api/routes.ts)

Definicje tras zawierają standardowe endpointy:

- **GET /api/health** - Health check
- **GET /api/hello** - Przykładowy endpoint

#### Struktura odpowiedzi
Wszystkie odpowiedzi zawierają:
```json
{
  "api": "api",
  "version": "v2",
  "...": "..."
}
```

### 3. Konfiguracja TypeScript (api/tsconfig.json)
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "allowJs": false,
    "outDir": "./dist",
    "rootDir": "./",
    "strict": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["./**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

Uwaga: projekt API jest budowany jako ESM (zależnie od `module` + `type: "module"` w root `package.json`). Z tego względu warto upewnić się, że środowisko uruchomieniowe obsługuje ESM (Node >= 18 zalecane). Jeśli chcesz targetować CommonJS zamiast ESM, zaktualizuj `api/tsconfig.json` (np. `module: "CommonJS"`) i rozważ zmianę `type` w `package.json`.

## Konfiguracja środowisk

### Zmienne środowiskowe

| Zmienna | Opis | Domyślna wartość |
|---------|------|------------------|
| `PORT` | Port serwera API | `3000` |
| `LOG_LEVEL` | Poziom logowania | `info` |
| `VERCEL` | Flaga środowiska Vercel | - |
| `NODE_ENV` | Tryb pracy | `development` |

### Przykład pliku .env
```bash
# API Configuration
PORT=3000
LOG_LEVEL=debug
NODE_ENV=development

# Database (opcjonalnie)
DATABASE_URL=postgresql://user:password@localhost:5432/dbname

# Auth (opcjonalnie)
JWT_SECRET=your-jwt-secret
```

## Deployment

### 1. Vercel (zalecane)

#### Konfiguracja vercel.json
```json
{
  "functions": {
    "api/dist/vercel.js": {
      "maxDuration": 10
    }
  },
  "routes": [
    {
      "src": "/assets/(.*)",
      "dest": "/dist/client/assets/$1"
    },
    {
      "src": "/api/(.*)",
      "dest": "/api/dist/vercel.js"
    },
    {
      "src": "/(.*)",
      "dest": "/dist/client/$1"
    }
  ]
}
```

#### Deployment krok po kroku
```bash
# 1. Build aplikacji
npm run build

# 2. Build API TypeScript
npm run api:v2:build

# 3. Deployment na Vercel
vercel --prod
```

### 2. Lokalny deployment

#### Build i uruchomienie
```bash
# Build TypeScript
npm run api:v2:build

# Uruchomienie produkcji
npm run api:v2:start
```

#### Development
```bash
# Uruchomienie w trybie deweloperskim
npm run dev

# Lub tylko API
npm run api
```

## Skrypty npm

| Skrypt | Opis |
|--------|------|
| `npm run dev` | Uruchomienie w trybie deweloperskim |
| `npm run build` | Build produkcyjny |
| `npm run api` | Uruchomienie tylko API |
| `npm run api:v2:build` | Kompilacja TypeScript API |
| `npm run api:v2:start` | Uruchomienie skompilowanego API |
| `npm run api:kill` | Zatrzymanie serwera API |
| `npm run api:logs` | Podgląd logów |

## Testowanie

### Testy lokalne
```bash
# Test health check
curl http://localhost:3000/api/health

# Test hello endpoint
curl http://localhost:3000/api/hello
```

### Testy zdalne
```bash
# Po deployment na Vercel
curl https://your-app.vercel.app/api/health
```

## Rozszerzanie API

### Dodawanie nowych tras

1. **Nowy plik z trasami** (api/users.ts)
```typescript
import express from "express";
const router = express.Router();

router.get("/users", (req, res) => {
  res.json({ users: [], api: "api", version: "v2" });
});

export default router;
```

2. **Rejestracja tras** (api/routes.ts)
```typescript
import usersRouter from "./users.js";
router.use("/users", usersRouter);
```

### Middleware autoryzacji
```typescript
const authMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) {
    return res.status(401).json({ error: "Unauthorized", api: "api", version: "v2" });
  }
  next();
};
```

## Monitoring i logowanie

### Struktura logów
```json
{
  "level": "info",
  "message": "http_request",
  "method": "GET",
  "url": "/api/health",
  "status": 200,
  "duration_ms": 12,
  "timestamp": "2024-01-01T12:00:00.000Z"
}
```

### Integracja z zewnętrznymi serwisami
- **Sentry**: Do śledzenia błędów
- **Datadog**: Do monitoringu
- **New Relic**: Do APM

## Bezpieczeństwo

### Lista kontrolna bezpieczeństwa
- [ ] Walidacja danych wejściowych
- [ ] Rate limiting
- [ ] CORS configuration
- [ ] Helmet.js dla nagłówków bezpieczeństwa
- [ ] Input sanitization
- [ ] HTTPS w produkcji

### Przykład konfiguracji CORS
```typescript
import cors from "cors";

const corsOptions = {
  origin: process.env.NODE_ENV === "production" 
    ? ["https://yourdomain.com"] 
    : ["http://localhost:5173"],
  credentials: true,
};

app.use(cors(corsOptions));
```

## Rozwiązywanie problemów

### Częste problemy

1. **Port zajęty**
```bash
# Zmiana portu
PORT=3001 npm run api:v2:start
```

2. **Błędy TypeScript**
```bash
# Sprawdzenie błędów
npx tsc -p api/tsconfig.json --noEmit
```

3. **Problem z Vercel**
```bash
# Test lokalny
vercel dev
```

### Debugowanie
```bash
# Szczegółowe logi
LOG_LEVEL=debug npm run api:v2:start

# Podgląd logów w czasie rzeczywistym
npm run api:logs
```

## Najlepsze praktyki

1. **Struktura kodu**
   - Separacja logiki biznesowej od tras
   - Używanie serwisów i repozytoriów
   - Middleware dla wspólnej logiki

2. **Obsługa błędów**
   - Konsystentny format odpowiedzi błędów
   - Logowanie z kontekstem
   - User-friendly komunikaty

3. **Wydajność**
   - Kompresja odpowiedzi
   - Cache headers
   - Connection pooling dla DB

4. **Testowanie**
   - Unit testy dla logiki biznesowej
   - Integration testy dla API
   - Load testing dla wydajności