# Codexgram — Implementation Plan

Status: planning complete — nothing implemented yet. Work through §7 step by step; every
checkbox starts unchecked.

This plan follows the structure and scope of
[burakorkmez/codexgram `PLAN.md`](https://github.com/burakorkmez/codexgram/blob/master/PLAN.md),
adapted to our constraints (no paid Apple/Google developer accounts yet, Android + iOS) and
the decisions we kept from our own interview. Differences from the reference are listed in §10.

## 1. Product and agreed scope

Build a polished Instagram-style demo for trusted testers, built to pass App Store / Play
Store review later.

### Chosen stack

- Expo SDK 57, React Native, TypeScript, Expo Router.
- Native tabs: **Home, Messages, Explore, Profile**.
- NativeWind for styling.
- Clerk for authentication and sessions (publishable key only in the client).
- Convex for application data, backend logic, media storage, and live updates.

### Platforms and testing

- **Android:** development build (`expo-dev-client`) built with EAS free tier, sideloaded as an APK.
- **iOS:** Expo Go until an Apple Developer account exists. Therefore v1 uses **only modules
  included in Expo Go**; anything else needs discussion first.
- Web is out of scope.

### Design

- Modern, clean, minimal, mobile-first.
- **Light and dark mode**, following the system setting.
- Visual reference in `design/` (see §3); tokens centralised so they apply consistently.

### Included

- Sign up / sign in with **Google**, **Apple** (browser-based OAuth) and **email + password**
  (6-digit email verification code, forgot-password reset code) through a custom welcome screen.
- Unique-username onboarding.
- **Single-image or single-video posts** with optional captions; **captions editable** after posting.
- Follow/unfollow, likes, comments with **one level of replies**.
- Delete your own posts and comments; **post authors can delete any comment on their posts**.
- One-to-one live text messages, with inbox search and an Unread filter.
- **Activity** list (likes, comments, replies, follows) behind a bell icon in the Home header.
- View/edit profiles.
- Clearly labeled fictional demo content (seed profiles).
- **Store compliance (built last):** accept Terms, report content/users, block users, in-app
  account deletion.

### Excluded

Web delivery, private accounts, follow requests, stories, carousels, bookmarks/saves, in-app
capture/editing, comment likes, "liked by" lists, push notifications, group chats, message
attachments/editing/deletion, typing indicators, read receipts, payments, background
uploads, persistent offline drafts, full UI automation, admin UI (reports reviewed in the
Convex dashboard), Explore category chips, People/Posts search toggle, composer extras
(location, tagging), profile location/website fields, a "Create" tab, native one-tap
Google/Apple buttons.

The demo is free.

## 2. Starting point and implementation constraints

The repository contains a minimal Stack layout and placeholder screen, ESLint (flat config),
iOS `bundleIdentifier` / Android `package` `com.kusalm.codexgram`, an EAS project with
`development` / `preview` / `production` profiles, and the `design/` reference images.

Declared versions:

| Package | Current project declaration |
| --- | --- |
| Expo | `~57.0.26` |
| Expo Router | `~57.0.24` |
| React | `19.2.3` |
| React Native | `0.86.3` |
| TypeScript | `~6.0.3` |

Candidate versions (verify when installing; always via `npx expo install`):

- NativeWind `4.2.7` (stable, adds SDK 57 support) with **Tailwind CSS v3**. Do not use NativeWind v5 (RC).
- `@clerk/expo` — current version at install time.
- `convex` — current version at install time.

Rules:

- Read the [Expo SDK 57 documentation](https://docs.expo.dev/versions/v57.0.0/) before writing code
  that touches Expo APIs. Use current [Clerk Expo](https://clerk.com/docs/expo/getting-started/quickstart),
  [Convex Clerk integration](https://docs.convex.dev/auth/clerk), and
  [NativeWind installation](https://www.nativewind.dev/docs/getting-started/installation) docs.
  Items marked **(verify)** are version-sensitive.
- Never edit generated `ios/` / `android/` folders; configure via `app.json` and config plugins.
- Do not silently change the selected stack or downgrade Expo.
- Definition of done for every step: `npx expo lint` and `npx tsc --noEmit` pass and the feature
  works on the Android dev build and iOS Expo Go. Commit at the end of each step.

Confirmed Expo Go availability (SDK 57 docs): `expo-image-picker`, `expo-video`.
Native tabs on SDK 57 import from `expo-router/unstable-native-tabs` **(verify in Expo Go)**.

### Authentication constraints

- Client only gets `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`. Clerk's secret key never ships in the app.
- Browser-based OAuth via Clerk `useSSO` (`oauth_google`, `oauth_apple`) works in Expo Go and
  dev builds on both platforms. Clerk **development** instances use shared Google/Apple
  credentials — no Google Cloud project or Apple account needed for development.
- Clerk **production** instances require custom Google (Google Cloud) and Apple (Services ID,
  Key ID, Team ID, private key) credentials — needed before launch (§7 Step 11).
- Clerk links an OAuth sign-in to an existing account with the same **verified** email.
  Apple "Hide My Email" users get a relay address and therefore a separate account.

### Environment variables

| Name | Where | Value |
| --- | --- | --- |
| `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY` | `.env.local` | Clerk dashboard → API keys |
| `EXPO_PUBLIC_CONVEX_URL` | `.env.local` | Written by `npx convex dev` |
| `CLERK_JWT_ISSUER_DOMAIN` | Convex dashboard env (dev + prod) | Clerk Frontend API URL **(verify)** |

`.env*.local` is gitignored.

## 3. Design reference

`design/` holds the visual reference (from
[burakorkmez/codexgram](https://github.com/burakorkmez/codexgram/tree/master/design), MIT).
Match layout, spacing, and components — **except** features excluded in §1.

| File | Use for |
| --- | --- |
| `app-design-ref.png` | Overview of every screen and state |
| `design-system-ref.png` | Tokens and components |
| `auth-screen-ref.png` | Step 2 |
| `home-screen-ref.png`, `post-detail-ref.png`, `comments-ref.png` | Steps 4–5 |
| `profile-screen-ref.png`, `edit-profile-ref.png` | Steps 3, 5 |
| `explore-screen-ref.png` | Step 5 |
| `messages-tab-ref.png`, `chat-screen-ref.png` | Step 6 |
| `settings-screen-ref.png` | Steps 2, 9 |

### Tokens (from `design-system-ref.png`)

| Token | Light | Use |
| --- | --- | --- |
| `primary` | `#3B82F6` | buttons, links, active states, own message bubbles |
| `text` | `#0F172A` | primary text |
| `secondary` | `#64748B` | secondary text, icons |
| `border` | `#E5E7EB` | dividers, input borders |
| `surface` | `#F8FAFC` | cards, inputs, secondary buttons, others' bubbles |
| `success` | `#22C55E` | "available", success |
| `warning` | `#F59E0B` | warnings |
| `destructive` | `#EF4444` | delete, report, errors |
| `info` | `#0EA5E9` | info / demo-content notices |

Dark values are not in the reference — derive them (e.g. slate-950 background, slate-900
surface, slate-800 border, slate-50 text, slate-400 secondary, same primary) and check contrast.

Typography (system font / SF Pro; Inter on Android optional — decide in Step 1):
H1 34 Bold · H2 28 Semibold · H3 22 Semibold · Body 17 Regular · Caption 15 Regular · Label 13 Medium.
Spacing: 4, 8, 12, 16, 24, 32, 48 (8pt grid). Icons: 24px outlined, rounded strokes.
Buttons: Primary (filled), Secondary (surface), Ghost (outline), Destructive (filled red).

## 4. Screens and user journeys

### Authentication and onboarding

1. Show a custom welcome screen with **Continue with Google**, **Continue with Apple**, and
   **Continue with email**.
2. Google/Apple use a combined sign-up/sign-in flow (one button handles new and returning users).
3. Email: sign-in and create-account forms; new accounts verify a 6-digit email code;
   forgot-password sends a reset code then sets a new password.
4. After authentication, resolve the corresponding Convex profile.
5. New users choose a unique username before entering the tabs.
6. Prefill an editable display name and avatar when the provider supplies them.
7. Photo and bio remain optional.
8. Returning users enter Home after session/profile loading completes.
9. Sign-out from Profile → Settings.

Handle cancellation, provider errors, wrong password, email already taken, invalid/expired
code, missing provider profile fields, session expiry, and interrupted onboarding.

### Home and posting

- Home shows the **current user's posts and followed users' posts**, newest first.
- Paginate results; pull to refresh.
- Header: logo/wordmark, **+** (opens composer), **bell** (opens Activity, unread dot).
- Select one image or video from the device library.
- Add an optional caption and publish.
- Show upload progress and prevent duplicate submissions.
- Publish only after successful upload and validation.
- Keep the composer open during upload; warn before abandoning an active upload.
- Failed uploads expose retry; unused uploads are cleaned up.
- Post authors can **edit the caption** (shows "edited") or delete the post.

Limits (enforced on client **and** backend):

- Images: maximum 10 MB.
- Videos: maximum 30 seconds and 50 MB. (`videoMaxDuration` only limits recording, so check
  the picked asset's `duration` — milliseconds — ourselves.)
- No automatic compression or video-processing service.
- Media keeps its original aspect ratio, displayed clamped between 4:5 portrait and 1.91:1
  landscape (store `width`/`height`).
- Videos show a preview, play inline on tap (`expo-video`), and stop when offscreen or when
  the app goes inactive.

Empty Home offers discovery (Explore) and post creation.

### Explore and profiles

- Profiles and posts are visible only to signed-in members; no anonymous browsing.
- Explore shows all members' posts, newest first, in a grid, with user search.
- Search usernames and display names; empty search shows "No results found".
- Selecting a result opens the member's profile.
- Profiles show avatar, username, display name, bio, post grid, and posts/follower/following
  counts; follower and following lists are tappable.
- Other profiles expose Follow/Unfollow and Message.
- Following takes effect immediately; there are no approval requests.
- Own profile exposes editing for all four profile fields (avatar, display name, username, bio).
- Username changes preserve relationships through stable IDs.

### Likes, comments, and deletion

- One like per user per post; support unlike; tap heart or double-tap media.
- Comments are displayed chronologically, with **one level of replies**: replying to a reply
  attaches to the same top-level comment with `@username` prefilled; replies collapse under
  "View N replies".
- Members can delete their own comments and posts; **a post author can also delete any
  comment on their post**.
- Deleting a top-level comment also deletes its replies.
- Confirm destructive actions.
- Removing a post also removes associated likes, comments, notifications, and media.

### Messages

- Any real member can start a one-to-one conversation from another member's profile.
- Reuse the existing conversation for that pair; no self-chat.
- Messages tab lists conversations by latest message, with **member search** and an
  **All / Unread** filter; unread count badge on the Messages tab.
- Conversation screens show paginated history and live text updates.
- Show pending and failed sends; allow manual retry without duplicates.
- Only participants can access a conversation.

No groups, attachments, typing indicators, read receipts, or message editing/deletion.

Fictional seed profiles cannot authenticate or reply; show a clear notice ("Fictional demo
content") before and inside such a chat.

### Activity

- Bell icon in Home header with an unread dot.
- List of likes, comments, replies, and follows on your content, grouped Today / This week / Earlier.
- Tap → post or profile. Opening the screen marks all as read.
- No notification for acting on your own content; undo (unlike, unfollow, delete) removes it.

### Settings and store compliance (Step 9)

- Settings: sign out, blocked accounts, Terms & Privacy links, contact email, delete account.
- Accept Terms at onboarding (stored timestamp).
- Report a post, comment, or profile with a reason (bottom-sheet menu).
- Block a user: hides their posts, comments, and profile from you and prevents DMs both ways.
- Delete account: confirm → delete all the user's data and media in Convex → delete the Clerk
  user from the client → signed out.

## 5. Data, interfaces, and backend responsibilities

Clerk owns authentication identity and sessions. Convex owns editable app profiles and all
social data.

| Entity | Minimum information |
| --- | --- |
| Profile (`users`) | Stable ID, optional Clerk ID (absent for seed profiles), unique normalized username, display name, avatar reference, bio, demo marker, denormalized counts, terms-accepted time |
| Post | Author ID, media storage reference, media type (`image`/`video`), width, height, video duration, optional caption, edited time, like/comment counts, creation time |
| Follow | Follower ID, followed-user ID |
| Like | User ID, post ID |
| Comment | Post ID, author ID, optional parent (top-level) comment ID, text, creation time |
| Conversation | Canonical participant pair (sorted), latest-message time and preview, per-participant last-read time |
| Message | Conversation ID, sender ID, text, creation time, client-generated retry/deduplication ID |
| Upload | Owner ID, storage reference, intended use (`post`/`avatar`), lifecycle state (`pending`/`attached`/`abandoned`), creation time |
| Notification | Recipient ID, actor ID, type (`like`/`comment`/`reply`/`follow`), optional post/comment ID, read flag |
| Report | Reporter ID, target type/ID, reason, details, status |
| Block | Blocker ID, blocked ID |

Text limits: username `^[a-z0-9._]{3,30}$` (normalized lowercase), display name 1–50,
bio ≤ 150, caption ≤ 2,200, comment 1–500, message 1–1,000.

### Backend interfaces

Use typed Convex queries and mutations (`v.*` validators) for profile onboarding/editing,
feeds, search, follows, likes, comments, uploads, post publication/editing/deletion,
conversations, messages, activity, reports, blocks, and account deletion.

- Derive the acting user from verified authentication (`ctx.auth.getUserIdentity()`).
- Never trust a client-supplied author or sender identity.
- Enforce ownership and conversation membership on the backend.
- Enforce unique usernames, follow pairs, like pairs, and conversation pairs transactionally.
- Use stable IDs rather than usernames for relationships.
- Authenticate upload creation, publication, and media access.
- Validate text, media metadata (type, size, duration), and resource existence.
- Make retries safe for publication (an upload can be attached once) and message sending
  (dedupe by client ID).
- Clean up abandoned uploads and deleted media through retry-safe scheduled work (Convex cron).
- Bootstrap profiles idempotently after authentication; app-profile edits remain authoritative in Convex.
- Apply blocks in feed, explore, search, profile, comment, and message queries.
- Feed strategy: fan-in on read (following IDs + own ID, paged by recency). Acceptable at
  demo scale; switch to fan-out-on-write if it becomes slow.

No separate REST server, billing service, analytics service, or video-processing service is required.

## 6. State, failure behavior, and operating defaults

- Convex is the source of truth; use live subscriptions where appropriate.
- Keep already loaded content visible during connection loss.
- Show connection status (offline banner) and actionable errors.
- Optimistic likes/follows must roll back on failure.
- Retain unsent text in the active screen for retry.
- Do not promise draft recovery after app termination.
- Stop media playback when offscreen or the app becomes inactive.
- Handle library-picker cancellation and unavailable media gracefully.
- Request only permissions required for selected features.
- Every list has loading, empty, and error states.
- Accessibility: labels on icon-only buttons, 44pt touch targets, contrast in both themes.
- Use development diagnostics and Convex logs without recording tokens, private message
  bodies, or unnecessary personal data.
- Start with a development environment (Clerk dev instance, Convex dev deployment).

## 7. Step-by-step implementation checklist

### Step 1 — Verify foundation

- [ ] Preserve existing repository changes; create Clerk (dev) and Convex projects; fill `.env.local`.
- [ ] Verify SDK/package compatibility; confirm current free-tier limits (Clerk, Convex, EAS).
- [ ] Configure NativeWind 4.2.7 + Tailwind v3 with light **and dark** design tokens from §3.
- [ ] Establish the four native tabs and supporting stack/modal navigation (fallback: JS tabs if native tabs fail in iOS Expo Go).
- [ ] Build and install the Android development build; run iOS in Expo Go.

**Complete when:** The app launches on both platforms with functioning native navigation and styling in both themes.

### Step 2 — Configure authentication

- [ ] Configure Clerk (Google, Apple, email + password, email-code verification).
- [ ] Add the custom welcome screen, Google/Apple (`useSSO`) and email sign-in/sign-up, verification, and forgot-password screens.
- [ ] Configure secure session persistence (`expo-secure-store`) and protected navigation.
- [ ] Add sign-out and authentication failure states.

**Complete when:** All three methods work on Android dev build and iOS Expo Go, and sessions survive restart.

### Step 3 — Establish Convex and profiles

- [ ] Configure Clerk authentication in Convex (`convex/auth.config.ts`).
- [ ] Add data definitions, indexes, and authorization helpers.
- [ ] Implement idempotent profile creation and username onboarding.
- [ ] Implement profile editing and retrieval.

**Complete when:** Two real users have distinct profiles and cannot edit each other's data.

### Step 4 — Build media and posts

- [ ] Add library selection and media validation (type, size, duration).
- [ ] Implement upload tracking, progress, publication, retry, and abandonment behavior.
- [ ] Implement post display, video previews, inline playback, caption editing, and deletion.
- [ ] Add storage cleanup (cron for abandoned uploads, cascade on delete).

**Complete when:** Supported images/videos publish and play, and failed uploads never create visible incomplete posts.

### Step 5 — Build discovery and social interactions

- [ ] Implement follow/unfollow and follower/following lists.
- [ ] Implement Home, Explore, search, and profile grids.
- [ ] Add likes (incl. double-tap), comments with one level of replies, and comment deletion rules.
- [ ] Add pagination, empty states, and missing/deleted-content handling.

**Complete when:** Two accounts can discover each other and complete every social interaction.

### Step 6 — Build messaging

- [ ] Create/reuse conversations from profiles.
- [ ] Implement ordered chat lists with search, Unread filter, and tab badge; paginated live conversations.
- [ ] Add pending/failed sends and duplicate-safe retries.
- [ ] Verify participant-only access.

**Complete when:** Two devices exchange messages live without duplicate conversations or retry-generated messages.

### Step 7 — Build activity

- [ ] Create/remove notifications inside like, comment, reply, and follow mutations.
- [ ] Build the Activity screen and the bell unread dot; mark all read on open.

**Complete when:** Each action creates exactly one notification for the right person and undo removes it.

### Step 8 — Seed and polish

- [ ] Add an idempotent development-only seed routine.
- [ ] Source licensed sample imagery and a short video for a few fictional profiles.
- [ ] Clearly mark fictional profiles and their messaging limitations.
- [ ] Apply the design reference across all screens, in light and dark mode.
- [ ] Verify keyboard behavior, accessibility labels, contrast, and touch targets.

**Complete when:** The demo feels populated and polished without presenting fictional activity as real.

### Step 9 — Store compliance

- [ ] Terms acceptance at onboarding; Terms/Privacy pages and contact email in Settings.
- [ ] Report post, comment, and profile.
- [ ] Block/unblock users and a blocked-accounts list; apply blocks everywhere.
- [ ] In-app account deletion (Convex data + media, then Clerk user).

**Complete when:** Each feature works end-to-end and a deleted account leaves no orphaned data.

### Step 10 — Validate and deliver

- [ ] Run type checking and lint.
- [ ] Run focused backend permission and invariant tests (§8).
- [ ] Complete the two-account acceptance walkthrough on Android dev build and iOS Expo Go.
- [ ] Verify cancellation, connection loss, retries, app restart, and deletion.
- [ ] Document setup, environment variables, seeding, and known limitations.

**Complete when:** The app passes the agreed acceptance checks on real Android and iOS devices.

### Step 11 — Launch prep (when accounts exist)

- [ ] Apple Developer ($99/yr) and Google Play Console ($25) accounts.
- [ ] Clerk production instance with custom Google and Apple credentials.
- [ ] Convex production deployment and `CLERK_JWT_ISSUER_DOMAIN` for production.
- [ ] iOS development build / TestFlight testing; EAS production builds.
- [ ] Store listings, privacy labels / data safety, account-deletion web link (Google Play); `eas submit`.

## 8. Test scenarios

Automate critical backend cases (e.g. `convex-test` + Vitest **(verify)**):

- Unauthenticated access is rejected.
- Users cannot alter another user's profile, posts, or comments (except post authors deleting comments on their own posts).
- Nonparticipants cannot read or send conversation messages.
- Concurrent username claims cannot produce duplicates.
- Repeated likes/follows cannot produce duplicate relationships; counts stay correct.
- Concurrent chat initiation reuses one conversation.
- Message/publication retries do not duplicate content.
- Post deletion removes dependent records and schedules media cleanup.
- Deleting a comment removes its replies and keeps counts correct.
- Invalid or unauthorized uploads cannot be published; size/duration limits are enforced.
- Blocked users cannot message or see each other's content.
- Account deletion removes all of the user's data.

Manually verify:

- Google, Apple, and email first sign-up, returning login, verification code, password reset, cancellation, and logout.
- Missing provider name/avatar and interrupted onboarding.
- Image/video selection, size/duration boundaries, playback, and failed upload.
- Feed ordering, following changes, search, and pagination.
- Comment replies, caption editing, and activity entries.
- Empty screens and content removed while being viewed.
- Live chat across two accounts and failed-send retry.
- Connection loss, restoration, and session persistence.
- Layout, keyboard, safe areas, native tabs, and both themes on real Android and iOS devices.

## 9. Assumptions and unresolved risks

### Assumptions/defaults

- Working name: Codexgram.
- English only, one member role, no admin UI.
- No fixed deadline; stay within free tiers of Clerk, Convex, and EAS.
- Modest trusted-tester usage.
- Basic accessibility is included.
- Development-only environment initially.
- Deleting a post removes dependent comments, likes, notifications, and media.
- No separate analytics or alerting integration.
- Usernames can be changed; the old one is freed immediately.
- Counts are denormalized and updated in mutations.
- Avatar defaults to the provider photo; otherwise initials.
- Composer entry point is the `+` in the Home header (exactly 4 tabs).

### Open risks

- Native tabs (`unstable-native-tabs`) in iOS Expo Go are unverified — fallback JS tabs.
- NativeWind cannot style the native tab bar.
- OAuth redirect differs between Expo Go (`exp://`) and the dev build (`codexgram://`); both must work with Clerk.
- iOS testing is limited to Expo Go until an Apple Developer account exists.
- Production Google/Apple credentials are required at launch (Apple needs the paid account).
- Uncompressed images (≤10 MB) and videos (≤50 MB) make Convex storage/bandwidth the main free-tier risk.
- Original-video formats, previews, upload handling, and playback need real-device testing on both platforms.
- Member-only media access must be enforced beyond navigation; Convex file URLs are unguessable but not access-controlled — do not assume possession of a file URL proves authorization.
- Licensed sample assets remain to be selected.
- Fan-in feed may slow down at larger scale.
- Store review may ask for EULA, contact info, and moderation response details beyond Step 9.

## 10. Differences from the reference plan

| Area | Reference | This plan |
| --- | --- | --- |
| Platform | iPhone only, native dev build | Android dev build + iOS Expo Go |
| Sign-in | Native Google/Apple | Browser-based Google/Apple + email/password |
| Caption editing | No | Yes |
| Comments | Flat; authors delete only their own | One level of replies; post authors can delete any comment on their post |
| Activity | None | Bell-icon activity list |
| Theme | Light only | Light + dark |
| Store compliance | Excluded | Report, block, account deletion, terms (Step 9) |
| Launch | Internal demo only | Launch prep planned (Step 11) |
