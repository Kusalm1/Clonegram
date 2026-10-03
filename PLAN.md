# Codexgram — v1 Implementation Plan

A minimal, mobile-first, Instagram-style photo sharing app. Portfolio project, built to pass
App Store / Play Store review when we're ready to submit.

This document is the single source of truth for v1. Work through the phases in order; each
phase ends with acceptance criteria that must pass before moving on.

---

## 0. Ground rules for every phase

- **Verify before coding.** Expo SDK 57, Clerk, Convex, and NativeWind change often. Before
  touching an API, check the current docs (`https://docs.expo.dev/versions/v57.0.0/`,
  Clerk Expo docs, Convex docs, NativeWind docs). Items marked **(verify)** below are known
  to be version-sensitive.
- **Install with `npx expo install <pkg>`** (never `npm install`) so versions match SDK 57.
  Dev-only tools go in `devDependencies`.
- **Never edit `ios/` or `android/`** — they are generated. Configure via `app.json` and
  config plugins.
- **Definition of done for each step:** `npx expo lint` and `npx tsc --noEmit` pass, and
  the feature works on **Android (development build)** and **iOS (Expo Go)**.
- **Only Expo Go–compatible modules in v1** (so iOS can still be tested without an Apple
  Developer account). If a module isn't in Expo Go, stop and discuss before adding it.
- Commit at the end of every step with a clear message.

---

## 1. Product scope

### In v1
| Area | Features |
|---|---|
| Auth | Sign up / sign in with **Google**, **Apple**, or **email + password**. Email sign-up verified with a 6-digit code. Forgot password (reset code by email). |
| Onboarding | First sign-in → choose unique `@username`, display name, optional avatar. |
| Posts | Carousel of **1–5 images** + caption. Square (1:1) or portrait (4:5) per post. Caption editable after posting. Delete own posts. |
| Feed (Home) | Posts from people you follow, newest first, infinite scroll. Own posts not included. |
| Likes | Tap heart or double-tap image. Optimistic UI. |
| Comments | One level of threading (replies to a reply attach to the top-level comment with `@username` prefilled). Delete own comments; post owner can delete any comment on their post. Deleting a top-level comment deletes its replies. |
| Follow | One-tap follow/unfollow. All accounts public. |
| Explore | Search people by username or name + grid of recent posts from everyone. |
| Messages | 1:1 text DMs, realtime. Anyone can message anyone. Unread badge on the Messages tab. |
| Activity | In-app list of likes, comments, follows on your content (bell icon in Home header). |
| Profile | View/edit profile (display name, username, bio, avatar). Followers / following / posts counts. Tappable followers & following lists. |
| Store compliance (built last) | Accept Terms at sign-up, report post/comment/user, block user, in-app account deletion. |
| General | Light + dark mode (follows system). English only, strings centralised. Offline banner. |

### Out of v1
Video, private accounts / follow requests, push notifications, group DMs, media in DMs,
sharing posts to DMs, comment likes, "liked by" list, offline reading, web app,
ranked/popular feeds, in-app admin panel (reports reviewed in the Convex dashboard),
change-password screen, native one-tap Google/Apple buttons.

Also excluded even though they appear in the `design/` reference images: stories row,
bookmark/save, comment hearts, composer extras (location, tag people, add testers), Explore
category chips and People/Posts search toggle, profile location & website fields, a
"Create" tab.

---

## 2. Tech stack & decisions

