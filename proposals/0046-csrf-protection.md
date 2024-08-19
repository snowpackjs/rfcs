- Start Date: 2024-04-02
- Reference Issues: https://github.com/withastro/roadmap/issues/811
- Implementation PR: 

# Summary

Provide the infrastructure to protect Astro websites from CSRF attacks

# Example

```js
// astro.config.mjs
export default defineConfig({
    security: {
        checkOrigin: true
    }
})
```

# Background & Motivation


Most background is available here: https://owasp.org/www-community/attacks/csrf

Astro should provide some level of security to users.

# Goals

- Add the required checks to prevent CSRF, by checking the routes that can be called via `POST`, `PUT`, `PATCH`, or `DELETE` requests.
- Add the required checks to prevent CSRF in case the application behind a reverse proxy.

# Non-Goals

- Give the users the possibility to customise the implementation of the protection at route level.

# Detailed Design

The solution should work only on on-demand applications (SSR), and applications with `server: "hybrid"` for routes that opt-out prerendering. 

The solutions proposed should work only on request meant to **modify** data: `POST`, `PUT`, `PATCH`, or `DELETE`. Other requests should be exempt from the CSRF check.

We will call these requests as **known requests**.

The header `origin` is a header that is preset and set by all modern browsers and there's no way to temper it. If the header `origin` won't match
the origin of the URL, Astro will return a 403.

This solution should fit most websites.

# Testing Strategy

Testing this feature will require e2e tests.

# Drawbacks

Having these security checks could prevent some applications from running, so users should know when to opt-in these options.

# Alternatives

N/A

# Adoption strategy

- initial experimental flag;
- remove the flag once we're confident in the implementation of the feature;
