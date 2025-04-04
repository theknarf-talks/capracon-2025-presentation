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
src/index.html   // Default index file in Vite
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

`index.html`:
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>TheKnarf</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/main.tsx"></script>
  </body>
</html>
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

`package.json`:

```json
{
  "scripts": {
    "build": "./ssg.tsx",
  }
}
```

To run the script:

```bash
$ pnpm build
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
import {
  createStaticHandler,
  createStaticRouter,
  StaticRouterProvider,
} from 'react-router';
```

```typescript
const renderHtml = async (path, routes) => {
  const { query, dataRoutes } = createStaticHandler(routes);
  const url = new URL(path, 'http://localhost/');
  url.pathname = path;

  const context = await query(new Request(url.href, {
    signal: new AbortController().signal,
  }));
  const router = createStaticRouter(dataRoutes, context);
  
  // continues on next slide
```

---

```typescript showLineNumbers{11}
  // continued from last slide

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

  // continues on next slide
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

```bash
$ pnpm build

> @ build /Users/knarf/projects/theknarf/theknarf.github.io
> ./ssg.tsx

Running Vite build with SSG setup (Static site generation)

vite v5.3.1 building for production...
✓ 143 modules transformed.
../dist/post/index.html                                1.21 kB │ gzip:   0.64 kB
../dist/index.html                                     1.32 kB │ gzip:   0.70 kB
../dist/post/macos-gif/index.html                      5.23 kB │ gzip:   1.62 kB
../dist/post/post1/index.html                         12.52 kB │ gzip:   4.04 kB
../dist/assets/screen-recording-tool-CVLr4XD-.png  1,229.17 kB
../dist/assets/ssg-main-BYAw0POh.css                   4.05 kB │ gzip:   1.15 kB
../dist/assets/vendor-D8m-Dqkf.css                    14.91 kB │ gzip:   3.12 kB
../dist/assets/ssg-main-BhWi9ZoR.js                   57.19 kB │ gzip:  13.09 kB
../dist/assets/vendor-CzoGOkGm.js                    478.51 kB │ gzip: 149.43 kB
✓ built in 2.90s
```

---

# Conclusion

- Vite Runner to write `tsx` files in our plugin

- Uses React's built inn `renderToPipeableStream` to make HTML files

- Vite does the rest for us

- The finished build can be deployed on Github Page's

---

https://github.com/TheKnarf/theknarf.github.io

--

https://theknarf.com/
