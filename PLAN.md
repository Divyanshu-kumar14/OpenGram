# OpenGram v1 — Spec + Implementation Plan (locked 2026-10-02, rev2: no-Convex)

> Scaffold: `expo ~57.0.26`, `expo-router ~57.0.24`, React 19.2.3, RN 0.86.3.
> Docs verified 2026-10-02: native-tabs (SDK 57 → `expo-router/unstable-native-tabs`) + using-clerk + using-supabase + tailwind.
> Locked decisions: Android-only · Clerk Google-only with **publishable key only** · **No Convex** → Supabase Postgres + Realtime · Cloudinary unsigned for all media · 3min/100MB background uploads · Public-only, no stories/reels.
> Status: PLAN READY, no code written yet.

## 1. Spec summary

### Product
OpenGram — Instagram-style, Android-only MVP. Modern/clean/minimal, mobile-first, native tabs (Home, Messages, Explore, Profile). Clerk = identity only. Supabase Postgres = source of truth + realtime. Cloudinary = bytes (images + video).

### Scope IN
- Clerk Google native sign-in (`@clerk/expo` 4.x + `expo-secure-store` tokenCache + `@clerk/expo-google-signin` + `expo-crypto`), publishable key only, no secret anywhere
- Onboarding carousel (simplified): unique username + avatar + bio, no follow suggestions
- Create: camera + gallery (`expo-image-picker` + `expo-camera`), basic crop, caption + `#hashtags` + `@tags` (parse + clickable + search, no mention pushes), ≤3min/100MB, background upload with progress + retry, online-only
- Home: chronological following feed, optimistic like (heart + double-tap), flat comments (no threads/edit)
- Explore: all-posts grid + search users + hashtags, hashtag feed
- Profile: view/edit own (avatar/bio/name; username unique), counts + own grid, any-user profile route
- Follow/unfollow public (no requests), DMs 1-1 text-only realtime + read receipts + unsend, conversation list by lastMessageAt
- Delete: hard delete + cascade (post → likes/comments + Cloudinary asset; comment/unsend → instant remove)

### Scope OUT
iOS/web, Apple/email login, stories/reels/notifications tab, trending, mention notifs, threaded comments, groups, media DMs, push, block/report, soft-delete, offline queue, analytics/Sentry, CI.

### Core flows
1. First run: launch → Clerk check → Google → `ensureUser(clerk_id)` (select → insert if missing) → no username → `/onboarding` → save → `/(tabs)`
2. Create: pick → validate size/duration (`expo-file-system` stat) → unsigned upload to Cloudinary (`https://api.cloudinary.com/v1_1/<cloud>/auto/upload` + `upload_preset`, XHR progress) → insert `posts` row (url + publicId + parsed hashtags) → prepend to Home
3. Home: `posts + follows` join (followees, newest first, paginated) → optimistic like → comment sheet
4. Explore: `posts` grid (thumbnailUrl) + `searchUsers` (ilike username/display) + `searchHashtags` (hashtag overlap) → hashtag feed
5. DMs: `conversations` by participant → `messages` realtime channel per conversation → send (optimistic) → `markRead` → unsend (delete if sender)
6. Delete post: delete Cloudinary asset (unsigned → must use **delete via server-side or keep orphan** — see risk R3) + delete row (DB cascade deletes likes/comments)