| Concern | Choice | Notes |
|---|---|---|
| Framework | Expo SDK 57, React Native 0.86, React 19.2 | Already installed. React Compiler + typed routes enabled. |
| Navigation | Expo Router, routes in `src/app/` | |
| Tabs | **Native tabs** — `expo-router/unstable-native-tabs` on SDK 57 **(verify)** | Fallback: Expo Router JS `Tabs` if Phase 1 spike fails in Expo Go on iOS. Native tab bar is **not** styleable with NativeWind (uses system look). |
| Styling | **NativeWind v4.2.7** + **Tailwind CSS v3** | v4.2.7 is the stable release with SDK 57 support. Do **not** use v5 (RC). |
| Auth | **Clerk** via `@clerk/expo` **(verify package name)**, publishable key only | Custom-built screens (Clerk's prebuilt native components don't run in Expo Go). Token cache in `expo-secure-store`. |
| Social sign-in | Browser-based OAuth (`useSSO`, strategies `oauth_google`, `oauth_apple`) | Works in Expo Go and dev builds, iOS + Android. Clerk dev instance supplies shared Google/Apple credentials — no Google Cloud / Apple account needed for development. |
| Email/password | Clerk `useSignUp` / `useSignIn` custom flows **(verify hook API)** | Email code verification; reset-password code flow. |
| Backend / DB | **Convex** (queries, mutations, file storage, realtime) | Clerk ↔ Convex via `ConvexProviderWithClerk` from `convex/react-clerk` + `convex/auth.config.ts`. |
| Images | `expo-image-picker` (multi-select, up to 5), `expo-image-manipulator` (center-crop to chosen ratio, resize ~1080px wide, JPEG ~0.8), `expo-image` for display | Stored in Convex file storage. No separate thumbnails in v1. |
| Connectivity | `@react-native-community/netinfo` **(verify Expo Go inclusion)** | For the offline banner. |
| Builds | EAS Build free tier; `development` profile → Android APK (sideload). `expo-dev-client` | iOS dev build blocked until an Apple Developer account exists — iOS tested in Expo Go. |
| Budget | Free tiers of Clerk, Convex, EAS | Confirm current limits on pricing pages in Phase 0. |

### Environment variables
| Name | Where | Value |
|---|---|---|
| `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY` | `.env.local` (app) | Clerk dashboard → API keys |
| `EXPO_PUBLIC_CONVEX_URL` | `.env.local` (app) | Written by `npx convex dev` |
| `CLERK_JWT_ISSUER_DOMAIN` | Convex dashboard env vars (dev + prod) | Clerk Frontend API URL **(verify exact name/format)** |

`.env*.local` is already gitignored. **Never** put Clerk's secret key in the app.

---

## 3. Auth model

