# Kompleksowa dokumentacja systemu motywów SCSS

## Wprowadzenie

Ten dokument opisuje zaawansowany system motywów SCSS wykorzystywany w projekcie. System oparty jest na CSS Custom Properties (CSS Variables) z fallbackiem do SASS variables dla starszych przeglądarek. Obsługuje motywy ciemny, jasny oraz automatyczny tryb systemowy.

## Architektura systemu motywów

### Struktura plików
```
src/
├── colors.scss          # Zmienne kolorów i motywów
├── typography.scss      # Zmienne i mixiny typograficzne
├── app.scss            # Główny plik stylów
└── lib/
    └── Theme.ts        # TypeScript - zarządzanie motywami
```

## System kolorów

### Podstawowe zmienne SASS

#### Kolory bazowe
```scss
// Kolory podstawowe
$color-white: #ffffff;
$color-black: #000000;

// Kolory brandu
$color-primary: #5966ff;
$color-primary-hover: #4754f0;
$color-primary-light: #6e75ff;

// Powierzchnie
$color-surface: #0f1724;
$color-elevation-1: #0b1220;
$color-muted: rgba(255, 255, 255, 0.65);
```

#### Kolory tekstów
```scss
// Na ciemnym tle
$color-text-dark: rgba(255, 255, 255, 0.92);
$color-text-subtle: rgba(255, 255, 255, 0.68);
$color-text-light: #213547;
```

### CSS Custom Properties

System wykorzystuje CSS Variables dla dynamicznych motywów:

#### Motyw domyślny (ciemny)
```css
:root {
    --color-primary: #5966ff;
    --color-primary-hover: #4754f0;
    --color-primary-alpha: rgba(89, 102, 255, 0.1);
    
    --color-surface: #0f1724;
    --color-elevation-1: #0b1220;
    --color-bg: #0f1724;
    --color-on-bg: rgba(255, 255, 255, 0.92);
    
    --color-muted: rgba(255, 255, 255, 0.68);
    --color-header-bg: rgba(11, 18, 32, 0.65);
    --color-card-bg: rgba(255, 255, 255, 0.02);
    
    --color-button-bg: #0d1320;
    --color-button-text: #ffffff;
    
    --color-border: rgba(255, 255, 255, 0.06);
    
    // Statusy
    --color-success: #22c55e;
    --color-warning: #f59e0b;
    --color-danger: #ef4444;
}
```

#### Motyw jasny
```css
:root[data-theme="light"] {
    --color-primary: #4754f0;
    --color-primary-hover: #3a45d6;
    --color-primary-alpha: rgba(71, 84, 240, 0.1);
    
    --color-surface: #ffffff;
    --color-elevation-1: #f5f7fb;
    --color-bg: #ffffff;
    --color-on-bg: #213547;
    
    --color-muted: rgba(33, 53, 71, 0.6);
    --color-header-bg: rgba(255, 255, 255, 0.9);
    --color-card-bg: #f9fbff;
    
    --color-button-bg: #f3f5fb;
    --color-button-text: #213547;
    
    --color-border: rgba(33, 53, 71, 0.08);
    
    // Statusy - dostosowane do jasnego tła
    --color-success: #12b76a;
    --color-warning: #d97706;
    --color-danger: #dc2626;
}
```

## System typograficzny

### Zmienne
```scss
// Font family
$font-family: system-ui, Avenir, Helvetica, Arial, sans-serif;
$font-weight: 400;
$line-height: 1.5;

// Font rendering
$font-synthesis: none;
$text-rendering: optimizeLegibility;
$webkit-font-smoothing: antialiased;
$moz-osx-font-smoothing: grayscale;
```

### Mixiny typograficzne

#### Podstawowa typografia
```scss
@mixin typography-base {
    font-family: $font-family;
    font-weight: $font-weight;
    line-height: $line-height;
    font-synthesis: $font-synthesis;
    text-rendering: $text-rendering;
    -webkit-font-smoothing: $webkit-font-smoothing;
    -moz-osx-font-smoothing: $moz-osx-font-smoothing;
}
```

#### Linki
```scss
@mixin typography-link {
    font-weight: 500;
    text-decoration: inherit;
    color: var(--color-primary);
    transition: color 0.25s;
}

@mixin typography-link-hover {
    color: var(--color-primary-hover);
}
```

#### Przyciski
```scss
@mixin typography-button {
    border-radius: 8px;
    border: 1px solid transparent;
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    background-color: var(--color-button-bg);
    color: var(--color-button-text);
    cursor: pointer;
    transition: border-color 0.25s;
}

@mixin typography-button-hover {
    border-color: var(--color-primary);
}

@mixin typography-button-focus {
    outline: 4px auto var(--color-primary);
}
```

