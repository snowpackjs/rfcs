<!--
  Note: You are probably looking for `stage-1--discussion-template.md`!
  This template is reserved for anyone championing an already-approved proposal.

  Community members who would like to propose an idea or feature should begin
  by creating a GitHub Discussion. See the repo README.md for more info.

  To use this template: create a new, empty file in the repo under `proposals/${ID}.md`.
  Replace `${ID}` with the official accepted proposal ID, found in the GitHub Issue
  of the accepted proposal.
-->

**If you have feedback and the feature is released as experimental, please leave it on the Stage 3 PR. Otherwise, comment on the Stage 2 issue (links below).**

- Start Date: 2024-11-15
- Reference Issues: <!-- related issues, otherwise leave empty -->
- Implementation PR: https://github.com/withastro/astro/pull/12441
- Stage 2 Issue: https://github.com/withastro/roadmap/issues/1050
- Stage 3 PR: https://github.com/withastro/roadmap/pull/1055
  <!-- related roadmap PR, leave it empty if you don't have a PR yet -->

# Summary

A first class `Astro.session` primitive, with pluggable storage backends. Inspired by PHP sessions and Rails sessions, this will allow on-demand rendered pages and API endpoints to use a new `Astro.session` object to access arbitrary data, scoped to that browser session. A session ID cookie is used to associate the browser with the session, but the actual data is stored on the backend.

# Example

Usage in an Astro component:

```astro
---
// src/components/CartButton.astro
export const prerender = false; // Not needed in 'server' mode
const cart = await Astro.session.get('cart');
---

<a href="/checkout">🛒 {cart?.length ?? 0} items</a>
```

Usage in an API endpoint:

```ts
// src/pages/api/addToCart.ts
import type { APIContext } from "astro";

export async function POST(req: Request, context: APIContext) {
  const cart = await context.session.get("cart");
  cart.push(req.body.item);
  await Astro.session.set("cart", cart);
  return Response.json(cart);
}
```

Usage in an Action:

```ts
// src/actions/addToCart.ts
import { defineAction } from "astro:actions";
import { z } from "astro:schema";

export const server = {
  addToCart: defineAction({
    input: z.object({ productId: z.string() }),
    handler: async (input, context) => {
      const cart = await context.session.get("cart");
      cart.push(input.productId);
      await context.session.set("cart", cart);
      return cart;
    },
  }),
};
```

# Background & Motivation

HTTP is a stateless protocol, so sharing data between requests is a problem that most server-rendered web apps need to solve. The standard way is using cookies, but these have limitations. Firstly, they are limited to 4kB in size, so can't be used for anything complex. The full data also needs to be sent along with every request and response, increasing their size. Finally, even when encrypted they can be vulnerable to replay attacks. We have encountered the limitations of cookie storage when building Astro Actions, where we need response data to persist between page redirects. It is easy to reach the 4kB limit in these cases.

For this reasons, most non-trivial apps need to implement some kind of server-side session handling. In most cases these work by storing a random session id in a cookie, and using that to retrieve the full session data on the server. Monolithic frameworks such as Rails and Laravel normally implement sessions at the framework or server level, and PHP has built-in session handling which heavily influenced this proposal. However it isn't a common primitive for modern JS frameworks, in part due to the fact that these need to work in serverless environments, so simple default backends such as filesystem storage cannot be relied upon. We propose addressing this by allowing adapters to provide default implementations, relying on the primitives that they have available.

# Goals

- Simple key/value API, similar to Astro cookies (`Astro.session.get('key')`, `Astro.session.set('key', 'value')`)
- Accessible in Astro components, actions, and API endpoints via Astro global and context objects.
- Additional control over the session via methods such as `Astro.session.regenerate()` and `Astro.session.destroy()`
- Storage drivers for popular providers available out of the box.
- Adapters can specify a default storage driver, with Node using filesystem storage by default.
- Other settings such as cookie name, session expiry etc are also optionally configurable.
- Sessions are lazy-loaded and auto-generated when first accessed. Session ID cookie is get and set automatically.

# Non-Goals

- Type-safety. This may be added later, but there are questions about the best way to do this, because of issues with the way that the config is loaded.
- User management, auth etc. This is just raw sessions, but they may be useful if someone is implementing auth
- Building or maintaining our own driver backends. We will rely on existing libraries for this.
- Strong consistency, atomic writes

# Detailed Design

The session object is a wrapper around pluggable storage backends. It is lazy-loaded: the contents of the session is fetched from the data store when `get()` is first called, and is `set()` at the end of the request. This means each request is treated as a single transaction, with read-your-writes consistency through the local object. Persistent storage is last-write-wins, with no attempt made to reconcile transactions between multiple requests. Each backend has its own consistency model for the persistent storage, and where this is configurable this will be exposed as part of the driver options.

## API

The session object is available in all Astro contexts, including components, actions, and API endpoints. In components, it is accessed via the global `Astro` object, and in actions and API endpoints it is available on the `context` object. The API is the same in all cases.

