# React Dual-Language (i18n) Without Framework Using JSON + Context (Lazy-loading)

**Date:** 2026-01-14  
**Goal:** Add multi-language support in a React app **without** i18n frameworks by using **JSON translation files**, **React Context**, and **lazy-loading** (load only what is needed).

---

## Why do this?

### Purpose
- Centralize all UI text in JSON translation files.
- Allow users to switch language at runtime.
- Scale to large apps by loading translations **on demand** (lazy-loading), instead of downloading a giant file up front.

### Key Benefits
- **Performance:** Faster initial load (only default language and only needed sections loaded).
- **Scalability:** Works better as your app grows (more pages/features, more keys).
- **Maintainability:** Translators/editors can change JSON without touching React code.

---

## Concept Overview

1. **Store translations** in JSON files under `/public/lang/<lang>/<section>.json`.
2. **LanguageProvider (Context)** holds:
   - current language (`lang`)
   - method to change language (`setLang`)
   - translation function (`t(key)`)
   - a **loader** that lazy-loads JSON sections per language
3. Components call:
   - `t("menu.home")` for text
   - `loadSection("menu")` to ensure a section is loaded before rendering

---

## Recommended Folder Structure

```text
/public
  /lang
    /en
      menu.json
      common.json
    /es
      menu.json
      common.json

/src
  App.jsx
  Menu.jsx
  LanguageProvider.jsx
```

**Purpose:** Keep translation files static, cacheable, and served by the web server/CDN.

---

## Translation Files (Examples)

### `public/lang/en/menu.json`
```json
{
  "menu": {
    "home": "Home",
    "about": "About",
    "contact": "Contact"
  }
}
```

### `public/lang/es/menu.json`
```json
{
  "menu": {
    "home": "Inicio",
    "about": "Acerca de",
    "contact": "Contacto"
  }
}
```

### `public/lang/en/common.json`
```json
{
  "common": {
    "loading": "Loading...",
    "language": "Language"
  }
}
```

### `public/lang/es/common.json`
```json
{
  "common": {
    "loading": "Cargando...",
    "language": "Idioma"
  }
}
```

**Purpose:** Split translations by feature/section (`menu`, `common`, `dashboard`, etc.) to avoid loading huge files.

---

## Core Implementation (Context + Lazy-loading + Cache)

### `src/LanguageProvider.jsx`
```jsx
import React, { createContext, useContext, useMemo, useRef, useState } from "react";

const LangContext = createContext(null);

/**
 * Safely read nested keys using dot notation.
 * Example: getByPath({a:{b:1}}, "a.b") -> 1
 */
function getByPath(obj, path) {
  return path.split(".").reduce((acc, key) => (acc && acc[key] !== undefined ? acc[key] : undefined), obj);
}

/**
 * Merge translation objects deeply-ish (simple merge for nested objects).
 * Good enough for typical i18n structures.
 */
function deepMerge(target, source) {
  const out = { ...target };
  for (const k of Object.keys(source)) {
    if (
      typeof source[k] === "object" &&
      source[k] !== null &&
      !Array.isArray(source[k]) &&
      typeof target[k] === "object" &&
      target[k] !== null &&
      !Array.isArray(target[k])
    ) {
      out[k] = deepMerge(target[k], source[k]);
    } else {
      out[k] = source[k];
    }
  }
  return out;
}

export function LanguageProvider({ children, defaultLang = "en" }) {
  const [lang, setLang] = useState(defaultLang);

  /**
   * `translationsByLang.current` stores loaded sections for each language in memory.
   *
   * Shape:
   * {
   *   en: { menu: {...}, common: {...} },
   *   es: { menu: {...} }
   * }
   *
   * Purpose: prevents refetching already-loaded sections.
   */
  const translationsByLang = useRef({});

  /**
   * `merged.current` stores the merged translation tree for the current language
   * so `t("menu.home")` can be fast.
   *
   * Shape:
   * {
   *   menu: {...},
   *   common: {...}
   * }
   */
  const merged = useRef({});

  /**
   * Tracks in-flight requests to avoid duplicate fetches
   * if multiple components request the same section simultaneously.
   */
  const inflight = useRef({}); // key: `${lang}:${section}` -> Promise

  /**
   * Load a translation section file like:
   * /lang/en/menu.json
   *
   * Purpose:
   * - Lazy-load only the pieces you need.
   * - Cache results in memory.
   * - Avoid double-fetch with inflight tracking.
   */
  async function loadSection(section) {
    const inflightKey = `${lang}:${section}`;

    // Already loaded?
    const existing = translationsByLang.current[lang]?.[section];
    if (existing) return existing;

    // Already fetching?
    if (inflight.current[inflightKey]) return inflight.current[inflightKey];

    // Start fetch
    inflight.current[inflightKey] = (async () => {
      const res = await fetch(`/lang/${lang}/${section}.json`);

      if (!res.ok) {
        // Fail soft: keep UI working with keys as fallback.
        // You can also throw to show an error boundary.
        console.warn(`Missing translation file: /lang/${lang}/${section}.json`);
        return null;
      }

      const json = await res.json();

      // Cache per language + section
      if (!translationsByLang.current[lang]) translationsByLang.current[lang] = {};
      translationsByLang.current[lang][section] = json;

      // Merge into the translation tree for quick lookup
      merged.current = deepMerge(merged.current, json);

      return json;
    })();

    try {
      return await inflight.current[inflightKey];
    } finally {
      delete inflight.current[inflightKey];
    }
  }

  /**
   * Reset merged cache when language changes.
   * Purpose: ensure UI uses translations for the active language only.
   */
  function changeLanguage(nextLang) {
    setLang(nextLang);

    // Reset merged tree; we will rebuild as sections are loaded.
    merged.current = {};
  }

  /**
   * Translation function with fallback:
   * - If key exists, return translation.
   * - If missing, return the key itself (useful during development).
   */
  function t(key) {
    const value = getByPath(merged.current, key);
    return value !== undefined ? value : key;
  }

  const value = useMemo(() => {
    return { lang, setLang: changeLanguage, t, loadSection };
  }, [lang]);

  return <LangContext.Provider value={value}>{children}</LangContext.Provider>;
}

export function useLanguage() {
  const ctx = useContext(LangContext);
  if (!ctx) throw new Error("useLanguage must be used inside <LanguageProvider>");
  return ctx;
}
```