## TypeScript - zarządzanie motywami

### Podstawowe funkcje

#### Inicjalizacja motywu
```typescript
import { initTheme } from './lib/Theme';

// Automatyczna inicjalizacja przy starcie aplikacji
const currentTheme = initTheme();
```

#### Zmiana motywu
```typescript
import { setTheme } from './lib/Theme';

// Ustawienie konkretnego motywu
setTheme('dark');
setTheme('light');
setTheme('system'); // Używa preferencji systemowych
```

#### Subskrypcja zmian
```typescript
import { subscribe } from './lib/Theme';

// Nasłuchiwanie na zmiany motywu
const unsubscribe = subscribe((theme) => {
    console.log('Motyw zmieniony na:', theme);
});

// Anulowanie subskrypcji
unsubscribe();
```

### Theme.ts — API i uwagi implementacyjne

Plik `src/lib/Theme.ts` eksportuje lekki, typowany interfejs do zarządzania motywami. Dla deweloperów warto znać następujące punkty:

- Eksportowane typy i funkcje:
    - `type Theme = "dark" | "light" | "system";`
    - `export function initTheme(): "dark" | "light"` — inicjalizuje motyw (localStorage lub system) i ustawia `data-theme` na `<html>`; zwraca rozstrzygnięty motyw.
    - `export function setTheme(theme: Theme): void` — zapisuje preferencję w localStorage i aplikuje motyw.
    - `export function getStoredTheme(): Theme | null` — zwraca zapisany wybór lub null.
    - `export function subscribe(cb: (theme: "dark" | "light") => void): () => void` — umożliwia subskrypcję zmian; zwraca funkcję do usunięcia subskrypcji.
    - `export function applyResolvedTheme(resolved: "dark" | "light")` — aplikuje motyw bez zapisu do storage (użyteczne w testach/SSR injection).

Uwagi implementacyjne:
- `initTheme()` rejestruje listener na `matchMedia('(prefers-color-scheme: dark)')`, obsługując zarówno `addEventListener` jak i starsze `addListener`, dzięki czemu aplikacja reaguje na zmianę preferencji systemowych.
- Funkcje zabezpieczają dostęp do `window`/`localStorage`, więc są bezpieczne dla środowisk isomorphic — jednak wywołuj je po stronie klienta (np. w `onMount` w Svelte).

### SSR i zapobieganie FOUC (Flash of Unstyled Content)

Aby uniknąć krótkiego błysku nieprawidłowego motywu przed wykonaniem aplikacyjnego JS, warto ustawić `data-theme` na `<html>` możliwie najwcześniej — przed załadowaniem bundla. Przykładowy, lekki skrypt do umieszczenia w `index.html` przed skryptem aplikacji:

```html
<script>
    (function(){
        try {
            var t = localStorage.getItem('theme');
            if (t) {
                if (t === 'system') {
                    if (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) {
                        document.documentElement.setAttribute('data-theme','dark');
                    } else {
                        document.documentElement.setAttribute('data-theme','light');
                    }
                } else {
                    document.documentElement.setAttribute('data-theme', t);
                }
            }
        } catch (e) {
            /* ignore storage errors */
        }
    })();
</script>
```

Uwaga: w aplikacji Svelte wywołuj `initTheme()` w `onMount()` (jak pokazano wcześniej). `initTheme()` jest implementowane defensywnie względem SSR, ale manipulacja DOM nastąpi dopiero po stronie klienta — pre-injection powyżej zapobiegnie FOUC.

### Praktyczne wskazówki

- Importuj `src/colors.scss` jako pierwszy w głównym pliku SCSS, żeby CSS Custom Properties były dostępne globalnie.
- Dla bardzo starych przeglądarek używaj SASS variables jako fallback (repo zawiera wartości SASS, np. `$color-primary`).
- Jeśli chcesz wymusić określony motyw podczas testów lub w trybie developer, użyj `setTheme('dark')`/`setTheme('light')` zamiast manipulować DOM bezpośrednio.

### Przechowywanie preferencji

System automatycznie zapisuje preferencje w localStorage:

```typescript
// Odczyt zapisanej preferencji
const storedTheme = localStorage.getItem('theme');
// Możliwe wartości: "dark", "light", "system" lub null
```

## Przykłady użycia w komponentach