- **One account per verified email.** Clerk automatically links a Google sign-in to an
  existing account with the same verified email (Clerk docs: *"Clerk links the OAuth account
  to the existing account and signs the user in"*).
- **Apple "Hide My Email"** users get a relay address → they become a separate account. Expected.
- **After any successful sign-in/up:** app has a Clerk session → query `users.me` in Convex:
  - `null` → `/onboarding`
  - user row exists → tabs (Home)
- **Convex identity:** every function calls `ctx.auth.getUserIdentity()`; `identity.subject`
  = Clerk user ID = `users.clerkId`. Shared helpers `requireIdentity(ctx)` and
  `requireUser(ctx)` (throws if no Convex user row yet).
- **Clerk dashboard settings (dev instance):**
  - Enable: Google, Apple (social connections); Email address + Password.
  - Email verification: **email code** at sign-up.
  - Disable: username, phone, magic links (username lives in Convex, not Clerk).
  - Allow users to delete their own account (needed in Phase 14) **(verify setting name)**.
  - Add the app's redirect URL(s) for native OAuth (`codexgram://…` and Expo Go `exp://…`) to the allowlist if required **(verify)**.

---

## 4. Data model (Convex `convex/schema.ts`)

All timestamps use Convex `_creationTime` unless noted. Counts are **denormalised** and
updated in the same mutation that changes the underlying data.

### `users`
| Field | Type | Rules |
|---|---|---|
| `clerkId` | string | unique — index `by_clerkId` |
| `username` | string | unique, lowercase, `^[a-z0-9._]{3,30}$`, no leading/trailing `.` — index `by_username` |
| `displayName` | string | 1–50 chars |
| `bio` | string? | ≤ 150 chars |
| `avatarStorageId` | Id<"_storage">? | uploaded avatar |
| `avatarUrl` | string? | fallback from Google/Clerk image; initials if neither |
| `followersCount`, `followingCount`, `postsCount` | number | default 0 |
| `termsAcceptedAt` | number? | set at onboarding (Phase 14 enforces) |
| `searchName` | string | lowercase `username + " " + displayName` — search index `search_name` |

### `posts`
| Field | Type | Rules |
|---|---|---|
| `authorId` | Id<"users"> | index `by_author` |
| `images` | Id<"_storage">[] | length 1–5 |
| `aspect` | `"1:1" \| "4:5"` | |
| `caption` | string | ≤ 2,200 chars |
| `editedAt` | number? | set when caption edited |
| `likesCount`, `commentsCount` | number | default 0 |

### `follows`
`followerId`, `followingId` (both Id<"users">). Indexes: `by_follower`, `by_following`,
`by_pair` (`followerId`, `followingId`) — enforces uniqueness. Cannot follow self.

### `likes`
`userId`, `postId`. Indexes: `by_post`, `by_user_post` (uniqueness).

### `comments`
| Field | Type | Rules |
|---|---|---|
| `postId` | Id<"posts"> | index `by_post` |
| `authorId` | Id<"users"> | |
| `parentId` | Id<"comments">? | must be a **top-level** comment; index `by_parent` |
| `text` | string | 1–500 chars |

### `conversations`
| Field | Type | Rules |
|---|---|---|
| `participantA`, `participantB` | Id<"users"> | stored sorted (A < B) → index `by_pair` guarantees one per pair |
| `lastMessageAt` | number | |
| `lastMessagePreview` | string | first ~80 chars |
| `lastReadA`, `lastReadB` | number | per-participant read timestamp |

Indexes: `by_participantA_last`, `by_participantB_last` (for the inbox list sorted by recency).
*(Alternative to evaluate in Phase 10: a `conversationMembers` table — simpler querying. Pick one, document why.)*

### `messages`
`conversationId`, `senderId`, `text` (1–1,000 chars). Index `by_conversation`.

### `notifications`
| Field | Type | Rules |
|---|---|---|
| `recipientId` | Id<"users"> | index `by_recipient` |
| `actorId` | Id<"users"> | |
| `type` | `"like" \| "comment" \| "reply" \| "follow"` | |
| `postId` | Id<"posts">? | |
| `commentId` | Id<"comments">? | |
| `read` | boolean | |

No notification when acting on your own content. Unlike / unfollow / delete removes the matching notification.

### Phase 14 tables
- `reports`: `reporterId`, `targetType` (`post|comment|user`), `targetId` (string), `reason`, `details?`, `status` (`open|reviewed`).
- `blocks`: `blockerId`, `blockedId`. Indexes `by_blocker`, `by_blocked`, `by_pair`.

---

## 5. Convex functions (planned API)

`convex/` at project root. Every public function validates args with `v.*` validators and
checks auth. Mutations enforce all rules server-side (never trust the client).

| File | Functions |
|---|---|
| `users.ts` | `me` (q), `getByUsername` (q), `isUsernameAvailable` (q), `completeOnboarding` (m), `updateProfile` (m), `generateAvatarUploadUrl` (m), `search` (q, paginated) |
| `posts.ts` | `generateUploadUrl` (m), `create` (m), `updateCaption` (m), `remove` (m — cascades likes, comments, notifications, storage files, decrements `postsCount`), `getById` (q), `feed` (q, paginated), `explore` (q, paginated), `byAuthor` (q, paginated) |
| `likes.ts` | `toggle` (m) |
| `follows.ts` | `toggle` (m), `followers` (q, paginated), `following` (q, paginated), `isFollowing` (q) |
| `comments.ts` | `list` (q — top-level paginated, replies grouped), `add` (m), `remove` (m — cascade replies, fix `commentsCount`) |
| `conversations.ts` | `getOrCreate` (m), `list` (q, paginated), `unreadCount` (q), `markRead` (m) |
| `messages.ts` | `list` (q, paginated, newest first), `send` (m) |
| `notifications.ts` | `list` (q, paginated), `unreadCount` (q), `markAllRead` (m) |
| `lib/auth.ts` | `requireIdentity`, `requireUser` helpers |
| `lib/validation.ts` | username regex, length limits (shared constants) |
| Phase 14 | `reports.create`, `blocks.toggle`, `blocks.list`, `users.deleteAccount` |

Post queries return a **hydrated** shape the UI can render directly:
`{ post, author: { username, displayName, avatar }, imageUrls[], likedByMe, isMine }`.

**Feed strategy (v1): fan-in on read.** `feed` loads the viewer's following IDs and pages
through recent posts filtered to those authors. Fine at portfolio scale. If it becomes slow,
switch to fan-out-on-write (a `feedItems` table written on post create). Document the trade-off in code.

---

## 6. App structure (`src/`)

```
src/
  app/
    _layout.tsx                 # ClerkProvider → ConvexProviderWithClerk → auth gate
    (auth)/
      _layout.tsx               # stack, only when signed out
      welcome.tsx               # Google / Apple / Email buttons
      sign-in.tsx               # email + password
      sign-up.tsx               # email + password
      verify-email.tsx          # 6-digit code
      forgot-password.tsx       # request code → new password
    onboarding.tsx              # signed in, no Convex user row yet
    (tabs)/
      _layout.tsx               # NativeTabs: home, messages, explore, profile
      home/        _layout.tsx (Stack), index.tsx
      messages/    _layout.tsx, index.tsx
      explore/     _layout.tsx, index.tsx
      profile/     _layout.tsx, index.tsx
    create.tsx                  # modal: pick → ratio → caption → post
    post/[id].tsx               # post detail (+ edit caption / delete menu)
    comments/[postId].tsx       # form-sheet modal, threaded comments
    user/[username].tsx         # other user's profile
    user/[username]/followers.tsx
    user/[username]/following.tsx
    chat/[conversationId].tsx   # DM thread
    activity.tsx                # notifications list
    edit-profile.tsx            # modal
    settings.tsx                # sign out (+ Phase 14: terms, blocked users, delete account)
  components/                   # PostCard, ImageCarousel, Avatar, Button, TextField, EmptyState, OfflineBanner, ...
  hooks/                        # useCurrentUser, useDoubleTap, useNetworkStatus, ...
  lib/                          # clerk token cache, image processing, formatters (relative time, counts)
  constants/strings.ts          # all user-facing copy
convex/                         # backend (see §5)
```

Shared routes (`post/[id]`, `user/[username]`, …) are pushed on top of the current tab's
stack. Exact grouping/route sharing with native tabs **(verify)** in Phase 1.

**Auth gate:** in the root layout, use Expo Router's protected routes (`Stack.Protected` with
`guard`) **(verify on SDK 57)** — signed out → `(auth)`; signed in without user row →
`onboarding`; otherwise → `(tabs)`.

