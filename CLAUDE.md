# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a browser extension for Chrome, Firefox, and Edge that copies links, images, and tab information as Markdown code. The extension supports multiple browser platforms with separate manifests for Chrome (MV3), Firefox (MV2 and MV3), and Edge.

## Key Directories

- `/src/` - Shared TypeScript source code
  - `background.ts` - Main extension background script
  - `content-script.ts` - Content script for page interaction
  - `handlers/` - Message/command/context-menu handlers
  - `services/` - Browser-agnostic logic with browser adapters (injected via `createBrowser*` factory functions)
  - `ui/` - Popup/options page scripts
  - `contracts/` - Type definitions for messages and commands
  - `storage/` - Storage management
  - `lib/` - Core libraries (Markdown, settings, etc.)
- `/chrome/` - Chrome/Chromium specific files (MV3)
- `/firefox-mv2/` - Firefox Manifest V2 files
- `/firefox-mv3/` - Firefox Manifest V3 files
- `/test/` - Test files (unit, browser UI, E2E)
- `/e2e_test/` - Legacy Python E2E tests
- `/build/` - Build output directory
- `/dist/` - TypeScript compilation output
- `/fixtures/` - Test fixtures and QA HTML
- `/scripts/` - Build and development scripts

## Architecture Patterns

1. **Dependency Injection**: Services receive browser APIs as dependencies via `createBrowser*` factory functions for testability.
2. **Browser Abstraction**: Thin adapters wrap browser-specific APIs, allowing services to remain browser-agnostic.
3. **Clear Separation**: Handlers orchestrate user entry points, services contain pure logic, UI scripts handle presentation.
4. **Type Safety**: Comprehensive TypeScript contracts define messages and commands between components.

## Development Commands

### Building
- `npm run compile` - Compile TypeScript for all platforms
- `npm run build-chrome` - Build Chrome extension package (`build/chrome.zip`)
- `npm run build-firefox-mv2` - Build Firefox MV2 extension (XPI)
- `npm run build-firefox-mv3` - Build Firefox MV3 extension (XPI)

### Debugging (Auto-reload)
- `npm debug-chrome` - Debug Chrome with auto-reload
- `npm debug-firefox-mv3` - Debug Firefox MV3 with auto-reload
- `npm debug-firefox-mv2` - Debug Firefox MV2 with auto-reload
- `npm debug-edge` - Debug Edge with auto-reload

### Testing
- `npm test` - Run unit tests with Vitest
- `npm run test:unit` - Run only unit tests
- `npm run test:browser` - Run browser UI tests (Vitest + Playwright)
- `npm run test:e2e` - Run Playwright E2E tests (requires building test extension first)
- `npm run test:e2e:docker` - Run E2E tests in Docker (CI parity)
- `npm run test:all` - Run all tests (unit + E2E)

### Code Quality
- `npm run lint` - Run ESLint
- `npm run typecheck` - TypeScript type checking
- `npm run lint:fix` - Fix ESLint issues automatically

## Build Process

1. TypeScript compilation to `/dist/` (`npm run build:ts`)
2. Platform-specific compilation via `scripts/compile.js` (copies `/dist/` to platform directories)
3. Browser-specific builds:
   - Chrome: ZIP archive creation
   - Firefox: `web-ext build` for XPI packages

## Testing Strategy

### Unit Tests (`test/**/*.test.ts`)
- Vitest with Node.js environment
- Test browser-agnostic service logic
- Mock browser APIs via dependency injection

### Browser UI Tests (`test/ui/**/*.spec.ts`)
- Vitest with Playwright browser environment
- Test UI components in actual browser
- Configured in `vitest.config.ts` with separate project

### E2E Tests (`test/e2e/`)
- Playwright for extension testing
- Requires building test extension via `scripts/build-test-extension.js`
- Runs Chromium in headed mode with persistent profile

### Legacy Python E2E (`e2e_test/`)
- Older Python-based tests (consider migrating to Playwright)

## Development Workflow

1. **Setup**: `npm install` (requires Node.js >= 20.0)
2. **Development**: Use auto-reload debugging commands for rapid iteration
3. **Testing**: Run `npm test` during development, `npm run test:e2e` before commits
4. **Building**: Use platform-specific build commands for distribution

## Browser-Specific Considerations

### Chrome/Edge
- Uses Manifest V3 with service workers
- ZIP packaging for distribution

### Firefox
- Supports both Manifest V2 (legacy) and V3
- XPI packaging with `web-ext`
- Requires signing for release builds (AMO)
- Debugging unsigned XPIs requires Developer Edition or setting `xpinstall.signatures.required` flag

## CI/CD Pipeline

GitHub Actions workflow (`.github/workflows/nodejs.yml`):
1. **Build job**: Unit tests, linting, building all platform packages
2. **Playwright job**: Browser UI tests and E2E tests in Docker container

## Key Configuration Files

- `tsconfig.json` - TypeScript targeting ES2020 with browser extension types
- `eslint.config.js` - ESLint using @antfu/eslint-config with TypeScript support
- `vitest.config.ts` - Dual project setup (unit + browser tests)
- `playwright.config.ts` - Playwright configuration for extension testing
- `.node-version` - Node.js >= 20.0 requirement

## Manual Testing

Use `fixtures/qa.html` for manual QA testing. Open in browser and test extension functionality with various edge cases.

## Firefox XPI Signing

For Firefox release builds:
1. Get API keys from Firefox Add-On Developer Hub
2. Bump version in manifest (X.Y.Z format, no zero prefixes)
3. Run: `web-ext sign --channel=unlisted --api-key=... --api-secret=...`

Signed XPIs can be sideloaded on release Firefox builds.

