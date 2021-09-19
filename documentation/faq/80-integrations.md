---
question: How do I use X with SvelteKit?
---

### How do I setup library X?

Please see the community site [sveltesociety.dev](https://sveltesociety.dev/templates) for examples of using many popular libraries like Tailwind, PostCSS, Firebase, GraphQL, mdsvex, and more. We recommend using [Svelte adders](https://sveltesociety.dev/templates#category-Svelte%20Add) which allow you to run a script to automatically add popular technologies to a newly created SvelteKit project.

### How do I use `svelte-preprocess`?

`svelte-preprocess` provides support for Babel, CoffeeScript, Less, PostCSS / SugarSS, Pug, scss/sass, Stylus, TypeScript, `global` styles, and replace. Adding [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) to your [`svelte.config.js`](#configuration) is the first step. It is provided by the template if you're using TypeScript. JavaScript users will need to add it. For many of the tools listed above, you will then only need to install the corresponding library such as `npm install -D sass`or `npm install -D less`. See the [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) docs for full details.

### How do I use Firebase?

Please use SDK v9 which provides a modular SDK approach that's currently in beta. The old versions are very difficult to get working especially with SSR and also resulted in a much larger client download size. Even with v9, most users need to set `kit.ssr: false` until [vite#4425](https://github.com/vitejs/vite/issues/4425) and [firebase-js-sdk#4846](https://github.com/firebase/firebase-js-sdk/issues/4846) are solved.

### How do I use a client-side only library that depends on `document` or `window`?

Vite will attempt to process all imported libraries and may fail when encountering a library that is not compatible with SSR. [This currently occurs even when SSR is disabled](https://github.com/sveltejs/kit/issues/754).

If you need access to the `document` or `window` variables or otherwise need it to run only on the client-side you can wrap it in a `browser` check:

```js
import { browser } from '$app/env';

if (browser) {
	// client-only code here
}
```

You can also run code in `onMount` if you'd like to run it after the component has been first rendered to the DOM:

```js
import { onMount } from 'svelte';

onMount(async () => {
	const { method } = await import('some-browser-only-library');
	method('hello world');
});
```

If the library you'd like to use is side-effect free you can also statically import it and it will be tree-shaken out in the server-side build where `onMount` will be automatically replaced with a no-op:

```js
import { onMount } from 'svelte';
import { method } from 'some-browser-only-library';

onMount(() => {
	method('hello world');
});
```

Otherwise, if the library has side effects and you'd still prefer to use static imports, check out [vite-plugin-iso-import](https://github.com/bluwy/vite-plugin-iso-import) to support the `?client` import suffix. The import will be stripped out in SSR builds. However, note that you will lose the ability to use VS Code Intellisense if you use this method.

```js
import { onMount } from 'svelte';
import { method } from 'some-browser-only-library?client';

onMount(() => {
	method('hello world');
});
```

### How do I setup a database?

Put the code to query your database in [endpoints](/docs#routing-endpoints) - don't query the database in .svelte files. You can create a `db.js` or similar that sets up a connection immediately and makes the client accessible throughout the app as a singleton. You can execute any one-time setup code in `hooks.js` and import your database helpers into any endpoint that needs them.

### How do I use middleware?

`adapter-node` builds a middleware that you can use with your own server for production mode. In dev, you can add middleware to Vite by using a Vite plugin. For example:

```js
const myPlugin = {
  name: 'log-request-middleware',
  configureServer(server) {
    server.middlewares.use((req, res, next) => {
      console.log(`Got request ${req.url}`);
      next();
    })
  }
}

/** @type {import('@sveltejs/kit').Config} */
const config = {
	kit: {
		target: '#svelte',
		vite: {
			plugins: [ myPlugin ]
		}
	}
};

export default config;
```

See [Vite's `configureServer` docs](https://vitejs.dev/guide/api-plugin.html#configureserver) for more details including how to control ordering.

### Does it work with Yarn 2?

Sort of. The Plug'n'Play feature, aka 'pnp', is broken (it deviates from the Node module resolution algorithm, and [doesn't yet work with native JavaScript modules](https://github.com/yarnpkg/berry/issues/638) which SvelteKit — along with an [increasing number of packages](https://blog.sindresorhus.com/get-ready-for-esm-aa53530b3f77) — uses). You can use `nodeLinker: 'node-modules'` in your [`.yarnrc.yml`](https://yarnpkg.com/configuration/yarnrc#nodeLinker) file to disable pnp, but it's probably easier to just use npm or [pnpm](https://pnpm.io/), which is similarly fast and efficient but without the compatibility headaches.
