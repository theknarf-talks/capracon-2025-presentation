---
style: "dark"
---

# Building your own Static Site Generator with Vite & React Router

---

# Me

**Bredland, Frank Lyder** <br />
_Senior Software Developer @ Capra_

Re-inventor of wheels

https://theknarf.com/

---

# Goal

<Reveal currentId={1}>

- Create a Static Site Generator

</Reveal>
<Reveal currentId={2}>

- It should work with Github Pages

</Reveal>
<Reveal currentId={3}>

- Using Vite as bundler

</Reveal>
<Reveal currentId={4}>

- React Router as routing library

</Reveal>
<Reveal currentId={5}>

- Create html files for each route

</Reveal>
<Reveal currentId={6}>

- All in under **250 lines of code**!

</Reveal>

---

# First steps

Installing packages:

```bash
$ pnpm create vite my-blog --template react-swc-ts

$ pnpm add react-router

$ pnpm add -D vite-node stream-buffers
```

---

File structure:

```javascript
src/routes.tsx   // React Router routes for our app
src/main.tsx     // Normal main file
src/ssg-main.tsx // Main file for ssg
src/...          // the rest of the blog
ssg.tsx          // build script
ssg-for-vite.tsx // our plugin
```

---

`src/routes.tsx`:

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

`src/main.tsx`:
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

`src/ssg-main.tsx`:
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

# let's look at our build script

---

`ssg.tsx`:
```typescript
#!/usr/bin/env -S vite-node --script
import { build } from 'vite';
import ssgPlugin from './ssg-for-vite.tsx';

const viteConfig = (await import('./vite.config.ts')).default;

await build({
  ...viteConfig,
  plugins: [
    ssgPlugin({
      routes: (await import('./src/routes')).routes,
    }),
    ...(viteConfig.plugins),
  ],
});
```

---

`ssg.tsx`:
```typescript
#!/usr/bin/env -S vite-node --script
```

To run the script:

```bash
$ ./ssg.tsx
```

---

Structure of `ssg.tsx` build script:

```
┌──────────────┐     ┌───────┐                    
│ ./src/routes ├────►│       │  ┌────────────────┐
└──────────────┘     │ssg.tsx├─►│ssg-for-vite.tsx│
┌─────────────────┐  │       │  └────────────────┘
│./vite.config.ts ├─►│       │                    
└─────────────────┘  └───────┘                    
```

---

`ssg-for-vite.tsx`:
```typescript
function ssgPlugin({ routes, mainScript }) : Plugin {
  const paths = routesToPaths(routes);

  return {
    name: 'ssg-plugin',

    // Add all routes in the `paths` array as chunks
    buildStart() { /* .. */},

    // resolveId just matches up files that we later want to handle in load
    // without it we don't get to load the files in this plugin
    resolveId(id) { /* .. */ },

    // build page
    async load(id) { /* .. */ },
  };
}

export default ssgPlugin;
```

---

`ssg-for-vite.tsx`:
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

`ssg-for-vite.tsx`:
```typescript
function ssgPlugin({ routes, mainScript }) : Plugin {
  const paths = routesToPaths(routes);

  return {
    name: 'ssg-plugin',

    // Add all routes in the `paths` array as chunks
    buildStart() { /* .. */ },

    // resolveId just matches up files that we later want to handle in load
    // without it we don't get to load the files in this plugin
    resolveId(id) { /* .. */ },

    // build page
    async load(id) {
      // Lets look at this part
    },
  };
}

export default ssgPlugin;
```

---

`ssg-for-vite.tsx`:
```typescript
    // build page
    async load(id) {
      const path = paths.find(path => {
        const idFromPath = fileNameFromPath(path);					
        return idFromPath === id;
      });

      if(path) {
        return await renderHtml(path, routes);
      }

      return null;
    },
```

---

```typescript
const renderHtml = async (path, routes) => {
  const { query, dataRoutes } = createStaticHandler(routes);
  const url = new URL(path, 'http://localhost/');
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
      <script type="module" src="/ssg-main.tsx"></script>
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

--

https://theknarf.com/
