# URL Copying Design: Copy as Markdown Browser Extension

## Overview

This document details the architecture and implementation of how the Copy as Markdown browser extension copies URLs from the Chrome address bar (and other browsers) into Markdown format. The extension supports multiple copy sources: current tab URLs, links on pages, images, selected text, and tab lists.

## Architecture Components

### 1. Core Services (Browser-Agnostic Logic)

#### `LinkExportService` (`src/services/link-export-service.ts`)
- **Purpose**: Formats individual links into Markdown
- **Key Methods**:
  - `exportLink(options: LinkExportOptions): Promise<string>`
  - Supports two formats: `'link'` (standard Markdown link) and `'custom-format'` (user-defined templates)
- **Dependencies**:
  - `MarkdownFormatter` for link formatting
  - `CustomFormatsProvider` for custom templates

#### `TabExportService` (`src/services/tab-export-service.ts`)
- **Purpose**: Exports browser tabs (single tab or lists) as Markdown
- **Key Methods**:
  - `exportTabs(options: ExportTabsOptions): Promise<string>`
  - Supports formats: `'link'`, `'title'`, `'url'`, `'custom-format'`
  - List types: `'list'` (unordered), `'task-list'` (GitHub-flavored task lists)
- **Architecture**:
  - Uses pure functions for conversion (`convertBrowserTabsToTabs`, `groupTabsIntoLists`)
  - Separates data fetching from formatting via `TabDataFetcher` interface

#### `ClipboardService` (`src/services/clipboard-service.ts`)
- **Purpose**: Abstracts clipboard operations with multiple fallback strategies
- **Key Features**:
  - Three copy methods in order of preference:
    1. `navigator.clipboard.writeText()` (modern clipboard API)
    2. `document.execCommand('Copy')` via textarea (legacy)
    3. Iframe-based copying (for restrictive contexts)
  - Mock mode for testing
- **Dependency Injection**: Receives browser APIs (`ScriptingAPI`, `TabsAPI`, `ClipboardAPI`)

#### `SelectionConverterService` (`src/services/selection-converter-service.ts`)
- **Purpose**: Converts selected HTML to Markdown using Turndown library
- **Note**: Not directly used for URL copying but part of the extension's feature set

### 2. Handlers (User Entry Points)

#### `ContextMenuHandler` (`src/handlers/context-menu-handler.ts`)
- **Purpose**: Handles context menu click events
- **Key Logic**:
  - Routes based on `menuItemId` from `browser.contextMenus.OnClickData`
  - Extracts URL and title from different contexts:
    - `ContextMenuIds.Link`: Uses `info.linkUrl` and `info.selectionText`/`info.linkText`
    - `ContextMenuIds.CurrentTab`: Uses `tab.url` and `tab.title`
    - `ContextMenuIds.Image`: Uses `info.srcUrl`
- **Browser Differences**: Handles Firefox vs Chrome API variations

#### `KeyboardCommandHandler` (`src/handlers/keyboard-command-handler.ts`)
- **Purpose**: Handles keyboard shortcut commands
- **Similar routing logic** to context menu handler

#### `RuntimeMessageHandler` (`src/handlers/runtime-message-handler.ts`)
- **Purpose**: Handles messages from popup UI and other extension parts

### 3. Browser Abstraction Layer

#### Factory Functions (`createBrowser*`)
- **Pattern**: Each service has a `createBrowser*` factory that injects browser APIs
- **Examples**:
  - `createBrowserTabExportService()`: Injects `createBrowserTabDataFetcher()`
  - `createBrowserContextMenuHandler()`: Injects `browser.bookmarks` API
- **Benefits**:
  - Testability: Browser APIs can be mocked
  - Browser compatibility: Abstracts Chrome/Firefox/Edge differences

#### Browser Utilities (`src/services/browser-utils.ts`)
- Helper functions for common browser operations
- `mustGetCurrentTab()`, `requireWindowId()`, etc.

### 4. Core Libraries

#### `Markdown` (`src/lib/markdown.ts`)
- **Purpose**: Pure Markdown formatting library
- **Key Methods**:
  - `linkTo(title: string, url: string): string` - Creates `[title](url)`
  - `escapeLinkText(text: string): string` - Escapes Markdown special characters
  - `list(items: NestedArray): string` - Formats lists
  - `taskList(items: NestedArray): string` - Formats task lists
