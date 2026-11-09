# What can Knock's in-app feed do in React Router SSR?

Research for [#2](https://github.com/BenAffleck/aihero-cohort-005-project-fork/issues/2), part of the map [#1](https://github.com/BenAffleck/aihero-cohort-005-project-fork/issues/1). Checked on 2026-11-09.

## Answer

Knock's prebuilt React feed works in Cadence's stack. It server-renders without crashing, and Knock hosts the websocket that delivers live updates, so Cadence runs no real-time infrastructure.

Three things shape the integration:

1. **The feed is client-only data.** Server-rendered HTML has an empty feed and no badge. Items and counts load in the browser after hydration.
2. **`KnockProvider` does real work during server render.** Left at its defaults, it sends `PUT /v1/users/:id` to Knock on every SSR pass. Set `identificationStrategy="skip"` and identify users from the server instead.
3. **Do not sign a new user token on every layout loader run.** Layout loaders revalidate on every navigation, and a changed token tears down and rebuilds the socket.

## SDK versions

Latest on npm at the time of checking.

| Package                 | Version | Use                                                             |
| ----------------------- | ------- | --------------------------------------------------------------- |
| `@knocklabs/node`       | 1.41.0  | Server: trigger workflows, identify users, sign user tokens     |
| `@knocklabs/react`      | 0.15.1  | Browser: prebuilt feed components (peer range includes React 19) |
| `@knocklabs/react-core` | 0.17.1  | Providers and hooks, re-used by `@knocklabs/react`              |
| `@knocklabs/client`     | 0.23.1  | Underlying JS client, feed store and socket                     |
| `@knocklabs/cli`        | 1.5.2   | Workflows as code                                               |

`@knocklabs/react` has optional `next` and `@tanstack/react-router` peers and matching entry points. The `next` entry point holds a Guides location sensor rather than feed code. There is no React Router entry point, and none is needed for the feed.

## Components and headless use

- **Prebuilt:** `KnockProvider` (`apiKey`, `user`, `userToken`) wraps `KnockFeedProvider` (`feedId`), which wraps `NotificationIconButton` and `NotificationFeedPopover`, or the inline `NotificationFeed`. Import `@knocklabs/react/dist/index.css` unless replacing all styles. [Feed docs](https://docs.knock.app/in-app-ui/react/feed)
- **Headless:** `useKnockClient`, `useNotifications` and `useNotificationStore` return `items` and `metadata`. The caller must run `feedClient.fetch()` and mark items as seen, because only the prebuilt components do that automatically. [Headless docs](https://docs.knock.app/in-app-ui/react/headless/feed)
- **Placement in the sidebar:** `NotificationFeedPopover` positions with Popper using `strategy: "fixed"` and accepts a `placement` prop, default `bottom-end` (`@knocklabs/react` 0.15.1 source). A left sidebar will want a different placement.
- **Payload mode differs by entry point:** the REST API defaults to `rich` ([API reference](https://docs.knock.app/api-reference/users/feeds/list_items)), while `@knocklabs/client` defaults to `compact` (`clients/feed/feed.ts`). Compact omits nested arrays and objects from `data`, so custom cell rendering that reads nested trigger data needs `mode: "rich"`.

## SSR behaviour

Knock's docs do not cover SSR, hydration or React Router; the feed, headless, client and React overview pages say nothing on them. Everything below comes from the SDK source and a local experiment.

**Experiment.** Rendered `KnockProvider` > `KnockFeedProvider` > `NotificationIconButton` + `NotificationFeedPopover` with `react-dom/server` `renderToString` on Node 22 and React 19, through both the ESM and CJS builds of `@knocklabs/react` 0.15.1, with `host` pointed at a local stub server and fake credentials.

| `KnockProvider` props                     | Render | Requests during SSR | New timers |
| ----------------------------------------- | ------ | ------------------- | ---------- |
| `user` (default strategy), no token       | OK     | `PUT /v1/users/u1`  | 0          |
| `user` + `userToken`                      | OK     | `PUT /v1/users/u1`  | 0          |
| `identificationStrategy="skip"`           | OK     | none                | 0          |
| `enabled={false}`                         | OK     | none                | 0          |
| `skip` + `onUserTokenExpiring`            | OK     | none                | 1          |

What the source explains:

- `useAuthenticatedKnockClient` calls `knock.authenticate()` inside `useMemo`, so it runs on the server. The default `"inline"` strategy then sends an identify request whenever `user` is an object with an `id`. Errors are caught, so it does not fail the render, but it is an unintended Knock call on the render path.
- `onUserTokenExpiring` schedules a `setTimeout` inside `authenticate()`. The cleanup is in a `useEffect`, which never runs on the server, so each server render leaves a timer that fires shortly before token expiry and calls the callback there. Pass this callback only in the browser.
- `ApiClient` creates the Phoenix socket only when `window` and `WebSocket` exist. `Feed` checks for `self` before opening a `BroadcastChannel`. No socket or browser global is touched on the server.
- `feedClient.fetch()` is called from a `useEffect` in `NotificationFeed`, so feed data never loads on the server. The store starts empty, with counts of 0. The popover renders hidden but mounted, so the fetch happens on page load, not on first open.
- The package ships `"use client"`, which is inert in non-RSC React Router.

**Pre-filling the badge.** The store has no initial-data prop on the providers. `client.users.feeds.listItems` exists in `@knocklabs/node`, so a loader could fetch counts for a custom first-paint badge, at the cost of a Knock call in the layout loader. Since Notifications are best effort, the simpler default is no badge until the client has loaded.

## Real-time updates

- Knock runs a [managed websocket service](https://docs.knock.app/in-app-ui/feeds/overview), and the SDK connects to it itself. `useNotifications` calls `listenForUpdates()` automatically. **No custom infrastructure is needed.**
- The socket is opened in the browser, directly from there to Knock. Cadence's single Node server plays no part.
- By default the client disconnects the socket when the page is hidden (`disconnectOnPageHidden`, `api.ts`).
- Cross-tab sync through `BroadcastChannel` sits behind `__experimentalCrossBrowserUpdates`. [Feed docs](https://docs.knock.app/in-app-ui/react/feed)

## Authentication

- **Enhanced security mode is off by default.** While off, a request with only the public key can act as any user. Knock recommends enabling it before release. [Security docs](https://docs.knock.app/in-app-ui/security-and-authentication)
- With it on, the server signs a per-user RS256 JWT with a private signing key and passes it to the browser. `signUserToken(userId, { expiresInSeconds })` comes from `@knocklabs/node/lib/tokenSigner`, defaults to one hour, and reads `KNOCK_SIGNING_KEY`. Pass the result as the `userToken` prop.
- The signing key and the secret API key stay on the server. `@knocklabs/client` throws if given an `sk_` key.
- `userToken` belongs on `KnockProvider`. The feed docs mention passing it to `KnockFeedProvider`, but that provider has no such prop in 0.17.1.

**Loader interaction.** In framework mode with SSR, [loaders revalidate after all navigations and form submissions](https://reactrouter.com/start/framework/route-module). `signUserToken` sets `iat` and `exp` from the current time, so a fresh signature per loader run produces a different token each time. A changed token makes `authenticate()` tear down and rebuild the feed connections. The token needs to stay stable, for example by caching it until near expiry or by loading it outside the layout loader. The `enabled` prop exists for tokens that arrive asynchronously.

## Read and unread state

- Statuses are held by Knock per message: seen, read, interacted, archived, and link clicked. They can be combined. Seen means shown in the feed at least once; read is an explicit interaction. [Message statuses](https://docs.knock.app/send-notifications/message-statuses)
- The feed response returns `total_count`, `unread_count` and `unseen_count`. [API reference](https://docs.knock.app/api-reference/users/feeds/list_items)
- The badge counts unseen by default, and `badgeCountType="unread"` switches it. Opening the popover marks items as seen by default; use `onOpen` to mark as read instead. [Feed docs](https://docs.knock.app/in-app-ui/react/feed)
- `feedClient` has `markAsSeen`, `markAsRead`, `markAsArchived` and `markAsInteracted`, plus unset and mark-all variants. Mark-all updates the local store at once and is processed on the server later. [FeedClient docs](https://docs.knock.app/in-app-ui/javascript/sdk/feed-client)

## Workflows

- Workflows can live in code or the dashboard. The CLI has `knock workflow pull` and `push`, `knock commit`, and `knock commit promote --to=<env>`; `knock.json` sets the resources directory; a service token supports CI. [CLI docs](https://docs.knock.app/cli)
- The Management API upserts with `PUT /v1/workflows/{key}` and JSON bodies, with `commit=true` to commit in the same request. [Management API](https://docs.knock.app/mapi-reference/workflows/upsert)
- The server triggers with `client.workflows.trigger(key, { recipients, data })` from `@knocklabs/node`.

## Identifying users

- Identification is an upsert, by `PUT /v1/users/{id}`, inline in a workflow trigger, or in bulk. Only `id` is required. Knock identifies recipients before running the rest of the request, so a trigger can name a recipient not yet known to Knock, as long as recipients are passed as objects. [Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients)
- A feed request for a user Knock has not identified returns an empty feed rather than an error. [API reference](https://docs.knock.app/api-reference/users/feeds/list_items)
- This fits the SSR finding: inline identification from server-side triggers, with `identificationStrategy="skip"` on the client. Cadence's numeric user IDs must be converted to strings.

## Environments

- Knock's terms are **development** and **production**, not sandbox. Sandbox mode is a separate per-channel setting. An account starts with production only. Enabling environments adds development and [cannot be undone](https://docs.knock.app/concepts/environments).
- Users, messages and API keys are per environment, and data is never copied between them. Workflows move by commit and promote. Production stays directly editable unless "Modify resources in any environment" is turned off.
- Once environments are on, the CLI and Management API default to development, so production must be named explicitly.
- A [channel has the same ID in every environment](https://docs.knock.app/concepts/channels), though its settings are per environment.

## Retention

[In-app feed items are kept indefinitely](https://docs.knock.app/manage-your-account/data-retention), and status updates keep working after message logs expire. Message and workflow-run logs are kept for 30 days on non-Enterprise plans, after which the message no longer appears in the dashboard. Custom retention is not offered. This supports Knock being the only store of Notifications.

## Free plan

From [Knock's pricing page](https://knock.app/pricing). The plan is now called **Free**; "Developer" appears only in older Knock content.

- 10,000 messages per month, where a message is one delivery to one user on one channel.
- No hard cap: on overage, Knock contacts you to set up billing for the next month.
- In-app feeds are included: Feed API, prebuilt components, unlimited retention.
- Excluded: **removing Knock branding** and translations. The prebuilt feed shows "Powered by Knock" when the feed settings report `branding_required`.
- Isolated environments are listed as unlimited.
- 30-day log retention.
- [Rate limits are per endpoint, not per plan](https://docs.knock.app/api-reference/overview). Listing feed items is Tier 2, scoped per user and environment.

## Not confirmed

- **SSR support is not stated by Knock.** The behaviour above is from source and a `renderToString` test of these versions. It may change between SDK releases.
- **Not run in a browser or through Vite.** Hydration should match, since both sides start from an empty store, but this was not observed. Neither were the socket connection, CSS import under Vite, or Tailwind interaction. The app has no `app/entry.server.tsx`, so it uses React Router's default entry, which calls `renderToPipeableStream`. The test used `renderToString`; effects do not run on the server in either.
- **Free-plan real-time limits.** Neither the pricing page nor the docs state any concurrent connection or websocket limit, on any plan.
- **Maximum feed `page_size`.** The general API limit is 50, and the feed endpoint documents only the default of 50.
- **Client-side identify under enhanced security.** Not established whether the token grants it. Using `skip` avoids the question.
- **Live status sync across clients.** The docs do not say whether read or seen changes reach a user's other open clients in real time.
- **Stale docs.** The feed docs describe `auto_manage_socket_connection`, which does not appear in `@knocklabs/client` 0.23.1 source. Only `disconnectOnPageHidden` does.
- **Exact CLI directory format and CI promotion flow.** The CLI page implies workflows live in a repo but does not describe the on-disk layout. The map keeps promotion as not yet specified.
