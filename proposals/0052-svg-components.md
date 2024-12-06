**If you have feedback and the feature is released as experimental, please leave it on the Stage 3 PR. Otherwise, comment on the Stage 2 issue (links below).**

- Start Date: 2024-10-01
- Reference Issues: <!-- related issues, otherwise leave empty -->
- Implementation PR: https://github.com/withastro/astro/pull/12067
- Stage 2 Issue: https://github.com/withastro/roadmap/issues/699
- Stage 3 PR: https://github.com/withastro/roadmap/pull/1035

# Summary

This RFC proposes adding native support for importing and rendering `.svg` files as Astro components, with optimized rendering techniques to minimize performance impact. It aims to allow `.svg` files to be treated as components that accept props and be optimized for repeated use on a page using `<symbol>` and `<use>` elements.

# Example

An SVG can be imported directly into an Astro or MDX file and treat the default export as an Astro component.

```astro
---
import Logo from '../assets/logo.svg'
---

<Logo />
```

# Background & Motivation

Currently, using `.svg` files in Astro requires workarounds that can result in unnecessary complexity or bloated client-side JavaScript. As `.svg` files are widely used for icons and graphics, an optimized solution is necssary to simplify usage and improve performance.

Previously, `astro-icon` was introduced to experiment with `.svg` support but, with the relase of `astro:assets`, there's now an opportunity to provide optimized `.svg` imports that simplify the developer experience and reduce client-side bloat.

The core motivation is to automate best practices around `.svg` usage, enabled developers to effortlessly use and optimize their `.svg` assets in Astro projects.

# Goals

- Enable importing and rendering `.svg` files as Astro components.
- Enable importing and rendering `.svg` files in MDX files
- Allow users to pass props to the root `svg` element while allowing for prop overrides.
- Inline `.svg` file directly without using optimized sprite rendering behavior.
- Allow users to opt-in to optimized `.svg` rendering by utilizing `symbol` and `use` for repeated instances.
- Maintain backwards compatibility with existing `.svg` import behavior.
- Strip unnecessary attributes like `xmlns` and `version` from `.svg` files to reduce size.
- Allow third-party plugins to hook into the `.svg` import behavior to generate their own component libraries.

# Non-Goals

- Out-of-scope: Advanced tree-shaking or SSR optimizations for large icon sets.
- Out-of-scope: Advanced `.svg` file optimization such as `svgo`.
- Future: Adding support for `.svg` files as framework components.
- Non-goal: Adding support for `.json` icon sets such as `@iconify/json`.
- Non-goal: SVG content analysis for accessibility.

# Detailed Design

## Importing SVG Files as Components

The core design of this proposal allows `.svg` files to be imported and used as components in Astro. This means developers can import `.svg` files into their Astro components and render them directly as if they were Astro components. The goal is to maintain ease of use while automating performance optimizations under the hood.

**Example:**

```astro
---
import Logo from '../assets/logo.svg';
---

<Logo />
```

This would generate the following output:

```astro
<svg width="24" height="24" role="img">
    <!-- SVG Content -->
</svg>
```

## Render Methods

There are two ways that the SVG component can render the `.svg` file, inline or as a sprite.

### Inline

Each time a `.svg` file is imported and rendered in Astro, Astro will directly inject the optimized `.svg` file contents into the location with the root `<svg>` element properties overridden by the props passed to the SVG Component.

This follows the most common implementation of rendering SVGs into the HTML. This method is the safest option as it does the least manipulation to the `.svg` file. However, the tradeoff is that repeated usages of the same `.svg` file will result in multiple copies in the HTML.

**Example:**

```astro
---
import Logo from '../assets/logo.svg';
---

<Logo class="text-slate-500" />
```

This would generate the following output:

```astro
<svg width="24" height="24" role="img" class="text-slate-500" >
    <!-- SVG Content -->
</svg>
```

### Sprite

An SVG Sprite is when an `<svg>` element defines multiple SVGs (or symbols) using the `<symbol>` element and attaching a unique identifier to each SVG as an `id`. These are then able to be referenced using the `<use>` element with an `href` attribute that points to the `id` defined on the `<symbol>` element.

When a `.svg` file is first imported and rendered in Astro, the system will convert the file into a `<symbol>` element. This will be inserted into the `<svg>` element. This approach ensures that all subsequent renders of that `.svg` can use a `<use>` element, referencing the ID of the initial `<symbol>`.

