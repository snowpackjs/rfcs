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

- Start Date: 2024-11-04
- Reference Issues: <!-- related issues, otherwise leave empty -->
- Implementation PR: https://github.com/withastro/astro/pull/12377
- Stage 2 Issue: https://github.com/withastro/roadmap/issues/1042
- Stage 3 PR: https://github.com/withastro/roadmap/pull/1051

# Summary

The current Astro image component offers a lot of flexibility for displaying images. It supports `densities` and `widths` props to help generate the correct `img` attributes, and the default image service supports modern formats such as AVIF and WebP. While this gives users the tools to create performant and responsive images, it does not give guidance in how to use them - and requires that they are set on all images. This proposal is for a more opinionated image component. It would offer all of the tools from the current component, and also introduce new props and config options that follow best practices by default.

# Example

Responsive images will be enabled by setting the `layout` prop to `responsive`, `fixed` or `full-width`.

```astro
---
import { Image } from "astro:assets"
import rocket from "./rocket.jpg"
---
<Image src={rocket} width={800} height={600} layout="responsive" />
```

A new `layout` option for the `image` config will default all images to that layout. This can be overridden on each image.

```js
import { defineConfig } from "astro/config";

export default defineConfig({
  image: {
    layout: "responsive",
  },
});
```

# Background & Motivation

Displaying images on the web is difficult, even for the most experienced developers. Users suffer slower page loads, and poor experience as the page layout jumps around. Meanwhile sites experience poor core web vitals scores for performance, cumulative layout shift (CLS) and largest contentful paint (LCP).

