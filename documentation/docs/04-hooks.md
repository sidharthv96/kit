---
title: Hooks
---

An optional `src/hooks.js` (or `src/hooks.ts`, or `src/hooks/index.js`) file exports four functions, all optional, that run on the server — **handle**, **handleError**, **getSession**, and **externalFetch**.

> The location of this file can be [configured](#configuration) as `config.kit.files.hooks`

### handle

This function runs every time SvelteKit receives a request — whether that happens while the app is running, or during [prerendering](#ssr-and-javascript-prerender) — and determines the response. It receives the `request` object and a function called `resolve`, which invokes SvelteKit's router and generates a response (rendering a page, or invoking an endpoint) accordingly. This allows you to modify response headers or bodies, or bypass SvelteKit entirely (for implementing endpoints programmatically, for example).

> Requests for static assets — which includes pages that were already prerendered — are _not_ handled by SvelteKit.

If unimplemented, defaults to `({ request, resolve }) => resolve(request)`.

```ts
// Declaration types for Hooks
// * declarations that are not exported are for internal use

// type of string[] is only for set-cookie
// everything else must be a type of string
type ResponseHeaders = Record<string, string | string[]>;
type RequestHeaders = Record<string, string>;

export type RawBody = null | Uint8Array;
export interface IncomingRequest {
	method: string;
	host: string;
	path: string;
	query: URLSearchParams;
	headers: RequestHeaders;
	rawBody: RawBody;
}

type ParameterizedBody<Body = unknown> = Body extends FormData
	? ReadOnlyFormData
	: (string | RawBody | ReadOnlyFormData) & Body;
// ServerRequest is exported as Request
export interface ServerRequest<Locals = Record<string, any>, Body = unknown>
	extends IncomingRequest {
	params: Record<string, string>;
	body: ParameterizedBody<Body>;
	locals: Locals; // populated by hooks handle
}

type StrictBody = string | Uint8Array;
// ServerResponse is exported as Response
export interface ServerResponse {
	status: number;
	headers: ResponseHeaders;
	body?: StrictBody;
}

export interface Handle<Locals = Record<string, any>, Body = unknown> {
	(input: {
		request: ServerRequest<Locals, Body>;
		resolve(request: ServerRequest<Locals, Body>): ServerResponse | Promise<ServerResponse>;
	}): ServerResponse | Promise<ServerResponse>;
}
```

To add custom data to the request, which is passed to endpoints, populate the `request.locals` object, as shown below.

```js
/** @type {import('@sveltejs/kit').Handle} */
export async function handle({ request, resolve }) {
	request.locals.user = await getUserInformation(request.headers.cookie);

	const response = await resolve(request);

	return {
		...response,
		headers: {
			...response.headers,
			'x-custom-header': 'potato'
		}
	};
}
```

You can add call multiple `handle` functions with [the `sequence` helper function](#modules-sveltejs-kit-hooks).

### handleError

If an error is thrown during rendering, this function will be called with the `error` and the `request` that caused it. This allows you to send data to an error tracking service, or to customise the formatting before printing the error to the console.

During development, if an error occurs because of a syntax error in your Svelte code, a `frame` property will be appended highlighting the location of the error.

If unimplemented, SvelteKit will log the error with default formatting.

```ts
// Declaration types for handleError hook

export interface HandleError<Locals = Record<string, any>, Body = unknown> {
	(input: { error: Error & { frame?: string }; request: ServerRequest<Locals, Body> }): void;
}
```

```js
/** @type {import('@sveltejs/kit').HandleError} */
export async function handleError({ error, request }) {
	// example integration with https://sentry.io/
	Sentry.captureException(error, { request });
}
```

> `handleError` is only called in the case of an uncaught exception. It is not called when pages and endpoints explicitly respond with 4xx and 5xx status codes.

### getSession

This function takes the `request` object and returns a `session` object that is [accessible on the client](#modules-$app-stores) and therefore must be safe to expose to users. It runs whenever SvelteKit server-renders a page.

If unimplemented, session is `{}`.

```ts
// Declaration types for getSession hook

export interface GetSession<Locals = Record<string, any>, Body = unknown, Session = any> {
	(request: ServerRequest<Locals, Body>): Session | Promise<Session>;
}
```

```js
/** @type {import('@sveltejs/kit').GetSession} */
export function getSession(request) {
	return request.locals.user
		? {
				user: {
					// only include properties needed client-side —
					// exclude anything else attached to the user
					// like access tokens etc
					name: request.locals.user.name,
					email: request.locals.user.email,
					avatar: request.locals.user.avatar
				}
		  }
		: {};
}
```

> `session` must be serializable, which means it must not contain things like functions or custom classes, just built-in JavaScript data types

### externalFetch

This function allows you to modify (or replace) a `fetch` request for an external resource that happens inside a `load` function that runs on the server (or during pre-rendering).

For example, your `load` function might make a request to a public URL like `https://api.yourapp.com` when the user performs a client-side navigation to the respective page, but during SSR it might make sense to hit the API directly (bypassing whatever proxies and load balancers sit between it and the public internet).

```ts
// Declaration types for externalFetch hook

export interface ExternalFetch {
	(req: Request): Promise<Response>;
}
```

```js
/** @type {import('@sveltejs/kit').ExternalFetch} */
export async function externalFetch(request) {
	if (request.url.startsWith('https://api.yourapp.com/')) {
		// clone the original request, but change the URL
		request = new Request(
			request.url.replace('https://api.yourapp.com/', 'http://localhost:9999/'),
			request
		);
	}

	return fetch(request);
}
```