- **Configurable**: List styles, indentation, bracket escaping

#### `CustomFormat` (`src/lib/custom-format.ts`)
- **Purpose**: User-defined templates using Mustache.js
- **Slots**: `single-link`, `multiple-links` contexts

### 5. Content Script

#### `copy()` Function (`src/content-script.ts`)
- **Purpose**: Executes in page context to write to clipboard
- **Three-tier fallback strategy**:
  1. **Navigator API**: `navigator.clipboard.writeText()` with permission checks
  2. **Textarea Method**: `document.execCommand('Copy')` via temporary textarea
  3. **Iframe Method**: Creates isolated iframe for restrictive pages
- **Note**: Entire function must be serializable for `scripting.executeScript()`

## Data Flow for URL Copying

### Scenario: Copying a Link via Context Menu

```text
User Action → Browser API → Handler → Service → Clipboard → User
```

#### Step 1: User Initiates Copy
- User right-clicks on a link and selects "Copy as Markdown"
- Browser triggers `browser.contextMenus.onClicked` event

#### Step 2: Background Script Handling (`src/background.ts`)
```typescript
// Event listener in background.ts
browser.contextMenus.onClicked.addListener((info, tab) => {
  const text = await contextMenuHandler.handleMenuClick(info, tab);
  await clipboardService.copy(text, tab);
});
```

#### Step 3: Context Menu Handler Routing
```typescript
// In createContextMenuHandler()
if (menuItemId === ContextMenuIds.Link) {
  const linkText = info.selectionText || info.linkText || '';
  return services.linkExportService.exportLink({
    format: 'link',
    title: linkText,
    url: info.linkUrl || '',
  });
}
```

#### Step 4: Link Export Service Formatting
```typescript
// In LinkExportService.exportLink()
case 'link':
  return this.markdown.linkTo(options.title, options.url);
```

#### Step 5: Markdown Formatting
```typescript
// In Markdown.linkTo()
linkTo(title: string, url: string): string {
  let titleToUse: string;
  if (title === '') {
    titleToUse = Markdown.DefaultTitle(); // "(No Title)"
  } else {
    titleToUse = this.escapeLinkText(title);
  }
  return `[${titleToUse}](${url})`;
}
```

#### Step 6: Clipboard Service Copy
```typescript
// In createClipboardService()
async function copy_(text: string, tab?: browser.tabs.Tab): Promise<void> {
  if (clipboardAPI) {
    await clipboardAPI.writeText(text); // Direct API if available
    return;
  }

  // Otherwise use content script
  let targetTab = tab;
  if (!targetTab) {
    targetTab = await mustGetCurrentTab(tabsAPI);
  }

  await copyUsingContentScript(targetTab, text);
}
```

#### Step 7: Content Script Execution
```typescript
// In copy() function (content-script.ts)
const results = await scriptingAPI.executeScript({
  target: { tabId: tab.id },
  func: copy, // The entire copy function
  args: [text, iframeUrl],
});
```

#### Step 8: Clipboard Write (Content Script)
The `copy()` function attempts three methods:
1. `navigator.clipboard.writeText()` with permission check
2. `document.execCommand('Copy')` via temporary textarea
3. Iframe-based copying as last resort

### Scenario: Copying Current Tab URL

Similar flow but different data source:
- `ContextMenuIds.CurrentTab` handler uses `tab.url` and `tab.title`
- Same formatting and clipboard pipeline

## Key Implementation Details

### 1. Dependency Injection Pattern
```typescript
// Services receive dependencies via constructor
export class TabExportService {
  constructor(
    private markdown: Markdown,
    private customFormatsProvider: CustomFormatsProvider,
    private tabDataFetcher: TabDataFetcher,
  ) { }
}

// Browser-specific factories inject real browser APIs
export function createBrowserTabExportService(
  markdown: Markdown,
  customFormatsProvider: CustomFormatsProvider,
): TabExportService {
  return new TabExportService(
    markdown,
    customFormatsProvider,
    createBrowserTabDataFetcher(), // Injects browser.tabs API
  );
}
```

### 2. Pure Functions for Testability
- `convertBrowserTabsToTabs()`, `groupTabsIntoLists()`, `getFormatter()` are pure
- Easy to unit test without browser dependencies

