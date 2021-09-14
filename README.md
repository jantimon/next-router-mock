# `next-router-mock`

An implementation of the Next.js Router that keeps the state of the "URL" in memory (does not read or write to the
address bar). Useful in tests. Inspired
by [`react-router > MemoryRouter`](https://github.com/ReactTraining/react-router/blob/master/packages/react-router/docs/api/MemoryRouter.md)
.

# API

`next-router-mock` is a drop-in replacement for `next/router`. It exports both a default (singleton) router and
the `useRouter` hook.

Install via NPM: `npm install --save-dev next-router-mock`

### Jest

Simply drop-in the `next-router-mock` like this:

```jsx
jest.mock('next/router', () => require('next-router-mock'));
// or this:
jest.mock('next/dist/client/router', () => require('next-router-mock'));
```

> Note: it's better to mock `next/dist/client/router` instead of  `next/router`, because both `next/router` and `next/link` depend directly on this nested path. It's also perfectly fine to mock both.

Here's a full working example:

```jsx
import singletonRouter, { useRouter } from 'next/router';
import NextLink from 'next/link';
import { render, act, fireEvent, screen, waitFor } from '@testing-library/react';
import { renderHook } from '@testing-library/react-hooks';
import mockRouter from 'next-router-mock';

// This is all you need:
jest.mock('next/dist/client/router', () => require('next-router-mock'));

describe('next-router-mock', () => {
  beforeEach(() => {
    mockRouter.setCurrentUrl("/initial");
  });
  
  it('supports `push` and `replace` methods', () => {
    singletonRouter.push('/foo?bar=baz');
    expect(singletonRouter).toMatchObject({
      asPath: '/foo?bar=baz',
      pathname: '/foo',
      query: { bar: 'baz' },
    });
  });
  
  it('supports URL objects with templates', () => {
    singletonRouter.push({
      pathname: '/[id]/foo',
      query: { id: '123', bar: 'baz' },
    });
    expect(singletonRouter).toMatchObject({
      asPath: '/123/foo?bar=baz',
      pathname: '/[id]/foo',
      query: { bar: 'baz' },
    });
  });

  it('mocks useRouter', () => {
    const { result } = renderHook(() => useRouter());
    expect(result.current).toMatchObject({ asPath: "" });
    act(() => {
      result.current.push("/example");
    });
    expect(result.current).toMatchObject({ asPath: "/example" })
  });

  it('works with next/link', () => {
    render(<NextLink href="/example?foo=bar"><a>Example Link</a></NextLink>);
    fireEvent.click(screen.getByText('Example Link'));
    expect(singletonRouter).toMatchObject({ asPath: '/example?foo=bar' });
  });
});
```

# Sync vs Async

By default, `next-router-mock` handles route changes synchronously. This is convenient for testing, and works for most
use-cases.  
However, Next normally handles route changes asynchronously, and in certain cases you might actually rely on that
behavior. If that's the case, you can use `next-router-mock/async`. Tests will need to account for the async behavior
too; for example:

```jsx
it('next/link can be tested too', async () => {
  render(<NextLink href="/example?foo=bar"><a>Example Link</a></NextLink>);
  fireEvent.click(screen.getByText('Example Link'));
  await waitFor(() => {
    expect(singletonRouter).toMatchObject({
      asPath: '/example?foo=bar',
      pathname: '/example',
      query: { foo: 'bar' },
    });
  });
});
```

# Supported Features

- `useRouter()`
- `router.push(url, as, options)`
- `router.replace(url, as, options)`
- `router.pathname`
- `router.asPath`
- `router.query`
- `router.locale`
- `router.locales`
- Works with `next/link` (see Jest notes)
- `router.events` supports:
  - `routeChangeStart(url, { shallow })`
  - `routeChangeComplete(url, { shallow })`

## Not yet supported:

PRs welcome!  
These fields have default values; these methods do nothing.

- `router.isReady`
- `router.route`
- `router.basePath`
- `router.isFallback`
- `router.prefetch()`
- `router.back()`
- `router.beforePopState(cb)`
- `router.reload()`
- `router.events` not implemented:
  - `routeChangeError`
  - `beforeHistoryChange`
  - `hashChangeStart`
  - `hashChangeComplete`