**Create button:** `+` icon in the Home header (and Profile header), opens `create` as a modal.
**Activity:** bell icon in the Home header with an unread dot.

---

## 7. UI / UX guidelines

### Design reference
`design/` holds the visual reference (from
[burakorkmez/codexgram](https://github.com/burakorkmez/codexgram/tree/master/design), MIT).
Match these screens for layout, spacing, and components — **except** the features excluded
in §1.

| File | Use for |
|---|---|
| `app-design-ref.png` | Overview of every screen and state (splash, welcome, auth states, onboarding, feed + empty feed, composer, upload progress/error, post detail, explore, search results/empty, profiles, edit profile, followers, inbox, chat, empty states, delete confirmation, bottom sheet) |
| `design-system-ref.png` | Tokens and components (logo, palette, type, spacing, icons, buttons, inputs, avatars, cards, post card, comment rows, profile stats, empty states, upload progress, nav bars, message bubbles, bottom sheets, destructive confirmation) |
| `auth-screen-ref.png` | Phase 2 |
| `home-screen-ref.png`, `post-detail-ref.png`, `comments-ref.png` | Phases 6, 7, 9 |
| `profile-screen-ref.png`, `edit-profile-ref.png` | Phase 8 |
| `explore-screen-ref.png` | Phase 9 |
| `messages-tab-ref.png`, `chat-screen-ref.png` | Phase 10 |
| `settings-screen-ref.png` | Phases 2 & 14 |

### Design tokens (from `design-system-ref.png`)
| Token | Light value | Use |
|---|---|---|
| `primary` | `#3B82F6` | buttons, links, active tab, own message bubbles |
| `text` | `#0F172A` | primary text |
| `secondary` | `#64748B` | secondary text, icons |
| `border` | `#E5E7EB` | dividers, input borders |
| `surface` | `#F8FAFC` | cards, inputs, secondary buttons, other-user bubbles |
| `success` | `#22C55E` | "available", success states |
| `warning` | `#F59E0B` | warnings |
| `destructive` | `#EF4444` | delete, report, errors |
| `info` | `#0EA5E9` | info notices |

Dark-mode values aren't in the reference — derive them (e.g. slate-950 background, slate-900
surface, slate-800 border, slate-50 text, slate-400 secondary, same primary) and verify contrast.

| Type | Size / weight |
|---|---|
| H1 | 34 Bold |
| H2 | 28 Semibold |
| H3 | 22 Semibold |
| Body | 17 Regular |
| Caption | 15 Regular |
| Label | 13 Medium |

Font: system font (SF Pro on iOS) — Inter optional on Android **(decide in Phase 4; custom
fonts load via `expo-font`)**. Spacing scale: 4, 8, 12, 16, 24, 32, 48 (8pt grid).
Icons: 24px outlined, rounded strokes. Buttons: Primary (filled blue), Secondary (surface),
Ghost (outline), Destructive (filled red); fully rounded corners on chips and pill buttons.

### Guidelines
- **Look:** clean, minimal, lots of whitespace, edge-to-edge images, neutral palette with one
  accent colour. Typography-led, thin dividers, no heavy shadows.
- **Tokens** above go in `tailwind.config.js` (colours with light/dark variants, spacing, radius).
  Dark mode via NativeWind `dark:` classes following system setting.
- **Every list has** a loading skeleton, an empty state with a clear next action, and an error
  state with retry.
- **Optimistic updates** for like, follow, comment add/delete, send message.
- **Accessibility:** labels on icon-only buttons, minimum 44pt touch targets, sufficient
  contrast in both themes, support Dynamic Type where practical.
- **Safe areas** handled with `react-native-safe-area-context`.
- **Performance:** `FlatList`/`FlashList`-style virtualised lists (stick to `FlatList` unless
  a perf issue appears — FlashList availability in Expo Go **(verify)**), `expo-image` with
  caching, paginated queries, no unbounded `.collect()` on large tables.

---

## 8. Implementation phases

### Phase 0 — Housekeeping
1. Commit pending changes (ESLint config, `tsconfig.json` exclude, iOS `bundleIdentifier`).
2. Decide on `example/` folder (delete or keep as reference — it's gitignored).
3. Create Clerk application (dev instance) and configure per §3.
4. Create Convex project: `npx convex dev` (creates `convex/`, writes `EXPO_PUBLIC_CONVEX_URL`).
5. Confirm free-tier limits (Clerk MAU, Convex storage/bandwidth/function calls, EAS builds/month) and note them here.

**Done when:** clean working tree, both dashboards exist, `.env.local` populated (not committed).

### Phase 1 — Foundation spike (de-risk before building features)
1. Install NativeWind 4.2.7 + Tailwind CSS v3 following the official SDK-57 guide (`tailwind.config.js`, `global.css`, Babel/Metro config, `nativewind-env.d.ts`).
2. Set up native tabs with 4 placeholder screens, SF Symbols (iOS) / Material icons (Android), and a test badge on Messages.
3. Install `expo-dev-client`; build Android dev build: `npx eas-cli@latest build --profile development --platform android`; install the APK.
4. Run on **iOS Expo Go** and **Android dev build**.

**Done when:** NativeWind classes (incl. `dark:`) render on both; native tabs + badge work on both.
**If native tabs fail in iOS Expo Go:** switch to Expo Router JS `Tabs`, record the decision here.

### Phase 2 — Auth
1. Install `@clerk/expo`, `expo-secure-store`, `expo-auth-session`/`expo-crypto` if required by `useSSO` **(verify deps)**, `convex`.
2. Root providers: `ClerkProvider` (publishable key + secure-store token cache) → `ConvexProviderWithClerk` (`useAuth` from Clerk).
3. `convex/auth.config.ts` with `CLERK_JWT_ISSUER_DOMAIN`, `applicationID: "convex"`.
4. Screens: `welcome` (Google, Apple, "Continue with email"), `sign-in`, `sign-up`, `verify-email`, `forgot-password`.
5. Auth gate in root layout (signed out → `(auth)`).
6. Sign out (temporary button on Profile).
7. Friendly error mapping for Clerk errors (wrong password, email taken, invalid code, cancelled OAuth).

**Done when:** on both platforms a user can sign up and sign in with Google, Apple, and email+password (incl. verification code and password reset), session survives app restart, sign-out works, and `ctx.auth.getUserIdentity()` returns the user in a test Convex query.

### Phase 3 — Users & onboarding
1. Schema: `users` table + indexes.
2. `users.me`, `isUsernameAvailable` (debounced live check), `completeOnboarding`, `generateAvatarUploadUrl`.
3. `onboarding` screen: username (validated + availability), display name (prefilled from Clerk/OAuth name), optional avatar (prefilled from OAuth image).
4. Gate: signed in + `me === null` → onboarding.

**Done when:** a new user lands on onboarding exactly once; usernames are unique (server-enforced, race-safe); a returning user goes straight to tabs.

### Phase 4 — Design system & app shell
1. Tokens in Tailwind config; base components: `Button`, `TextField`, `Avatar`, `IconButton`, `EmptyState`, `ErrorState`, `Skeleton`, `OfflineBanner`, `Divider`.
2. `constants/strings.ts`; formatters (compact counts `1.2k`, relative time `3h`).
3. Tab stacks with headers; Home header with logo/wordmark, `+` and bell icons.
4. Offline banner via NetInfo.

**Done when:** components look right in light & dark on both platforms; offline banner appears in airplane mode.

### Phase 5 — Create post
1. `posts` schema; `generateUploadUrl`, `create`.
2. `create` modal: multi-select up to 5 → choose 1:1 or 4:5 (preview) → center-crop + resize ~1080px + compress → caption → Post.
3. Upload progress, per-image failure handling, retry; disable Post until all uploaded. Clean up orphaned storage files on failure.
4. Increment `postsCount`.

**Done when:** posting 1 and 5 images works on both platforms; >5 is blocked; post appears on own profile.

### Phase 6 — Home feed, post card, likes
1. `PostCard`: header (avatar, username, `…` menu), `ImageCarousel` (paging + dot indicator, fixed aspect), actions (like, comment), likes count, caption (expandable), "View all N comments", relative time, "edited".
2. `posts.feed` paginated; infinite scroll; pull-to-refresh.
3. `likes.toggle` + optimistic update + double-tap heart animation (Reanimated).
4. Empty state → "Find people on Explore".
5. Owner menu: edit caption, delete post (confirm).

**Done when:** following someone shows their posts newest-first; likes are instant and consistent across devices; edit/delete work and update everywhere.

### Phase 7 — Comments
1. `comments` schema; `list`, `add`, `remove`.
2. `comments/[postId]` sheet: top-level list, replies under each (collapsed "View N replies"), reply button prefills `@username` and attaches to the top-level parent, input bar above keyboard.
3. Delete rules: own comment, or any comment on own post; cascade replies; `commentsCount` stays correct.

**Done when:** threading behaves as specified; counts are accurate after add/delete/cascade.

### Phase 8 — Profiles & follow
1. Own profile: avatar, name, username, bio, counts, Edit Profile, Settings, 3-column post grid.
2. Other profile `user/[username]`: Follow/Unfollow (optimistic) + Message.
3. `follows.toggle` updating both users' counts; followers/following lists (paginated, with follow buttons).
4. `edit-profile`: display name, username (re-validated), bio, avatar.

**Done when:** counts stay correct under rapid follow/unfollow; username change is reflected everywhere.

### Phase 9 — Explore & post detail
1. Search bar → `users.search` (debounced); results list.
2. Recent posts grid (`posts.explore`, paginated) → tap → `post/[id]`.
3. `post/[id]` detail screen reusing `PostCard`.

**Done when:** search finds users by username and display name; grid paginates smoothly.

### Phase 10 — Messages
1. Finalise conversation schema choice (see §4).
2. `getOrCreate` (from profile Message button), `list` (inbox, by recency, with preview & unread dot), `messages.list` / `send`, `markRead` on open/focus.
3. Chat UI: bubbles, timestamps grouping, input bar, keyboard handling, optimistic send.
4. `unreadCount` → badge on the Messages native tab.

**Done when:** two devices chat in realtime; unread badge increments and clears correctly.

### Phase 11 — Activity
1. Create notifications inside like/comment/reply/follow mutations; remove on undo.
2. `activity` screen: grouped by time (Today / This week / Earlier), tap → post or profile.
3. Unread dot on Home header bell; `markAllRead` when opened.

**Done when:** each action creates exactly one notification for the right person and undo removes it.

### Phase 12 — Polish
Empty/error/loading states everywhere, accessibility pass, dark-mode pass, performance pass
(long feed scroll, image memory), app icon & splash, haptics on like/follow.

### Phase 13 — QA pass
Run every flow in §1 on Android dev build and iOS Expo Go, two accounts, including:
slow network, airplane mode, app kill/restart, deleting a post someone else is viewing,
deleting a comment with replies, username race, 5-image post on slow network.

### Phase 14 — Store compliance (required before submission)
1. **Terms:** checkbox/notice at sign-up/onboarding linking to Terms of Use & Privacy Policy (host simple pages); store `termsAcceptedAt`.
2. **Report:** `…` menu on posts, comments, profiles → reason picker → `reports.create`. Review in Convex dashboard.
3. **Block:** from profile menu; hides the blocked user's posts/comments/profile and prevents DMs in both directions; Settings → Blocked accounts (unblock).
4. **Delete account:** Settings → Delete account (confirm) → `users.deleteAccount` deletes all user data and storage files and fixes counts on others → Clerk `user.delete()` from the client → signed out.
5. Contact/support email visible in Settings.

**Done when:** each feature works end-to-end and a deleted user leaves no orphaned data.

### Phase 15 — Launch prep (when accounts exist)
1. Buy Apple Developer account ($99/yr) and Google Play Console ($25 one-time).
2. Clerk **production instance**: custom Google OAuth credentials (Google Cloud, app "In production") and Apple credentials (Services ID, Key ID, Team ID, private key) — **required**, shared dev credentials don't work in production.
3. Convex production deployment + `CLERK_JWT_ISSUER_DOMAIN` for the prod Clerk instance.
4. EAS `production` builds; iOS dev build/TestFlight testing; store listings, screenshots, privacy policy URL, data-safety / privacy nutrition labels, account-deletion web link for Google Play.
5. `eas submit`.

---

## 9. Assumptions (agreed defaults — revisit if wrong)

1. Store-compliance features are in v1 but built last (Phase 14).
2. Your own posts don't appear in your Home feed.
3. Avatar defaults to the OAuth profile photo; otherwise initials.
4. Usernames can be changed; the old one is freed immediately.
5. Counts are denormalised and updated in mutations.
6. Full-size images are used in grids (no thumbnails in v1).
7. No app-level rate limits beyond input limits (5 images, text lengths).
8. Two Convex deployments (dev, prod) and two Clerk instances; no CI in v1 — lint + typecheck before each commit.
9. "Done" = every flow works on Android dev build + iOS Expo Go; manual testing, no automated test suite in v1.
10. No analytics or crash reporting in v1 — Convex dashboard logs only.
11. Image cropping is automatic center-crop to the chosen ratio (no manual crop UI in v1).
12. The create-post entry point is a `+` in the Home (and Profile) header since there are exactly 4 tabs.

## 10. Open risks

1. **Native tabs in iOS Expo Go** unconfirmed on SDK 57 (`unstable-native-tabs`) — de-risked in Phase 1, fallback JS tabs.
2. **NativeWind can't style the native tab bar** or other system UI.
3. **OAuth redirect differences** between Expo Go (`exp://`) and the dev build (`codexgram://`) — both must be allowed in Clerk.
4. **iOS testing limited to Expo Go** until an Apple Developer account exists; no native-only modules in v1.
5. **Production OAuth credentials required** for Google and Apple at launch (Apple needs the paid account).
6. **Free-tier limits** (Convex storage/bandwidth from images is the main driver) — monitor in dashboards.
7. **Fan-in feed** may slow down at larger scale — migration path to fan-out documented.
8. **Store review** may ask for EULA link, contact info, and moderation response details beyond Phase 14.
9. **Apple Hide-My-Email** users can't be linked to an email/password account with their real email.