This method was chosen by `astro-icon` to optimize SVG usage by default. It is riskier because it does more manipulation to the `.svg` file and relies on the `id` reference being available. The benefit to this method is that we now only have a single definition of repeated usages of the same `.svg` file and only minimal code to reference the definition in other locations.

**Example:**

```astro
---
import Logo from '../assets/logo.svg';
---

<!-- First Usage -->
<Logo class="text-gray-700" />

<!-- Second Usage -->
<Logo class="text-yellow-50" />

```

This would generate the following output:

```astro
<svg width="24" height="24" role="img" class="text-gray-700">
    <symbol id="a:0">
        <!-- SVG Content -->
    </symbol>
    <use href="#a:0" />
</svg>
<svg width="24" height="24" role="img"  class="text-yellow-50">
    <use href="#a:0" />
</svg>
```

## Config Options

To control the default rendering method for the SVG Components, a `mode` option will be added to the `svg` config flag. This will allow the values `inline` or `sprite` and provide the default rendering method for SVG Components.

**Example:**

```js
// astro.config.js
{
  //...
  experimental: {
    svg: {
      mode: "sprite";
    }
  }
  //...
}
```

## Prop Inheritance and Overriding

When a `.svg` is treated as a component, users can pass attributes such as `width`, `height`, `fill`, `stroke`, and other common SVG attributes. These attributes will automatically be applied to the root `<svg>` element unless they conflict with existing properties in the original `.svg` file, in which case the passed props will override the existing ones.

**Example:**

```astro
---
import Logo from '../assets/logo.svg'
---

<Logo />

<!-- Pass width and height to override default size -->
<Logo width={100} height={100} fill="blue" />
```

This would generate the following output:

```astro
<svg width="24" height="24" role="img">
    <!-- SVG Content -->
</svg>
<svg width="100" height="100" role="img" fill="blue">
    <!-- SVG Content -->
</svg>
```

### Rendering Mode

To override the default rendering method defined in the `astro.config.js` file, you can utilize the `mode` prop. This accepts the same values as the config (`inline` or `sprite`).

**Example:**

```astro
---
import ChevronRight from '../assets/chevron-right.svg';
---
<!-- Assuming config has `sprite` defined as default rendering mode -->
<ChevronRight />
<!-- This next usage will re-use the existing definition -->
<ChevronRight />
<!-- This usage will override the default and explicitly `inline` the SVG -->
<ChevronRight mode="inline" />
```

### Sizing

To simplify the process of setting the dimensions of SVG components, this proposal introduces a `size` prop that developers can use to uniformly scale the width and height of an SVG. This prop provides a convenient shorthand for situations where both width and height need to be set to the same value, offering a more concise API.

**Example:**

```astro
---
import Logo from '../assets/logo.svg';
---

<Logo />

<!-- Using the size prop to set both width and height -->
<Logo size={48} />
```

This would generate the following output:

```astro
<svg width="24" height="24" role="img">
    <!-- SVG Content -->
</svg>
<svg width="48" height="48" role="img">
    <!-- SVG Content -->
</svg>
```

Caution: This property will not apply if you have set a `height` and/or `width` prop.

**Example:**

```astro
---
import Logo from '../assets/logo.svg';
---
<Logo />
<Logo size={50} width={20} />
<Logo width={20} size={50} />
```

This would generate the following output:

```astro
<svg width="24" height="24" role="img">
    <!-- SVG Content -->
</svg>
<svg width="20" height="24" role="img">
    <!-- SVG Content -->
</svg>
<svg width="20" height="24" role="img">
    <!-- SVG Content -->
</svg>
```

## Accessibility Considerations

One key aspect of this design is ensuring that the SVG components generated from `.svg` files are accessible by default, following best practices for web accessibility. This includes the following considerations:

### ARIA Attributes

- **Role Attribute:** By default, Astro could set `role="img"` on SVGs when they are used for non-decorative purposes.
- **Title Element:** If the `.svg` file needs and accessible title/label, Astro can inject a `<title>` element.

**Example:**

```astro
---
import Logo from '../assets/logo.svg'
---

<!-- Pass an accessible title that will be injected  -->
<Logo title="Company Logo" />
```

