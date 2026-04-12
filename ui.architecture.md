## Architecture: Confi Manager Frontend

### Goals

**Component independence.** Each component is a self-contained unit with its own data fetching, state, rendering, and styling. It can run alone in a playground with zero external setup beyond a provider.

**No framework lock-in.** Components are Web Components (browser standard). They work in any HTML context — a vanilla page, a Vue app, a React app, a VS Code extension webview. The "app" is just HTML that composes components.

**Explicit over implicit.** Components don't rely on naming conventions, global CSS class agreements, or assumed environment. All configuration passes through providers with explicit attributes.

**Demoability.** Every component has its own runnable playground. You can develop and test a component without running the full app.

**Portability.** The same components run in the web SaaS app and in a VS Code extension webview. The only difference is the theme values passed to providers.

**Evolutionary resilience.** When you come back in 6 months and dislike something, you can replace any single component's internals without touching other components. You can replace the shell without touching any component. The blast radius of any change is bounded.

---

### Part 1: How to Design a Component

A component is a custom element built with Lit. It uses Shadow DOM for encapsulation. It consists of two pieces: a **provider** and the **component itself**.

#### The Provider

The provider is a Lit element that holds configuration for its component. It uses `@lit/context` to make values available to any descendant of that provider in the DOM tree.

```ts
import { LitElement, html } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import { provide } from '@lit/context';
import { createContext } from '@lit/context';

export interface ConfigEditorConfig {
  client: HttpClient;
  accentColor: string;
  bgColor: string;
}

export const configEditorContext = createContext<ConfigEditorConfig>('config-editor');

@customElement('config-editor-provider')
export class ConfigEditorProvider extends LitElement {
  @provide({ context: configEditorContext })
  @property({ type: Object })
  config: ConfigEditorConfig = {
    client: new HttpClient(''),
    accentColor: '#3b82f6',
    bgColor: '#ffffff',
  };

  render() { return html`<slot></slot>`; }
}
```

The provider defines **only what this component needs**. It knows nothing about global themes or other components. The names (`accentColor`, `bgColor`) are whatever makes sense for this component — no shared naming convention.

#### The Component

The component consumes its provider's context and does everything else: fetches data, manages state, renders UI.

```ts
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { consume } from '@lit/context';
import { configEditorContext, ConfigEditorConfig } from './config-editor-provider.js';

@customElement('config-editor')
export class ConfigEditor extends LitElement {
  @consume({ context: configEditorContext, subscribe: true })
  private config!: ConfigEditorConfig;

  @property({ attribute: 'config-key' }) configKey = '';
  @state() private data = '';
  @state() private loading = true;

  static styles = css`
    :host { display: block; }
    textarea {
      width: 100%;
      min-height: 16rem;
      font-family: monospace;
      padding: 0.5rem;
      border: 1px solid #e4e4e7;
      border-radius: 0.375rem;
    }
    button {
      margin-top: 0.5rem;
      padding: 0.5rem 1rem;
      border: none;
      border-radius: 0.375rem;
      color: white;
      cursor: pointer;
    }
  `;

  async connectedCallback() {
    super.connectedCallback();
    this.data = await this.config.client.getText(`/configs/${this.configKey}`);
    this.loading = false;
  }

  private async save() {
    await this.config.client.putJson(`/configs/${this.configKey}`, this.data);
  }

  render() {
    if (this.loading) return html`<p>Loading...</p>`;
    return html`
      <textarea
        .value=${this.data}
        @input=${(e: Event) => this.data = (e.target as HTMLTextAreaElement).value}
      ></textarea>
      <button
        style="background:${this.config.accentColor}"
        @click=${this.save}
      >Save</button>
    `;
  }
}
```

