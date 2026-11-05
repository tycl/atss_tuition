# CLAUDE.md — Online Tuition Video Platform

## 1. Project overview
A web platform for primary and secondary school online tuition in Malaysia.
Teachers upload lesson videos and notes. Students watch videos by level and subject.
Parents pay a monthly subscription and track their child's progress.

- Target levels: Standard 1–6 (primary), Form 1–5 (secondary)
- Subjects: configurable list (e.g. Bahasa Melayu, English, Mathematics, Science, Chinese)
- Language of UI: English first, with i18n-ready strings (BM and Chinese later)
- Device: web app, mobile-friendly (responsive). No native app in v1.

## 2. Tech stack (do not change without asking)
- Frontend: React + TypeScript, Vite, React Router
- Styling: Tailwind CSS
- Backend: Supabase (Postgres, Auth, Storage, Row Level Security, Edge Functions)
- Video: Cloudflare Stream (direct creator uploads, adaptive HLS, signed URLs)
- Payments: Stripe (FPX + card), subscription billing via webhooks
- Package manager: npm
- Lint/format: ESLint + Prettier
- Tests: Vitest (+ React Testing Library for components)
- Hosting: Cloudflare Pages (frontend), Supabase (backend)

## 3. Coding rules
- Functional style only. No classes.
- Strict TypeScript. No `any` unless commented with a reason.
- Small, focused files. One component per file.
- All secrets in environment variables. Never hard-code keys.
- Run `npm run lint` and `npm test` before saying a task is done.
- NEVER run `git commit` or `git push` without asking the user first.
- Ask before adding any new npm dependency.

## 4. User roles
| Role | Can do |
|---|---|
| admin | Manage teachers, subjects, levels, pricing, view all reports |
| teacher | Upload/edit own videos, notes, quizzes; see their students' progress |
| parent | Register, give consent, pay subscription, manage child accounts, view reports |
| student | Watch videos, download notes, take quizzes (only if subscription active) |

Enforce access with Supabase Row Level Security (RLS) on every table.

## 5. Data model (Postgres)
- `profiles` (id, role, full_name, phone, created_at)
- `parents_students` (parent_id, student_id) — one parent, many children
- `levels` (id, code e.g. STD1…STD6, F1…F5, name, sort_order)
- `subjects` (id, name)
- `courses` (id, level_id, subject_id, teacher_id, title, description, published)
- `lessons` (id, course_id, title, stream_video_uid, duration_sec, sort_order, published)
- `lesson_notes` (id, lesson_id, file_path) — PDFs in Supabase Storage (private bucket)
- `quizzes` (id, lesson_id, title)
- `quiz_questions` (id, quiz_id, question, options jsonb, answer_index)
- `quiz_attempts` (id, quiz_id, student_id, score, submitted_at)
- `watch_progress` (student_id, lesson_id, seconds_watched, completed, updated_at)
- `subscriptions` (id, parent_id, student_id, plan, status, stripe_subscription_id, current_period_end)
- `active_sessions` (id, user_id, device_label, last_seen_at) — device limit
- `consents` (parent_id, student_id, consent_version, consented_at)

## 6. Key rules and security
- **Video protection:** Cloudflare Stream with `requireSignedURLs = true`.
  Signed playback tokens are created only by a Supabase Edge Function after checking
  the student has an active subscription. Tokens expire after 2 hours.
- **Watermark:** overlay the student's name + ID on the player (moving position every 30s).
- **Device limit:** max 2 active devices per student account. Oldest session is signed out.
- **Children's data (Malaysia PDPA 2010):** parent must accept a consent notice before
  a student account (under 18) is created. Store consent version and time.
  Collect only needed data. No public student profiles.
- **Payments:** subscription status is updated ONLY from verified Stripe webhooks,
  never from the frontend.

## 7. Build phases (do one phase at a time)
### Phase 1 — Foundation
- Project setup (Vite, Tailwind, ESLint, Prettier, Vitest)
- Supabase auth (email + password), role-based routing
- Admin: manage levels and subjects

### Phase 2 — Video core
- Teacher: create course, upload video via Cloudflare Stream direct upload
- Student: browse by level → subject → course → lesson
- Video player with signed URL + watermark
- Watch progress saving

### Phase 3 — Payments and access
- Parent registration + PDPA consent + add children
- Stripe subscription (FPX + card), webhook Edge Function
- Block content when subscription is inactive
- Device limit

### Phase 4 — Learning features
- PDF notes upload/download (signed Storage URLs)
- Quizzes and scoring
- Parent dashboard: progress and quiz results per child
- Teacher dashboard: student progress

### Phase 5 — Launch
- Error handling, loading states, empty states
- Tests for auth, access control, webhook handling
- Deployment to Cloudflare Pages + Supabase production

## 8. Folder structure
```
src/
  app/            routes and layouts
  features/       auth, courses, lessons, player, payments, quizzes, dashboard
  components/     shared UI
  lib/            supabase client, api helpers, types
supabase/
  migrations/     SQL schema + RLS policies
  functions/      edge functions (stream-token, stripe-webhook, upload-url)
```

## 9. Working rules to save usage
- Work on ONE small task per request. Confirm the plan in 3–5 bullet points first.
- Do not read `node_modules`, `dist`, or lock files.
- Read only the files needed for the current task.
- After finishing a task, summarise changed files in a short list.
- If a requirement is unclear, ask instead of guessing.
