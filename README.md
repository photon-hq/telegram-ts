# @photon-ai/telegram

TypeScript SDK for the Telegram Bot API via Photon's Spectrum gRPC gateway.

## Installation

```bash
bun add @photon-ai/telegram
```

## Quick start

```ts
import { createClient } from "@photon-ai/telegram";

const client = createClient({
  botToken: process.env.TELEGRAM_BOT_TOKEN!,
});

// Send a text message
await client.messages.send({
  chatId: "-100123456789",
  text: "Hello from the SDK!",
});

// Subscribe to inbound events
const stream = client.events.subscribe();
for await (const event of stream) {
  if (event.type === "heartbeat") continue;
  if (event.type === "message") {
    console.log(`${event.message.from?.username}: ${event.message.text}`);
  }
}

await client.close();
```

## Configuration

| Environment variable | Description | Default |
| --- | --- | --- |
| `TELEGRAM_ENDPOINT` | gRPC server address (host:port or full URL) | `telegram-grpc.spectrum.photon.codes:443` |
| `SPECTRUM_TELEGRAM_ENDPOINT` | Legacy alias for `TELEGRAM_ENDPOINT` | — |

`endpoint` accepts:
- `host:port` (plaintext, e.g. for local dev)
- `https://host[:port]` (TLS)
- `http://host[:port]` (plaintext over HTTP)

## Receiving events

`events.subscribe()` returns a `TypedEventStream` you iterate with `for await ... of`. The stream auto-reconnects with full-jitter backoff and resumes from the last delivered cursor.

```ts
const stream = client.events.subscribe({ fromCursor: lastSavedCursor });
for await (const event of stream) {
  switch (event.type) {
    case "message":
      // event.message: InboundMessage
      break;
    case "reaction":
      // event.reaction: MessageReaction
      break;
    case "poll":
    case "poll_answer":
      // event.poll / event.answer
      break;
    case "heartbeat":
      // ignore; emitted every ~30s
      break;
  }
  await saveCheckpoint(event.cursor);
}
await stream.close();
```

For bounded historical replay, use `events.replay({ fromCursor, untilCursor?, limit? })`.

## Sending messages

Beyond text, `send()` accepts `photo`, `video`, `audio`, `voice`, `document`, `sticker`, `location`, `contact`, `poll`, or `attachment`. Media may be referenced by `fileId`, `url`, or raw `data`.

```ts
await client.messages.send({
  chatId,
  photo: { url: "https://example.com/cat.jpg", caption: "look" },
});

import { attachment } from "@photon-ai/telegram";
await client.messages.send({
  chatId,
  attachment: await attachment("./photo.jpg"),
});
```

See [`bot.proto`](./proto/spectrum/telegram/v1/bot.proto) for the full surface.

## Errors

All RPC errors are wrapped in typed subclasses of `TelegramTransportError`:

- `AuthenticationError` — `UNAUTHENTICATED` / `PERMISSION_DENIED`
- `RateLimitError` — `RESOURCE_EXHAUSTED` (carries `retryAfterSec` when present)
- `NotFoundError` — `NOT_FOUND`
- `ValidationError` — `INVALID_ARGUMENT` / `FAILED_PRECONDITION`
- `UnavailableError` — `UNAVAILABLE` / `DEADLINE_EXCEEDED`

`retryable` and `grpcCode` are exposed on every instance.

## Development

```bash
bun install
bun run generate    # regenerate protobuf types from proto/
bun run build       # generate + bundle with tsup
bun run check       # type-check
bun run lint        # lint with Biome
bun test            # run tests
```

`src/_gen/` is generated from `proto/spectrum/telegram/v1/bot.proto` and is git-ignored; `bun run build` regenerates it as part of the build.

## Releases

Every push to `main` runs the [Release workflow](.github/workflows/release.yaml) to build, type-check, lint, and publish a staging candidate through the pinned [Buildspace package workflows](https://github.com/photon-hq/buildspace/tree/c1fb06e4ae457a8e053642a914bd37b8f5ce9e1b/.github/workflows). Manual dispatch defaults to `dry-run`, which verifies and packs both channels without publishing or creating tags or Releases. To release, dispatch `promote` from `main` with a staging version and obtain approval in the `production` environment. Promotion publishes the stored production candidate under npm's stable `latest` tag without rebuilding or committing a source version bump.

The pack script stamps a temporary copy of the built package and checks that the archive's file list contains every declared export, entry point, and type declaration. The source manifest and lockfile stay unchanged.

Before the first npm publication, an npm administrator must [configure trusted publishing](https://docs.npmjs.com/trusted-publishers/) for `photon-hq/telegram-ts`, caller workflow `release.yaml`, and two environment-specific entries: `staging` and `production`, each allowing `npm publish` and `npm dist-tag`. Publishing uses GitHub OIDC; these workflows do not use `NPM_TOKEN`. Keep any legacy token until the first real promotion succeeds and no active legacy consumer or documented rollback needs it.

The first staging run can create `staging` automatically. Before the first production promotion, a repository administrator must run the [Buildspace environment setup command](https://github.com/photon-hq/buildspace/blob/main/README.md#package-stage-and-promote). The command restricts both environments to `main` and requires at least one production reviewer, using CODEOWNERS by default; append `@USER` or `@ORG/TEAM` after the repository argument if CODEOWNERS supplies no reviewers. [Required reviewers in private repositories need GitHub Enterprise](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments).

## License

MIT
