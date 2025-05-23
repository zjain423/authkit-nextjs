# AuthKit Next.js Library

The AuthKit library for Next.js provides convenient helpers for authentication and session management using WorkOS & AuthKit with Next.js.

> Note: This library is intended for use with the Next.js App Router.

## Installation

Install the package alongside the WorkOS Node SDK (a peer dependency):

```
pnpm i @workos-inc/authkit-nextjs @workos-inc/node
```

or

```
yarn add @workos-inc/authkit-nextjs @workos-inc/node
```

## Video tutorial

<a href="https://youtu.be/W8TmptLkEvA?feature=shared" target="_blank">
  <img src="https://github.com/user-attachments/assets/08c77835-1140-412a-baa9-a587ab27fc5e" alt="YouTube tutorial: Next.js App Router Authentication with AuthKit" style="display: block; width: 100%; max-width: 720px; height: auto; aspect-ratio: 16/9; object-fit: cover; object-position: center; margin: 1em auto;" onerror="this.onerror=null; this.src='https://i3.ytimg.com/vi/W8TmptLkEvA/maxresdefault.jpg'" />
</a>

## Pre-flight

Make sure the following values are present in your `.env.local` environment variables file. The client ID and API key can be found in the [WorkOS dashboard](https://dashboard.workos.com), and the redirect URI can also be configured there.

```sh
WORKOS_CLIENT_ID="client_..." # retrieved from the WorkOS dashboard
WORKOS_API_KEY="sk_test_..." # retrieved from the WorkOS dashboard
WORKOS_COOKIE_PASSWORD="<your password>" # generate a secure password here
NEXT_PUBLIC_WORKOS_REDIRECT_URI="http://localhost:3000/callback" # configured in the WorkOS dashboard
```

`WORKOS_COOKIE_PASSWORD` is the private key used to encrypt the session cookie. It has to be at least 32 characters long. You can use the [1Password generator](https://1password.com/password-generator/) or the `openssl` library to generate a strong password via the command line:

```
openssl rand -base64 24
```

To use the `signOut` method, you'll need to set a default Sign-out URI in your WorkOS dashboard under your application's "Redirects" tab.

### Optional configuration

Certain environment variables are optional and can be used to debug or configure cookie settings.

| Environment Variable     | Default Value         | Description                                                                                                                         |
| ------------------------ | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `WORKOS_COOKIE_MAX_AGE`  | `34560000` (400 days) | Maximum age of the cookie in seconds                                                                                                |
| `WORKOS_COOKIE_DOMAIN`   | None                  | Domain for the cookie. When empty, the cookie is only valid for the current domain                                                  |
| `WORKOS_COOKIE_NAME`     | `'wos-session'`       | Name of the session cookie                                                                                                          |
| `WORKOS_API_HOSTNAME`    | `'api.workos.com'`    | Base WorkOS API URL                                                                                                                 |
| `WORKOS_API_HTTPS`       | `true`                | Whether to use HTTPS in API calls                                                                                                   |
| `WORKOS_API_PORT`        | None                  | Port to use for API calls. When not set, uses standard ports (443 for HTTPS, 80 for HTTP)                                           |
| `WORKOS_COOKIE_SAMESITE` | `'lax'`               | SameSite attribute for cookies. Options: `'lax'`, `'strict'`, or `'none'`                                                           |
| `WORKOS_ISSUER`          | None                  | Expected `iss` claim of access tokens (comma-separated to accept multiple issuers). When not set, the issuer claim is not validated |

Example usage:

```sh
WORKOS_COOKIE_MAX_AGE='600'
WORKOS_COOKIE_DOMAIN='example.com'
WORKOS_COOKIE_NAME='my-auth-cookie'
```

> [!WARNING]
> Setting `WORKOS_COOKIE_SAMESITE='none'` allows cookies to be sent in cross-origin contexts (like iframes), but reduces protection against CSRF attacks. This setting forces cookies to be secure (HTTPS only) and should only be used when absolutely necessary for your application architecture.

> [!TIP] >`WORKOS_COOKIE_DOMAIN` can be used to share WorkOS sessions between apps/domains. Note: The `WORKOS_COOKIE_PASSWORD` would need to be the same across apps/domains. Not needed for most use cases.

## Setup

### Callback route

WorkOS requires that you have a callback URL to redirect users back to after they've authenticated. In your Next.js app, [expose an API route](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) and add the following.

```ts
import { handleAuth } from '@workos-inc/authkit-nextjs';

export const GET = handleAuth();
```

Make sure this route matches the `WORKOS_REDIRECT_URI` variable and the configured redirect URI in your WorkOS dashboard. For instance if your redirect URI is `http://localhost:3000/auth/callback` then you'd put the above code in `/app/auth/callback/route.ts`.

You can also control the pathname the user will be sent to after signing-in by passing a `returnPathname` option to `handleAuth` like so:

```ts
export const GET = handleAuth({ returnPathname: '/dashboard' });
```

If your application needs to persist data upon a successful authentication, like the `oauthTokens` from an upstream provider, you can pass in a `onSuccess` function that will get called after the user has successfully authenticated:

```ts
export const GET = handleAuth({
  onSuccess: async ({ user, oauthTokens, authenticationMethod, organizationId, state }) => {
    await saveTokens(oauthTokens);
    if (authenticationMethod) {
      await saveAuthMethod(user.id, authenticationMethod);
    }
    // Access custom state data passed through the auth flow
    const customData = state ? JSON.parse(state) : null;
    if (customData?.teamId) {
      await addUserToTeam(user.id, customData.teamId);
    }
  },
});
```

When running in environments like Docker, set the `baseURL` explicitly to ensure the redirects point to the correct location.

```ts
export const GET = handleAuth({
  baseURL: 'http://localhost:3000',
});
```

`handleAuth` can be used with the following options.

| Option           | Default     | Description                                                                                                                                                                                                 |
| ---------------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `returnPathname` | `/`         | The pathname to redirect the user to after signing in                                                                                                                                                       |
| `baseURL`        | `undefined` | The base URL to use for the redirect URI instead of the one in the request. **Required** if the app is being run in a container like docker where the hostname can be different from the one in the request |
| `onSuccess`      | `undefined` | A function that receives successful authentication data and can be used for side-effects like persisting tokens                                                                                             |
| `onError`        | `undefined` | A function that can receive the error and the request and handle the error in its own way.                                                                                                                  |

#### onSuccess callback data

The `onSuccess` callback receives the following data:

| Property               | Type                        | Description                                                                                        |
| ---------------------- | --------------------------- | -------------------------------------------------------------------------------------------------- |
| `user`                 | `User`                      | The authenticated user object                                                                      |
| `accessToken`          | `string`                    | JWT access token                                                                                   |
| `refreshToken`         | `string`                    | Refresh token for session renewal                                                                  |
| `impersonator`         | `Impersonator \| undefined` | Present if user is being impersonated                                                              |
| `oauthTokens`          | `OauthTokens \| undefined`  | OAuth tokens from upstream provider                                                                |
| `authenticationMethod` | `string \| undefined`       | How the user authenticated (e.g., 'password', 'google-oauth'). Only available during initial login |
| `organizationId`       | `string \| undefined`       | Organization context of authentication                                                             |
| `state`                | `string \| undefined`       | Custom state string passed through the authentication flow (parse with JSON.parse if needed)       |

**Note**: `authenticationMethod` is only provided during the initial authentication callback. It will not be available in subsequent requests or session refreshes.

### Initiate login URI

Create a route that initiates the AuthKit sign-in flow. This route is used as the **[Initiate login URI](https://workos.com/docs/authkit/nextjs#configure-initiate-login-uri)** (`initiate_login_uri`) for your application in the WorkOS dashboard.

```ts
// app/sign-in/route.ts (or app/login/route.ts)
import { getSignInUrl } from '@workos-inc/authkit-nextjs';
import { redirect } from 'next/navigation';

export const GET = async () => {
  const signInUrl = await getSignInUrl();
  return redirect(signInUrl);
};
```

In the [WorkOS dashboard](https://dashboard.workos.com), open **Applications**, select your application, go to the **Redirects** tab, and set the **Initiate login URI** to match this route (e.g., `http://localhost:3000/sign-in`). Use a normal `<a>` link to this route so navigation, not prefetching, starts sign-in.

> [!IMPORTANT]
> The Initiate login URI is required for features like [impersonation](https://workos.com/docs/user-management/impersonation) to work correctly. Without it, WorkOS-initiated flows (such as impersonating a user from the dashboard) will fail because they cannot complete the PKCE/CSRF verification that this library enforces on every callback.

### Proxy / Middleware

This library relies on Next.js proxy (called "middleware" in Next.js ≤15) to provide session management for routes.

**For Next.js 16+:** Create a `proxy.ts` file in the root of your project.
**For Next.js ≤15:** Create a `middleware.ts` file in the root of your project.

```ts
// proxy.ts (Next.js 16+)
import { authkitProxy } from '@workos-inc/authkit-nextjs';

export default authkitProxy();

// Match against pages that require auth
export const config = { matcher: ['/', '/admin'] };
```

```ts
// middleware.ts (Next.js ≤15)
import { authkitMiddleware } from '@workos-inc/authkit-nextjs';

export default authkitMiddleware();

// Match against pages that require auth
export const config = { matcher: ['/', '/admin'] };
```

> [!WARNING]
> Using a catch-all matcher pattern can intercept static assets (CSS, images, fonts), causing styles to break—particularly with Tailwind CSS v4. If you need a broad matcher, exclude Next.js static paths:
>
> ```ts
> export const config = {
>   matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
> };
> ```

The proxy/middleware can be configured with several options.

| Option                 | Default                                                     | Description                                                                                                                                                     |
| ---------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `redirectUri`          | `undefined`                                                 | Used in cases where you need your redirect URI to be set dynamically (e.g. Vercel preview deployments)                                                          |
| `middlewareAuth`       | `undefined`                                                 | Used to configure proxy/middleware auth options. See [middleware auth](#middleware-auth) for more details.                                                      |
| `debug`                | `false`                                                     | Enables debug logs.                                                                                                                                             |
| `signUpPaths`          | `[]`                                                        | Used to specify paths that should use the 'sign-up' screen hint when redirecting to AuthKit.                                                                    |
| `eagerAuth`            | `false`                                                     | Enables synchronous access token availability for third-party services. See [eager auth](#eager-auth) for more details.                                         |
| `refreshBufferSeconds` | `60` (`30` for tokens with a lifetime of 5 minutes or less) | Seconds before access token expiry at which the session is proactively refreshed. See [proactive session refresh](#proactive-session-refresh) for more details. |

#### Custom redirect URI

In cases where you need your redirect URI to be set dynamically (e.g. Vercel preview deployments), use the `redirectUri` option in `authkitMiddleware`:

```ts
import { authkitMiddleware } from '@workos-inc/authkit-nextjs';

export default authkitMiddleware({
  redirectUri: 'https://foo.example.com/callback',
});

// Match against pages that require auth
export const config = { matcher: ['/', '/admin'] };
```

Custom redirect URIs will be used over a redirect URI configured in the environment variables.

#### Composable proxy/middleware

If you need to combine AuthKit with other proxy logic (rate limiting, redirects, etc.), use the `authkit()` function with `handleAuthkitHeaders()` helper:

```ts
// proxy.ts (Next.js 16+) or middleware.ts (Next.js ≤15)
import { NextRequest } from 'next/server';
import { authkit, handleAuthkitHeaders } from '@workos-inc/authkit-nextjs';

export default async function proxy(request: NextRequest) {
  // For Next.js ≤15, use: export default async function middleware(request: NextRequest) {
  // Get session, headers, and the WorkOS authorization URL for sign-in redirects
  const { session, headers, authorizationUrl } = await authkit(request);

  const { pathname } = request.nextUrl;

  // Redirect unauthenticated users on protected routes
  if (pathname.startsWith('/app') && !session.user && authorizationUrl) {
    return handleAuthkitHeaders(request, headers, { redirect: authorizationUrl });
  }

  // Custom redirects (relative URLs supported)
  if (pathname === '/old-path') {
    return handleAuthkitHeaders(request, headers, { redirect: '/new-path' });
  }

  // Continue request with properly merged headers
  return handleAuthkitHeaders(request, headers);
}

export const config = { matcher: ['/', '/app/:path*'] };
```

> [!IMPORTANT]
> Always use `handleAuthkitHeaders()` when returning a response. This helper ensures:
>
> - AuthKit headers are properly passed to your pages (so `withAuth()` works)
> - Internal headers (session data, URLs) are never leaked to the browser
> - Only safe response headers (`Set-Cookie`, `Cache-Control`, `Vary`) are forwarded
> - `Cache-Control: no-store` is automatically set when cookies are present
> - `Vary` headers are properly merged when multiple values exist
> - Relative redirect URLs are automatically normalized to absolute URLs
> - POST/PUT redirects use 303 status to prevent form resubmission

> [!NOTE]
> The `redirect` option should only be used with trusted values (e.g., `authorizationUrl` from `authkit()` or hardcoded paths). Never pass user-controlled input directly to `redirect` without validation, as this could enable open redirect attacks.

##### Redirect options

```ts
handleAuthkitHeaders(request, headers, {
  redirect: '/login', // URL to redirect to (string or URL object)
  redirectStatus: 307, // 302 | 303 | 307 | 308 (default: 307 for GET, 303 for POST)
});
```

##### Advanced: Composing with rewrites

For advanced use cases like rewrites, use the lower-level `partitionAuthkitHeaders()` and `applyResponseHeaders()`:

```ts
// proxy.ts (Next.js 16+) or middleware.ts (Next.js ≤15)
import { NextRequest, NextResponse } from 'next/server';
import { authkit, partitionAuthkitHeaders, applyResponseHeaders } from '@workos-inc/authkit-nextjs';

export default async function proxy(request: NextRequest) {
  // For Next.js ≤15, use: export default async function middleware(request: NextRequest) {
  const { headers } = await authkit(request);
  const { requestHeaders, responseHeaders } = partitionAuthkitHeaders(request, headers);

  // Create your own response (rewrite, etc.)
  const response = NextResponse.rewrite(new URL('/app/dashboard', request.url), {
    request: { headers: requestHeaders },
  });

  // Apply AuthKit response headers (Set-Cookie, etc.)
  applyResponseHeaders(response, responseHeaders);

  return response;
}
```

##### Internal headers reference

AuthKit uses internal headers to pass data between proxy/middleware and server components. These are automatically handled by the helpers above, but understanding them helps with debugging.

**Request headers** (passed to server components, never sent to browser):

| Header                | Purpose                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------ |
| `x-workos-middleware` | Flag indicating AuthKit proxy/middleware is active. Required for `withAuth()` to function. |
| `x-workos-session`    | Encrypted session data. Contains user info, access token, and refresh token.               |
| `x-url`               | Current request URL. Used for redirect-after-login and generating sign-in URLs.            |
| `x-redirect-uri`      | OAuth callback URI. Used by `getAuthorizationUrl()` for the OAuth flow.                    |
| `x-sign-up-paths`     | Paths configured to trigger sign-up instead of sign-in flow.                               |

> **Security:** These headers contain sensitive session data. The `handleAuthkitHeaders()` helper ensures they're forwarded to your pages (so `withAuth()` works) but never leaked to the browser. Client-injected `x-workos-*` headers are stripped and replaced with trusted values.

**Response headers** (safe to send to browser):

| Header               | Purpose                                                                                |
| -------------------- | -------------------------------------------------------------------------------------- |
| `Set-Cookie`         | Session cookies (e.g., `wos-session`). Multiple cookies are properly appended.         |
| `Cache-Control`      | Caching directives. Auto-set to `no-store` when cookies are present.                   |
| `Vary`               | Cache variation keys. Values are deduplica