Values are serialized and deserialized using [devalue](https://github.com/Rich-Harris/devalue), which is the same library used by content layer and actions. This means that supported types are the same, and include strings, numbers, Dates, Maps, Sets, Arrays and plain objects.

### `Astro.session.get(key: string): Promise<any>`

Returns the value of the given key in the session. If the key does not exist, it returns `undefined`.

### `Astro.session.set(key: string, value: any, options?: { ttl: number }): void`

Sets the value of the given key in the session. The value can be any serializable type.

### `Astro.session.regenerate(): void`

Regenerates the session ID. Best practice is to call this when a user logs in or escalates their privileges, to prevent session fixation attacks.

### `Astro.session.destroy(): void`

Destroys the session, deleting the cookie and the object from the backaned. This should be called when a user logs out or their session is otherwise invalidated.

## Configuration

The session object is configured using the `session` key in the Astro config. The `driver` option is required, unless an adapter provides a default. The type for `options` depends on the driver, and may or may not be required. TypeScript types will show this to the user.

```js
// astro.config.ts
import { defineConfig } from "astro/config";
import { z } from "astro/schema";

export default defineConfig({
  // Sessions require an adapter to be used
  adapter: node({
    mode: "standalone",
  }),
  session: {
    // Required: the name of the Unstorage driver
    driver: "redis",
    // The required options depend on the driver
    options: {
      base: "session",
      host: process.env.REDIS_HOST,
      password: process.env.REDIS_PASSWORD,
    },
    // If set to a string, this will be used as the cookie name
    cookie: "my-session-id",
    // If set to an object, this will allow advanced options to be set
    // cookie: {
    //  name: "my-session-id"
    //  maxAge: 86400,
    //  sameSite: "Strict",
    //},
    // The default time-to-live for session entries, in seconds
    ttl: 86400,
  },
});
```

## Cookies

Sessions are generated when first accessed, and a session ID cookie is set in the response. It can be regenerated at any time with `Astro.session.regenerate()`, and destroyed with `Astro.session.destroy()`. The ID is a 36-character v4 UUID generated using [`crypto.randomUUID()`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) from the Web Crypto API. The session cookie options are configurable, with the defaults being `astro-session-id` for the name, and `HttpOnly` and `SameSite=Lax` for the flags. The expiry is set to 1 hour by default, but this is also configurable.

## Storage

The backend drivers are implemented using [Unstorage](https://unstorage.unjs.io/), which provides a storage abstraction with drivers for multiple providers.

The session is implemented as a single object stored in the backend, rather than individual entries, to avoid the need to make multiple requests to the backend for each read or write. This is a tradeoff between performance and consistency, but is a common pattern in session handling. The session is fetched in full or created when first accessed, and is written back in full at the end of the request.

When `session.get()` is called, the session ID is read from the cookie. If there is no session ID, a new one is generated. The full session data is then fetched as a single value from the backend and deserialized into using devalue. Future reads and writes within that request context are done on this local object. If any entries have an expiry time set, they are checked and deleted at this point if required.

When `session.set()` is called, if there is no session ID, a new one is generated. If a local session object has already been fetched, the value is set on that. If not, the data is set on a partial local session object. If there is a TTL specified, either as an argument to `set()` or in the session config, an expiry time is calculated and included as metadata, stored alongside the value. This is merged with the full session if `get()` is called later in the request. Otherwise this is done at the end of the request, and the merged object is serialized using devalue and written back to the backend.

## Adapter and integration support

Because of the variety of environments to which Astro can be deployed, there is no single approach to storage that can be relied upon in all cases. For this reason, adapters should provide default session storage drivers where possible. Sessions are only available in server-rendered contexts, so there will always be an adapter available. The Node adapter will use filesystem storage by default, but this is not suitable for serverless environments. For these, the adapter can default to any storage service that is available. For example, Netlify may use Netlify Blobs, Vercel may use Vercel KV or Upstash, Cloudflare may use Cloudflare Workers KV and Deno Deploy may use Deno KV. Integrations can also provide their own storage drivers, and these can be auto-configured by the integration.

This is done using the normal integration API, and should be handled in the [`astro:config:done` hook](https://docs.astro.build/en/reference/integrations-reference/#astroconfigdone). The adapter or integration is responsible for ensuring that they do not overwrite any user-defined driver configuration. Adapters may choose to accept their own configuration options which they can apply to the storage driver where needed. Adapters may provide a storage driver for use in development, or rely on the built-in node adapter which is provided by the dev server.

## Local development

A user may or may not wish to use the same driver in local development and prod. They may want to test a particular driver locally, for example. However in most cases users would probably just want to use a default local driver. An adapter

# Testing Strategy

The session object API can be tested using a mock driver. e2e tests can be used to test the dev server and cookie handling. Individual adapters can include tests for their default drivers.

# Drawbacks

- There is no universal fallback available for persistent storage, so adapters must provide their own default drivers. Vercel KV is no longer automatically available, so it does not have a baseline storage driver.
- Sessions can increase request latency, particularly if a slow storage backend is used.
- Allowing arbitrary drivers may be a footgun, as users may choose drivers that are unsuitable for session storage.

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

- Allow users to implement their own session handling using Astro.cookie and their own backends, such as unstorage.

# Adoption strategy

The initial release will be behind an experimental flag, and will be opt-in. It will be a non-breaking change when introduced, and will not execute any code unless the session object is accessed. We will implement automatic driver support in the node adapter using the filesystem driver, with other adapters added later.

# Unresolved Questions

How should edge middleware be handled?

# References

- [PHP Sessions](https://www.php.net/manual/en/intro.session.php)
- [Laravel sessions](https://laravel.com/docs/11.x/session)
- [How Rails sessions work](https://www.justinweiss.com/articles/how-rails-sessions-work/)
- [Securing Rails sessions](https://guides.rubyonrails.org/security.html#sessions)
