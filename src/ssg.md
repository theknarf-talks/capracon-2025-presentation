---
style: "dark"
---

# Building your own Static Site Generator with Vite & React Router

---

# Me

Frank Lyder Bredland <br />
_Senior Software Developer @ Capra Consulting_

Re-inventor of wheels

https://theknarf.com/

---

# Goal

- Create a Static Site Generator
- It should work with Github Pages
- Using Vite as bundler
- React Router as routing library
- Create html files for each route
- All in under **250 lines of code**!

---

# First steps

Installing packages:

```bash
$ pnpm create vite my-blog --template react-swc-ts

$ pnpm add react-router

$ pnpm add -D vite-node stream-buffers
```

---

`routes.tsx`:

```typescript
import App, { Home, Blog, Post  } from './app.tsx';
import { routes as blogRoutes } from './blog';

export const routes = [{
  path: '/',
  element: <App />,
  children: [
    { path: '/',      element: <Home /> },
    { path: 'post/',  element: <Blog /> },
    { path: 'post/*', element: <Post />,
      children: [...blogRoutes,],
    },
  ]
}]
```

---

`main.tsx`:
```typescript
import React from 'react';
import { createRoot } from 'react-dom/client';
import { createBrowserRouter, RouterProvider } from 'react-router';
import { routes } from './routes';

const router = createBrowserRouter(routes);
const domNode = document.getElementById('root')!;
const root = createRoot(domNode);

root.render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>,
);
```

---

`ssg-main.tsx`:
```typescript
import React from 'react';
import { hydrateRoot } from 'react-dom/client';
import { createBrowserRouter, RouterProvider } from 'react-router';
import { routes } from './routes';

const router = createBrowserRouter(routes);
const domNode = document.getElementById('root')!;

hydrateRoot(
  domNode,
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>,
);
```

---

`ssg.tsx`:
```typescript
#!/usr/bin/env -S VITE_NODE=true vite-node --script
import { build } from 'vite';
import ssgPlugin from './vite-react-router-ssg-plugin.tsx';

const viteConfig = (await import('./vite.config.ts')).default;
const ssgPluginConfig = {
  routes: (await import('./src/routes')).routes,
  mainScript: '/ssg-main.tsx',
}

await build({
  ...viteConfig,
  plugins: [
    ssgPlugin(ssgPluginConfig),
    ...(viteConfig.plugins),
  ],
});
```

---

`vite-react-router-ssg-plugin.tsx`:
```typescript
function ssgPlugin({ routes, mainScript }) : Plugin {
  const paths = routesToPaths(routes);

  return {
    name: 'ssg-plugin',

    // Add all routes in the `paths` array as chunks
    buildStart() {
    },
    // resolveId just matches up files that we later want to handle in load
    // without it we don't get to load the files in this plugin
    resolveId(id) {
    },
    // build page
    async load( id ) {
    },
  };
}

export default ssgPlugin;
```

---

`vite-react-router-ssg-plugin.tsx`:
```typescript
    // Add all routes in the `paths` array as chunks
    buildStart() {
      paths.forEach(path => {
        const id = fileNameFromPath(path);					
        this.emitFile({ type: 'chunk', id });
      });

    },

    // resolveId just matches up files that we later want to handle in load
    // without it we don't get to load the files in this plugin
    resolveId(id) {
      const file = paths.find(path => {
        const idFromPath = fileNameFromPath(path);					
        return idFromPath === id;
      });

      if(file)
        return id;
    },
```

---

Transform this:

```typescript
export const routes = [{
  path: '/',
  element: <App />,
  children: [
    { path: '/',      element: <Home /> },
    { path: 'post/',  element: <Blog /> },
    { path: 'post/*', element: <Post />,
      children: [...blogRoutes,],
    },
  ]
}]
```

into this:

```typescript
[
  { type: 'chunk', id: 'index.html' },
  { type: 'chunk', id: 'post/index.html' },
  { type: 'chunk', id: 'post/blog-post1/index.html' },
  { type: 'chunk', id: 'post/blog-post2/index.html' },
]
````

---

`vite-react-router-ssg-plugin.tsx`:
```typescript
function ssgPlugin({ routes, mainScript }) : Plugin {
  const paths = routesToPaths(routes);

  return {
    name: 'ssg-plugin',

    // Add all routes in the `paths` array as chunks
    buildStart() {
    },
    // resolveId just matches up files that we later want to handle in load
    // without it we don't get to load the files in this plugin
    resolveId(id) {
    },
    // build page
    async load( id ) {
    },
  };
}

export default ssgPlugin;
```

---

`vite-react-router-ssg-plugin.tsx`:
```typescript
    // build page
    async load( id ) {
      const path = paths.find(path => {
        const idFromPath = fileNameFromPath(path);					
        return idFromPath === id;
      });

      if(path) {
        return await renderHtml(path, routes, mainScript);
      }

      return null;
    },
```

---

```typescript
const renderHtml = async (path, routes, mainScript) => {
  const { query, dataRoutes } = createStaticHandler(routes);
  const url = new URL(path, 'http://localhost/')
  url.pathname = path;

  const context = await query(new Request(url.href, {
    signal: new AbortController().signal,
  }));
  const router = createStaticRouter(dataRoutes, context);

  const app = <html lang="en">
    <head>
      <meta charSet="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    </head>
    <body>
      <div id="root">
        <React.StrictMode>
          <StaticRouterProvider router={router} context={context} />
        </React.StrictMode>
      </div>
      <script type="module" src={mainScript}></script>
    </body>
  </html>;
```

---

```typescript
import { renderToPipeableStream } from 'react-dom/server';
import { finished } from 'node:stream/promises';
import { WritableStreamBuffer } from 'stream-buffers';
```

```typescript
  // continued from last slide
  const writableStream = new WritableStreamBuffer();
  const { pipe } = renderToPipeableStream(app, {
    onError(e) {
      throw e;
    },
    onAllReady() {
      pipe(writableStream);
    }
  });

  await finished(writableStream);
  return writableStream.getContentsAsString('utf8');
}
```

---

https://github.com/TheKnarf/theknarf.github.io

---

TODO

- project structure
- `./src/main.tsx` and `./src/main_ssg.tsx`
- The router file
- The Vite file
- Our own "ssg_for_vite"
  - explain why we need `vite-node`
  - go through each part of that code
  - explain what React provides
  - explain the trick of making multiple `index.html` file for Vite to build