### Data model — Supabase Postgres (replaces Convex)
```sql
users(id uuid pk default gen_random_uuid(), clerk_id text unique not null,
  username text unique not null, display_name text not null,
  avatar_url text, avatar_public_id text, bio text, created_at timestamptz default now());
posts(id uuid pk, author_id uuid refs users(id) on delete cascade, type text check (type in ('image','video')),
  cloudinary_public_id text not null, url text not null, thumbnail_url text,
  width int, height int, duration_sec int, bytes int,
  caption text, hashtags text[] default '{}', tagged_user_ids uuid[] default '{}',
  like_count int default 0, comment_count int default 0, created_at timestamptz default now());
likes(post_id uuid refs posts(id) on delete cascade, user_id uuid refs users(id) on delete cascade,
  created_at timestamptz default now(), primary key (post_id, user_id));
comments(id uuid pk, post_id uuid refs posts(id) on delete cascade, author_id uuid refs users(id) on delete cascade,
  body text check (char_length(body) between 1 and 500), created_at timestamptz default now());
follows(follower_id uuid refs users(id) on delete cascade, following_id uuid refs users(id) on delete cascade,
  created_at timestamptz default now(), primary key (follower_id, following_id));
conversations(id uuid pk, participant_ids uuid[] not null, last_message_at timestamptz default now(),
  last_message_preview text, created_at timestamptz default now());
messages(id uuid pk, conversation_id uuid refs conversations(id) on delete cascade,
  sender_id uuid refs users(id) on delete cascade, body text check (char_length(body) between 1 and 1000),
  read_at timestamptz, created_at timestamptz default now());
-- indexes: users(clerk_id), users(username), posts(author_id, created_at desc), posts using gin(hashtags),
-- likes(post_id), comments(post_id, created_at), follows(follower_id), follows(following_id), messages(conversation_id, created_at)
-- count triggers: after insert/delete on likes → posts.like_count; on comments → posts.comment_count
-- conversations trigger: after message insert → update last_message_at/preview
```
- Auth: Clerk JWT via Supabase **third-party auth** (Dashboard → Auth → Third-party → add Clerk issuer). Client sets Supabase token from Clerk: `getToken({ template: 'supabase' })` → `supabase.auth.setSession` / `accessToken` callback. RLS policies compare `(auth.jwt() ->> 'sub')` to `users.clerk_id` (verify exact claim in Phase 0 against Clerk template output — do not guess).
- RLS on EVERY table (missing policy = empty array, no error — hardest Supabase footgun). Grants: `grant select/insert/update/delete on <table> to anon, authenticated` + policies per action.
- Source of truth: Supabase rows; Cloudinary holds bytes (publicId mirrored). Clerk holds identity only.