Styles live inside the component (Shadow DOM's `static styles`). Dynamic values that come from the theme (like `accentColor`) come through the provider, applied via inline `style` bindings.

#### The Playground

Each component has an `index.html` at its package root. This is the development entry point.

```html
<!DOCTYPE html>
<html>
<head>
  <title>Config Editor Playground</title>
</head>
<body>
  <config-editor-provider>
    <config-editor config-key="demo-settings"></config-editor>
  </config-editor-provider>

  <script type="module">
    import './config-editor.ts';
    import './config-editor-provider.ts';
    import { HttpClient } from './http-client.ts';

    const provider = document.querySelector('config-editor-provider');
    provider.config = {
      client: new HttpClient('http://localhost:5000'),
      accentColor: '#3b82f6',
      bgColor: '#fff',
    };
  </script>
</body>
</html>
```

Run with `npx vite`. The component is fully functional in isolation. No app shell, no router, no other components. The playground creates its own `HttpClient` — no auth hooks needed for local development.

#### HTTP Client

Components need to make API calls. Auth tokens, base URLs, and other request-level concerns are cross-cutting — components shouldn't manage them directly.

The approach: an `HttpClient` class wrapping `ky` (~3KB, built on native `fetch`, supports hooks for auth injection). Each component receives an `HttpClient` instance through its provider. The app shell creates the client with auth configuration; the component just calls `getJson` / `putJson`.

Why `ky`: small, `fetch`-based (not legacy `XMLHttpRequest` like axios), supports before/after hooks (equivalent to C#'s `HttpClientHandler`), actively maintained. If `ky` is ever replaced, only the `HttpClient` internals change — components call the same methods.

```ts
import ky, { type KyInstance, type Options } from 'ky';

export class HttpClient {
  private client: KyInstance;

  constructor(baseUrl: string, options?: Options) {
    this.client = ky.create({ prefixUrl: baseUrl, ...options });
  }

  get(path: string): Promise<Response> { return this.client.get(path); }
  getJson<T>(path: string): Promise<T> { return this.client.get(path).json<T>(); }
  getText(path: string): Promise<string> { return this.client.get(path).text(); }
  putJson<T>(path: string, body: unknown): Promise<T> { return this.client.put(path, { json: body }).json<T>(); }
  postJson<T>(path: string, body: unknown): Promise<T> { return this.client.post(path, { json: body }).json<T>(); }
  delete(path: string): Promise<void> { return this.client.delete(path).then(() => {}); }
}
```

The component's provider carries the client instead of a raw API URL:

```ts
export interface ConfigEditorConfig {
  client: HttpClient;
  accentColor: string;
  bgColor: string;
}
```

The component uses it without auth awareness:

```ts
const data = await this.config.client.getText(`/configs/${this.configKey}`);
await this.config.client.putJson(`/configs/${this.configKey}`, data);
```

Auth is injected at the app level when creating the client — components never see tokens:

```ts
const client = new HttpClient('https://api.confi.app', {
  hooks: {
    beforeRequest: [
      request => request.headers.set('Authorization', `Bearer ${getToken()}`),
    ],
  },
});
```

In theory, each component could use a different `HttpClient` implementation — as long as it has the same method signatures. The provider is the injection point. You could start with `ky`-based, swap to a custom `fetch` wrapper later, or use a mock client in playgrounds. Components don't change.

#### How a Component Stays Independent

The component never imports or references anything from the app or from other components. Its dependency graph:

```mermaid
graph TD
    A[config-editor] --> B[lit]
    A --> C[@lit/context]
    A --> D[config-editor-provider]
    A --> E[HttpClient interface]
    D --> B
    D --> C
```

No path leads to the app, the router, global CSS, other components, or Momentum Design. If you delete every other package and the entire app shell, this component still works in its playground.

---

### Part 2: How to Build the App

The app is the thin layer that composes components, maps a global theme to individual providers, and sets up routing.

#### Routing

Framework-free, using `@vaadin/router`:

```ts
import { Router } from '@vaadin/router';

const router = new Router(document.getElementById('outlet'));
router.setRoutes([
  { path: '/editor/:key', component: 'config-editor' },
  { path: '/versions/:key', component: 'version-history' },
  { path: '/', redirect: '/editor/default' },
]);
```

Each route maps to a custom element tag name. The router creates the element and puts it in the outlet. No framework needed.

#### Theme → Provider Mapping

The app defines global theme values and maps them into each component's provider. This is the **only place** that knows about both the global theme and component-specific token names.

```html
<style>
  :root {
    --brand-bg: #ffffff;
    --brand-fg: #18181b;
    --brand-primary: #3b82f6;
    --brand-danger: #ef4444;
    --brand-radius: 0.375rem;
  }
</style>

<!-- HttpClient is created once in the app setup script with auth hooks -->
<!-- Then passed into each provider's config object -->

<config-editor-provider>
<version-history-provider>

  <div style="display:grid; grid-template-columns:250px 1fr; height:100vh">
    <aside>
      <config-nav></config-nav>
    </aside>
    <main id="outlet"></main>
  </div>

</version-history-provider>
</config-editor-provider>
```

The providers receive their config objects (including the shared `HttpClient`) from the app's setup script:

```ts
import { HttpClient } from './http-client.js';

const client = new HttpClient('https://api.confi.app', {
  hooks: {
    beforeRequest: [
      request => request.headers.set('Authorization', `Bearer ${getToken()}`),
    ],
  },
});

const editorProvider = document.querySelector('config-editor-provider');
editorProvider.config = {
  client,
  accentColor: getComputedStyle(document.documentElement).getPropertyValue('--brand-primary'),
  bgColor: getComputedStyle(document.documentElement).getPropertyValue('--brand-bg'),
};
```

Adding a new component means: add its provider to this nesting, map global tokens and the client to its specific props, add a route.

#### Using Momentum Design Components

Momentum components are used directly alongside your own. They share the same tech (Lit, Shadow DOM, `@lit/context`). Momentum's `<mdc-themeprovider>` wraps the app to provide Momentum's theme tokens to Momentum primitives (buttons, inputs, dialogs):

```html
<mdc-themeprovider themeclass="mds-theme-stable-lightWebex">
  <!-- your providers nest inside -->
  <config-editor-provider>
    <!-- Momentum components (mdc-button, mdc-input) pick up the Momentum theme automatically -->
    <!-- Your components pick up config from your providers -->
    <main id="outlet"></main>
  </config-editor-provider>
</mdc-themeprovider>
```

Your components can use Momentum primitives internally:

```ts
import '@nicholasgasior/momentum-design/dist/components/button';

render() {
  return html`
    <textarea .value=${this.data} @input=${...}></textarea>
    <mdc-button @click=${this.save}>Save</mdc-button>
  `;
}
```

This does create a dependency on Momentum for that component. If you want maximum independence, use plain `<button>` and style it yourself. If you want to move fast and look good, use `<mdc-button>`. This is a per-component decision, not an architectural one.

#### VS Code Extension

The webview loads the same components. The only differences:

- `theme.css` maps `--vscode-*` variables (provided by VS Code) to your `--brand-*` variables
- No router (the extension opens specific views directly)
- The extension host (`extension.ts`) just creates the webview panel — ~20 lines

```css
/* vscode-theme.css */
:root {
  --brand-bg: var(--vscode-editor-background);
  --brand-fg: var(--vscode-editor-foreground);
  --brand-primary: var(--vscode-button-background);
  --brand-danger: var(--vscode-errorForeground);
}
```

```html
<!-- webview index.html -->
<link rel="stylesheet" href="${themeUri}">
<config-editor-provider>
  <config-editor config-key="${configKey}"></config-editor>
</config-editor-provider>
<script type="module" src="${componentsUri}"></script>
```

Same components. Different theme source. No code changes in the components themselves.

#### What the App Is NOT

The app does not contain business logic, data fetching, state management, or component styling. Those live in the components. The app is:

- A set of `<style>` definitions for global brand tokens
- A nesting of providers that map brand tokens to component-specific config
- A router that puts the right component in the outlet
- Layout CSS (grid/flexbox)

If you rewrote the entire app shell from scratch, it would take a day. The components are untouched.