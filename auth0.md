# Kompleksowy przewodnik integracji Auth0 z Svelte

## Wprowadzenie

Integracja **Auth0** z aplikacją Svelte zapewnia solidny, bezpieczny i elastyczny system uwierzytelniania. Użyjemy oficjalnego pakietu **`@auth0/auth0-spa-js`**, który jest przeznaczony do aplikacji jednostronicowych (SPA) i doskonale współpracuje z Svelte.

### Co osiągniemy?

1. **Konfigurację** aplikacji w panelu Auth0
2. **Instalację** i stworzenie reużywalnego serwisu Auth0 w Svelte
3. Implementację funkcji **logowania** i **wylogowywania**
4. Dostęp do **danych użytkownika** (profil, token)
5. **Ochronę tras** (komponentów)
6. **Zaawansowane funkcje** Auth0

---

## Krok 1: Konfiguracja aplikacji w panelu Auth0

Zanim zaczniesz pisać kod, musisz skonfigurować aplikację w swoim panelu Auth0.

1. **Zaloguj się** do [panelu Auth0](https://manage.auth0.com/)
2. Przejdź do **Applications → Applications** i kliknij **Create Application**
3. Wybierz typ aplikacji: **Single Page Web Applications** i nadaj jej nazwę (np. "Svelte App")
4. Po utworzeniu aplikacji przejdź do zakładki **Settings** i skopiuj następujące wartości:
   - **Domain**
   - **Client ID**

5. W tej samej zakładce skonfiguruj adresy URL. Vite domyślnie uruchamia serwer deweloperski pod adresem `http://localhost:5173`:
   - **Allowed Callback URLs**: `http://localhost:5173`
   - **Allowed Logout URLs**: `http://localhost:5173`
   - **Allowed Web Origins**: `http://localhost:5173`

Zapisz zmiany na dole strony.

---

## Krok 2: Konfiguracja projektu Vite

Bezpieczeństwo nakazuje, aby klucze i domena Auth0 nie były przechowywane bezpośrednio w kodzie. Użyjemy do tego zmiennych środowiskowych w Vite.

1. W głównym katalogu projektu utwórz plik `.env`:

```env
# .env
VITE_AUTH0_DOMAIN="TWOJA_DOMENA_Z_AUTH0"
VITE_AUTH0_CLIENT_ID="TWÓJ_CLIENT_ID_Z_AUTH0"
```

**Ważne**: Zmienne środowiskowe w Vite, które mają być dostępne w kodzie front-endowym, muszą zaczynać się od prefiksu `VITE_`.

2. Aby TypeScript rozpoznawał te zmienne, zaktualizuj plik `src/vite-env.d.ts` (lub go utwórz):

```typescript
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_AUTH0_DOMAIN: string;
  readonly VITE_AUTH0_CLIENT_ID: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## Krok 3: Instalacja i stworzenie serwisu Auth0

Teraz zainstalujemy potrzebną bibliotekę i stworzymy centralne miejsce do zarządzania logiką Auth0.

1. **Instalacja zależności**:

```bash
npm install @auth0/auth0-spa-js
```

2. **Stworzenie serwisu Auth0**: Utwórz plik `src/services/auth.ts`, który będzie zarządzał klientem Auth0 i stanem uwierzytelnienia:

```typescript
// src/services/auth.ts
import { Auth0Client, createAuth0Client, User } from '@auth0/auth0-spa-js';
import { writable, derived, type Readable } from 'svelte/store';

// Sklepy Svelte do przechowywania stanu
export const isAuthenticated = writable<boolean>(false);
export const user = writable<User | undefined>(undefined);
export const popupOpen = writable<boolean>(false);
export const error = writable<any>(null);
export const isLoading = writable<boolean>(true);

// Sklep pochodny (derived store) do łatwego dostępu
export const authState: Readable<{
  isAuthenticated: boolean;
  user?: User;
  isLoading: boolean;
}> = derived([isAuthenticated, user, isLoading], ([$isAuthenticated, $user, $isLoading]) => ({
  isAuthenticated: $isAuthenticated,
  user: $user,
  isLoading: $isLoading,
}));

let client: Auth0Client;

async function createClient(): Promise<Auth0Client> {
  if (client) return client;

  client = await createAuth0Client({
    domain: import.meta.env.VITE_AUTH0_DOMAIN,
    clientId: import.meta.env.VITE_AUTH0_CLIENT_ID,
    authorizationParams: {
      redirect_uri: window.location.origin,
      // Dodatkowe parametry autoryzacji
      scope: 'openid profile email', // Domyślne zakresy
      audience: 'your-api-identifier', // Opcjonalne: dla API
    },
    // Opcje konfiguracyjne klienta
    cacheLocation: 'localstorage', // 'memory' lub 'localstorage'
    useRefreshTokens: true, // Wykorzystanie refresh tokenów
    useCookiesForTransactions: false, // Użycie ciasteczek zamiast sessionStorage
  });

  return client;
}

async function handleRedirectCallback(): Promise<void> {
  isLoading.set(true);
  const client = await createClient();
  try {
    await client.handleRedirectCallback();
    const currentUser = await client.getUser();
    user.set(currentUser);
    isAuthenticated.set(!!currentUser);
  } catch (e) {
    error.set(e);
    console.error('Error handling redirect callback:', e);
  } finally {
    isLoading.set(false);
  }
}

// Inicjalizacja stanu uwierzytelnienia
async function initAuth(): Promise<void> {
  isLoading.set(true);
  try {
    const client = await createClient();
    const isAuth = await client.isAuthenticated();
    
    if (isAuth) {
      const currentUser = await client.getUser();
      user.set(currentUser);
      isAuthenticated.set(true);
    }
  } catch (e) {
    error.set(e);
    console.error('Error initializing auth:', e);
  } finally {
    isLoading.set(false);
  }
}

// Sprawdzenie, czy URL zawiera parametry code i state po powrocie z Auth0
if (window.location.search.includes('code=') && window.location.search.includes('state=')) {
  handleRedirectCallback();
} else {
  initAuth();
}

// Funkcja logowania z przekierowaniem
const loginWithRedirect = async (options?: {
  appState?: any;
  authorizationParams?: {
    screen_hint?: 'signup' | 'login';
    prompt?: 'none' | 'login' | 'consent' | 'select_account';
    connection?: string;
    [key: string]: any;
  };
}): Promise<void> => {
  const client = await createClient();
  await client.loginWithRedirect({
    appState: options?.appState,
    authorizationParams: {
      ...options?.authorizationParams,
    }
  });
};

// Funkcja logowania z popup
const loginWithPopup = async (options?: {
  authorizationParams?: {
    screen_hint?: 'signup' | 'login';
    prompt?: 'none' | 'login' | 'consent' | 'select_account';
    connection?: string;
    [key: string]: any;
  };
}): Promise<void> => {
  const client = await createClient();
  popupOpen.set(true);
  try {
    await client.loginWithPopup({
      authorizationParams: {
        ...options?.authorizationParams,
      }
    });
    const currentUser = await client.getUser();
    user.set(currentUser);
    isAuthenticated.set(true);
  } catch (e) {
    error.set(e);
    console.error('Login with popup failed:', e);
  } finally {
    popupOpen.set(false);
  }
};

// Funkcja wylogowania
const logout = async (options?: {
  logoutParams?: {
    returnTo?: string;
    federated?: boolean;
  };
}): Promise<void> => {
  const client = await createClient();
  client.logout({
    logoutParams: {
      returnTo: window.location.origin,
      ...options?.logoutParams,
    }
  });
  user.set(undefined);
  isAuthenticated.set(false);
};

// Pobieranie tokena dostępowego
const getAccessTokenSilently = async (options?: {
  authorizationParams?: {
    audience?: string;
    scope?: string;
    [key: string]: any;
  };
  detailedResponse?: boolean;
  timeoutInSeconds?: number;
}): Promise<string> => {
  const client = await createClient();
  return await client.getTokenSilently({
    authorizationParams: {
      audience: 'your-api-identifier',
      scope: 'read:current_user',
      ...options?.authorizationParams,
    },
    ...options,
  });
};

// Pobieranie tokena z popup
const getAccessTokenWithPopup = async (options?: {
  authorizationParams?: {
    audience?: string;
    scope?: string;
    [key: string]: any;
  };
}): Promise<string> => {
  const client = await createClient();
  return await client.getTokenWithPopup({
    authorizationParams: {
      audience: 'your-api-identifier',
      scope: 'read:current_user',
      ...options?.authorizationParams,
    }
  });
};

// Sprawdzenie sesji
const checkSession = async (): Promise<void> => {
  const client = await createClient();
  try {
    await client.checkSession();
    const currentUser = await client.getUser();
    user.set(currentUser);
    isAuthenticated.set(!!currentUser);
  } catch (e) {
    error.set(e);
    console.error('Check session failed:', e);
  }
};

const auth = {
  loginWithRedirect,
  loginWithPopup,
  logout,
  getAccessTokenSilently,
  getAccessTokenWithPopup,
  checkSession,
};

export default auth;
```

---

## Szczegółowa dokumentacja parametrów Auth0Client

### Parametry konfiguracyjne createAuth0Client()

#### Wymagane parametry:
- **`domain`** (string): Twoja domena Auth0 (np. `your-tenant.auth0.com`)
- **`clientId`** (string): Client ID Twojej aplikacji Auth0

#### Parametry autoryzacji (authorizationParams):
- **`redirect_uri`** (string): URL przekierowania po logowaniu
- **`scope`** (string): Zakresy uprawnień (domyślnie: `'openid profile email'`)
- **`audience`** (string): Identyfikator API dla którego token jest przeznaczony
- **`response_type`** (string): Typ odpowiedzi (domyślnie: `'code'`)
- **`response_mode`** (string): Sposób dostarczenia odpowiedzi (`'query'`, `'fragment'`, `'form_post'`)
- **`state`** (string): Losowy string dla bezpieczeństwa
- **`nonce`** (string): Losowy string zapobiegający atakom replay
- **`prompt`** (string): Sposób wyświetlenia formularza logowania
  - `'none'`: Nie wyświetla formularza logowania
  - `'login'`: Wyświetla formularz logowania
  - `'consent'`: Wyświetla formularz zgody
  - `'select_account'`: Pozwala wybrać konto
- **`screen_hint`** (string): Wskazówka o tym, który ekran wyświetlić (`'login'` lub `'signup'`)
- **`connection`** (string): Konkretne połączenie do użycia (np. `'google-oauth2'`)

#### Opcje klienta:
- **`cacheLocation`** (`'memory'` | `'localstorage'`): Miejsce przechowywania tokenów
- **`useRefreshTokens`** (boolean): Czy używać refresh tokenów
- **`useCookiesForTransactions`** (boolean): Użycie ciasteczek zamiast sessionStorage
- **`cookieDomain`** (string): Domena dla ciasteczek
- **`leeway`** (number): Tolerancja czasu dla tokenów (w sekundach)
- **`httpTimeoutInSeconds`** (number): Timeout dla żądań HTTP
- **`legacySameSiteCookie`** (boolean): Wsparcie dla starszych przeglądarek
- **`sessionCheckExpiryDays`** (number): Ile dni token sesji jest ważny
- **`nowProvider`** (() => number): Funkcja dostarczająca aktualny czas

### Metody Auth0Client

#### loginWithRedirect(options?)
Przekierowuje użytkownika do strony logowania Auth0.

**Parametry:**
```typescript
{
  appState?: any; // Stan aplikacji do przywrócenia po logowaniu
  authorizationParams?: {
    screen_hint?: 'signup' | 'login';
    prompt?: 'none' | 'login' | 'consent' | 'select_account';
    connection?: string;
    ui_locales?: string; // Język interfejsu (np. 'pl')
    max_age?: number; // Maksymalny wiek sesji w sekundach
    [key: string]: any;
  };
}
```

#### loginWithPopup(options?)
Otwiera popup z formularzem logowania.

**Parametry:**
```typescript
{
  authorizationParams?: {
    screen_hint?: 'signup' | 'login';
    prompt?: 'none' | 'login' | 'consent' | 'select_account';
    connection?: string;
    [key: string]: any;
  };
  config?: {
    timeoutInSeconds?: number; // Timeout dla popup
    popup?: Window; // Niestandardowe okno popup
  };
}
```

#### getTokenSilently(options?)
Pobiera token dostępowy bez interakcji z użytkownikiem.

**Parametry:**
```typescript
{
  authorizationParams?: {
    audience?: string;
    scope?: string;
    [key: string]: any;
  };
  detailedResponse?: boolean; // Zwraca szczegółową odpowiedź z tokenem
  timeoutInSeconds?: number;
  cacheMode?: 'on' | 'off' | 'cache-only'; // Tryb cache'owania
}
```

#### logout(options?)
Wylogowuje użytkownika.

**Parametры:**
```typescript
{
  logoutParams?: {
    returnTo?: string; // URL powrotu po wylogowaniu
    federated?: boolean; // Wylogowanie z dostawcy federowanego
    client_id?: string;
    [key: string]: any;
  };
  openUrl?: (url: string) => void; // Niestandardowa funkcja otwierania URL
}
```

---

## Krok 4: Zaawansowana integracja z aplikacją Svelte

```svelte
<script lang="ts">
  import auth from './services/auth';
  import { authState, error, isLoading } from './services/auth';
  
  // Funkcje pomocnicze
  const handleLogin = () => auth.loginWithRedirect();
  const handleSignup = () => auth.loginWithRedirect({ 
    authorizationParams: { screen_hint: 'signup' } 
  });
  const handlePopupLogin = () => auth.loginWithPopup();
  const handleLogout = () => auth.logout();
  
  // Pobieranie tokena dla API
  const getToken = async () => {
    try {
      const token = await auth.getAccessTokenSilently({
        authorizationParams: {
          audience: 'your-api-identifier',
          scope: 'read:profile write:profile'
        }
      });
      console.log('Access token:', token);
    } catch (err) {
      console.error('Error getting token:', err);
    }
  };
</script>

<main>
  <h1>Aplikacja Svelte + Auth0</h1>

  {#if $error}
    <div class="error">
      <p>Błąd: {$error.message}</p>
    </div>
  {/if}

  {#if $isLoading}
    <div class="loading">
      <p>Ładowanie...</p>
    </div>
  {:else if $authState.isAuthenticated}
    <div class="user-info">
      <h2>Witaj, {$authState.user?.name ?? 'użytkowniku'}!</h2>
      
      {#if $authState.user?.picture}
        <img src={$authState.user.picture} alt="Avatar" width="80" height="80" />
      {/if}
      
      <div class="user-details">
        <p><strong>Email:</strong> {$authState.user?.email}</p>
        <p><strong>ID:</strong> {$authState.user?.sub}</p>
        <p><strong>Verified:</strong> {$authState.user?.email_verified ? 'Tak' : 'Nie'}</p>
        <p><strong>Last update:</strong> {$authState.user?.updated_at}</p>
      </div>
      
      <div class="actions">
        <button on:click={handleLogout} class="btn btn-danger">Wyloguj</button>
        <button on:click={getToken} class="btn btn-primary">Pobierz token</button>
      </div>
      
      <details>
        <summary>Pełne dane użytkownika</summary>
        <pre>{JSON.stringify($authState.user, null, 2)}</pre>
      </details>
    </div>
  {:else}
    <div class="login-options">
      <h2>Zaloguj się lub zarejestruj</h2>
      <div class="login-buttons">
        <button on:click={handleLogin} class="btn btn-primary">Zaloguj</button>
        <button on:click={handleSignup} class="btn btn-secondary">Zarejestruj</button>
        <button on:click={handlePopupLogin} class="btn btn-outline">Zaloguj (Popup)</button>
      </div>
    </div>
  {/if}
</main>

<style>
  main {
    max-width: 800px;
    margin: 0 auto;
    padding: 2rem;
    font-family: system-ui, sans-serif;
  }
  
  .error {
    background-color: #fee;
    border: 1px solid #fcc;
    color: #c00;
    padding: 1rem;
    border-radius: 4px;
    margin-bottom: 1rem;
  }
  
  .loading {
    text-align: center;
    padding: 2rem;
  }
  
  .user-info {
    text-align: center;
  }
  
  .user-info img {
    border-radius: 50%;
    margin: 1rem 0;
  }
  
  .user-details {
    background-color: #f8f9fa;
    padding: 1rem;
    border-radius: 4px;
    margin: 1rem 0;
    text-align: left;
  }
  
  .actions {
    margin: 1rem 0;
  }
  
  .login-buttons {
    display: flex;
    gap: 1rem;
    justify-content: center;
    flex-wrap: wrap;
  }
  
  .btn {
    padding: 0.75rem 1.5rem;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1rem;
    transition: background-color 0.2s;
  }
  
  .btn-primary {
    background-color: #007bff;
    color: white;
  }
  
  .btn-primary:hover {
    background-color: #0056b3;
  }
  
  .btn-secondary {
    background-color: #6c757d;
    color: white;
  }
  
  .btn-secondary:hover {
    background-color: #545b62;
  }
  
  .btn-outline {
    background-color: transparent;
    color: #007bff;
    border: 1px solid #007bff;
  }
  
  .btn-outline:hover {
    background-color: #007bff;
    color: white;
  }
  
  .btn-danger {
    background-color: #dc3545;
    color: white;
  }
  
  .btn-danger:hover {
    background-color: #c82333;
  }
  
  pre {
    text-align: left;
    background-color: #f8f9fa;
    padding: 1rem;
    border-radius: 4px;
    overflow-x: auto;
    font-size: 0.875rem;
  }
  
  details {
    margin-top: 1rem;
    text-align: left;
  }
  
  summary {
    cursor: pointer;
    font-weight: bold;
    margin-bottom: 0.5rem;
  }
</style>
```

---

## Krok 5: Zaawansowana ochrona tras

```svelte
<!-- src/components/AuthGuard.svelte -->
<script lang="ts">
  import { authState, isLoading } from '../services/auth';
  import auth from '../services/auth';
  
  export let fallback: 'login' | 'message' = 'message';
  export let requiredScopes: string[] = [];
  export let customMessage = 'Musisz być zalogowany, aby zobaczyć tę zawartość.';
  
  // Sprawdzanie czy użytkownik ma wymagane uprawnienia
  $: hasRequiredScopes = requiredScopes.length === 0 || 
    requiredScopes.every(scope => 
      $authState.user?.['https://myapp.com/roles']?.includes(scope) ||
      $authState.user?.scope?.split(' ').includes(scope)
    );
  
  const handleLogin = () => auth.loginWithRedirect();
</script>

{#if $isLoading}
  <div class="loading">Sprawdzanie uprawnień...</div>
{:else if $authState.isAuthenticated && hasRequiredScopes}
  <slot />
{:else if !$authState.isAuthenticated}
  {#if fallback === 'login'}
    <div class="auth-required">
      <h3>Wymagane logowanie</h3>
      <p>{customMessage}</p>
      <button on:click={handleLogin} class="btn btn-primary">Zaloguj się</button>
    </div>
  {:else}
    <div class="auth-required">
      <p>{customMessage}</p>
    </div>
  {/if}
{:else}
  <div class="insufficient-permissions">
    <h3>Niewystarczające uprawnienia</h3>
    <p>Nie masz uprawnień do przeglądania tej zawartości.</p>
    <p>Wymagane: {requiredScopes.join(', ')}</p>
  </div>
{/if}

<style>
  .loading, .auth-required, .insufficient-permissions {
    text-align: center;
    padding: 2rem;
    background-color: #f8f9fa;
    border-radius: 4px;
    margin: 1rem 0;
  }
  
  .insufficient-permissions {
    background-color: #fff3cd;
    border: 1px solid #ffeaa7;
    color: #856404;
  }
  
  .btn {
    padding: 0.75rem 1.5rem;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1rem;
    background-color: #007bff;
    color: white;
    transition: background-color 0.2s;
  }
  
  .btn:hover {
    background-color: #0056b3;
  }
</style>
```

---

## Integracja z API

Przykład serwisu do komunikacji z zabezpieczonym API:

```typescript
// src/services/api.ts
import auth from './auth';

class ApiService {
  private baseURL = 'https://api.example.com';
  
  private async getAuthHeaders(): Promise<Record<string, string>> {
    try {
      const token = await auth.getAccessTokenSilently({
        authorizationParams: {
          audience: 'your-api-identifier',
          scope: 'read:data write:data'
        }
      });
      
      return {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      };
    } catch (error) {
      console.error('Error getting access token:', error);
      throw error;
    }
  }
  
  async get<T>(endpoint: string): Promise<T> {
    const headers = await this.getAuthHeaders();
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      headers
    });
    
    if (!response.ok) {
      throw new Error(`API Error: ${response.status}`);
    }
    
    return response.json();
  }
  
  async post<T>(endpoint: string, data: any): Promise<T> {
    const headers = await this.getAuthHeaders();
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      method: 'POST',
      headers,
      body: JSON.stringify(data)
    });
    
    if (!response.ok) {
      throw new Error(`API Error: ${response.status}`);
    }
    
    return response.json();
  }
}

export const apiService = new ApiService();
```

---

## Dodatkowe funkcje Auth0

### Multi-Factor Authentication (MFA)
Auth0 automatycznie obsługuje MFA skonfigurowane w panelu administracyjnym. Użytkownik zostanie przekierowany do odpowiedniego kroku MFA podczas logowania.

### Social Login
Aby dodać logowanie społecznościowe, skonfiguruj połączenia w panelu Auth0 i użyj parametru `connection`:

```typescript
// Logowanie przez Google
auth.loginWithRedirect({
  authorizationParams: {
    connection: 'google-oauth2'
  }
});

// Logowanie przez Facebook
auth.loginWithRedirect({
  authorizationParams: {
    connection: 'facebook'
  }
});
```

### Niestandardowe domeny
Jeśli używasz niestandardowej domeny Auth0, wystarczy zmienić wartość `VITE_AUTH0_DOMAIN` w pliku `.env`.

### Obsługa błędów
Rozszerz obsługę błędów w serwisie auth:

```typescript
// Dodaj do auth.ts
export const errorMessages = {
  'login_required': 'Wymagane logowanie',
  'consent_required': 'Wymagana zgoda użytkownika',
  'interaction_required': 'Wymagana interakcja użytkownika',
  'access_denied': 'Dostęp zabroniony',
  'unauthorized': 'Brak autoryzacji',
  'popup_closed_by_user': 'Okno popup zostało zamknięte przez użytkownika'
};

export const getErrorMessage = (error: any): string => {
  if (error?.error) {
    return errorMessages[error.error as keyof typeof errorMessages] || error.error_description || error.error;
  }
  return error?.message || 'Wystąpił nieznany błąd';
};
```

---

## Podsumowanie

Gratulacje! Pomyślnie zaimplementowałeś zaawansowane uwierzytelnianie Auth0 w swojej aplikacji Svelte. Masz teraz dostęp do:

- Różnych metod logowania (redirect, popup)
- Ochrony tras z weryfikacją uprawnień
- Bezpiecznej komunikacji z API
- Obsługi tokenów i sesji
- Zaawansowanej obsługi błędów
- Wsparcia dla MFA i logowania społecznościowego

Możesz teraz rozbudowywać swoją aplikację o dodatkowe funkcje bezpieczeństwa i integracje z ekosystemem Auth0.