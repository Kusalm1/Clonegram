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
- **Image carousel posts — 1 to 5 images** (square or portrait, compressed on device) with optional captions; **captions editable** after posting.
- Follow/unfollow, likes, comments with **one level of replies**.
- Delete your own posts and comments; **post authors can delete any comment on their posts**.
- One-to-one live text messages, with inbox search and an Unread filter.
- **Activity** list (likes, comments, replies, follows) behind a bell icon in the Home header.
- View/edit profiles.
- Clearly labeled fictional demo content (seed profiles).
- **Store compliance (built last):** accept Terms, report content/users, block users, in-app
  account deletion.

### Excluded

**Video posts (deferred until a development build that can compress video exists)**,
web delivery, private accounts, follow requests, stories, bookmarks/saves, in-app
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

Confirmed Expo Go availability (SDK 57 docs): `expo-image-picker`, `expo-image-manipulator`
(use `ImageManipulator.manipulate()` / `useImageManipulator` — `manipulateAsync` is deprecated).
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

### Assets

`assets/` includes the files from
[burakorkmez/codexgram `assets/`](https://github.com/burakorkmez/codexgram/tree/master/assets) (MIT).
The original source of the sample photos is unknown — fine for a demo, but replace them with
images you own or that are clearly licensed before a public store release.

**Branding**

| File | Size | Use | Step |
| --- | --- | --- | --- |
| `images/logo.png` | 1254×1254, glossy camera mark, transparent edges | Source for app icon, adaptive icon foreground, and splash image | 1 |
| `images/codexgram-mark.png` | 149×145, flat camera mark | Home header logo next to the "Codexgram" wordmark; welcome screen | 2, 4 |
| `images/auth-demo-img.png` | 1402×1122, mountain photo in a rounded blob | Welcome-screen hero illustration | 2 |
| `images/community-note.png` | 225×136, handwritten "Good people are built with great people." | Decorative note on welcome / empty states | 2, 8 |

**Seed content** (fictional demo profiles — Step 8; seed routine uploads them to Convex storage
through the same compression pipeline so they behave like real posts)

| Folder | Files | Use |
| --- | --- | --- |
| `images/feed/` | `alex`, `casey`, `jordan`, `maya`, `taylor` (avatars), `dog-avatar`, `dog`, `lake` | Fictional users' avatars and Home feed posts |
| `images/explore/` | `alex`, `casey`, `jordan`, `maya`, `taylor` (avatars), `santorini` | Explore grid / search results |
| `images/profile/` | `avatar`, `edit-avatar`, `photo-1` … `photo-9` | A fictional profile with a full 9-post grid; edit-profile preview |
| `images/chat/` | `avatar`, `lake` | Fictional chat partner avatar and shared-photo context |

**Template leftovers** (unused — delete in Step 8 polish): `expo-badge*.png`, `expo-logo.png`,
`react-logo*.png`, `tutorial-web.png`, `tabIcons/*` (native tabs use SF Symbols / Material icons).
`expo.icon/` is the iOS 26 icon bundle — regenerate from `logo.png` in Step 1.

**Icon & splash work (Step 1):**
- App icons must be square with **no transparency** on iOS: export `logo.png` onto a solid
  `#3B82F6`-family background at 1024×1024 → `images/icon.png`; clean the stray dark pixels at
  the transparent edges first.
- Android adaptive icon: foreground = camera mark centered inside the 66% safe zone on
  transparency; background = solid primary blue; monochrome = white silhouette of the mark.
- Splash: `expo-splash-screen` plugin image = the mark, background `#FFFFFF` (light) /
  slate-950 (dark) — update `app.json`.
- Large source images (`auth-demo-img.png` 2.1 MB, `logo.png` 1.2 MB) — compress/resize
  before shipping in the bundle.

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
- Select **1–5 images** from the device library (multi-select, `selectionLimit: 5`), shown in
  selection order; the composer allows removing/reordering before posting.
- Choose the post's shape: **square (1:1)** or **portrait (4:5)**; every image in the post is
  center-cropped to that shape so the carousel height never jumps.
- Add an optional caption and publish.
- In the feed and post detail, images display as a swipeable **carousel** with a dot indicator
  and an "n/5" counter; grids show the first image with a small multi-image icon.
- Show upload progress and prevent duplicate submissions.
- Publish only after successful upload and validation.
- Keep the composer open during upload; warn before abandoning an active upload.
- Failed uploads expose retry; unused uploads are cleaned up.
- Post authors can **edit the caption** (shows "edited") or delete the post.

Image processing and limits (sized to stay inside Convex's free 1 GB storage / 1 GB egress):

- Before upload, on device with `expo-image-manipulator`, center-crop each picked image to the
  chosen shape, then produce JPEGs:
  - **Full image (each of 1–5):** **1080×1080** (square) or **1080×1350** (portrait), JPEG
    quality **0.75** (typically 150–400 KB).
  - **Thumbnail (first image only):** **400 px** wide, same shape, JPEG quality **0.7**
    (typically 30–60 KB) — used in profile and Explore grids, Activity, and anywhere the post
    is shown small.
- **Hard caps after compression** (enforced on client **and** backend via upload metadata):
  each full image ≤ **1 MB**, thumbnail ≤ **150 KB**, **max 5 images per post**. Reject with
  a clear message otherwise.
- Accept any picker image type the manipulator can read (JPEG, PNG, HEIC); output is always JPEG.
- Images upload in parallel (max 2 at a time) with per-image progress; a failed image can be
  retried individually; the post is published only when every image is uploaded.
- Avatars: square crop, ≤ **400 px**, quality 0.75, ≤ **150 KB** (no separate thumbnail).
- Feed carousels load only the visible image and its neighbour, so unswiped images are never downloaded.
- Rely on `expo-image` disk caching so repeat views don't re-download.

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
| Post | Author ID, ordered list of 1–5 full-image storage references, thumbnail storage reference (first image), shape (`1:1`/`4:5`), optional caption, edited time, like/comment counts, creation time |
| Follow | Follower ID, followed-user ID |
| Like | User ID, post ID |
| Comment | Post ID, author ID, optional parent (top-level) comment ID, text, creation time |
| Conversation | Canonical participant pair (sorted), latest-message time and preview, per-participant last-read time |
| Message | Conversation ID, sender ID, text, creation time, client-generated retry/deduplication ID |
| Upload | Owner ID, storage reference, intended use (`post`/`postThumb`/`avatar`), size in bytes, lifecycle state (`pending`/`attached`/`abandoned`), creation time |
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
- Validate text, image metadata (content type `image/jpeg`, byte size against the caps — read
  from Convex storage metadata, not trusted from the client), and resource existence.
- Make retries safe for publication (an upload can be attached once) and message sending
  (dedupe by client ID).
- Clean up abandoned uploads and deleted media through retry-safe scheduled work (Convex cron).
- Bootstrap profiles idempotently after authentication; app-profile edits remain authoritative in Convex.
- Apply blocks in feed, explore, search, profile, comment, and message queries.
- Feed strategy: fan-in on read (following IDs + own ID, paged by recency). Acceptable at
  demo scale; switch to fan-out-on-write if it becomes slow.

No separate REST server, billing service, analytics service, or image/video-processing service is required.

## 6. State, failure behavior, and operating defaults

- Convex is the source of truth; use live subscriptions where appropriate.
- Keep already loaded content visible during connection loss.
- Show connection status (offline banner) and actionable errors.
- Optimistic likes/follows must roll back on failure.
- Retain unsent text in the active screen for retry.
- Do not promise draft recovery after app termination.
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
- [ ] Replace the Expo placeholder app icon, adaptive icon, `expo.icon`, and splash with Codexgram branding from `assets/images/logo.png` (see §3 Assets).
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

- [ ] Add multi-image selection (max 5), shape choice, on-device crop/compression (full images + thumbnail), and size validation.
- [ ] Implement upload tracking, progress, publication, retry, and abandonment behavior.
- [ ] Implement post display (swipeable carousel with indicator in feed/detail, thumbnail + multi-image icon in grids), caption editing, and deletion (removes all of the post's images).
- [ ] Add storage cleanup (cron for abandoned uploads, cascade on delete).

**Complete when:** 1-image and 5-image posts publish within the size caps, a 6th image is blocked, (verify real sizes in the Convex dashboard), and failed uploads never create visible incomplete posts.

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
- [ ] Seed fictional profiles from `assets/images/{feed,explore,profile,chat}/` (run through the same compression).
- [ ] Remove unused template assets (see §3 Assets).
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
- Invalid or unauthorized uploads cannot be published; non-JPEG files, files over the size caps, and posts with 0 or more than 5 images are rejected.
- Blocked users cannot message or see each other's content.
- Account deletion removes all of the user's data.

Manually verify:

- Google, Apple, and email first sign-up, returning login, verification code, password reset, cancellation, and logout.
- Missing provider name/avatar and interrupted onboarding.
- Image selection (JPEG, PNG, HEIC, very large photos), 1 vs 5 images, square vs portrait crop, compression output sizes, size-cap boundaries, and one image failing mid-upload.
- Carousel swiping, indicator, and double-tap like on any slide.
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
- Free-tier usage is controlled by compression, thumbnails, lazy carousel loading, and caching
  (≈700 worst-case 5-image posts, ≈3,000+ single-image posts fit in 1 GB storage) — still check the Convex usage dashboard weekly; confirm whether file serving
  counts toward the 1 GB egress allowance and how often it resets.
- Member-only media access must be enforced beyond navigation; Convex file URLs are unguessable but not access-controlled — do not assume possession of a file URL proves authorization.
- Sample photos come from the MIT-licensed reference repo but their original source is unknown — replace with owned/clearly licensed images before a public store release.
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
| Post media | Single image or video (≤10 MB image, ≤30 s / 50 MB video), no compression | Carousel of 1–5 images (square or 4:5), compressed on device (1080 px, ≤1 MB each) plus a 400 px thumbnail; video deferred |
