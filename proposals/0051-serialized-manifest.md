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

- Start Date: 2025-01-22
- Reference Issues: 
- Implementation PR: 
- Stage 2 Issue: https://github.com/withastro/roadmap/issues/1099
- Stage 3 PR:

# Summary

Expose common Astro config properties to users and integrations

# Example

> [!NOTE]
> The following example don't reflect the final naming of the module 

```js
import { trailingSlash, i18n, build } from 'astro:config/routing';

console.log(trailingSlash);
console.log(i18n.locales);
console.log(build.format);
```

```js
import { srcDir, build } from 'astro:config/paths';

console.log(srcDir);
console.log(build.client)
console.log(build.server)
```

# Background & Motivation

It happens quite often in userland to need the trailingSlash setting, or the base etc. Things that are available at import.meta.env and some that are not. The goal is to unify the way we expose this data.

Many integrations need this data and have to create virtual modules for this anwyays.

# Goals

- Have a unified way of exposing Astro config data.
- Expose information that could be directly by our users without the use of integrations.
- Expose information that can be used in client-side scripts.

# Non-Goals

- Expose non-serializable information such as integrations (they are functions).
- Expose all information coming from the configuration.

# Detailed Design

> [!IMPORTANT]
> The proposal will use some paths/names that might not be official, only to provide some scenarios or examples. Please refer to [#proposed-apis] to understand the final and proposed modules.  

The main idea is to provide a virtual module with sub-paths (e.g. `astro:config/client`, `astro:config/server`), each sub path will expose information that won't be available in the other sub paths, so information won't be repeated. These modules will be compiled into code, so a piece information e.g. `trailingSlash` should be available only once in order to avoid polluting the final bundle with repeated code.

Some sub-paths can't be used inside client-side code, because they could expose sensitive information, such as file system paths. Hence, we should divide the information based on where they could be used. We can't track if binding e.g. `config.srcDir` is used inside a client script, but we can track if a `import { srcDir } from astro:config/paths` is used inside a client side code. 

## Use the manifest

Internally, we already have this information, which is the `SSRManifest`, **not** the `AstroConfig`. The `AstroConfig` disappears after the build, instead we have
the `SSRManifest` that is created during the build, and passed to the Astro application even when it has on-demand pages.

The source code already does a very good job to serialise the information across pipelines (dev, SSG, SSR and container) and make sure that all of them contain the same information. Many of this information are computed from the user configuration.

## What to expose

As a rule of thumb, the information to expose should adhere to the following guidelines:
- Information that can be serialised. Functions, `Map`, `Set`, etc. shouldn't be exposed. As long as a field can go through `JSON.strigify`, it's a good candidate. 
- Information that can be used directly by our end-users. Integration developers have access to `AstroConfig`.
- Information that isn't present inside `AstroConfig`. As mentioned previously, the `SSRManifest` contains useful information that are often used by on-demand pages and adapters.
  An example is `SSRManifest.i18n.domainLookupTable`, which is used by adapters to understand the mapping of a domain to a locale.

These are guidelines, so exceptions are possible, as long as they have a good reason to be exposed. 

## Proposed APIs

Proposed name of the module: `astro:config`

Proposed submodules

### `astro:config/client`

Exported bindings:

  - `i18n`
  - `trailingSlash`
  - `base`
  - `build.format`
  - `site`
  - `redirects`[^1]

### `astro:config/server`

Exported bindings:
  - `srcDirc`
  - `cacheDir`
  - `outDir`
  - `build.client`
  - `build.server`
  - `build.serverEntry`
  - `build.assetsPrefix`
  - `publicDir`
  - `root`
  - `i18n`
  - `trailingSlash`
  - `base`
  - `build.format`
  - `site`
  - `redirects`[^1]


To note that `/routing`/`/client` can be used anywhere in the code, even in client scripts.

[^1]: To evaluate, probably not needed

# Testing Strategy

- The virtual modules will be "usable" via experimental flag. Attempting to use these virtual modules without the experimental flag turned on will result into a hard error.
- After a gracing period where we stabilise the APIs, the experimental flag will be removed.
- We will deprecate existing means to read this information. The following will be deprecated:
  - `import.meta.env.BASE_URL`
  - `import.meta.env.SITE`
  - `import.meta.env.ASSETS_PREFIX`

# Drawbacks

Exposing these kinds of information requires "drawing a line" for when something can be exposed to our end-users, instead of telling them to use an integration.

I believe it will be more difficult to come up with important and firm guidelines, but once we have them, maintainers will know when something is worth exposing to the users.

# Alternatives

Many of the information available to the user are related to routing e.g. `trailingSlash`, `build.format`, `i18n`, `base`, etc. 
so I also evaluated exposing *utility functions* to create links, however we don't know what are the use cases, and even if we did, there are many other information at play,
such as the pipeline (dev VS build VS on-demand pages), prerendering, server islands, and more. 

I feel that Astro should provide the primitives to integrations, so they can create utility functions based on their use cases.

# Adoption strategy

- The experimental flag `experimental.serializeConfig` will be shipped in a minor.
- Once the RFC is stable and approved, a new minor will remove the flag.

# Unresolved Questions

- The name of the module
- The name of the sub-paths
- The list of bindings to expose


