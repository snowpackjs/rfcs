**If you have feedback and the feature is released as experimental, please leave it on the Stage 3 PR. Otherwise, comment on the Stage 2 issue (links below).**

- Start Date: 2024-07-23
- Implementation PR: https://github.com/withastro/astro/pull/11360
- Stage 2 Issue: https://github.com/withastro/roadmap/issues/946
- Stage 3 PR: https://github.com/withastro/roadmap/pull/982

# Summary

- Explore a new and improved content layer for Astro.
- Improve the current experience of loading/defining data into content collections
- Improve the current experience of querying data from content collections

# Example

Collections are defined using a new `loader` property. There are built-in `file` and `glob` loaders, which load data and content from the filesystem, and users can define their own loaders.

```ts
// src/content/config.ts
import { defineCollection, z } from "astro:content";
import { glob, file } from "astro/loaders";

// The `glob()` loader loads multiple files, with one entry per file
const spacecraft = defineCollection({
  loader: glob({ pattern: "*.md", base: "src/data/spacecraft" }),
  // A schema is optional, but provides validation and type safety for data.
  // It can also be used to transform data before it is stored.
  schema: ({ image }) =>
    z.object({
      title: z.string(),
      description: z.string(),
      heroImage: image().optional(),
    }),
});

// The `file()` loader loads multiple entries from one file
const dogs = defineCollection({
  loader: file("src/data/dogs.json"),
  schema: z.object({
    id: z.string(),
    breed: z.string(),
    temperament: z.array(z.string()),
  }),
});

export const collections = { spacecraft, dogs };
```

This is then used in Astro pages in the same way as current content collections.

# Background & Motivation

Content collections are a key primitive that brings people to Astro. Content Collections make it easy to work with local content (MD, MDX, Markdoc, etc) inside of your Astro project. They give you structure (`src/content/[collection-name]/*`), schema validation for frontmatter, and querying APIs. However they are limited in a few ways:

- They can only be used with local data
- The data must be in a specific location in the project
- Large collections can be slow to load and use a lot of memory. They can also take up a lot of space when bundling for server deployment. This places an upper limit on the number of entries that can be practically included in a collection.

Content layer is designed to be a successor to content collections that addresses these limitations and opens up more use cases. It is inspired by the Gatsby data layer, but with a simpler API and no GraphQL or graph data store.

# Goals

- Create a successor to content collections that can be used with local and remote data.
- Allow data to be cached between builds.
- Improve performance and scalability by decoupling data from Vite.
- Provide a simple API for defining collections with a migration path from content collections.
- Support local files in user-defined locations with built-in file and glob loaders.
- Support Markdown, MDX and Markdoc rendering and JSON data for local files.
- Provide a flexible API for defining custom loaders.
- Make the implementation scalable to tens of thousands of entries.

# Non-Goals

- Allowing loaders to define multiple collections automatically. e.g. separate collections would need to be manually defined for posts and categories in a blog.
- Dependency tracing for entries.
- Hot-reloading remote data.
- Rendering Markdown from remote data. A loader could store rendered HTML, but it would be up to the loader to handle this.
- Custom `Content` components.
- Support for queries more complex than get by ID.

## Stretch Goals/Future Work

- SQLite-based backend for collections.
- Expressive query API.

# Detailed Design

## Glossary

- **Collection**: A set of entries that share a common schema. Each entry has a unique ID.
- **Entry**: A single piece of data in a collection.
- **Loader**: A function or object that loads data into a collection.
  - **Inline Loader**: A loader defined as a function that returns an array of entries which are then inserted into the store.
  - **Loader Object**: A loader defined as an object with a `load` method that loads data into the store.

## Collection Definition

Collections are defined in a similar way to current content collections, using `defineCollection()` in `src/content/config.ts`. There is a new `loader` property that defines how data is loaded into the collection. At its simplest, the loader can be a function that returns an array of entries.

```ts
const countries = defineCollection({
  loader: async () => {
    const response = await fetch("https://restcountries.com/v3.1/all");
    const data = await response.json();
    // Must return an array of entries with an id property, or an object with IDs as keys and entries as values
    return data.map((country) => ({
      id: country.cca3,
      ...country,
    }));
  },
});
```

The returned entries are stored in the collection, and can be queried using the `getCollection()` and `getEntry()` functions.

## Loaders