### Svelte - komponent z dynamicznym motywem
```svelte
<script lang="ts">
    import { onMount } from 'svelte';
    import { initTheme, setTheme } from '$lib/Theme';
    
    let currentTheme: 'dark' | 'light' = 'dark';
    
    onMount(() => {
        currentTheme = initTheme();
    });
    
    function toggleTheme() {
        const newTheme = currentTheme === 'dark' ? 'light' : 'dark';
        setTheme(newTheme);
        currentTheme = newTheme;
    }
</script>

<div class="theme-container">
    <h1>Aktualny motyw: {currentTheme}</h1>
    <button on:click={toggleTheme}>
        Przełącz na {currentTheme === 'dark' ? 'jasny' : 'ciemny'}
    </button>
</div>

<style>
    .theme-container {
        background-color: var(--color-bg);
        color: var(--color-on-bg);
        padding: 2rem;
        border-radius: 8px;
        border: 1px solid var(--color-border);
    }
    
    button {
        background-color: var(--color-button-bg);
        color: var(--color-button-text);
        border: 1px solid var(--color-border);
        padding: 0.5rem 1rem;
        border-radius: 4px;
        cursor: pointer;
    }
    
    button:hover {
        background-color: var(--color-surface-hover);
    }
</style>
```

### SCSS - użycie zmiennych
```scss
// Użycie CSS Custom Properties
.my-component {
    background-color: var(--color-surface);
    color: var(--color-on-bg);
    border: 1px solid var(--color-border);
    
    &:hover {
        background-color: var(--color-surface-hover);
    }
}

// Fallback dla starszych przeglądarek
@supports not (--css: variables) {
    .my-component {
        background-color: $color-surface;
        color: $color-text-dark;
    }
}
```

### Responsive design z motywami
```scss
// Mixin dla responsywności z motywami
@mixin responsive-theme {
    @media (max-width: 768px) {
        :root {
            --spacing-base: 0.5rem;
            --font-size-base: 14px;
        }
    }
    
    @media (min-width: 769px) {
        :root {
            --spacing-base: 1rem;
            --font-size-base: 16px;
        }
    }
}

// Użycie w komponencie
.responsive-card {
    padding: var(--spacing-base);
    font-size: var(--font-size-base);
    
    @include responsive-theme;
}
```

## Zaawansowane techniki

### Dynamiczne generowanie kolorów
```scss
// Funkcja do generowania odcieni
@function generate-shade($color, $percentage) {
    @return mix(black, $color, $percentage);
}

@function generate-tint($color, $percentage) {
    @return mix(white, $color, $percentage);
}

// Użycie
$primary-dark: generate-shade($color-primary, 20%);
$primary-light: generate-tint($color-primary, 20%);
```

### Animacje z motywami
```scss
// Animacje dostosowane do motywu
.theme-transition {
    transition: 
        background-color 0.3s ease,
        color 0.3s ease,
        border-color 0.3s ease;
}

// Preferencje użytkownika dla redukcji ruchu
@media (prefers-reduced-motion: reduce) {
    .theme-transition {
        transition: none;
    }
}
```

### Dark mode z systemową preferencją
```scss
// Automatyczny dark mode na podstawie systemu
@media (prefers-color-scheme: dark) {
    :root:not([data-theme]) {
        @include color-scheme-dark;
    }
}

@media (prefers-color-scheme: light) {
    :root:not([data-theme]) {
        @include color-scheme-light;
    }
}
```

## Troubleshooting

### Częste problemy

1. **Flash of unstyled content (FOUC)**
   - Użyj `initTheme()` przed renderowaniem aplikacji
   - Dodaj klasę na `<html>` element zamiast `<body>`

2. **Nie działa w starszych przeglądarkach**
   - Dodaj fallback do SASS variables
   - Użyj PostCSS dla autoprefixer

3. **Kolory nie są spójne**
   - Sprawdź kolejność importów plików SCSS
   - Upewnij się że CSS Custom Properties są zdefiniowane przed użyciem

### Debugowanie
```javascript
// Sprawdzenie aktualnych wartości CSS
console.log(getComputedStyle(document.documentElement)
    .getPropertyValue('--color-primary'));

// Lista wszystkich zmiennych CSS
console.log(getComputedStyle(document.documentElement));
```

## Narzędzia developerskie

### Browser DevTools
- **Chrome**: Elements > Styles > CSS Variables
- **Firefox**: Inspector > Variables
- **Safari**: Elements > Styles > Variables

### VS Code Extensions
- **SCSS IntelliSense** - autouzupełnianie zmiennych
- **CSS Variables** - podgląd wartości zmiennych
- **Color Highlight** - podgląd kolorów w edytorze

## Najlepsze praktyki

1. **Nazewnictwo**
   - Używaj konwencji BEM dla klas
   - Prefiksuj zmienne CSS: `--component-state-property`

2. **Organizacja**
   - Grupuj powiązane zmienne
   - Używaj komentarzy do dokumentacji

3. **Wydajność**
   - Ogranicz liczbę zmiennych
   - Używaj shorthand properties gdzie możliwe

4. **Dostępność**
   - Testuj kontrast kolorów
   - Używaj semantic naming zamiast kolorów
   - Dodaj support dla high contrast mode