### 3. Type Safety
- `contracts/` directory defines message and command types
- Discriminated unions for menu/item IDs
- Type guards like `isTabListMenuId()`

### 4. Error Handling
- Services throw descriptive `TypeError` for invalid options
- Clipboard service has comprehensive fallback strategy
- Content script catches and reports failures

## Manifest and Permissions

### Required Permissions
The extension declares minimal permissions in `manifest.json`:

```json
"permissions": [
  "activeTab",      // Access current tab for clipboard operations
  "alarms",         // Schedule context menu refreshes
  "contextMenus",   // Add "Copy as Markdown" to right-click menus
  "scripting",      // Inject content scripts for clipboard access
  "storage"         // Store user preferences and custom formats
],
"optional_permissions": [
  "tabGroups",      // Chrome-only tab group support
  "tabs"            // Access all tabs for tab list exports
]
```

### Permission Justification
- **activeTab**: Granted temporarily when user interacts with extension
- **clipboardWrite**: Implicit permission for `navigator.clipboard` (Firefox) or via `document.execCommand`
- **tabs**: Optional, only requested when user exports tab lists

### Manifest Differences
- **Chrome/Edge**: Manifest V3 with service worker background
- **Firefox MV2**: Background page with ES module loading
- **Firefox MV3**: Similar to Chrome but with Firefox-specific APIs

## Cross-Browser Considerations

### Manifest Versions
- **Chrome/Edge**: Manifest V3 (service workers)
- **Firefox**: Supports both MV2 (legacy) and MV3
- **Platform-specific directories**: `chrome/`, `firefox-mv2/`, `firefox-mv3/`

### API Differences
- **Clipboard Permissions**: Firefox doesn't support `clipboard-write` permission query
- **Context Menu Data**: `info.linkText` vs `info.selectionText` across browsers
- **Tab Groups**: Chrome-only feature, gracefully handled in Firefox

### Build System
- Separate compilation for each platform (`scripts/compile.js`)
- `web-ext` tool for Firefox packaging

## Testing Strategy

### Unit Tests (`test/**/*.test.ts`)
- Vitest with Node.js environment
- Mock browser APIs via dependency injection
- Test pure functions in isolation

### Browser UI Tests (`test/ui/**/*.spec.ts`)
- Vitest with Playwright browser environment
- Test UI components in actual browser

### E2E Tests (`test/e2e/`)
- Playwright for extension testing
- Tests complete user flows including clipboard operations
- Uses mock clipboard mode for verification

### Mock Clipboard Service
```typescript
// Enabled during testing
const clipboardService = createBrowserClipboardService(
  clipboardAPI,
  iframeUrl,
  true, // mockMode = true
);

// Tests can verify copy calls were made
const calls = await clipboardService.getCalls();
expect(calls[0].text).toBe('[Example](https://example.com)');
```

## Performance Considerations

### Lazy Initialization
- Services created on demand via factories
- Browser APIs only accessed when needed

### Efficient Data Flow
- Minimal data passing between components
- Pure functions avoid unnecessary side effects

### Clipboard Optimization
- Content script injection only when needed
- Permission checks cached by browser

## Security Considerations

### Content Script Isolation
- Copy function runs in isolated page context
- Iframe method for restrictive CSP environments

### Permission Model
- Minimal required permissions
- `clipboardWrite` permission declared in manifest
- Runtime permission checks where possible

### Input Sanitization
- `escapeLinkText()` prevents Markdown injection
- URL validation by browser APIs

## Extension Points

### Custom Formats
Users can define custom templates via options page:
```mustache
{{title}} - {{url}} ({{number}})
```

### New Export Formats
1. Add format to `ExportFormat` type union
2. Implement formatter in `getFormatter()`
3. Add handler routing logic

### Additional Copy Sources
1. Create new service following dependency injection pattern
2. Add handler for new user entry point
3. Register appropriate browser event listener

## Conclusion

The Copy as Markdown extension demonstrates a well-architected browser extension with:
- Clear separation of concerns (handlers, services, libraries)
- Dependency injection for testability and browser compatibility
- Comprehensive fallback strategies for clipboard operations
- Type-safe design with extensive contracts
- Multi-platform support through abstracted browser APIs

The URL copying flow efficiently transforms browser data into Markdown format while handling edge cases and browser differences gracefully.
