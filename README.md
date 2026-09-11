# Beacon SDK

TypeScript SDK for publishing notification requests to [Beacon](https://github.com/sisques-labs/beacon-api), Sisques Labs' platform notification service.

> **Status: idea stage.** This repository currently only captures the motivation and intended scope. No code has been written yet.

## Motivation

Beacon ingests notification requests asynchronously via Kafka: any internal app (e.g. Gardenia) that wants to notify a user publishes an event to Beacon's ingestion topic, and Beacon consumes it, persists it, and delivers it through the configured channel.

Today, that means every producer app has to know, by hand:

- The exact Kafka topic name (`beacon-api.notification-requests` by default).
- How to connect to the shared broker.
- The exact JSON payload shape Beacon expects (`tenantId`, `recipientUserId`, `channel`, `title`, `body`, `sourceService`, `dedupeKey`).

None of that is enforced anywhere outside Beacon's own consumer-side validation. If the schema drifts, or a producer typos a field, nothing catches it until an event silently fails to create a notification. That's the gap this SDK is meant to close: give every producer app a single, typed, versioned way to talk to Beacon, instead of re-deriving the contract from Beacon's source or docs each time.

## Intended scope (not yet built)

A thin client that wraps the Kafka producer side of the contract, roughly:

```ts
import { BeaconClient } from '@sisques-labs/beacon-sdk';

const beacon = new BeaconClient({ brokers: [...] });

await beacon.notify({
  tenantId: '...',
  recipientUserId: '...',
  channel: 'DISCORD',
  title: 'Deploy completed',
  body: 'Gardenia finished deploying v1.4.2',
  sourceService: 'gardenia',
  dedupeKey: 'gardenia-deploy-1.4.2',
});
```

### Resolved

- **NestJS module, framework-agnostic core.** The SDK ships a NestJS module (`BeaconModule.forRootAsync({...})`), registered once and marked global, so any provider anywhere can `@Inject()` a `BeaconClient` without re-importing the module — matching how `nestjs-kit`'s own `MessagingModule`/`AuthClientModule` are consumed today. Internally, the actual Kafka-producer logic (connect, build the payload, publish) lives in a plain class with zero `@nestjs/*` imports; the NestJS module is a thin registration layer on top of it. This keeps today's DX idiomatic for NestJS producers (every known producer app is NestJS) without hard-locking out a future non-NestJS caller (a plain script, a Lambda) from using the same core class directly.

Open questions still to resolve before implementation starts:

- **Kafka client dependency.** Wrap `kafkajs` directly, or depend on `@sisques-labs/nestjs-kit`'s messaging primitives (today outbound-only — see [nestjs-kit#170](https://github.com/sisques-labs/nestjs-kit/issues/170) for the inbound-consumer counterpart discussion)?
- **Schema versioning.** How does this package stay in lockstep with Beacon's ingestion contract as it evolves (new channels, new fields)? Needs a deliberate versioning/compatibility policy, not just "bump when it breaks."
- **No deliverable-address field.** By design, the payload never carries a destination address (email/push-token/Discord ID) — Beacon resolves the Discord destination from its own server-side config, deliberately, to avoid an SSRF vector on an unauthenticated ingestion topic. The SDK's types should make this omission obvious, not just silently absent.
- **Non-Node producers.** If a future producer app isn't Node/TypeScript, does this SDK's contract (topic name, JSON shape) need to be published language-agnostically (e.g. a JSON Schema) alongside the TS package?

## Relationship to other repos

- [`beacon-api`](https://github.com/sisques-labs/beacon-api) — the service this SDK talks to. The authoritative source for the current ingestion contract lives in its `openspec/changes/beacon-mvp/specs/notification-ingestion/` delta spec.
- [`nestjs-kit`](https://github.com/sisques-labs/nestjs-kit) — shared NestJS building blocks used by `beacon-api` and other Sisques Labs services.