### Architecture (who does what)
- **Client (SDK 57):** screens, `unstable-native-tabs`, NativeWind styling, validation, Cloudinary unsigned XHR upload, Supabase queries + realtime channels, Clerk hooks.
- **Clerk:** Google OAuth + session. Publishable key only. No secret.
- **Supabase:** Postgres + RLS + Realtime + (optional) Edge not needed in v1. Logic lives in SQL (policies, triggers, functions). No custom server.
- **Cloudinary unsigned preset** (`folder=opengram`): direct-from-device upload, eager `f_auto,q_auto` + poster/thumbnail. No secret in app.
- **EAS:** `development` + `preview` Android profiles (dev builds required for Clerk native + camera + video). Supabase dev = hosted project (local CLI optional; physical device can't reach 127.0.0.1 — use LAN or hosted for device testing).

## 2. Phases

### Phase 0 — Foundations
1. `npx expo install @clerk/expo expo-secure-store expo-crypto @clerk/expo-google-signin @supabase/supabase-js expo-sqlite expo-image-picker expo-camera expo-video expo-file-system` + NativeWind (nativewind.dev Expo guide, Tailwind v3). Plugins: `expo-secure-store`, `@clerk/expo` (appleSignIn false), `expo-router`, `expo-sqlite`, `expo-camera`, `expo-video`.
2. `eas integrations:supabase:connect` (or manual) → `.env.local`: `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`, `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, `EXPO_PUBLIC_CLOUDINARY_CLOUD_NAME`, `EXPO_PUBLIC_CLOUDINARY_PRESET`. Set `android.package` in app.json (required for Clerk callback + builds).
3. SQL migration: tables + indexes + RLS + grants + count triggers (single reviewable `.sql`). Configure Clerk third-party auth in Supabase dashboard; verify `auth.jwt()` sub claim with one authed query before building features.
4. `src/lib/supabase.ts` (client + AppState auto-refresh per Expo guide) + `src/lib/clerk-supabase.ts` (token injection) + root `_layout.tsx`: `ClerkProvider(tokenCache)` → auth guard (signed-out → `/sign-in`, no username → `/onboarding`, else tabs).
5. `(tabs)/_layout.tsx` with `expo-router/unstable-native-tabs` (SDK 57!): index(home)/messages/explore/profile, md icons `home/chat/search/person`.
- Verify: `tsc --noEmit`, `expo lint`, tabs render on Android dev build, Google sign-in creates `users` row, RLS denies anon write (prove with negative test).

### Phase 1 — Media + Home
6. Unsigned upload util (XHR progress, 100MB/180s pre-check, Cloudinary `auto/upload`) + `createPost` (insert row, server re-parses hashtags via trigger/function — never trust client array) + `deletePost` (row delete; see R3).
7. `create.tsx` modal + Home feed (paginated `author_id in (followees)` + realtime invalidation for likes/comments counts) + optimistic like + comment sheet.
- Verify: oversize reject, airplane-mode fail + retry, delete cascades likes/comments, counts converge after re-query.

### Phase 2 — Explore + Profile + DMs
8. Explore grid + user/hashtag search + hashtag route. Profile + edit (avatar re-upload same path). DMs: `getOrCreate` by sorted pair, list by lastMessageAt, per-conversation `postgres_changes` channel, send/markRead/unsend.
- Verify: two-emulator DM live + receipts + unsend-for-both, search <500ms on seed, ordering correct.

### Phase 3 — Harden & ship
9. Empty states (no follows → Explore CTA + recent-users directory, NOT algorithm), quota errors (Cloudinary/Supabase 402/429 → friendly copy), error boundaries.
10. `tsc` + `lint` clean, `eas build --profile preview --platform android`, Play internal-track checklist (package, versionCode, camera/media rationale).
- Done = manual QA per tab on preview build + lint/type green + cascade proven + quota path screenshotted.

## 3. ASSUMPTIONS
A1. General consumers, no special compliance. A2. Core loop = post→feed→like/comment. A3. Reuse `/OpenGram` SDK 57 template (overwrite `src/app`, keep configs). A4. NativeWind v4 + Tailwind v3. A5. Clerk template named `supabase` (confirm in dashboard; claim mapping verified in Phase 0, not assumed). A6. Cloudinary unsigned preset is acceptable (quota-abuse risk accepted for <1k MVP). A7. Server-side compression via Cloudinary eager only. A8. No push/analytics/CI. A9. Free-tier caps → friendly error, no auto-upgrade. A10. English-only, portrait. A11. Conversation `participant_ids` sorted array length==2 (group-ready shape). A12. No deadline.

## 4. OPEN RISKS
R1. **Unsigned-preset abuse:** anyone with preset name can burn quota. Mitigate: strict preset (folder, max size, resource type), client validation, monitor dashboard; switch to signed (secret in Supabase Edge Function only) before public launch.
R2. **RLS silent empty:** missing policy/grant returns `[]`, not error — looks like "no data" bug. Mitigate: per-table RLS checklist + negative tests (anon read/write denied) in Phase 0.
R3. **Unsigned delete gap:** Cloudinary unsigned cannot delete from client (needs signed/rest API). MVP options: (a) leave orphan bytes + scheduled cleanup (documented), or (b) tiny signed-delete via Supabase Edge Function (adds one secret). Default: (a) orphans accepted for MVP, cleanup before scale.
R4. **Clerk↔Supabase claim mismatch:** `sub` vs `user_id` vs email template differences break all RLS. Mitigate: print `auth.jwt()` in dev, pin template, re-verify on Clerk/Supabase dashboard changes.
R5. **Android fragmentation:** duration/size metadata varies. Mitigate: `expo-file-system` stat pre-check + test 2 emulator profiles + 1 device.
R6. **Realtime fan-out cost:** per-conversation channels scale fine <1k; feed-wide realtime would not — feed uses polling/invalidation, only DMs use channels.
R7. **Cold-start empty Home:** no suggestions → empty feed. Mitigate: empty-state CTA + recent-users directory.
R8. **No block/report:** spam has no remedy. Mitigate: rate-limit sends/creates (Supabase function check) if abuse appears; report flow is first v2 item.
R9. **Hashtag consistency:** single `parseHashtags()` shared client preview + DB trigger re-parse, always lowercase.
R10. **SDK 58 rename:** `unstable-native-tabs` → `native-tabs`. Pin SDK 57 for v1.

## 5. Verification (mandatory)
- `npx tsc --noEmit` → 0 errors · `npx expo lint` → 0 new warnings
- RLS proof: anon insert denied, authed own-row write allowed (paste Supabase logs)
- Edge cases cited: oversize video, offline upload, dup username, double-like race, unsend-after-read
- Evidence: EAS preview build ID + Supabase project ref + manual QA per tab
- Secrets scan: `grep -r "SERVICE_ROLE\|CLOUDINARY_API_SECRET\|sk_live" --exclude-dir=node_modules .` → 0 hits (no secrets in repo)

*Next: approve this rev2, then Phase 0 begins.*