The most common [`img`tag attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/img#attributes) are well known: `src`, `alt`, `width` and `height`, there are several lesser-known attributes that are needed if an image is to have the best performance. All of these are optional according to the spec, but best practices require most of them. The most important are `srcset`, `sizes`, `loading`, `decoding` and `fetchpriority`.

These are a lot of attributes to remember and understand, though the final three have values that are usually safe to think of as dependent on just whether the image is onscreen when the page loads. Astro Image already sets `loading` and `decoding` to `lazy` and `async` by default. However `srcset` and `sizes` have no simple rules because they depend on how the image will be displayed, and can be very hard to do correctly. Images also need to be styled correctly if they are to be responsive and avoid CLS.

# Goals

- A new `layout` prop for the `Image` and `Picture` components that sets all attributes that will make an image responsive and follow best practices, including `srcset` entries and `sizes`
- Config options to change the defaults for all images
- Backwards-compatible, so that existing images are unaffected unless they set the props or config options.
- Add support for optional cropping in image services
- Images displayed correctly, even if image service doesn't support cropping

# Non-goals

- Placeholder support
- Automatic provider detection
- Art direction
- Implementing crop support for all existing image services

# Detailed Design

When a user sets the layout, either via the prop or as a default in the config, the image component will auto-generate defaults for `srcset` and `sizes`. It will also apply styles to determine the resizing behavior. This will be done according to these rules for each layout.

For each of these, `<width>` is a placeholder for the value passed by the user as the image `width` prop. The **widths** value refers to the widths of the images in the `srcset`. The **sizes** value is the string value set as the `sizes` attribute.

### Image `srcset`

The image `srcset` tells the browser which images are available. We want to generate a list of sources that allow the browser to always download an image that gives the best balance of quality and file size. There `srcset` is specified as a list of image candidates and conditions. A condition is either the width of the image, or the pixel density. The best result is almost always achieved by using widths rather than densities. This is because it allows the browser to choose the best size, and it can use any criteria it wants to choose that. For example it can choose a lower resolution image if the device is on a slow or metered network. Conversely, if using a density value, the browser _must_ download the resolution that matches the screen.

For this reason, we will generate a srcset with width conditions, based on the width and layout props.

These are indicative implementations of the functions that generate the image widths for the `srcset`. The `breakpoints` argument is an array of possible screen resolutions, which it uses to choose candidate breakpoints. The default list is shown in the section below.

```ts
/**
 * Gets the breakpoints for an image, based on the layout and width
 */
export const getWidths = ({
  width,
  layout,
  breakpoints,
  originalWidth,
}: {
  width?: number;
  layout: ImageLayout;
  breakpoints?: Array<number>;
  originalWidth?: number;
}): Array<number> => {
  const smallerThanOriginal = (w: number) =>
    !originalWidth || w <= originalWidth;

  if (layout === "full-width") {
    return breakpoints.filter(smallerThanOriginal);
  }

  if (!width) {
    return [];
  }
  const doubleWidth = width * 2;
  const maxSize = originalWidth
    ? Math.min(doubleWidth, originalWidth)
    : doubleWidth;
  if (layout === "fixed") {
    // If the image is larger than the original, only include the original width
    // Otherwise, include the image width and the double-resolution width, unless
    // the double-resolution width is larger than the original
    return originalWidth && width > originalWidth
      ? [originalWidth]
      : [width, maxSize];
  }
  if (layout === "responsive") {
    return (
      [
        // Always include the image at 1x and 2x the specified width
        width,
        doubleWidth,
        ...breakpoints,
      ]
        // Sort the resolutions in ascending order
        .sort((a, b) => a - b)
        // Filter out any resolutions that are larger than the double-resolution image or source image
        .filter((w) => w <= maxSize)
    );
  }

  return [];
};
```

When the list of widths has been generated, the component uses the sites chosen image service to generate the set of URLs for the srcset. For services that support height cropping, the height will be set to a value that preserves the requested image aspect ratio.

### Image `sizes`

The `sizes` attribute tells the browser the size at which the image will be displayed at different screen widths. The default behavior for this needs to be different for each layout, so it is generated based on the size and layout. This is an indicative implementation.

```ts
/**
 * Gets the `sizes` attribute for an image, based on the layout and width
 */
export const getSizes = (
  width?: number,
  layout?: Layout
): string | undefined => {
  if (!width || !layout) {
    return undefined;
  }
  switch (layout) {
    // If screen is wider than the max size then image width is the max size,
    // otherwise it's the width of the screen
    case `responsive`:
      return `(min-width: ${width}px) ${width}px, 100vw`;

    // Image is always the same width, whatever the size of the screen
    case `fixed`:
      return `${width}px`;

    // Image is always the width of the screen
    case `full-width`:
      return `100vw`;

    default:
      return undefined;
  }
};
```

### Image styling

It is important that an image is displayed at the correct size before the source has loaded, otherwise the page will need to re-layout. This causes annoying jumps in the page layout, and poor CLS scores. Because of this, we don't rely on the intrinsic size of the loaded image, and instead use CSS to set the correct sizing. We do not rely on just the image `width` and `height`, because we want the responsive images to resize according to the container width.

Shared styles will be generated for all sites that use images, which are then applied to images according to the chosen layout, using data attributes to target the styles, with CSS variables to set the image-specific options.

```astro
<img [data-astro-image]="responsive" {/* ...other props */} style="--w: 800; --h: 600; --fit: cover; --pos: center;" />
```

CSS vars would be used to set the width, height and crop options for each image. The classes for each layout would be as follows:

```css
[data-astro-image] {
  width: 100%;
  height: auto;
  object-fit: var(--fit);
  object-position: var(--pos);
  aspect-ratio: var(--w) / var(--h);
}
/* Styles for responsive layout */
[data-astro-image="responsive"] {
  max-width: calc(var(--w) * 1px);
  max-height: calc(var(--h) * 1px);
}
/* Styles for fixed layout */
[data-astro-image="fixed"] {
  width: calc(var(--w) * 1px);
  height: calc(var(--h) * 1px);
}
```

Users can override these styles if they prefer, by passing `class` or `style` props to the component.

### Breakpoints

The default list of breakpoints is chosen to give coverage of all common screen resolutions. They do not need to include all sizes because the browser will download a larger one if needed. However to avoid unnecessarily large downloads (and poor Core Web Vitals scores) these should always aim to have sizes as close as possible to the correct ones.

Local image services that resize the images at build time need to balance the number of images generated against the time taken to build them. Remote image services are not restricted in this way, because images are resized on demand. For this reason different default breakpoint lists will be used for local and remote services. This list is the full set, which would only be used for full-width images served from a remote image service. Other layouts would filter this list according to the rules given above.

While the comments list the common screen resolution that matches these, bear in mind that the browser can also use these sizes for other screen sizes depending on conditions such as network speed or display pixel density.

```ts
// Common screen widths. This full list is used for image services that transform at runtime
export const DEFAULT_RESOLUTIONS = [
  640, // older and lower-end phones
  750, // iPhone 6-8
  828, // iPhone XR/11
  960, // older horizontal phones
  1080, // iPhone 6-8 Plus
  1280, // 720p
  1668, // Various iPads
  1920, // 1080p
  2048, // QXGA
  2560, // WQXGA
  3200, // QHD+
  3840, // 4K
  4480, // 4.5K
  5120, // 5K
  6016, // 6K
];

// A more limited set of screen widths, for statically generated images
export const LIMITED_RESOLUTIONS = [
  640, // older and lower-end phones
  750, // iPhone 6-8
  828, // iPhone XR/11
  1080, // iPhone 6-8 Plus
  1280, // 720p
  1668, // Various iPads
  2048, // QXGA
  2560, // WQXGA
];
```

You may be expecting the list to include smaller resolutions such as 480 or 320. These are not needed however, as all devices that have screens that apparent size have a higher pixel density. There have been no devices made in the past decade or more that actually have a 320px wide screen with a 1x pixel density.

## Lazy and priority loading

Astro images currently default to `loading="lazy"` and `decoding="async"`, and for priority images you should set these to `eager` and `sync` respectively. Best practice is to also set the `fetchpriority` attribute, but this does not currently have a default in Astro. This proposal adds a new `priority` prop to handle these all at once, to make it easier to eagerly load your LCP and other priority images. It is a boolean prop, which sets `loading="eager"`, `decoding="sync"` and `fetchpriority="high"` for an image. Preloading is out of scope for this RFC, but it could conceivably used for this in future.

## Image service crop support

Currently, Astro image services do not support cropping, and if the target image aspect ratio does not match the source image it will be stretched. While image services do support the height property, the built-in image service ignores it, and other services do not get passed the properties needed to handle cropping. Most of the underlying services do support cropping though, so could implement cropping in their image services if needed, normally with a single parameter. The sharp library that powers the default service supports cropping, with a wide range of options.

While these responsive images do not rely on crop support in the services, it will give better results, with smaller image sizes when the aspect ratio is different. Currently the full image is served, with the image needing to be cropped using CSS in the browser. This means a lot of wasted pixels being sent. Adding crop support means only the needed image sizes would be sent.

#### New `ImageTransform` properties

- `fit`: Allowed values are the supported values for CSS `object-fit`. When unset (which is the default for existing images) this gives the current behavior, meaning image is not cropped and only the width is used to resize. This is passed to the image service which should use it to crop the image in a way that matches the CSS value.
- `position`: matches the CSS `object-position` property, this specifies the alignment of the image when cropped. Defaults to `center`. At minimum, this supports all `css-position` values, but also will pass through arbitrary values that may be supported by the image service. For example, some services support `auto`, which automatically detects the main point of interest, or `face` which focusses on human faces when present. The allowed values of these depend on the service and are not checked.

#### New image props

There are new props added to the image component which expose this crop support.

- `fit`: uses the `fit` property in the image transform to crop the image, and sets the `object-fit` CSS property. Requires both `width` and `height` to be set. For responsive images this defaults to `cover`, which is also the CSS value that is used if the value is not one that is supported by CSS. While this may be different from the one used by the image service, the image service should be returning an image that matches the size, so this will not be used in those cases. Existing images which are not cropped will have this set to `fill`, which matches the current behavior.
- `position`: if `fit` is `cover` or a custom value, this specifies the focal point for the crop. Allowed values are the CSS values plus any custom values supported by the image service.

# Testing Strategy

The existing image component and service tests can be extended to cover the new cases. E2E tests would need to be added to ensure the correct dimensions are used at various viewport sizes, and the image service can be tested to ensure the request dimensions are used correctly.

# Drawbacks

- This is an opinionated implementation, which by definition will not meet everyone's needs
- Not all image services support cropping, most importantly Vercel
- Most of the value requires both width and height to be set

# Alternatives

The current component allows users to implement most of these options by setting values manually

# Adoption strategy

- This will initially be enabled with an `experimental.responsiveImages` config option. Configuration of defaults use prefixed option names: `image.experimentalLayout` and `image.experimentalFit`.
- When unflagged, it will be backwards-compatible. If `layout` is not set as a prop or default config value, the component will behave exactly as now.
- In future we may decide to make this the default, but that would be in a future major, not Astro 5.

# Unresolved Questions

- The crop attributes and values may not work well once we look at different image services.
- It may be possible to combine responsive with full-width layouts, though this would need some thought
