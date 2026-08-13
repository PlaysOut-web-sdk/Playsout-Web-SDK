# Playsout Web SDK Integration Guide

Playsout Web SDK lets HTML, Vue 3, and React applications embed the Playsout game list widget. It also provides SDK initialization, platform login, login state, user information, locale switching, and user points display.

This guide is organized so that integrators can select their project type and use the corresponding code directly.

## Key Points

- `appId` is not required.
- Game details use iframe mode by default, so `detailMode` does not need to be provided.
- Persistent SDK image caching is currently disabled. Images use their online URLs directly.
- `user-points` seeds the widget's frontend gem balance. The SDK does not fetch or persist the balance through a backend.
- In iframe detail mode, games can request simulated gem payment through the `privy-bridge` postMessage protocol.
- Vue 3 projects built with Vite must configure `isCustomElement`.
- React projects do not need Vue's `isCustomElement` configuration. Import `playsout-web-sdk/web-components` once instead.

## Installation

```bash
npm install playsout-web-sdk
```

## Choose an Integration

| Project type | Guide |
| --- | --- |
| HTML / IIFE | [HTML integration](#html-integration) |
| Vue 3 + Vite | [Vue 3 integration](#vue-3-integration) |
| React | [React integration](#react-integration) |

## Recommended Integration Flow

Use the following sequence when the application starts:

1. Initialize the SDK.
2. Read the current login state.
3. If the user is not logged in, call `Login()`.
4. Render `<playsout-widget>`.
5. Use `locale` to control the language and `user-points` to display the gem amount.

During initialization, the SDK checks `expiresAt` and `refreshExpiresAt`. It uses a valid access token directly, refreshes an expired access token when the refresh token is still valid, and clears the session when the refresh token has expired. Do not use `getUserInfo()` as a startup gate. Call it only when a page explicitly needs fresh backend user data, and handle that request error separately so the widget can still render.

## Login Parameters

Grab and Eros use the same public login method:

```js
await Playsout.Login(params);
```

Required parameters:

| Parameter | Description |
| --- | --- |
| `platform` | Platform identifier. Use `'grab'` or `'eros'`. |
| `platformUserId` | Unique user ID from the host platform. |
| `platformToken` | Random string. |
| `username` | User display name. |

Example:

```js
await Playsout.Login({
  platform: 'eros',
  platformUserId: '10',
  platformToken: 'random-string',
  username: 'TestUser',
});
```

`platformToken` is a required, non-empty random string.

## HTML Integration

Use this method for a page that does not use a Vue or React build setup.

Place the following code in `index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Playsout HTML Demo</title>
  </head>
  <body>
    <div id="game-container"></div>

    <script src="https://unpkg.com/playsout-web-sdk/index.iife.js"></script>
    <script>
      let loginPromise = null;

      function login() {
        if (!loginPromise) {
          loginPromise = window.Playsout.Login({
            platform: 'eros',
            platformUserId: '10',
            platformToken: 'random-string',
            username: 'TestUser'
          }).finally(function () {
            loginPromise = null;
          });
        }

        return loginPromise;
      }

      async function ensureLogin() {
        if (!window.Playsout.isLoggedIn) {
          return login();
        }
      }

      async function bootstrap() {
        await window.Playsout.init({ locale: 'en' });

        window.Playsout.on('authExpired', function () {
          login().catch(function (error) {
            console.error('Playsout re-login failed:', error);
          });
        });

        await ensureLogin();

        window.Playsout.mount('#game-container');

        document
          .querySelector('playsout-widget')
          ?.setAttribute('user-points', '1000');
      }

      bootstrap().catch(function (error) {
        console.error('Playsout bootstrap failed:', error);
      });
    </script>
  </body>
</html>
```

Common HTML / IIFE APIs:

```js
window.Playsout.init({ locale: 'en' });
window.Playsout.isLoggedIn;
window.Playsout.Login(params);
window.Playsout.getUserInfo();
window.Playsout.getUser();
window.Playsout.setLocale('zh');
window.Playsout.getLocale();
```

## Vue 3 Integration

### 1. Configure `vite.config.js`

Add this configuration to `vite.config.js` or `vite.config.ts` in the Vue project.

It tells the Vue compiler that `<playsout-widget>` is a native Web Component rather than a Vue component.

```js
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          isCustomElement: (tag) => tag.startsWith('playsout-'),
        },
      },
    }),
  ],
});
```

Without this configuration, Vue may report:

```text
Failed to resolve component: playsout-widget
```

### 2. Initialize the SDK in `src/main.js`

Place this code in the Vue entry file, usually `src/main.js` or `src/main.ts`.

```js
import { createApp } from 'vue';
import { createPlaysoutPlugin } from 'playsout-web-sdk/vue';
import 'playsout-web-sdk/web-components';
import App from './App.vue';

const app = createApp(App);

app.use(createPlaysoutPlugin({
  config: {
    locale: 'en',
  },
}));

app.mount('#app');
```

`createPlaysoutPlugin({ config })` automatically calls `init(config)` when the plugin is installed. Vue components normally should not call `init()` again.

### 3. Use the SDK in the game page

Place this code in the page component that displays the game list. In a new Vue project, it can be placed directly in `src/App.vue`.

```vue
<script setup>
import { watch } from 'vue';
import { usePlaysout } from 'playsout-web-sdk/vue';

const {
  isInitialized,
  isLoggedIn,
  locale,
  Login,
} = usePlaysout();

let loginPromise = null;

function login() {
  if (!loginPromise) {
    loginPromise = Login({
      platform: 'eros',
      platformUserId: '10',
      platformToken: 'random-string',
      username: 'TestUser',
    }).finally(() => {
      loginPromise = null;
    });
  }

  return loginPromise;
}

watch([isInitialized, isLoggedIn], ([initialized, loggedIn]) => {
  if (!initialized || loggedIn) return;

  login().catch((error) => {
    console.error('Playsout login flow failed:', error);
  });
}, { immediate: true });
</script>

<template>
  <playsout-widget
    :locale="locale"
    user-points="1000"
  />
</template>
```

Vue notes:

- The page waits for plugin initialization, then checks the login state and runs the login flow automatically.
- `locale` controls the widget language.
- For a fixed default language, `config: { locale: 'en' }` in the entry file is sufficient. If the application later needs dynamic locale switching, call `setLocale('zh')` and bind `locale`.
- `user-points="1000"` seeds the widget's frontend gem balance. Iframe game payments can deduct from this in-session balance only.
- If the application already has a game page, place the logic in that page component.

## React Integration

### 1. Initialize the SDK in `src/main.jsx`

Place this code in the React entry file, usually `src/main.jsx` or `src/main.tsx`.

React does not require `isCustomElement` configuration. Tags containing a hyphen, such as `<playsout-widget>`, are handled as Custom Elements.

```jsx
import { createRoot } from 'react-dom/client';
import { PlaysoutProvider } from 'playsout-web-sdk/react';
import 'playsout-web-sdk/web-components';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <PlaysoutProvider config={{ locale: 'en' }}>
    <App />
  </PlaysoutProvider>
);
```

`PlaysoutProvider` automatically calls `init(config)` after receiving `config`. Page components normally should not call `init()` again.

### 2. Use the SDK in the game page

Place this code in the page component that displays the game list. In a new React project, it can be placed directly in `src/App.jsx`.

```jsx
import { useEffect, useRef } from 'react';
import { usePlaysout } from 'playsout-web-sdk/react';

export default function App() {
  const loginInFlight = useRef(false);
  const {
    isInitialized,
    isLoggedIn,
    locale,
    Login,
  } = usePlaysout();

  useEffect(() => {
    if (!isInitialized || isLoggedIn || loginInFlight.current) return;
    loginInFlight.current = true;

    Login({
      platform: 'eros',
      platformUserId: '10',
      platformToken: 'random-string',
      username: 'TestUser',
    })
      .catch((error) => {
        console.error('Playsout login flow failed:', error);
      })
      .finally(() => {
        loginInFlight.current = false;
      });
  }, [isInitialized, isLoggedIn, Login]);

  return (
    <playsout-widget
      locale={locale}
      user-points="1000"
    />
  );
}
```

React notes:

- `PlaysoutProvider` initializes the SDK.
- After Provider initialization, the page checks the login state and runs the login flow once.
- `usePlaysout()` provides the login state, login method, user information method, and locale APIs.
- For a fixed default language, `<PlaysoutProvider config={{ locale: 'en' }}>` is sufficient. If the application later needs dynamic locale switching, call `setLocale('zh')` and bind `locale`.
- React does not need Vue's `isCustomElement` configuration.

## Common API Reference

| Capability | HTML / IIFE | Vue 3 | React |
| --- | --- | --- | --- |
| Initialize | `Playsout.init(config)` | `createPlaysoutPlugin({ config })` | `<PlaysoutProvider config={...}>` |
| Check login state | `Playsout.isLoggedIn` | `isLoggedIn.value` | `isLoggedIn` |
| Log in | `Playsout.Login(params)` | `Login(params)` | `Login(params)` |
| Fetch latest user information | `Playsout.getUserInfo()` | `Playsout.getUserInfo()` | `getUserInfo()` |
| Read locally stored user information | `Playsout.getUser()` | `Playsout.getUser()` | `user` |
| Change locale | `Playsout.setLocale('en')` | `setLocale('en')` | `setLocale('en')` |
| Read current locale | `Playsout.getLocale()` | `locale.value` | `locale` |
| Log out | `Playsout.logout()` | `logout()` | `logout()` |

## Supported Locales

```ts
'zh' | 'en' | 'ja' | 'ko' | 'vi' | 'th' | 'id' | 'ms'
```

## User Points and Simulated Payment

Pass the value through the `user-points` attribute:

```html
<playsout-widget user-points="1000"></playsout-widget>
```

The SDK currently does not fetch, persist, or recharge the user's gem amount through a backend. `user-points` is supplied by the host application and becomes the widget's in-session frontend balance.

When a game is opened in iframe detail mode, the widget listens for `privy-bridge` `postMessage` requests from the current iframe only. Supported methods are:

- `bridge.handshake`
- `auth.getUser`
- `pay.createOrder`
- `pay.request`
- `pay.query`

`pay.request` opens a localized confirmation dialog. Confirm deducts gems from the in-session balance and emits `gem-balance-change`; cancel and insufficient balance do not deduct gems.

## Authentication Expiration

The SDK handles an expired access token internally:

1. A protected API reports that the token is invalid.
2. The SDK automatically calls the refresh token endpoint.
3. After a successful refresh, the SDK stores the new token and retries the original request.

If the refresh token is also invalid, the SDK:

- Clears the local token.
- Changes the login state to logged out.
- Emits the `authExpired` event.

HTML / IIFE example:

```js
window.Playsout.on('authExpired', function () {
  // Get a new platform credential, then call Playsout.Login() again.
});
```

React and Vue adapters synchronize their reactive login state when `authExpired` is emitted. A watcher or effect that logs in whenever initialization is complete and `isLoggedIn` becomes false will therefore handle both startup and runtime expiration.

## Image Loading

The SDK currently uses online image URLs directly. Persistent SDK image caching is disabled by default, so the SDK does not proactively store images in persistent browser storage.