There are two ways to define a loader, and the choice depends on the complexity of the loader and the features it requires. The example above uses the high-level API, which is an async function that returns an array of entries. This is useful for loaders that don't need to manually control how the data is loaded into the store. Whenever the loader is called, it will clear the store and reload all the entries.

If a loader needs more control over the loading process, it can use the low-level API. This allows, for example, entries to be updated incrementally, or for the store to be cleared only when necessary. The low-level API is an object with a `load` method that is called to load data into the store. This is similar to an Astro integration or Vite plugin, and similarly the recommended pattern is to define a function that accepts configuration options and returns the loader object. This is the recommended pattern for loaders that are distributed as packages, as it provides the simplest API for users. For example, this is how a loader for an RSS feed might be used:

```ts
const podcasts = defineCollection({
  loader: feedLoader({
    url: "https://feeds.99percentinvisible.org/99percentinvisible",
  }),
});
```

The `feedLoader` function in this example receives a configuration object and returns a loader object, pre-configured with the URL of the feed. The loader object has a `load` method that is called to load data into the store.

### Loader API

A loader is an object with a `load` method that is called to load data into the store. The `load` method is an async function that receives a context object which includes a number of helper functions and objects. The loader can use these to load data, store it in the data store, and persist metadata between builds. The object can also define a schema for the data, which is used to validate and transform the data before it is stored. This can optionally be an async function that returns a schema, allowing the loader to introspect the data source to determine the schema or otherwise dynamically define it at load time.

This is an example of a loader for an RSS feed:

```ts
import type { Loader } from "astro/loaders";
import { ItemSchema, type Item } from "./schema.js";
import { parseFeed } from "./feed.js";

export interface FeedLoaderOptions {
  /** URL of the feed */
  url: URL | string;
}

export function feedLoader({ url }: FeedLoaderOptions): Loader {
  const feedUrl = new URL(url);
  // Return a loader object
  return {
    // The name of the loader. This is used in logs and error messages.
    name: "feed-loader",
    // The load method is called to load data
    load: async ({ store, logger, parseData, meta, generateDigest }) => {
      logger.info("Loading posts");

      // The meta store is used to store metadata, such as sync tokens
      // etags or last-modified times. It is persisted between builds.
      // In this case, we store the last-modified time of the feed, so we
      // can make a conditional request for the data.
      const lastModified = meta.get("last-modified");

      // Make a conditional request for the feed
      const headers = lastModified ? { "If-Modified-Since": lastModified } : {};

      const res = await fetch(feedUrl, { headers });

      // If the feed hasn't changed, you do not need to update the store
      if (res.status === 304) {
        logger.info("Feed not modified, skipping");
        return;
      }
      if (!res.ok || !res.body) {
        throw new Error(`Failed to fetch feed: ${res.statusText}`);
      }

      // Store the last-modified header in the meta store so we can
      // send it with the next request
      meta.set("last-modified", res.headers.get("last-modified"));

      const feed = parseFeed(res.body);

      // If the loader doesn't handle incremental updates, clear the store before inserting new entries
      // In some cases the API might send a stream of updates, in which case you would not want to clear the store
      // and instead add, delete or update entries as needed.
      store.clear();

      for (const item of feed.items) {
        // The parseData helper uses the schema to validate and transform data
        const data = await parseData({
          id: item.guid,
          data: item,
        });

        // The generateDigest helper lets you generate a digest based on the content. This is an optional
        // optimization. When inserting data into the store, if the digest is provided then the store will
        // check if the content has changed before updating the entry. This will avoid triggering a rebuild
        // in development if the content has not changed.
        const digest = generateDigest(data);

        store.set({
          id,
          data,
          // If the data source provides HTML, it can be set in the `rendered` property
          // This will allow users to use the `<Content />` component in their pages to render the HTML.
          rendered: {
            html: data.description ?? "",
          },
          digest,
        });
      }
    },
    // A loader can optionally provide its own Zod schema. This can be static, or it can be an async function
    // that returns a schema. This allows an API to use introspection to determine the schema.
    schema: ItemSchema,
  };
}
```

### The data store

Each loader is provided with a data store object. This is an in-memory key/value store, scoped to that loader and is used to store collection entries. The store is persisted to disk between builds, so loaders can handle incremental updates. The store has the following interface:

```ts
export interface DataStore {
  get: (key: string) => DataEntry | undefined;
  entries: () => Array<[id: string, DataEntry]>;
  /**
   * Sets an entry in the store. Returns true if the entry was added or updated,
   * or false if the entry was not changed.
   */
  set: (opts: {
    /** The ID of the entry. Must be unique per collection. */
    id: string;
    /** The data to store. Any JSON-serializable object */
    data: TData;
    /** The raw body of the content, if applicable. */
    body?: string;
    /** The file path of the content, if applicable. Relative to the site root. */
    filePath?: string;
    /** An optional content digest, to check if the content has changed. */
    digest?: number | string;
    /** The rendered content, if applicable. */
    rendered?: RenderedContent;
  }) => boolean;
  values: () => Array<DataEntry>;
  keys: () => Array<string>;
  delete: (key: string) => void;
  clear: () => void;
  has: (key: string) => boolean;
}
```

### The meta store

Each loader is provided with a meta store object. This is a key/value store, scoped to that loader and is used to store metadata. This data isn't available to pages, but is instead used to store information such as sync tokens, etags, or last-modified times. The meta store has the following methods:

```ts
export interface MetaStore {
  get: (key: string) => string | undefined;
  set: (key: string, value: string) => void;
  has: (key: string) => boolean;
  delete: (key: string) => void;
}
```

The `reference()` helper can be used in the same way as content collections, to reference other collections.

## Using the data

The data is accessed in the same way as content collections, using the `getCollection()` or `getEntry()` functions.

```astro
---
// src/pages/spacecraft/[id].astro
import type { GetStaticPaths } from "astro";
import { getCollection } from "astro:content";
import { Image } from "astro:assets";

export const getStaticPaths: GetStaticPaths = async () => {
  const collection = await getCollection("spacecraft");
  if (!collection) return [];
  return collection.map((craft) => ({
    params: {
      id: craft.id,
    },
    props: {
      craft,
    },
  }));
}

const { craft } = Astro.props;
---

<h1>{craft.data.title}</h1>
<p>{craft.data.description}</p>

```

### Rendered content

Some entry types may have HTML content that can be rendered as a component. While this can be accessed like any other property, a loader can also store the rendered HTML in the `rendered.html` property. This allows users to use the `<Content />` component to render the HTML. The `rendered` property can also include metadata such as frontmatter or headings, which can be accessed as properties on the `rendered.metadata` object:

```ts
// src/content/config.ts
store.set({
  id,
  data,
  rendered: {
    // A raw HTML string
    html: data.description ?? "",
    metadata: {
      // Optionally, arbitrary metadata such as headings can be stored here
      headings: data.headings ?? [],
    },
  },
  digest,
});
```

This can then be accessed in the page like this:

```astro
---
// src/pages/spacecraft/[id].astro
import type { GetStaticPaths } from "astro";
import { getCollection, render } from "astro:content";
import { Image } from "astro:assets";

export const getStaticPaths: GetStaticPaths = async () => {
  const collection = await getCollection("spacecraft");
  if (!collection) return [];
  return collection.map((craft) => ({
    params: {
      id: craft.id,
    },
    props: {
      craft,
    },
  }));
}

const { craft } = Astro.props;
// The `render()` helper can be used to render the HTML content of an entry. If an entry doesn't have rendered content, it will return an empty component.
const { Content, headings } = await render(craft);
---

<h1>{craft.data.title}</h1>

<Content />

```

## Built-in loaders

There are two built-in loaders: `file()` and `glob()`, which load data from the local filesystem. The `glob()` loader covers the current use case of directories full of Markdown, MDX, Markdoc or JSON content. The `glob()` helper is more flexible than in current content collections, as it can load data from anywhere on the filesystem. The `file()` loader loads multiple entries from a single file. Both loaders can process Markdown in the same way as content collections. They can also extract images in the same way as content collections.

```ts
const spacecraft = defineCollection({
  // The glob loader can be used for either Markdown or JSON, as well as MDX and Markdoc if the integrations are enabled.
  // The pattern is any valid glob pattern. It is relative to the "base" directory.
  // "base" is optional and defaults to the project root. It is defined relative to the project root, or as an absolute path.
  // By default the ID is a slug of the entry filename, relative to `base`. Alternatively, the ID can be customized by passing
  // a `generateId` function which receives the entry path and data and returns a string ID.
  loader: glob({ pattern: "*.md", base: "src/data/spacecraft" }),
  schema: ({ image }) =>
    z.object({
      title: z.string(),
      description: z.string(),
      heroImage: image().optional(),
    }),
});

const dogs = defineCollection({
  // The file loader loads a single file which contains multiple entries. The path is relative to the project root, or an absolute path.
  // The data must be an array of objects, each with a unique `id` property, or an object with IDs as keys and entries as values.
  loader: file("src/data/dogs.json"),
  schema: z.object({
    id: z.string(),
    breed: z.string(),
    temperament: z.array(z.string()),
  }),
});
```

