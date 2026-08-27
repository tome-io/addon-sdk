# Tomeio Add-on SDK

The public TypeScript SDK for capability-based Tomeio add-ons.

Add-ons return normalized book metadata, resolution candidates, acquisition options,
reader progress, and host-rendered library actions. They do not inject UI or executable
code into Tomeio.

## Install

Until the npm package is published, pin the GitHub release:

```json
{
  "dependencies": {
    "@tomeio/addon-sdk": "github:tome-io/addon-sdk#v0.1.0"
  }
}
```

## Example

```ts
import { createAddonHandler, defineAddon } from '@tomeio/addon-sdk';

const addon = defineAddon(
  {
    manifestVersion: 1,
    id: 'dev.example.books',
    version: '1.0.0',
    name: 'Example Books',
    description: 'An example metadata provider.',
    types: ['book'],
    resources: [{ name: 'search', supportsPagination: true }],
    transport: {
      kind: 'http',
      baseUrl: 'https://example-addon.example.workers.dev',
    },
    permissions: {
      hosts: ['https://example-addon.example.workers.dev'],
    },
  },
  {
    search: async ({ query }) => ({
      items: query
        ? [{ id: '1', title: query, authors: [], subjects: [], identifiers: {} }]
        : [],
    }),
  }
);

export default {
  fetch: createAddonHandler(addon),
};
```

The request handler uses the Web `Request` and `Response` APIs and works with
Cloudflare Workers, Bun, Node, and serverless frameworks that expose Web-standard
handlers.

## Protocol routes

- `GET /manifest.json`
- `GET /catalog/book/:catalogId.json`
- `GET /search/book.json`
- `GET /meta/book/:id.json`
- `POST /resolve/book.json`
- `GET /acquisition/book/:id.json`
- `POST /reader/sync.json`
- `POST /action/library.json`

Configuration is sent by Tomeio through `X-Tomeio-Config`. Use
`readAddonConfiguration` only when integrating outside `createAddonHandler`;
normal handlers receive parsed configuration in their invocation context.
