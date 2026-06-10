# Codebase Summary

This is a **React 19 + TypeScript Chrome/Firefox extension** boilerplate using Vite and Turborepo. It follows a monorepo structure with shared packages and modular pages.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Root Level Files & Configuration](#root-level-files--configuration)
3. [Directory Structure](#directory-structure)
4. [Build Pipeline](#build-pipeline)
5. [Key Technologies](#key-technologies)
6. [Development Workflow](#development-workflow)

---

## Project Overview

A modern, production-ready Chrome/Firefox extension boilerplate built with:
- **React 19** for UI components
- **TypeScript** for type safety
- **Vite** for fast development and optimized builds
- **Turborepo** for efficient monorepo task orchestration
- **Manifest V3** for latest Chrome extension standards
- **Tailwind CSS** for styling
- **WebdriverIO** for end-to-end testing

The project uses a monorepo structure with shared packages and multiple isolated page bundles (popup, options, devtools, etc.), each compiled separately.

---

## Root Level Files & Configuration

| File | Purpose |
|------|---------|
| `AGENTS.md` | Guidance for Warp agent; development instructions and architecture overview |
| `package.json` | Root workspace manifest with shared dependencies and npm scripts |
| `pnpm-workspace.yaml` | Defines monorepo workspaces for pnpm |
| `pnpm-lock.yaml` | Locked dependency versions for reproducible installs |
| `turbo.json` | Turborepo configuration for task orchestration and caching |
| `tsconfig.json` | Root TypeScript configuration shared across packages |
| `eslint.config.ts` | ESLint configuration for code linting |
| `.prettierrc` | Code formatting rules for Prettier |
| `.prettierignore` | Files to exclude from Prettier formatting |
| `.env` | Environment variables (prefix all with `CEB_`) |
| `.example.env` | Example environment variables template |
| `.npmrc` | npm registry and configuration settings |
| `.nvmrc` | Node version requirement (>= 22.15.1) |
| `.gitignore` | Git ignore patterns |
| `.gitattributes` | Git attributes configuration |
| `.gitguardian.yaml` | Secret scanning configuration |
| `.husky/` | Git hooks configuration (pre-commit) |
| `.github/` | GitHub Actions workflows |
| `README.md` | Project documentation and setup instructions |
| `LICENSE` | MIT license |

---

## Directory Structure

### `chrome-extension/`

The core extension configuration and background service worker.

**Key Files:**
- `manifest.ts` - Generates `manifest.json` (Chrome Manifest V3 format)
- `src/background/index.ts` - Background service worker (runs in background context)
- `public/` - Static assets
  - `icon-34.png` - Extension toolbar icon (34px)
  - `icon-128.png` - Extension store icon (128px)
  - `content.css` - CSS injected into pages
- `vite.config.mts` - Vite build configuration for manifest generation
- `tsconfig.json` - TypeScript config for pre-build steps
- `package.json` - Local dependencies for this workspace
- `utils/plugins/make-manifest-plugin.ts` - Custom Vite plugin for manifest generation

**Purpose:** Defines the extension's entry point, permissions, content scripts, and background worker logic.

---

### `pages/` - UI Entry Points

Each sub-folder becomes a separate Vite bundle and represents one UI surface of the extension.

#### Popup Page
- **Path:** `pages/popup/`
- **Purpose:** Extension popup UI shown when clicking the extension icon in the toolbar
- **Output:** Single bundled page displayed in a small popup window
- **Files:**
  - `src/index.tsx` - React entry point
  - `vite.config.mts` - Build configuration
  - `package.json` - Local dependencies

#### Options Page
- **Path:** `pages/options/`
- **Purpose:** Settings/preferences page accessible from context menu
- **Output:** Full-page settings interface
- **Features:** User preferences, configuration storage

#### New Tab Page
- **Path:** `pages/new-tab/`
- **Purpose:** Override Chrome's default new tab page
- **Output:** Custom new tab replacement
- **Config:** `chrome_url_overrides.newtab` in manifest

#### Side Panel
- **Path:** `pages/side-panel/`
- **Purpose:** Chrome 114+ side panel (persistent sidebar UI)
- **Output:** Persistent panel alongside web pages
- **Config:** `side_panel.default_path` in manifest
- **Requirements:** Chrome 114+

#### DevTools Extension
- **Path:** `pages/devtools/`
- **Purpose:** Extends Chrome DevTools with custom panels
- **Output:** Custom DevTools page
- **Config:** `devtools_page` in manifest
- **Reference:** Works with `devtools-panel` for the actual panel UI

#### DevTools Panel
- **Path:** `pages/devtools-panel/`
- **Purpose:** Custom panel within DevTools (created by `devtools` page)
- **Output:** Panel content displayed in DevTools
- **Relationship:** Controlled by `pages/devtools/src/index.ts`

#### Content Scripts (Plain JavaScript)
- **Path:** `pages/content/`
- **Purpose:** Inject plain JavaScript into specified web pages
- **Output:** Console-accessible scripts
- **Routing System:** Directory-based multi-entry
  - Each folder under `src/matches/` becomes its own IIFE bundle
  - Example: `src/matches/all/index.ts` → `dist/content/all.iife.js`
  - Folder name must match entry in `manifest.ts` `content_scripts`
  - Every match folder must contain `index.ts` or `index.tsx`

#### Content UI (React Components)
- **Path:** `pages/content-ui/`
- **Purpose:** Inject React components into web pages
- **Output:** React UI rendered at the bottom of web pages
- **Use Case:** Interactive overlays, sidebars, or UI enhancements on existing websites
- **Routing System:** Directory-based multi-entry (like content)

#### Content Runtime
- **Path:** `pages/content-runtime/`
- **Purpose:** Content scripts with extended runtime functionality
- **Output:** Advanced content scripts with more capabilities
- **Injection:** Can be injected from other pages (e.g., popup)
- **Routing System:** Directory-based multi-entry
- **Difference from `content`:** Runtime provides more capabilities than standard content scripts

**Common Files in Each Page:**
- `src/index.tsx` or `src/index.ts` - React/TypeScript entry point
- `src/[PageName].tsx` - Main page component
- `vite.config.mts` - Vite build configuration
- `tailwind.config.ts` - Tailwind CSS configuration (where applicable)
- `tsconfig.json` - TypeScript configuration
- `package.json` - Local dependencies

---

### `packages/` - Shared Libraries & Tools

Shared code consumed by pages, background worker, and each other.

#### **@extension/storage**
- **Purpose:** Reactive Chrome storage wrapper
- **Key API:** `createStorage<T>(key, fallback, config)`
- **React Hook:** `useStorage(storageObject)`
- **Implementation:** Uses `subscribe`/`getSnapshot` interface for `useSyncExternalStore`
- **Features:**
  - Reactive state across extension contexts
  - Support for both local and session storage
  - `liveUpdate: true` option to sync across contexts
  - Automatic propagation of mutations to UI pages
- **Example:** Define storage objects in `packages/storage/lib/impl/` and use in any page

#### **@extension/shared**
- **Purpose:** Shared code for entire project
- **Exports:**
  - `useStorage()` - Hook for using storage objects
  - `withErrorBoundary` - HOC for error handling
  - `withSuspense` - HOC for suspense boundaries
  - Shared types and constants
  - Custom hooks
  - Common components
- **Usage:** Provides common functionality wrapped around every page component

#### **@extension/env**
- **Purpose:** Environment variables management
- **Features:**
  - Exposes `process.env` values to Vite
  - Type-safe access to environment variables
  - Support for two prefixes:
    - `CEB_` for `.env` file variables
    - `CLI_CEB_` for CLI-only variables (set via `pnpm set-global-env`)
  - Auto-set variables: `CLI_CEB_DEV`, `CLI_CEB_FIREFOX`
- **Usage:** Import constants like `IS_DEV`, `IS_PROD` from `@extension/env`
- **Access Syntax:** `process.env['CEB_MY_VAR']` (bracket notation for IDE autocomplete)

#### **@extension/vite-config**
- **Purpose:** Shared Vite configuration factory
- **Key Export:** `withPageConfig(config)` - Merges page-specific config with base config
- **Usage:** All page `vite.config.mts` files use this base configuration
- **Benefit:** Consistent build settings across all pages

#### **@extension/i18n**
- **Purpose:** Type-safe internationalization
- **Key API:** `t('key')` for translations
- **Locale Files:** `packages/i18n/locales/{locale}/messages.json`
- **Features:**
  - Type-safe translation keys
  - Multi-language support
  - Validation of translation keys
- **Setup:** Edit locale messages for your extension name and description

#### **@extension/ui**
- **Purpose:** Shared UI components and styling utilities
- **Exports:**
  - UI Components: `ToggleButton`, `LoadingSpinner`, `ErrorDisplay`
  - Utility: `cn()` function for Tailwind class merging
- **Benefit:** Consistent UI across all pages
- **Integration:** Components use Tailwind CSS from `@extension/tailwindcss-config`

#### **@extension/hmr**
- **Purpose:** Custom Hot Module Reload (HMR) for extension development
- **Features:**
  - Custom HMR plugin: `watchRebuildPlugin`, `makeEntryPointPlugin`
  - Injection script for extension reload/refresh
  - HMR dev-server
- **Benefit:** Fast development with automatic reload on code changes
- **Usage:** Automatically integrated in page Vite configs

#### **@extension/dev-utils**
- **Purpose:** Development utilities for Chrome extension development
- **Tools:**
  - Manifest parser
  - Logger utility
- **Benefit:** Simplifies debugging and manifest validation

#### **@extension/tailwindcss-config**
- **Purpose:** Shared Tailwind CSS configuration
- **Usage:** All pages that use Tailwind inherit this base configuration
- **Customization:** Pages can extend with their own `tailwind.config.ts`

#### **@extension/tsconfig**
- **Purpose:** Shared TypeScript configuration templates
- **Usage:** Referenced by all page `tsconfig.json` files
- **Benefit:** Consistent TypeScript settings across the project

#### **@extension/module-manager**
- **Purpose:** CLI tool to enable/disable pages
- **Command:** `pnpm module-manager`
- **Features:**
  - Enable/disable extension pages
  - Delete specific pages: `pnpm module-manager -d popup`
- **Benefit:** Customize which pages are included in your extension

#### **@extension/zipper**
- **Purpose:** Package extension for distribution
- **Command:** `pnpm zip` (Chrome) or `pnpm zip:firefox` (Firefox)
- **Output:** `dist-zip/extension-YYYYMMDD-HHmmss.zip`
- **Workflow:** Runs build automatically, then creates zip
- **Use Case:** Preparing extension for Chrome Web Store or Firefox Add-ons

---

### `tests/` - End-to-End Testing

**Path:** `tests/e2e/`

- **Framework:** WebdriverIO
- **Test Specs:** `tests/e2e/specs/`
- **Target:** Tests against the zipped (production) extension
- **Requirements:** Built extension zip required before running tests
- **Commands:**
  - `pnpm e2e` - Run tests on Chrome
  - `pnpm e2e:firefox` - Run tests on Firefox
- **Note:** No unit tests; testing is E2E focused

---

### `bash-scripts/` - Utility Scripts

Shell scripts for common development tasks:

- **`update_version.sh`** - Updates version across project files
  - Run with: `pnpm update-version <version>`
- **`set_global_env.sh`** - Sets environment variables for CLI
  - Sets `CLI_CEB_DEV` and `CLI_CEB_FIREFOX` flags
- **`copy_env.sh`** - Copies `.env` from template
  - Runs automatically on `pnpm install` (postinstall hook)

---

## Data Flow

```
Storage Objects (packages/storage/lib/impl/)
    ↓
Background Worker (chrome-extension/src/background)
    ↓
All UI Pages (pages/*)
    ↓
React Components via useStorage() hook
    ↓
Automatic Sync across extension contexts
```

**Key Point:** Storage mutations in one context (e.g., background) automatically propagate to all UI pages via `useSyncExternalStore`.

---

## Build Pipeline

### Task Execution Order (via Turborepo)

1. **`ready`** (Pre-build phase)
   - Generates manifest from `manifest.ts`
   - Prepares build environment
   - Output: Updated `manifest.json`

2. **`build`** (Main build phase)
   - Runs in dependency order
   - Builds all Vite bundles in parallel
   - Each page compiled independently
   - Background worker bundled
   - Output: All bundles to `dist/` directory

3. **Output**
   - All bundles output to shared top-level `dist/` directory
   - Not per-package `dist/` folders
   - Structure: `dist/popup/`, `dist/options/`, `dist/content/`, etc.

### Development Pipeline

```bash
pnpm dev
  ↓
set CLI_CEB_DEV=true
  ↓
pnpm clean:bundle
  ↓
turbo ready  # Generate manifest
  ↓
turbo watch dev  # Watch mode with HMR
  ↓
Load dist/ as unpacked extension
  ↓
Edit code → Auto-reload in browser
```

### Production Pipeline

```bash
pnpm build
  ↓
set CLI_CEB_DEV=false
  ↓
pnpm clean:bundle
  ↓
turbo build
  ↓
Optimized dist/
  ↓
Ready for Chrome Web Store
```

### Packaging Pipeline

```bash
pnpm zip
  ↓
Run pnpm build
  ↓
Run zipper package
  ↓
Create dist-zip/extension-YYYYMMDD-HHmmss.zip
  ↓
Ready for distribution
```

---

## Key Technologies

| Technology | Purpose | Version |
|---|---|---|
| **React** | UI framework | 19.1.0 |
| **TypeScript** | Type safety | 5.8.3+ |
| **Vite** | Build tool & dev server | 6.3.6+ |
| **Rollup** | Module bundler | (via Vite) |
| **Turborepo** | Monorepo orchestration | 2.5.3+ |
| **Tailwind CSS** | Utility-first styling | 3.4.17+ |
| **Chrome Extensions** | MV3 spec | Latest |
| **WebdriverIO** | E2E testing | Latest |
| **ESLint** | Code linting | 9.27.0+ |
| **Prettier** | Code formatting | 3.5.3+ |
| **Husky** | Git hooks | 9.1.7+ |
| **pnpm** | Package manager | 10.11.0+ |

---

## Development Workflow

### Prerequisites

- Node.js >= 22.15.1 (check `.nvmrc`)
- pnpm >= 10.0 (install globally: `npm install -g pnpm`)

### Installation

```bash
# Clone repository
git clone https://github.com/Jonghakseo/chrome-extension-boilerplate-react-vite
cd react-instagram-unfollow-chrome-extension

# Install dependencies
pnpm install

# Copy environment variables (automatic via postinstall)
# Manually if needed: pnpm copy-env
```

### Development Commands

```bash
# Watch mode with HMR (Chrome)
pnpm dev

# Watch mode with HMR (Firefox)
pnpm dev:firefox

# Production build (Chrome)
pnpm build

# Production build (Firefox)
pnpm build:firefox

# Type checking
pnpm type-check

# Linting
pnpm lint
pnpm lint:fix

# Code formatting
pnpm format

# Package as zip
pnpm zip
pnpm zip:firefox

# End-to-end tests
pnpm e2e
pnpm e2e:firefox

# Module management
pnpm module-manager              # Interactive menu
pnpm module-manager -d popup     # Delete popup page

# Install dependencies
pnpm i <package> -w                    # Root workspace
pnpm i <package> -F <module-name>      # Specific package
```

### Loading into Browser

#### Chrome

1. Run `pnpm dev` (or `pnpm build` for production)
2. Open `chrome://extensions`
3. Enable "Developer mode" (top right)
4. Click "Load unpacked"
5. Select the `dist/` directory

#### Firefox

1. Run `pnpm dev:firefox` (or `pnpm build:firefox` for production)
2. Open `about:debugging#/runtime/this-firefox`
3. Click "Load Temporary Add-on..."
4. Select `./dist/manifest.json`
5. Note: Firefox unloads on browser close; reload each session

### Environment Variables

**File Location:** `.env` at repository root

**Prefixes:**
- `CEB_` for `.env` file variables
- `CLI_CEB_` for CLI-only variables

**Example:**
```env
CEB_API_KEY=your_api_key
CEB_ENABLE_FEATURE=true
```

**Access in Code:**
```typescript
import { IS_DEV, IS_PROD } from '@extension/env'
// or
const apiKey = process.env['CEB_API_KEY']
```

**Set via CLI:**
```bash
pnpm set-global-env CEB_MY_VAR=value
```

---

## Project Structure Summary

```
react-instagram-unfollow-chrome-extension/
├── chrome-extension/          # Extension config & background worker
├── pages/                     # UI entry points
│   ├── popup/                # Popup UI
│   ├── options/              # Settings page
│   ├── new-tab/              # New tab override
│   ├── side-panel/           # Side panel UI
│   ├── devtools/             # DevTools extension
│   ├── devtools-panel/       # DevTools panel
│   ├── content/              # Content scripts (JS)
│   ├── content-ui/           # Content scripts (React)
│   └── content-runtime/      # Advanced content scripts
├── packages/                 # Shared libraries
│   ├── storage/              # Reactive storage
│   ├── shared/               # Common code & hooks
│   ├── env/                  # Environment variables
│   ├── vite-config/          # Shared Vite config
│   ├── i18n/                 # Internationalization
│   ├── ui/                   # UI components
│   ├── hmr/                  # Hot module reload
│   ├── dev-utils/            # Dev utilities
│   ├── tailwindcss-config/   # Tailwind config
│   ├── tsconfig/             # TypeScript config
│   ├── module-manager/       # Page management CLI
│   └── zipper/               # Packaging tool
├── tests/                    # End-to-end tests
│   └── e2e/                  # WebdriverIO tests
├── bash-scripts/             # Utility scripts
├── dist/                     # Build output (generated)
├── dist-zip/                 # Zipped extension (generated)
└── Configuration files       # ESLint, Prettier, etc.
```

---

## Troubleshooting

### HMR Frozen

If saving files doesn't trigger reload:

1. Stop dev server: `Ctrl+C`
2. Kill turbo process if needed: `pkill -f turbo`
3. Restart: `pnpm dev`

### Imports Not Resolving (WSL)

Use VS Code Remote - WSL extension for proper path resolution.

### Module Compilation Issues

Ensure TypeScript version matches:
- VS Code: Run `Typescript: Select Typescript version` → Use Workbench version
- Or: Use workspace's TypeScript: `/node_modules/.bin/tsc`

---

## Additional Resources

- [Chrome Extensions Documentation](https://developer.chrome.com/docs/extensions)
- [Vite Guide](https://vitejs.dev/)
- [Turborepo Documentation](https://turbo.build/repo/docs)
- [WebdriverIO](https://webdriver.io/)
- [Manifest V3](https://developer.chrome.com/docs/extensions/mv3/intro/)

---

**Project Version:** 0.5.0  
**License:** MIT  
**Repository:** github.com/Jonghakseo/chrome-extension-boilerplate-react-vite
