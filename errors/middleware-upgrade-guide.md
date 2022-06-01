# Middleware Upgrade Guide

As we work on improving Middleware for General Availability (GA), we've made some changes to the Middleware APIs (and how you define Middleware in your application) based on your feedback.

This upgrade guide will help you understand the changes and how to migrate your existing Middleware to the new API. This guide is for Next.js customers who:

- Currently use the beta Next.js Middleware features
- Choose to upgrade to the next stable version of Next.js

> **Note:** For customers using Next.js on Vercel, your existing deploys using Middlware will continue to work. Existing sites can be redeployed without needing to upgrade. Before upgrading to the next stable version of Next.js, you will need to follow the upgrade guide below.

## Breaking changes

1. [No Nested Middleware](#no-nested-middleware)
2. [No Response Body](#no-response-body)
3. [Cookies API Revamped](#cookies-api-revamped)
4. [No More Page Match Data](#no-more-page-match-data)
5. [Executing Middleware on Internal Next.js Requests](#executing-middleware-on-internal-next.js-requests)

## No Nested Middleware

### Summary of changes

- Define a single Middleware file at the root of your project
- No need to prefix the file with an underscore
- A custom matcher can be used to define matching routes using an exported config object

### Explanation

Previously, you could create a `_middleware.js` file under the `pages` directory at any level. Middleware execution was based on the file path where it was created. Beta customers found this route matching confusing. For example:

- Middleware in `pages/dashboard/_middleware.ts`
- Middlware in `pages/dashboard/users/_middleware.ts`
- A request to `/dashboard/users/*` **would match both.**

Based on customer feedback, we have replaced this API with a single root Middleware.

### How to upgrade

You should declare **one single Middleware file** in your application, which should be located at the root of the project directory (**not** inside of the `pages` directory) and named without an `_` prefix. Your Middleware file can still have either a `.ts` or `.js` extension.

Middleware will be invoked for **every route in the app**, and a custom matcher can be used to define matching filters. The following is an example for a Middleware that triggers for `/about/*`, the custom matcher is defined in an exported config object:

```typescript
// middleware.ts
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  return NextResponse.rewrite(new URL('/about-2', request.url))
}
// config with custom matcher
export const config = {
  pathname: '/about/:path*',
}
```

You can also use a conditional statement to only run the Middleware when it matches the `about/*` path:

```typescript
// <root>/middleware.js
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith('/about')) {
    // This logic is only applied to /about
  }

  if (request.nextUrl.pathname.startsWith('/dashboard')) {
    // This logic is only applied to /dashboard
  }
}
```

## No Response Body

### Summary of changes

- Middleware can no longer respond with a body
- If your Middleware _does_ respond with a body, a runtime error will be thrown
- Migrate to using `rewrites`/`redirects` to pages that handle authorization

### Explanation

Beta customers had explored using Middleware to handle authorization for their application. However, to ensure both the HTML and data payload (JSON file) are protected, we commend checking authorization at the Page level.

To help ensure security, we are removing the ability to send response bodies in Middleware. This ensures that Middleware is only used to `rewrite`, `redirect`, or modify the incoming request (e.g. [setting cookies](#cookies-api-revamped)).

The following patterns will no longer work:

```js
new Response('a text value')
new Response(streamOrBuffer)
new Response(JSON.stringify(obj), { headers: 'application/json' })
```

### How to upgrade

For cases where Middleware is used to respond (such as authorization), you should migrate to use `rewrites`/`redirects` to Pages that show an authorization error, login forms, or to an API Route.

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { credentialsValid } from './lib/auth'

export function middleware(request: NextRequest) {
  const basicAuth = request.headers.get('authorization')

  if (basicAuth) {
    const auth = basicAuth.split(' ')[1]
    const [user, pwd] = atob(auth).split(':')
    // Example function to validate auth
    if (credentialsValid(user, pwd)) {
      return NextResponse.next()
    }
  }

  const loginUrl = new URL('/login', request.url)
  loginUrl.searchParams.set('from', request.nextUrl.pathname)

  return NextResponse.redirect(loginUrl)
}
```

## Cookies API Revamped

### Summary of changes

**Added**

- `cookie.delete`
- `cookie.set`
- `cookie.getAll`

**Removed:**

- `cookie`
- `cookies`
- `clearCookie`

### Explanation

Based on beta feedback, we are changing the Cookies API in `NextRequest` and `NextResponse` to align more to a `get`/`set` model.

### How to upgrade

`NextResponse` now has a `cookies` instance with:

- `cookie.delete`
- `cookie.set`
- `cookie.getAll`

#### Before

```javascript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // create an instance of the class to access the public methods. This uses `next()`,
  // you could use `redirect()` or `rewrite()` as well
  let response = NextResponse.next()
  // get the cookies from the request
  let cookieFromRequest = request.cookies['my-cookie']
  // set the `cookie`
  response.cookie('hello', 'world')
  // set the `cookie` with options
  const cookieWithOptions = response.cookie('hello', 'world', {
    path: '/',
    maxAge: 1000 * 60 * 60 * 24 * 7,
    httpOnly: true,
    sameSite: 'strict',
    domain: 'example.com',
  })
  // clear the `cookie`
  response.clearCookie('hello')

  return response
}
```

#### After

```typescript
// middleware.ts
export function middleware() {
  const response = new NextResponse()

  // set a cookie
  response.cookies.set('vercel', 'fast')

  // set another cookie with options
  response.cookies.set('nextjs', 'awesome', { path: '/test' })

  // get all the details of a cookie
  const [value, options] = response.cookies.getWithOptions('vercel')
  console.log(value) // => 'fast'
  console.log(options) // => { Path: '/test' }

  // delete a cookie means mark it as expired
  response.cookies.delete('vercel')

  // clear all cookies means mark all of them as expired
  response.cookies.clear()
}
```

## No More Page Match Data

### Summary of changes

- Use [`URLPattern`](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) to check if a Middleware is being invoked for a certain page match

### Explanation

Currently, Middleware estimates whether you are serving an asset of a Page based on the Next.js routes manifest (internal configuration). This value is surfaced through `request.page`.

To make this more accurate, we are recommending to use the web standard `URLPattern` API.

### How to upgrade

Use [`URLPattern`](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern) to check if a Middleware is being invoked for a certain page match.

#### Before

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server'

export function middleware(request: NextRequest) {
  const { params } = event.request.page
  const { locale, slug } = params

  if (locale && slug) {
    const { search, protocol, host } = request.nextUrl
    const url = new URL(`${protocol}//${locale}.${host}/${slug}${search}`)
    return NextResponse.redirect(url)
  }
}
```

#### After

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server'

const PATTERNS = [
  [
    new URLPattern({ pathname: '/:locale/:slug' }),
    ({ pathname }) => pathname.groups,
  ],
]

const params = (url) => {
  const input = url.split('?')[0]
  let result = {}

  for (const [pattern, handler] of PATTERNS) {
    const patternResult = pattern.exec(input)
    if (patternResult !== null && 'pathname' in patternResult) {
      result = handler(patternResult)
      break
    }
  }
  return result
}

export function middleware(request: NextRequest) {
  const { locale, slug } = params(request.url)

  if (locale && slug) {
    const { search, protocol, host } = request.nextUrl
    const url = new URL(`${protocol}//${locale}.${host}/${slug}${search}`)
    return NextResponse.redirect(url)
  }
}
```

## Executing Middleware on Internal Next.js Requests

### Summary of changes

- Middleware will be executed for _all_ requests, including `_next`

### Explanation

Currently, we do not execute Middleware for `_next` requests, due the authorization use cases.

For cases where Middleware is used for authorization, you should migrate to use `rewrites`/`redirects` to Pages that show an authorization error, login forms, or to an API Route.