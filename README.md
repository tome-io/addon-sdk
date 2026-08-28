# Tomeio Add-on SDK

The public TypeScript SDK for capability-based Tomeio add-ons.

Add-ons return normalized book metadata, resolution candidates, acquisition options,
reader progress, and host-rendered library actions. They do not inject UI or executable
code into Tomeio.

An add-on with the `catalog` resource, declared `catalogs`, and `providerRoles: ['discovery']`
can be selected as Tomeio's Discovery provider. Catalog support alone does not opt an add-on into
the Home provider picker. Optional normalized `BookMetadata.offers` let Tomeio render prices and
purchase actions consistently; add-ons never provide their own cover overlays or buttons.
Manifest attribution may include an HTTPS `imageUrl` for provider-required branding.

## Install

Until the npm package is published, pin the GitHub release:

```json
{
  "dependencies": {
    "@tomeio/addon-sdk": "github:tome-io/addon-sdk#v0.4.0"
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

## GitHub-only declarative add-ons

Use `defineWorkflow` when an add-on should run without executable code or custom
hosting. TypeScript provides authoring and type checking, while the published artifact
is JSON interpreted by Tomeio. Workflows can make bounded HTTPS requests only to
manifest-approved origins and transform responses through the fixed protocol operations.

```ts
import { defineWorkflow } from '@tomeio/addon-sdk';

export const workflow = defineWorkflow({
  workflowVersion: 1,
  resources: {
    search: {
      steps: [
        {
          id: 'search',
          request: {
            urls: 'https://example.com/books',
            query: { q: { $op: 'path', path: 'input.query' } },
          },
        },
      ],
      output: { $op: 'path', path: 'steps.search.body' },
    },
  },
});
```

## Reviewed device integrations

Use `defineDeviceWorkflow` for reader integrations that need controlled access to a
user-selected folder or an Android reading app. These JSON workflows are accepted only
through Tomeio's reviewed community registry. They declare every device primitive and
Android package they need; Tomeio executes only those fixed operations and never runs
downloaded JavaScript.

Version 1 supports selected-directory scanning, bounded text/JSON/binary file reads,
ZIP entry reads, read-only SQLite queries, Android SharedPreferences parsing, and
allow-listed Android file-open intents. File operations can use only URIs supplied by
Tomeio or returned by an earlier directory scan. Device workflows cannot make network
requests.

A reader integration normally declares:

```json
{
  "resources": [{ "name": "reader" }, { "name": "libraryAction" }],
  "config": [{
    "key": "backup_directory",
    "type": "directory",
    "title": "Reader backup folder",
    "required": true
  }],
  "transport": {
    "kind": "device",
    "definitionUrl": "https://raw.githubusercontent.com/example/reader-addon/main/device-workflow.json"
  },
  "permissions": {
    "hosts": ["https://raw.githubusercontent.com"],
    "device": ["directory.read", "file.read", "android.open-file"],
    "androidPackages": ["com.example.reader"]
  }
}
```

Its workflow scans that folder, reads the reader's published backup format, and maps it
to normalized `ExtensionReaderBook` objects. A `libraryAction` can request the fixed
`android.open-file` operation; Tomeio renders the button and supplies the local file.

The complete Moon+ Reader reference implementation—including ZIP discovery, read-only
SQLite queries, progress mapping, package ids, MIME types, and its library action—lives
in [`tome-io/extensions/community/moon-reader`](https://github.com/tome-io/extensions/tree/main/community/moon-reader).

Device integrations must be submitted to `tome-io/extensions` because requested local
capabilities and packages are reviewed before they become browsable under Community.
Network-only HTTP/declarative add-ons can remain third-party and be installed by URL.

Library-action requests include the current platform and, when a local book exists, its format.
Network add-ons never receive the local URI or filename. A reviewed add-on may return
`{ kind: 'openLocalFile', packageName: 'com.example.reader' }`; Tomeio verifies that package
against `permissions.androidPackages` and performs the Android file handoff itself.

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
