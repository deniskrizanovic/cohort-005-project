# Research: how Knock's in-app feed works

Ticket: [#2 Research how Knock's in-app feed works](https://github.com/deniskrizanovic/cohort-005-project/issues/2). Map: [#1 In-app Notifications in Cadence](https://github.com/deniskrizanovic/cohort-005-project/issues/1).

Facts only. No recommendations or decisions. Research date: 2026-11-10.

## Sources and conventions

- Primary sources only: `docs.knock.app` pages, Knock's OpenAPI spec (`https://docs.knock.app/openapi.json`), the official SDK repos and npm packages (`knocklabs/javascript`, `knocklabs/knock-node`), and Knock's own pricing page (`https://knock.app/pricing`).
- Docs pages were read through the raw Markdown form that the site serves by appending `.md` to a page URL, for example `https://docs.knock.app/in-app-ui/react/feed.md`. The index of all pages is `https://docs.knock.app/llms.txt`. The citations below give the normal page URL.
- Package versions seen on npm on the research date: `@knocklabs/react` 0.15.1 (MIT), `@knocklabs/node` 1.41.0 (Apache-2.0).
- "Knock user" means Knock's own recipient entity. In Cadence terms the people involved are Students, Instructors, Platform Admins and Team Admins.
- Marker `[unverified]` means the docs did not state it and nothing was run to confirm it. Marker `[ambiguous]` means the docs contradict themselves or are vague.
- Cadence facts (from the repo, not Knock): React 19.2, React Router 8.3.0, Tailwind and shadcn/ui (radix-ui), numeric user ids, cookie session in `app/lib/session.ts` (`cadence_session`, `userId` stored as a number). `app/lib/mailer.server.ts` picks a Postmark transport when `POSTMARK_SERVER_TOKEN` is set, otherwise writes JSON files to `.outbox/`.

## 1. Feed UI: React SDK components vs headless

### What is available

Knock has two in-app channel types: **feeds** (a per-user list of messages) and **guides** (single contextual messages such as banners and modals). This ticket covers feeds only. Source: [In-app UI overview](https://docs.knock.app/in-app-ui/overview), [Feeds vs guides](https://docs.knock.app/in-app-ui/feeds-vs-guides).

Packages:

| Package | Role | Source |
| --- | --- | --- |
| `@knocklabs/react` | Prebuilt components plus hooks (web React only) | [React overview](https://docs.knock.app/in-app-ui/react/overview), [README](https://github.com/knocklabs/javascript/tree/main/packages/react) |
| `@knocklabs/client` | Framework-agnostic JS client: `Knock`, `FeedClient`, zustand store. `@knocklabs/react` depends on it | [JS SDK overview](https://docs.knock.app/in-app-ui/javascript/overview) |
| `@knocklabs/react-core` | Hooks and providers that `@knocklabs/react` re-exports | listed as a dependency in the `@knocklabs/react` package.json |
| `@knocklabs/node` | Server SDK (trigger, identify, sign user tokens) | [knock-node README](https://github.com/knocklabs/knock-node) |

### Prebuilt React components

From [React overview](https://docs.knock.app/in-app-ui/react/overview) and [Feed](https://docs.knock.app/in-app-ui/react/feed):

- `KnockProvider` (props `apiKey`, `user`, `userToken`, `i18n`, `enabled`, `host`, `logLevel`; see [KnockProvider ref](https://docs.knock.app/in-app-ui/react/sdk/components/knock-provider)).
- `KnockFeedProvider` (props `feedId` = the in-app channel ID, `colorMode`, `defaultFeedOptions`, and `userToken` per the Feed page).
- `NotificationIconButton`: the bell, with a badge. `badgeCountType="unread"` switches the badge from unseen count (default) to unread count.
- `NotificationFeedPopover`: managed popover anchored to a button ref (`buttonRef`, `isVisible`, `onClose`, `onOpen`, `onNotificationClick`).
- `NotificationFeed`: full-page list; `renderItem`, `renderHeader`, `initialFilterStatus` (`FilterStatus.Unread`), `onNotificationClick`, `onNotificationButtonClick(item, button)`.
- `NotificationCell` (default cell; accepts `avatar` and `children`) and `Avatar`.
- Pages for a notification inbox ([Inbox](https://docs.knock.app/in-app-ui/react/inbox), a full-page `NotificationFeed`) and real-time toasts ([Toasts](https://docs.knock.app/in-app-ui/react/toasts)).
- Required stylesheet unless overriding: `import "@knocklabs/react/dist/index.css"` (about 68 KB in the 0.15.1 tarball).
- The built-in `NotificationFeed` header has an All / Unread status filter. Cells show an "x" archive control; there is "mark all as read" ([In-app feeds integration page](https://docs.knock.app/integrations/in-app/feeds)).
- Only English ships; translations are supplied through `KnockProvider`'s `i18n` prop ([Feed page](https://docs.knock.app/in-app-ui/react/feed#localization)).

### Customising the prebuilt components

Source: [Styling](https://docs.knock.app/in-app-ui/feeds/styling), [Feed page, UI/UX overrides](https://docs.knock.app/in-app-ui/react/feed#uiux-overrides).

- CSS variables prefixed `--rnf-`; full list is `theme.css` in the [javascript repo](https://github.com/knocklabs/javascript/blob/main/packages/react/src/theme.css).
- Or replace the CSS entirely; all classes start with `rnf-`.
- `renderItem` replaces the whole cell; `children` on `NotificationCell` injects content into the default cell; `avatar` swaps the avatar; `renderHeader` replaces the header; `colorMode` is `light` (default) or `dark`.
- The Knock docs say the default in-app feed can be forked to change behaviour ([In-app feeds integration page](https://docs.knock.app/integrations/in-app/feeds#how-the-knock-in-app-feed-uses-status)).
- The package pulls in Radix and `@telegraph/*` UI packages plus `lucide-react` as dependencies (package.json, `@knocklabs/react` 0.15.1). Cadence already uses `radix-ui` and `lucide-react`. Whether those overlap or duplicate at install time was not checked `[unverified]`.
- Peer dependencies: `react` and `react-dom` ^17 || ^18 || ^19; `next` and `@tanstack/react-router` are declared optional peers (`peerDependenciesMeta`). The package exposes `./next` and `./tanstack` entry points. No React Router entry point exists and the docs do not mention React Router or Remix, so behaviour under React Router SSR is `[unverified]`. The `dist` output carries `"use client"` directives (checked in the 0.15.1 tarball).
- Tailwind and shadcn/ui: the docs do not discuss either. Styling is by CSS variables, class overrides, or custom components.

### Headless (bring your own UI)

Source: [Custom UI (headless)](https://docs.knock.app/in-app-ui/feeds/custom-ui), [React headless feed](https://docs.knock.app/in-app-ui/react/headless/feed), [JS SDK quick start](https://docs.knock.app/in-app-ui/javascript/sdk/quick-start).

- React hooks from `@knocklabs/react`: `useKnockClient()`, `useAuthenticatedKnockClient()`, `useNotifications(knockClient, feedChannelId, options?)` which returns a `Feed` client, and `useNotificationStore(feedClient)` which returns `{ items, metadata, ... }`. Call `feedClient.fetch()` in an effect.
- `metadata` has `total_count`, `unread_count`, `unseen_count`.
- Each `FeedItem` has `blocks` (rendered markdown or text as HTML or plaintext, plus optional button sets), `data` (the trigger payload), `actors`, `source` (workflow key), `tenant`, `inserted_at`, `seen_at`, `read_at`, `archived_at`, `link_clicked_at`. Full schema: [In-app APIs](https://docs.knock.app/in-app-ui/api-overview#feeditem).
- Feed `mode`: SDK default is `compact` (drops `activities`, extra actors, nested `data`); `rich` returns everything. The REST API itself defaults to `rich` per the OpenAPI spec (the two defaults differ, `[ambiguous]` which applies when a client does not set it).
- With the headless hooks the app is responsible for calling `markAllAsSeen()`; the prebuilt popover does it on open ([headless page](https://docs.knock.app/in-app-ui/react/headless/feed#marking-items-as-seen)).
- Non-React: `@knocklabs/client` exposes `new Knock(publicKey)`, `knock.authenticate(user, token)`, `knock.feeds.initialize(channelId, options)`, `feed.listenForUpdates()`, `feed.fetch()`, `feed.fetchNextPage()`, and `markAsRead/Seen/Archived` (plus un- variants) ([FeedClient ref](https://docs.knock.app/in-app-ui/javascript/sdk/feed-client)).
- Direct REST is also possible: `GET /v1/users/{user_id}/feeds/{id}` ([OpenAPI](https://docs.knock.app/openapi.json), [API reference](https://docs.knock.app/api-reference/users/feeds/list_items)). Realtime via the socket needs the client SDK; no standalone socket protocol doc was found `[unverified]`.
- Rendering `item.blocks[0].rendered` uses `dangerouslySetInnerHTML` in Knock's own example, because `rendered` is HTML for markdown blocks.
- A "pinned" list is not supported; the docs suggest multiple feed instances plus filtering ([Feeds overview FAQ](https://docs.knock.app/in-app-ui/feeds/overview)).

### Interactivity and deep links

Source: [Handling interactivity](https://docs.knock.app/in-app-ui/feeds/handling-interactivity), [In-app feeds integration page](https://docs.knock.app/integrations/in-app/feeds#action-url).

- A template can enable a click action ("Action URL"), static or a Liquid variable. If non-empty, `KnockFeedProvider` attaches a redirect handler. The docs describe this as a redirect to the URL; whether it does a full page load or a client-side navigation was not documented `[unverified]`. Overriding `onNotificationClick` and `onNotificationButtonClick` is the documented way to control navigation.
- Template variants: Standard (whole cell clickable), Single-action (one primary button), Multi-action (primary and secondary buttons).
- Clicking a cell records an `interacted` event; link clicks inside the body only record `link_clicked` if Knock link tracking is enabled.

## 2. Identity and auth

### Identity mapping

Source: [Users](https://docs.knock.app/concepts/users), [Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients), [UserIdentificationOptions](https://docs.knock.app/in-app-ui/react/sdk/types/user-identification-options).

- A Knock user needs only an `id`. The SDK types it as `string`. Max 256 characters, cannot contain `/` or `#`, and cannot be changed after creation. Docs recommend the internal primary key.
- Cadence ids are numbers, so a string form would be needed at the boundary. The docs do not say whether numeric ids are accepted or coerced `[unverified]`.
- Reserved properties: `name`, `email`, `avatar`, `locale`, `phone_number`, `timezone`, `created_at`. Custom properties allowed, up to 256 kB total.
- Three ways to identify: direct upsert (`PUT /v1/users/{user_id}`), inline (in a workflow trigger's recipients/actor, or by `KnockProvider` client-side), and bulk (`POST /v1/users/bulk/identify`). All are upserts with deep merge.
- A recipient must exist in Knock to receive a message. Inline identification in the trigger call guarantees identification before the run.
- Client-side inline identification (via `KnockProvider`'s `user` prop) works with the public key only and is documented as suited to displaying in-app notifications; sensitive or extensive properties should be set server-side.
- Users are per environment; they must be re-identified in each environment.
- All users that receive a notification are identified and persisted ([Users FAQ](https://docs.knock.app/concepts/users)). See [Deleting users](https://docs.knock.app/managing-recipients/deleting-users) for privacy controls (not read in detail).

### Keys

Source: [API keys](https://docs.knock.app/developer-tools/api-keys), [Environments](https://docs.knock.app/version-control/environments).

- Public keys start with `pk_`, secret keys with `sk_`. Keys are per environment. Any number of keys per environment.
- Browser uses the public key plus the user id; server uses the secret key (`KNOCK_API_KEY` is the Node SDK default env var).
- Also needed by the client: the feed channel ID (a UUID, copied from Dashboard > Settings > Channels and sources).

### Enhanced security and user tokens

Source: [Security and authentication](https://docs.knock.app/in-app-ui/security-and-authentication).

- Off by default. While off, a request with the public key can act as any user and no token is needed. The docs recommend turning it on before release, and say production should have it on ([Implementation guide, going to production](https://docs.knock.app/tutorials/implementation-guide#going-to-production)).
- When on, requests without a signed user token are rejected.
- Where to enable it: the implementation guide says Dashboard > Platform > API keys in the Production environment. The security page itself gives no toggle steps. The docs suggest enabling environments and keeping the mode off in development.
- Signing key: generated in Dashboard > Platform > API keys; the private key is shown once. Offered as base-64 PEM (single line, for env vars) and plain PEM. Convention: `KNOCK_SIGNING_KEY`.
- Token: JWT signed RS256. Required claim `sub` = Knock user id; `iat` and `exp` recommended. The Node SDK default lifetime is 1 hour (`expiresInSeconds`).
- Node helper: `signUserToken(userId, { signingKey?, expiresInSeconds?, grants?, shouldGenerateJti? })`. The docs page imports it from `@knocklabs/node/lib/tokenSigner`; the SDK 1.41.0 README imports it from `@knocklabs/node`. Both paths exist in the 1.41.0 tarball (`lib/tokenSigner.js` and the package index), so this is a docs/README mismatch only. The SDK uses `jose`; a CJS build may need `jose` v5 via `overrides` (README).
- The token is generated on the server, so no extra network request is needed. The page says to pass it to the front end, for example as a property or in a cookie.
- Client use: `knockClient.authenticate({ id }, token)` or `<KnockProvider userToken={...}>`. The Feed page also shows `userToken` on `KnockFeedProvider` `[ambiguous]`: the KnockProvider reference documents `userToken`, the KnockFeedProvider reference page does not list it.
- Refresh: `onUserTokenExpiring(oldToken)` callback (React `KnockOptions`, JS `AuthenticateOptions`), fires 30 s before expiry by default (`timeBeforeExpirationInMs`); a returned string becomes the new token. Calling `authenticate` again restarts the realtime connection. The docs advise matching token lifetime to the session token lifetime where possible.
- `KnockProvider` has an `enabled` prop. When false, no identify call, no API requests, no websocket. Documented as the way to wait for a token on first render.
- Grants (optional): `explicit_grants: true` restricts the token to listed grants such as `user/feed_read`, `user/feed_write`, tenant-scoped and object-scoped grants.
- Alternative: skip client auth by proxying every request through the backend; docs do not recommend it (extra latency).
- Cadence-specific wiring (cookie session to token) is not covered by Knock docs; the docs only say the backend mints the token.

## 3. Triggering workflows from the server

Source: [Triggering workflows via the API](https://docs.knock.app/send-notifications/triggering-workflows/api), [Workflows](https://docs.knock.app/concepts/workflows), [OpenAPI trigger endpoint](https://docs.knock.app/openapi.json).

### Call shape

Node SDK (README of [knock-node](https://github.com/knocklabs/knock-node)):

```ts
import Knock from '@knocklabs/node';
const client = new Knock({ apiKey: process.env['KNOCK_API_KEY'] });
const res = await client.workflows.trigger('dinosaurs-loose', {
  recipients: ['dnedry'],
  data: { dinosaur: 'triceratops' },
});
// res.workflow_run_id
```

REST: `POST /v1/workflows/{key}/trigger`. Runs asynchronously; returns only a `workflow_run_id` (UUID v4).

Payload fields: `recipients` (required), `actor`, `data`, `cancellation_key`, `tenant`, `settings`. The workflow key is the human-readable key set in the dashboard.

- `data`: any JSON object, 10 MB limit. Everything in `data` is copied into each generated feed item's `data`, so identifiers such as `{ project_id }` are available to the UI for deep links or filtering ([In-app APIs, passing through data](https://docs.knock.app/in-app-ui/api-overview#passing-through-data)).
- `actor`: a user or object that caused the event. It is attached to feed items (as `actors`) and excluded from receiving the notification when it is also a subscriber of an Object recipient.
- `tenant`: scopes feed items and can be used to filter the feed client-side ([Filtering feeds](https://docs.knock.app/in-app-ui/feeds/filtering-in-app-feeds)).

### One recipient vs many

- One: pass one identifier. Many: pass an array. FAQ states up to **1000 recipients per trigger**; above that the docs point to subscriptions.
- A recipient identifier is a string id, an Object reference `{ id, collection }`, or a full inline recipient object (identified before the run). A common mistake is passing id strings where inline identification expects objects.
- Per-recipient data: `$trigger_data` on each inline recipient, merged over the shared `data`.
- Each workflow run is executed for one recipient at a time; fan-out creates one run and one message per recipient. See [Workflows, Running workflows](https://docs.knock.app/concepts/workflows#running-workflows).
- Objects and subscriptions: triggering with an Object as recipient also runs for each subscriber ([Subscriptions](https://docs.knock.app/concepts/subscriptions); not read in detail).
- Bulk trigger endpoint: the FAQ says no bulk trigger API exists; use per-recipient trigger data instead. Triggering to an audience via the API is not supported either ([Audiences trigger FAQ](https://docs.knock.app/send-notifications/triggering-workflows/audiences) covers only audience-membership triggers, which fire for new members, not existing ones).
- Idempotency: optional `Idempotency-Key` header; a retry with the same key within 24 hours returns the original response, and mismatched parameters error.
- Cancellation: a `cancellation_key` allows cancelling a delayed or batched run. Trigger frequency ("once per recipient") can be configured on the workflow.
- Errors: inactive workflow returns `workflow_inactive`; data failing a configured JSON schema returns `422` ([Validating trigger data](https://docs.knock.app/designing-workflows/validating-trigger-data)). The Node SDK retries 408, 409, 429 and 5xx twice by default.
- Rate limits are per endpoint, scoped per environment (and per user when a signed user token is used). The tier numbers shown on the docs page did not render in the raw Markdown, so the per-endpoint limits for trigger and feed are not confirmed `[unverified]`. Source: [Rate limits](https://docs.knock.app/api-reference/overview/rate-limits).
- Preferences and opt-outs are evaluated automatically by Knock (out of scope for this effort, noted only because it applies to every trigger).

### Templates: authoring and versioning

Source: [In-app feeds integration page](https://docs.knock.app/integrations/in-app/feeds), [Version control overview](https://docs.knock.app/version-control/overview), [Commits](https://docs.knock.app/version-control/commits), [Workflows](https://docs.knock.app/concepts/workflows), [Knock CLI](https://docs.knock.app/developer-tools/knock-cli).

- A workflow (identified by `key`) contains steps; an in-app feed channel step holds the template. Templates exist only inside their step and cannot be shared across workflows.
- Template content is written in the dashboard template editor using Liquid; the in-app template body is Markdown (CLI file `in_app_feed_1/markdown_body.md`). Variables come from `data`, `recipient`, `actor`, `tenant`, and others ([Variables](https://docs.knock.app/template-editor/variables)).
- Content cannot be changed after Knock generates a message. To fix an outdated message: archive it and trigger again ([FAQ](https://docs.knock.app/integrations/in-app/feeds)).
- Trigger data validation: optional JSON schema on the trigger step; also used for generating TypeScript types via the CLI ([Type safety](https://docs.knock.app/developer-tools/type-safety)).
- A template render error (for example missing data) is retried up to 3 times and then halts the run ([Testing and debugging templates](https://docs.knock.app/template-editor/testing-and-debugging)).
- Versioning: every change starts as a draft; a **commit** makes it live and records it in a commit log with revert and per-version history. The API always runs the last *committed* version; the dashboard test runner uses the last *saved* version.
- Environments: new accounts start with a single environment (production). Accounts created before October 2026 start with development and production and the commit-and-promote flow. Enabling environments is a one-way switch. Branches (`knock branch`) isolate changes per developer or feature once environments are on ([Version control overview](https://docs.knock.app/version-control/overview), [Branches](https://docs.knock.app/version-control/branches)). The docs say accounts created before October 2026 get both; by that wording an account created now starts with a single environment.
- Code vs dashboard: workflows can be managed in the dashboard, the Management API, or the CLI. The CLI (`npm i -g @knocklabs/cli`) pulls and pushes workflows as files: `workflows/<key>/workflow.json` plus `in_app_feed_1/markdown_body.md`. `knock workflow push --all --commit` upserts and commits ([Push](https://docs.knock.app/cli/workflow/push), [File structure](https://docs.knock.app/cli/workflow/file-structure)). A CI recipe using `knock push`, `knock commit promote` and branches is in [Integrating into CI/CD](https://docs.knock.app/tutorials/integrating-into-cicd). Auth for CI uses a service token ([CLI page](https://docs.knock.app/developer-tools/knock-cli)).
- Workflow status (Active or Inactive) works as a kill switch; it is separate from commits.
- Translations (i18n) are listed as an Enterprise-plan feature on the pricing page; `locale` on the feed API is "only available for enterprise plan customers using custom translations" (OpenAPI).

## 4. Feed behaviour

### Realtime

Source: [In-app APIs and real-time service](https://docs.knock.app/in-app-ui/api-overview#real-time-delivery), [Socket behavior](https://docs.knock.app/in-app-ui/feeds/socket-behavior), [Feed page](https://docs.knock.app/in-app-ui/react/feed#socket-behavior-options).

- Managed websocket service run by Knock; the SDK handles connection lifecycle and reconnects. In the JS client, call `feed.listenForUpdates()`; the React provider does this for you.
- The only documented event is `new-message`. It does not carry the message content, only updated badge-count metadata, and "might be throttled". The client then fetches the items.
- Options: `auto_manage_socket_connection: true` disconnects inactive tabs after `auto_manage_socket_connection_delay` ms (default 2000); `__experimentalCrossBrowserUpdates` syncs tabs over a BroadcastChannel.
- Is the socket subject to the public key and user token check? Re-authenticating restarts the connection (see section 2); further detail not documented `[unverified]`.

### Statuses

Source: [Message statuses](https://docs.knock.app/send-notifications/message-statuses), [In-app feeds integration page](https://docs.knock.app/integrations/in-app/feeds#understanding-message-statuses).

- Engagement statuses are independent timestamps, not a single state: `seen_at`, `read_at`, `interacted_at`, `archived_at` (plus `link_clicked_at`). A message may have several.
- **Seen**: retrieved or displayed in the feed. The prebuilt popover marks all items seen on open. **Read**: explicitly opened or marked; in the default UI by clicking a cell or "mark all as read". **Interacted**: clicked the cell or a button. **Archived**: removed from the default feed list.
- Default bell badge counts unseen; `badgeCountType="unread"` counts unread. Opening the popover marks seen by default; passing a custom `onOpen` can mark read instead.
- Archived items are excluded from the feed by default. `archived=only|include|exclude` on the API retrieves them. The prebuilt UI offers no way for a person to view archived items.
- All statuses can be unset again (`markAsUnseen`, `markAsUnread`, `markAsUnarchived`). Batch and "mark all" endpoints exist (`/v1/messages/batch/{read,seen,archived,...}`). `markAllAsArchived` applies to all items in the current scope.
- The docs describe "interacted" only briefly; its relation to read is not spelled out `[ambiguous]`.

### Pagination

- Cursor-based. `GET /v1/users/{user_id}/feeds/{id}` takes `page_size` (default 50), `after`, `before`, and returns `entries`, `meta` (counts), `page_info` ({ `after`, `before`, `page_size` }), and `vars`. Source: [OpenAPI](https://docs.knock.app/openapi.json), [API reference](https://docs.knock.app/api-reference/users/feeds/list_items).
- In the JS client and React store, `feed.fetchNextPage()` appends the next page; `pageInfo` is in the store ([FeedStoreState](https://docs.knock.app/in-app-ui/javascript/sdk/types/feed-store-state)).
- The feeds overview page says nothing about pagination itself `[ambiguous]` (it only says results are reverse chronological). The API and SDK pages above are the source.

### Filtering and ordering

Source: [Filtering feeds](https://docs.knock.app/in-app-ui/feeds/filtering-in-app-feeds), OpenAPI.

- Server-side filters: `status` (unread, read, unseen, seen, all), `archived`, `source` (workflow key, a single value only), `tenant`, `has_tenant`, `workflow_categories`, `trigger_data` (top-level keys, exact match, AND logic, single value per key), `inserted_at.{gt,gte,lt,lte}`.
- Filters passed in `defaultFeedOptions` apply to real-time updates and badge counts too.
- Order is reverse chronological and not configurable. The overview page also says responses can be sorted by priority `[ambiguous]`; its FAQ and the API parameter list show no sort option.

### Retention

Source: [Data retention](https://docs.knock.app/manage-your-account/data-retention), [Pricing](https://knock.app/pricing).

- Feed items are available indefinitely: "In-app feed notifications are available indefinitely", even after message logs expire. Status updates on old items still work. The pricing page lists "Feed retention: Unlimited ... No limit, but can be configured" for all three plans. The docs did not say where or how it can be configured `[unverified]`.
- Message logs, workflow run logs, audit logs, and webhook delivery logs are kept 30 days on non-Enterprise plans (90 on Enterprise). API logs are 30 days for all plans. Once expired they vanish from the dashboard and Management API.

## 5. Free tier limits

Source: [Knock pricing page](https://knock.app/pricing) (read 2026-11-10; vendor page, subject to change). The plan is called **Free** on the page; the ticket calls it "Developer". No plan named "Developer" appears on the page `[ambiguous]`.

Included monthly on Free ($0, no card required):

- **10,000 messages sent.** A message is "A single message delivered to a single user on a single channel. Messages that bounce or fail to send are not included." Under this definition, one event fanned out to N recipients on the in-app channel counts as N messages.
- 500 guide recipients (not relevant to feeds); 500 agent credits.
- Unlimited stored profiles, workflows, broadcasts, in-app guides, delivery channels, team members, environments.
- Includes in-app feed, prebuilt feed components, Feed API, custom feed filtering, unlimited feed retention, batching, preferences, scheduled workflows, API and CLI management, version control, Google SSO, 99.9% uptime SLA stated on the page, SOC 2 Type 2, GDPR, CCPA.
- 30-day log retention.
- Not on Free: removing Knock branding, email client previews, translations (i18n), per-tenant branding and preferences, custom roles, audit logs, SAML, SCIM, observability streaming, priority support.
- Overage: "We don't hard cap your messages." Over the limit, Knock contacts you about a billing method for the next month. No Free-plan per-message overage price is stated; Starter ($250/month) lists 50,000 messages then $0.005 each.
- The pricing page does not state API rate limits per plan. See the rate-limit note in section 3.
- The "Powered by Knock" branding note on the Free plan refers to email; whether the in-app feed components carry branding was not found `[unverified]`.

## 6. Local dev and tests without keys

### What Knock documents

- **No offline mock, emulator or local server** is documented for Knock. A search of `llms.txt` found no such page `[unverified]` beyond that search.
- **Sandbox mode** exists per channel, but "Sandbox mode is supported across all channel types (except for in-app)" ([Integrations overview, sandbox mode](https://docs.knock.app/integrations/overview#sandbox-mode)). The trigger `settings.sandbox_mode: true` and the test-runner toggle are documented as forcing "all messages" in the run to sandbox ([Triggering via API, trigger settings](https://docs.knock.app/send-notifications/triggering-workflows/api#testing-workflows-with-trigger-settings), [Testing workflows](https://docs.knock.app/send-notifications/testing-workflows)). The two statements conflict for the in-app channel `[ambiguous]`; whether an in-app feed message is still created under sandbox is not stated.
- Other trigger setting: `skip_delay: true` skips delay steps.
- **Dashboard test runner**: runs a whole workflow or a single step for a chosen recipient, actor, tenant and data, without any app code. Uses the last saved (not committed) workflow version. Trigger frequency is bypassed. Debugging: workflow debugger, API logs and message logs (30-day retention) ([Debugging workflows](https://docs.knock.app/send-notifications/debugging-workflows)).
- **Environments**: separate keys, data, users and enhanced security setting per environment. The docs suggest a development environment with enhanced security off for building. Free plan lists unlimited environments; enabling environments is one-way ([Environments](https://docs.knock.app/version-control/environments)).
- Without enhanced security, only the public key and a user id are needed in the browser; with it, a signing key and a minted token are needed ([section 2](#2-identity-and-auth)).
- **Node SDK seams** (from the `@knocklabs/node` 1.41.0 typings and README): `new Knock({ apiKey, baseURL, fetch, maxRetries, timeout, branch })`; `baseURL` also reads `KNOCK_BASE_URL`; a custom `fetch` can be supplied. The docs do not describe these as a testing feature. `signUserToken` fails if `KNOCK_SIGNING_KEY` is missing ("No signing key provided...").
- **Browser SDK seam**: `KnockProvider` has `host` (API host override), `enabled` (when false: no network, no socket) and `logLevel`. `KnockProvider` requires `apiKey`; the `enabled` flag is the documented way to render without connecting.
- Type-safe triggers: with the CLI, trigger data schemas can generate TypeScript types ([Type safety](https://docs.knock.app/developer-tools/type-safety)). Not read in detail.
- Branches (`knock branch`, Node `branch` option) isolate workflow edits; they require environments.
- No documented way to read back "what was sent" in a test other than calling the Knock API (for example the messages list) or the dashboard logs `[unverified]`.

### Comparison point with `app/lib/mailer.server.ts` (Cadence side)

This is a statement about what exists, not a recommendation. The mailer chooses a transport by whether `POSTMARK_SERVER_TOKEN` is set; with no token it writes JSON to `.outbox/`. Knock documents no equivalent built-in "no key, write to disk" mode. Where Knock exposes a configuration seam (`baseURL`, custom `fetch`, `enabled`), it is documented as an SDK option, not as a test mode.

## 7. Gaps and open questions raised

1. Does a numeric Cadence id work as a Knock user id, or must it be stringified? Docs type it as string.
2. Does a feed message get created in sandbox mode for the in-app channel? Docs contradict themselves.
3. What are the real per-endpoint rate limits for trigger and feed reads? The tier table did not render in the raw page.
4. Does the prebuilt feed render correctly under React Router 8 SSR? Only Next.js and TanStack entry points are shipped.
5. Does the Action URL redirect do a client-side navigation, or a full page load?
6. How is Knock's "feed retention" configured, given the pricing page says "can be configured"?
7. Behaviour of the feed UI package and styling alongside Tailwind and shadcn/ui is undocumented.
8. Whether Knock's "single environment by default" wording applies to a free account created today (docs say accounts created before October 2026 start with development and production); not confirmed against a live signup.
