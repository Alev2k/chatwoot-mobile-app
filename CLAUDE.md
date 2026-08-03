# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Chatwoot mobile app (agent-side) for iOS and Android. React Native 0.79 + Expo SDK 53 (prebuild workflow, New Architecture **disabled**), TypeScript strict, Redux Toolkit. Requires a Chatwoot server ≥ 3.13.0 (the app hard-warns below `EXPO_PUBLIC_MINIMUM_CHATWOOT_VERSION`).

## Commands

Package manager is **pnpm 10.11.0** (`packageManager` field is enforced; two dependencies are patched via `pnpm.patchedDependencies`, so npm/yarn installs will break the build).

```bash
pnpm install

# ios/ and android/ are NOT committed — generate them before the first native run
pnpm generate            # expo prebuild --clean
pnpm generate:soft       # expo prebuild (keeps existing native dirs)

pnpm start               # Metro with dev client
pnpm run:ios             # expo run:ios -d (device picker)
pnpm run:android
pnpm run:doctor          # expo-doctor

pnpm test                # jest (jest-expo preset)
pnpm test src/store/auth/specs/authSlice.spec.ts     # single file
pnpm test -- -t "should handle logout"               # single test by name
pnpm lint                # eslint . (prettier runs as an eslint rule)

pnpm storybook:ios       # on-device Storybook
pnpm storybook-generate  # regenerate .storybook/storybook.requires

pnpm build:ios / build:android          # EAS cloud build, production profile
pnpm build:ios:local                    # local EAS build with dotenv
```

Copy `.env.example` → `.env` first. Everything the JS bundle reads must be `EXPO_PUBLIC_*`; `app.config.ts` consumes these at config-resolution time (Sentry, Firebase `google-services` paths, EAS project id).

CI (CircleCI) runs only `jest --maxWorkers=2`.

## Architecture

### Request pipeline — the single most important file

`src/services/APIService.ts` is a singleton axios instance with two interceptors that do a lot of implicit work:

- **baseURL is not configured statically.** It is read per-request from `state.settings.installationUrl` — the server URL the user typed on the Config URL screen. There is no compile-time API host.
- **Account scoping is automatic.** Any URL passed to `apiService.get('conversations')` is rewritten to `api/v1/accounts/{state.auth.user.account_id}/conversations`. Only the routes in the `nonAccountRoutes` allowlist (`profile`, `profile/availability`, `notification_subscriptions`, `profile/set_active_account`) become `api/v1/{url}`. **Never pass a full path** — pass the account-relative one.
- **Auth headers** (`access-token`, `uid`, `client` — devise_token_auth) are injected from `state.auth.headers`.
- **401 anywhere dispatches `auth/logout`**, which the root reducer turns into a full state wipe. Any other error shows a generic toast — services generally don't need their own error UI.

Because the interceptors need the store and the store imports services, the store is injected at boot via `src/store/storeAccessor.ts` (`setStore`/`getStore`). Don't `import { store }` inside `src/services`.

### Store layout and slice convention

Redux Toolkit, persisted to AsyncStorage via redux-persist (`src/store/index.ts`). Two behaviors to know:

- `persistConfig.version` (`CURRENT_VERSION`) — bumping it **discards all persisted state**. That is the migration strategy; there are no per-slice migrations.
- The root reducer intercepts `auth/logout` and resets everything **except `settings`** (so the installation URL and locale survive logout).

Every feature under `src/store/<feature>/` follows the same five-file shape, and new code is expected to match it:

| File | Role |
|---|---|
| `<feature>Service.ts` | Static class, the only place that calls `apiService`. Also where snake_case → camelCase transformation happens. |
| `<feature>Actions.ts` | `createAsyncThunk`s wrapping the service; errors go through `rejectWithValue(response.data)`. |
| `<feature>Slice.ts` | `createSlice` + `createEntityAdapter` where the data is a list. |
| `<feature>Selectors.ts` | Typed selectors off `RootState`. |
| `<feature>Types.ts` | `*Payload` (request), `*APIResponse` (raw, snake_case), `*Response` (transformed, camelCase). |
| `specs/` | Colocated jest tests, one per file above. |

`src/store/reducers.ts` lists all 27 slices; note the conversation domain is split across many slices (`conversationSlice`, `conversationFilterSlice`, `conversationHeaderSlice`, `conversationSelectedSlice`, `conversationActionSlice`, `conversationTypingSlice`, `sendMessageSlice`, `audioPlayerSlice`, `localRecordedAudioCacheSlice`).

