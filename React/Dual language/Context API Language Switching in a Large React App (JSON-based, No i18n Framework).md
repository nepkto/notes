# Context API Language Switching in a Large React App (JSON-based, No i18n Framework)

**Goal:** Implement language switching in a React application using:
- External JSON translation files
- React Context API (global access anywhere in the component tree)
- A simple `t("path.to.key")` translation function
- Optional lazy-loading + caching (recommended for large apps)

---

## 1) Why Context API for i18n?

### Purpose
- Provide `lang`, `setLang`, and `t()` globally without prop drilling.
- Let any component translate text by calling `t("menu.home")`.
- Enable runtime language switching and automatic re-rendering.

### When it’s a good fit
- You want a lightweight approach without installing i18n libraries.
- You control the translation format and app structure.
- You want to gradually scale (start small, add lazy-loaded sections later).

---

## 2) Recommended Folder Structure

```text
/public
  /lang
    en.json
    es.json

/src
  App.jsx
  Menu.jsx
  LanguageProvider.jsx
```

**Purpose:** files inside `/public` are served as static assets so `fetch("/lang/en.json")` works in production builds.

---

## 3) Translation JSON Files (examples)

### `public/lang/en.json`
```json
{
  "menu": {
    "home": "Home",
    "about": "About",
    "contact": "Contact"
  }
}
```

### `public/lang/es.json`
```json
{
  "menu": {
    "home": "Inicio",
    "about": "Acerca de",
    "contact": "Contacto"
  }
}
```

**Purpose:** keep translations structured by section (menu, common, pages, etc.).

---

## 4) Language Context + Provider

### `src/LanguageProvider.jsx`
```jsx
import React, { createContext, useContext, useEffect, useRef, useState } from "react";

// Create context
const LangContext = createContext(null);

// Translation getter using dot notation: t('menu.home')
function getTranslation(translations, key) {
  return (
    key
      .split(".")
      .reduce((obj, k) => (obj || {})[k], translations) ?? key
  );
}

export function LanguageProvider({ children }) {
  const [lang, setLang] = useState("en");
  const [translations, setTranslations] = useState({});

  // Optional: in-memory cache to avoid re-fetching already loaded languages
  const cache = useRef({}); // { en: {...}, es: {...} }

  useEffect(() => {
    let cancelled = false;

    async function load() {
      // Serve from cache if available
      if (cache.current[lang]) {
        setTranslations(cache.current[lang]);
        return;
      }

      // Fetch JSON file from /public/lang
      const res = await fetch(`/lang/${lang}.json`);
      const data = await res.json();

      cache.current[lang] = data;
      if (!cancelled) setTranslations(data);
    }

    load();
    return () => {
      cancelled = true;
    };
  }, [lang]);

  // Translation function exposed to the entire app
  const t = (key) => getTranslation(translations, key);

  return (
    <LangContext.Provider value={{ lang, setLang, t }}>
      {children}
    </LangContext.Provider>
  );
}

// Custom hook for simple usage
export function useLanguage() {
  const ctx = useContext(LangContext);
  if (!ctx) throw new Error("useLanguage must be used within LanguageProvider");
  return ctx;
}
```

### Purpose of this provider
- Loads translations when `lang` changes.
- Exposes `t(key)` to translate anywhere.
- Exposes `setLang` so any component can change language.

---

## 5) Menu Component (uses global translator)

### `src/Menu.jsx`
```jsx
import React from "react";
import { useLanguage } from "./LanguageProvider";

export default function Menu() {
  const { t } = useLanguage();

  return (
    <nav>
      <ul>
        <li>{t("menu.home")}</li>
        <li>{t("menu.about")}</li>
        <li>{t("menu.contact")}</li>
      </ul>
    </nav>
  );
}
```

**Purpose:** Demonstrates that any component can access translations without passing props.

---

## 6) App Component (wrap with provider + language selector)

### `src/App.jsx`
```jsx
import React from "react";
import { LanguageProvider, useLanguage } from "./LanguageProvider";
import Menu from "./Menu";

const languages = [
  { code: "en", label: "English" },
  { code: "es", label: "Español" }
];

function LanguageSelector() {
  const { lang, setLang } = useLanguage();

  return (
    <select value={lang} onChange={(e) => setLang(e.target.value)}>
      {languages.map((l) => (
        <option key={l.code} value={l.code}>
          {l.label}
        </option>
      ))}
    </select>
  );
}

export default function App() {
  return (
    <LanguageProvider>
      <LanguageSelector />
      <Menu />
      {/* Any other components can call useLanguage() */}
    </LanguageProvider>
  );
}
```

**Purpose:**  
- Wraps the component tree so everything can access translations.
- Provides a working language switcher for a menu.

---

## 7) How it works (flow)

1. App renders inside `<LanguageProvider>`.
2. Provider loads `/lang/en.json` (default `lang = "en"`).
3. Components call `t("menu.home")` and React renders English.
4. User selects `es` in the dropdown:
   - `setLang("es")` triggers effect
   - Provider fetches `/lang/es.json`
   - Context updates -> components re-render with Spanish text

---

## 8) Scaling tips for large apps

### A) Split translations by section (recommended)
Instead of one huge `en.json`, use:
```text
/public/lang/en/common.json
/public/lang/en/menu.json
/public/lang/en/dashboard.json
/public/lang/es/common.json
/public/lang/es/menu.json
/public/lang/es/dashboard.json
```

**Purpose:** load only the part needed per route/feature (faster and lower memory).

### B) Cache strategy
- In-memory cache (shown) is fast but grows with languages.
- If JSON files are large, consider:
  - caching only the active language
  - eviction (keep last N languages)
  - HTTP caching (CDN/browser cache headers)
  - IndexedDB for persistent caching (advanced)

### C) Fallback language
For production, you may want:
- if key missing in `es`, show value from `en`
- or show a readable fallback instead of the raw key

---

## 9) Example usage in any component

```jsx
import React from "react";
import { useLanguage } from "./LanguageProvider";

export function ProfileHeader() {
  const { t } = useLanguage();
  return <h1>{t("profile.title")}</h1>;
}
```

**Purpose:** shows how you can translate anywhere in the tree with zero prop drilling.

---