This would generate the following output:

```astro
<svg width="24" height="24" role="img">
    <title>Company Logo</title>
    <!-- SVG Content -->
</svg>
```

While this proposal offers strong defaults for accessibility, it is essential to give developers full control. Any automatically generated accessibility attributes (aria-label, role, etc.) should be overridable by the developer at the component level, ensuring that specific use cases or custom behaviors can be supported.

# Testing Strategy

In addition to standard unit and integration tests, the testing strategy should include:

- **Cross-browser testing:** Ensuring that the Sprite pattern works consistently across all major browsers (e.g., Chrome, Firefox, Safari, Edge).
- **Regression testing:** Verifying that this optimization doesn't unintentionally break any existing `.svg` imports.
- **Performance testing:** Measuring the performance improvements in pages with many `<svg>` elements and comparing the impact of the new rendering system against inlining every SVG individually.

# Drawbacks

- **Increased Complexity for Debugging**: Debugging issues with SVG rendering could become more difficult due to the added abstraction of the Sprite approach. Developers may need additional documentation to understand how SVGs are being rendered and optimized, especially when diagnosing rendering problems in complex applications.
- **Limited Use Cases:** The performance benefits of the Sprite approach may be minimal for projects that don't heavily rely on SVGs. For smaller projects or those with limited SVG usage, the optimization might introduce unnecessary complexity without providing significant gains in performance.
- **Edge Cases with Advanced SVG Features:** Some advanced features of SVGs, like `<filter>` or `<mask>`, _may_ have issues when used within a `<symbol>` and referenced by `<use>`. These features will need thorough testing to ensure they function properly in the optimized model.

# Alternatives

- **Status Quo:** Leave developers to their own decision for handling SVGs which has typically been either embedding into Astro components or reaching for [`astro-icon`](https://github.com/natemoo-re/astro-icon) / [`unplugin-icons`](https://github.com/unplugin/unplugin-icons).
- **SVG Spritesheet Plugin:** Leave SVG handling to userland entirely and provide an Astro plugin that automates SVG Sprite generation. This plugin could compile multiple SVGs into a single spritesheet at build time, allowing developers to manually include optimized SVGs where needed.
- **Partial Implementation:** If full integration into Astro's core is deemed too complex, a partial implementation could involve adding support for importing `.svg` files as components without the full Sprite optimization. This would allow developers to use `.svg` files more easily, while leaving optimization to future versions or third-party tools.

# Adoption strategy

To ensure that developers can easily adopt this feature:

- **Documentation:** The Astro documentation will be updated with detailed guides and examples of how to use `.svg` imports and leverage the new optimization strategies. Special focus will be placed on explaining the benefits of the Sprite pattern for performance-conscious developers.
  - **Background:** Documentation should be included to explain when to choose the SVG format over alternative image formats.
  - **Framework-Specific Docs:** Documentation will also need to address framework-specific usage. This will cover the limitations of the `.svg` imports in popular frameworks like React and Vue, ensuring smooth adoption for developers who are building Astro projects using these frameworks.
  - **Code Examples:** The core team can provide boilerplate examples and starter templates that showcase the optimized `.svg` handling. These templates could be adapted for common use-cases, such as building an icon library or optimizing assets for a marketing page.
- **Experimental Flag:** Initially, this feature could be released under an experimental flag in `astro.config.mjs` to gather feedback from early adopters. If the feedback is positive, the feature could later be enabled by default in a future Astro release.
- **Migration Path:** Since this feature is backwards-compatible, no migration will be required for existing projects. Developers can opt into the new functionality by updating their `.svg` imports, but their projects will continue working without any changes if they choose not to adopt the new behavior.

# Unresolved Questions

- **SVG Accessibility:** Should we automatically inject the `role="img"` attribute for better accessibility by default or leave this responsibility to developers?
  - What should happen when the `.svg` file includes a `<title>` element and they pass the `title` prop?
- **Custom IDs for Symbols:** By default, Astro would generate an ID for each `<symbol>`, but there might be cases where developers what to specify their own IDs for easier reference. Should we allow developers to manually assign custom IDs to the generated symbols?
- **Sizing:** Should the `size` property override `width` and `height` props? Should it override `width` and `height` attributes? What would be the expectation of the `size` property? Should we keep the size property?