## Integration support

Astro integrations can use `astro:server:setup` hook to reload data from the content layer. The `refreshContent` function is passed to the hook, which allows integrations to refresh the content layer during `astro dev`. Use cases include:

- Adding a sync button to the Astro toolbar
- Opening a web socket to a CMS that listens for updates during dev
- Creates a webhook URL that can be tunnelled to a public address for CMSs to trigger
- Allow services such as code sandboxes or hosted dev servers to automatically trigger reloads

### API

Adds a `refreshContent` function to the `astro:server:setup` hook options, with the following signature:

```ts
async refreshContent(options: {
   loaders?: Array<string>,
   context?: Record<string, any>
})
```

- `loaders`: an optional array of loader names. If set, only collections that use those loaders will be synced. This allows integrations to selectively sync their own content.
- `context`: an optional object with arbitrary data that is passed to the loader's `load` function as `refreshContextData`. This can be used to pass information such as events from a websocket or a webhook payload.

### Usage

An integration can use the `refreshContent` function to trigger a refresh of all collections, or just specific loaders, with optional context data. If the content layer is already loading, calls to `refreshContent` are queued and executed in series. The function returns a promise that resolves when that job has completed or throws if there is an error during sync.

This example shows an integration that creates a refresh webhook endpoint when running `astro dev`. Sending a `POST` request to the endpoint will trigger a refresh of the content for collections that use a specific loader:

```ts
 {
    name: 'my-integration',
    hooks: {
        'astro:server:setup': async ({ server, refreshContent }) => {
            // `server` is the Vite dev server instance
            server.middlewares.use('/_refresh', async (req, res) => {
                if(req.method !== 'POST') {
                  res.statusCode = 405
                  res.end('Method Not Allowed');
                  return
                }
                let body = '';
                req.on('data', chunk => {
                    body += chunk.toString();
                });
                req.on('end', async () => {
                    try {
                        const webhookBody = JSON.parse(body);
                        await refreshContent?.({
                          // The context can be any arbitrary object. We're calling it `webhookBody` here, but it could be anything.
                          context: { webhookBody },
                          // Only refresh collections that use the `my-loader` loader
                          loaders: ['my-loader']
                        });
                        res.writeHead(200, { 'Content-Type': 'application/json' });
                        res.end(JSON.stringify({ message: 'Content refreshed successfully' }));
                    } catch (error) {
                        res.writeHead(500, { 'Content-Type': 'application/json' });
                        res.end(JSON.stringify({ error: 'Failed to refresh content: ' + error.message }));
                    }
                });
            });
        }
    }
}
```

Then, inside the loader, the `refreshContextData` object can be accessed in the `load` function:

```ts
import type { Loader } from "astro/loaders";
export function myLoader(): Loader {
  return {
    name: "my-loader",
    load: async ({ store, logger, refreshContextData, meta }) => {
      if (refreshContextData?.webhookBody?.action) {
        logger.info("Received incoming webhook");
        // do something with the webhook body
      }
      // this is a normal sync...
    },
  };
}
```

# Testing Strategy

- Integration tests for the built-in loaders, covering Markdown and JSON data.
- Integration tests for image handling in Markdown.
- Integration tests for rendering components from Markdown.
- Integration tests for custom loaders, covering incremental updates and schema validation.
- Unit tests for the data store and meta store.

# Drawbacks

- A lot of the performance benefits will not be available for MDX, as that is code that is executed at runtime rather than content that can be pre-rendered and persisted in the store.
- The DX for loading data from APIs is already good, so it may be harder to show the benefits (mostly the ability to cache and query the data locally, and persist it between builds).

# Alternatives

- We could keep content collections with the current scope of local content, but add support for custom directories and multiple entries per file.

# Adoption strategy

- Experimental adoption via experimental flag in minor
- Unflagged in major release beta
- In future, implement existing content collections as a loader