**Purpose of this file:**
- Provides a global language state and helpers.
- Implements **lazy-loading** by section.
- Keeps an in-memory cache to reduce network requests.

---

## Example UI Components

### `src/Menu.jsx`
```jsx
import React, { useEffect, useState } from "react";
import { useLanguage } from "./LanguageProvider";

export default function Menu() {
  const { t, loadSection } = useLanguage();
  const [ready, setReady] = useState(false);

  // Ensure the menu section is loaded before rendering translated labels
  useEffect(() => {
    let cancelled = false;
    (async () => {
      await loadSection("menu");
      if (!cancelled) setReady(true);
    })();
    return () => {
      cancelled = true;
    };
  }, [loadSection]);

  // Fallback UI while loading menu translations
  if (!ready) return <div>{t("common.loading")}</div>;

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

**Purpose:**
- Demonstrates how a component can lazy-load its own translation section.
- Prevents rendering “raw keys” while still loading.

---

### `src/App.jsx`
```jsx
import React, { useEffect, useState } from "react";
import { LanguageProvider, useLanguage } from "./LanguageProvider";
import Menu from "./Menu";

const LANGUAGE_OPTIONS = [
  { code: "en", label: "English" },
  { code: "es", label: "Español" }
];

function LanguageSelector() {
  const { lang, setLang, loadSection, t } = useLanguage();
  const [commonReady, setCommonReady] = useState(false);

  // Load common strings once per language (label, loading text, etc.)
  useEffect(() => {
    let cancelled = false;
    (async () => {
      await loadSection("common");
      if (!cancelled) setCommonReady(true);
    })();
    return () => {
      cancelled = true;
    };
  }, [lang, loadSection]);

  if (!commonReady) return <div>Loading...</div>;

  return (
    <div style={{ marginBottom: 12 }}>
      <label style={{ marginRight: 8 }}>{t("common.language")}:</label>
      <select value={lang} onChange={(e) => setLang(e.target.value)}>
        {LANGUAGE_OPTIONS.map((opt) => (
          <option key={opt.code} value={opt.code}>
            {opt.label}
          </option>
        ))}
      </select>
    </div>
  );
}

export default function App() {
  return (
    <LanguageProvider defaultLang="en">
      <LanguageSelector />
      <Menu />
    </LanguageProvider>
  );
}
```

**Purpose:**
- Wraps the app with `LanguageProvider`.
- Shows language switching at top level.
- Loads `common` strings for every language.

---

## How Lazy-loading Helps When JSON Is Large

### Problem with huge JSON files
If you have one file like `/lang/en.json` containing *everything*:
- downloading is slow
- parsing is slower
- memory usage is high
- caching large objects in memory can degrade performance

### Lazy-loading + splitting solves it
By splitting translations:
- the Menu only loads `/lang/<lang>/menu.json`
- a Dashboard page only loads `/lang/<lang>/dashboard.json`
- users download only what they actually use

**Result:** faster initial load + less memory usage.

---

## Caching Considerations (Important)

### In-memory cache (what we did)
- Pros: very fast after first load, simplest
- Cons: memory grows as you visit more sections + languages

**Tip:** If memory becomes an issue, you can:
- cache only the active language
- evict old sections (LRU-like strategy)
- store cache in `indexedDB` for persistence (more complex)

### Browser/HTTP cache (recommended)
Because translation files are static in `/public`, you can also rely on:
- CDN caching
- browser caching via proper cache headers

**Purpose:** reduce repeat downloads even after refresh.

---

## Common Enhancements (Optional)

1. **Fallback language** (e.g., if `es` missing a key, use `en`)
2. **Pluralization rules** (one/many)
3. **Interpolation**
   - Example: `"hello": "Hello, {name}!"`
4. **Preload critical sections**
   - Load `common` at startup
   - Lazy-load everything else

---

## Quick Usage Summary

1. Put translations in:
   - `/public/lang/en/common.json`, `/public/lang/en/menu.json`
   - `/public/lang/es/common.json`, `/public/lang/es/menu.json`

2. Wrap your app:
   - `<LanguageProvider> ... </LanguageProvider>`

3. In any component:
   - `const { t, loadSection } = useLanguage();`
   - `await loadSection("menu");`
   - render: `{t("menu.home")}`

---

## Troubleshooting Notes

- If `fetch('/lang/...')` returns 404:
  - confirm files are in `/public/lang/...`
  - confirm correct path and server setup
- If you see keys like `menu.home` in UI:
  - the section may not be loaded yet
  - load it with `loadSection("menu")`
- Consider adding an Error Boundary if missing files should fail loudly.

---