### The snake_case / camelCase boundary

The Chatwoot API speaks snake_case; the app's domain types (`src/types/`) are camelCase. Conversion is **not** global — it happens explicitly in service methods and in the ActionCable handlers via named transformers in `src/utils/camelCaseKeys.ts` (`transformConversation`, `transformMessage`, `transformContact`, `transformNotification`, …). Outgoing payloads are hand-written in snake_case.

The notable exception is `src/types/User.ts`, which stays snake_case (`account_id`, `avatar_url`, `identifier_hash`) because it comes straight from `auth/sign_in`. Expect both conventions when touching auth.

### Realtime (ActionCable)

`src/utils/baseActionCableConnector.ts` + `src/utils/actionCable.ts`. Subscribes to `RoomChannel` with `{pubsub_token, account_id, user_id}`, filters every inbound frame by `account_id`, and performs `update_presence` on a 20s interval. Handlers **dispatch store actions directly** (not thunks) — `message.created`, `conversation.created/updated/status_changed/read`, `assignee.changed`, `conversation.typing_on/off`, `contact.updated`, `notification.created/deleted`, `presence.update`. Typing state self-expires after 30s.

Connection is initialized once in `src/navigation/tabs/AppTabs.tsx`, which is also where all first-load fetches happen (profile, inboxes, labels, dashboard apps, custom attributes, device registration, Sentry/analytics identify).

### Navigation

`src/navigation/index.tsx` owns the `NavigationContainer` and a hand-rolled `linking` config that is doing three jobs at once:

1. Conversation deep links (`app/accounts/:accountId/conversations/:conversationId/...`).
2. **SSO callbacks** (`chatwootapp://auth/saml`) — intercepted in `getStateFromPath`/`getInitialURL`/`subscribe` and handed to `SsoUtils.handleSsoCallback`, returning `undefined` so no navigation occurs.
3. **FCM push** — `getInitialNotification` (cold start) and `onNotificationOpenedApp` (background) are translated into conversation URLs by `src/utils/pushUtils.ts`.

`AppTabs.tsx` switches on `selectLoggedIn` between `AuthStack` and the logged-in root stack. The Inbox/Conversations tabs are conditionally mounted based on `CONVERSATION_PERMISSIONS` (see `src/utils/permissionUtils.ts`). `ChatScreen`, `ContactDetails`, `Dashboard` and `SearchScreen` deliberately live in the **root stack, outside the tab bar**.

### Styling

`twrnc` (Tailwind for RN), not StyleSheet. Import `tailwind` from `@/theme` and call `tailwind('bg-blue-500 p-4')`. Colors come from `src/theme/colors/{light,dark}.ts` (Radix-style scales). Typography uses five Inter variable-font cuts registered by name in `src/navigation/index.tsx` and exposed as `font-inter-{400-20,420-20,500-24,580-24,600-20}` — use those class names rather than `fontWeight`.

### Path aliases

`tsconfig.json` maps **both** `@/*` and bare `*` to `src/*`, so `@/store/auth` and `i18n` / `constants/permissions` are all valid imports and appear interchangeably in the codebase. Jest mirrors only the `@/` alias plus `moduleDirectories: ['node_modules', 'src']`.

### i18n

`i18n-js` with 42 locales in `src/i18n/*.json`, synced from Crowdin (`crowdin.yml`) — **only edit `en.json`**; other files are overwritten by translation sync. Locale is a persisted user setting applied in `src/navigation/index.tsx`.

### Platform-specific files

`.ios.tsx` / `.android.ts` suffixes are used where behavior genuinely diverges: `ImageBubble`, `audioConverter` (ffmpeg-kit transcodes ogg/aac → wav for playback). `with-ffmpeg-pod.js` is a config plugin that patches the ffmpeg pod during prebuild; iOS uses `useFrameworks: 'static'`.

## Conventions

- Functional components only; named exports for components; directories kebab-case.
- Prefer `interface` over `type`; avoid enums (use const maps — see `src/constants/index.ts`).
- Message sending is optimistic: `createPendingMessage`/`buildCreatePayload` in `src/utils/messageUtils.ts` generate an `echoId` that the server echoes back so the pending row can be reconciled.
- Lists use `@shopify/flash-list`; sheets use `@gorhom/bottom-sheet` with refs shared through `src/context/RefsContext.tsx` (`useRefsContext`) rather than local state.
- Analytics events are declared in `src/constants/analyticsEvents.ts` and sent through `src/utils/analyticsUtils.ts`.
