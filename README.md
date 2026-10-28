# StudyVault — Smart Student Notes Sharing & Learning Platform

StudyVault lets college students upload lecture PDFs, extract topics automatically, review and correct AI-assisted drafts, and share moderated notes by department, semester, subject, lesson, and topic. Contributions earn points that move students through grade levels and onto a leaderboard.

**Stack:** React 19 · Vite · React Router · Firebase Auth · Cloud Firestore · Firebase Storage · Cloud Functions (v2) · PDF.js (`pdfjs-dist`) · Lucide React · custom CSS design system (light and dark themes).

---

## 1. Quick start (demo mode, no backend needed)

```bash
npm install
npm run dev        # http://localhost:5173
```

Without Firebase configuration the app runs in **demo mode**. A clear banner says so. Data is stored in the browser's `localStorage`, PDFs and images live only for the current session, and nothing is secure. Demo mode is for trying the workflow.

- Students: register your own account. The seeded students on the leaderboard are sample profiles and cannot sign in.
- Admin: open **Admin sign-in**. The demo administrator's email and a password generated for your browser are shown on that page. No admin password is hard-coded in the source.

Production builds: `npm run build` creates a single self-contained `dist/index.html`.

---

## 2. Production setup (Firebase)

### 2.1 Create the Firebase project

1. In the [Firebase console](https://console.firebase.google.com), create a project.
2. **Authentication → Sign-in method:** enable **Email/Password**.
3. **Firestore Database:** create a database in production mode and choose a region.
4. **Storage:** get started (production mode).
5. **Project settings → Your apps → Web app:** register a web app and copy the config values.
6. Upgrade to the **Blaze** plan. Cloud Functions and Secret Manager require it, and usage for a campus pilot is typically within the free allowances.

### 2.2 Configure the frontend

```bash
cp .env.example .env
# fill in VITE_FIREBASE_* values from step 2.1
```

`VITE_*` values are public web identifiers. Security comes from Auth, the security rules, and Cloud Functions. Never put provider keys or service-account credentials in `VITE_*` variables.

### 2.3 Install and deploy the backend

```bash
npm install -g firebase-tools
firebase login
firebase use --add                     # select your project
cd functions && npm install && cd ..
firebase deploy --only firestore:rules,firestore:indexes,storage,functions
```

Wait for the `firestore:indexes` build to finish. Two composite indexes are required (leaderboard and duplicate checks) and are defined in `firestore.indexes.json`.

**Optional AI drafts.** Store the provider key as a secret. It is never bundled into the web app:

```bash
firebase functions:secrets:set AI_API_KEY      # paste an OpenAI-compatible API key
firebase deploy --only functions
```

Set `AI_MODEL` in the functions environment if you want a model other than `gpt-4o-mini`. Without a key, the app falls back to a clearly labelled offline draft generator.

### 2.4 Create the first administrator

Admins are granted with a Firebase Auth **custom claim**. Students cannot grant it to themselves. The Firestore and Storage rules and every admin Cloud Function check the claim.

1. Register the admin's account in the app (or create it in the Authentication console).
2. Create a service-account key: **Project settings → Service accounts → Generate new private key**. Keep the file out of source control.
3. From `functions/`:

```bash
GOOGLE_APPLICATION_CREDENTIALS=/secure/path/service-account.json node grantAdmin.js admin@university.edu
```

4. The admin signs in at **/admin/login**. The app refreshes the token automatically. Revoke with `--revoke`.

### 2.5 Host the frontend

```bash
npm run build
firebase deploy --only hosting        # firebase.json serves dist/ with SPA rewrites
```

Any static host works too. Serve the contents of `dist/`, and rewrite unknown paths to `index.html`.

### 2.6 First content

Sign in as an admin, open **Subjects & lessons**, and add departments' subjects and lessons. Students choose from this catalog when they upload.

---

## 3. Scripts

| Command | Purpose |
| --- | --- |
| `npm run dev` | Local development server |
| `npm run build` | Production build to `dist/` |
| `npm run preview` | Preview the production build |
| `cd functions && npm run serve` | Run Functions and emulators locally |

---

## 4. Workflow

1. **Upload** (`/app/upload`): validate the PDF (type and 15 MB limit), store it in Firebase Storage, extract text and headings with PDF.js in the browser, and save draft topics.
2. **Review** (`/app/notes/:id/review`): correct titles, explanations, key points, definitions, examples, lesson names, and images. Generate AI or offline study aids per topic. Every AI output is flagged as a draft.
3. **Submit**: the `submitNote` function validates completeness, applies a daily limit, checks duplicates (server-side hash of the PDF bytes plus a content fingerprint), and awards provisional points.
4. **Moderate** (`/admin/reviews`): approve, reject with a reason, or request changes. `reviewNote` awards points in a transaction and writes the ledger and audit entries.
5. **Publish**: approved topics appear in Explore and subject pages. Students can bookmark, mark topics complete, mark them helpful, report content, and read the source PDF.
6. **Edits after publication** go into `pendingRevision` and are shown only after an admin approves them. The live version stays unchanged in the meantime.

---

## 5. Points, grades, and leaderboard

Policy values live in `src/config/appConfig.js` and are mirrored in `functions/policy.js`. The server copy is authoritative.

| Event | Points | Notes |
| --- | --- | --- |
| Valid submission | +5 provisional | Not counted on the leaderboard until approved |
| Note approved | +20 | Plus **+5 per topic** with an explanation of at least 40 characters |
| Submission rejected or duplicate | provisional removed | No points |
| Helpful vote on a topic | +2 to the author | One vote per reader per topic; the author's helpful points are capped at 20 per ISO week |
| Note removed by moderators | points reversed | Restoring re-awards them |

**Grades (approved points):** Bronze Scholar 0 · Silver Scholar 100 · Gold Scholar 300 · Platinum Scholar 700 · Diamond Scholar 1500.

**Leaderboard:** all-time and weekly (ISO week) rankings use competition ranking, so tied scores share a rank and are labelled. Only approved points count.

**Abuse controls:** 10 submissions per student per day, duplicate detection on exact PDF bytes and content fingerprints, one helpful vote per reader per topic, no self-rating, no points for edits, a weekly cap on helpful points, a daily AI-draft quota, and suspensions that revoke sessions.

---

## 6. Security model

| Concern | Enforcement |
| --- | --- |
| Admin privileges | Firebase Auth custom claim `admin`, set only by `grantAdmin.js`. Checked by rules and by every admin function. Admin pages never trust client flags. |
| Editing points, grade, rank, counters | Clients cannot write these fields (`publicProfiles` update allow-list; create requires zeros). Only Cloud Functions write them. |
| Own drafts | Owners create, edit, and delete only drafts and returned notes. Status changes happen on the server. |
| Published content visibility | Readable by signed-in students only when `status == 'published'`. Drafts are visible to owners and admins. |
| Approvals and rejections | `reviewNote` and `moderateTopic` callables, admin only, transactional. |
| Email and private data | Stored in `users/{uid}` (owner and admin read). Public views use `publicProfiles`, which has no email. |
| Suspended accounts | Rules deny writes, sign-in is blocked, and refresh tokens are revoked. |
| Uploads | Storage rules enforce owner-only writes, PDF or image content types, and size limits (PDF 15 MB, images 3 MB, avatars 2 MB). |
| AI provider key | Cloud Functions secret (`AI_API_KEY`). The browser only calls `generateStudyAids`. |
| Passwords | Handled entirely by Firebase Auth. The app never stores or logs them. Password-reset responses do not reveal whether an account exists. |

**Caveat:** Firebase Storage download URLs include access tokens. Anyone holding a URL can open the file, so treat PDF links as shared-by-link. For stricter access, serve files through a function that issues short-lived signed URLs.

---

## 7. PDF processing and OCR

- **Extraction:** PDF.js reads selectable text, groups it into lines, and detects headings by font size, numbering (`1.2 Title`), and keywords (`Unit`, `Chapter`, `Module`). Running headers and footers that repeat across pages are removed.
- **Fallback:** if no headings are found, the PDF is split by page. The review screen warns about this.
- **Scanned PDFs:** when almost no selectable text exists, the app says so and lets the student add topics manually.
- **OCR (optional extension):** add `tesseract.js` and run recognition on the page canvases for pages without text. The analysis module (`src/lib/pdfAnalysis.js`) already accepts line objects, so OCR output can feed the same pipeline. OCR is not enabled by default because it is slow and large.

---

## 8. Known limitations

- The Firebase backend (rules, functions, storage) was written against the Firebase v10+ APIs but must be deployed and tested in a real project. Run the emulator suite before production use.
- Topic search and filtering run in the browser over the most recent 300 published topics. For a larger library, add Algolia or Typesense.
- Renaming a student does not update the denormalised `ownerName` on existing topics and notes.
- Demo mode is not secure and must not be used for real students.

---

## 9. Manual QA checklist

Run through these against a configured Firebase project:

1. **Registration:** create an account with name, email, department, and semester. Validation messages appear for empty or invalid fields.
2. **Login and logout:** sign in, refresh the page (session persists), sign out.
3. **Password reset:** request a reset. The same confirmation appears for any email.
4. **Profile:** edit display name, bio, skills, and photo. Email is visible only on your own profile.
5. **Upload:** try a non-PDF and an oversized file (both rejected). Upload a text PDF, check the detected topics, and edit them.
6. **AI draft:** generate study aids for a topic and confirm the draft warning appears.
7. **Submit:** submit a note. Confirm the status becomes "Pending review" and provisional points are shown. Submit the same PDF again and confirm the duplicate is blocked.
8. **Admin approval:** sign in at `/admin/login`, approve the submission, and confirm points move from pending to approved and the topics appear in Explore.
9. **Points and grades:** confirm the grade and progress bar update. Confirm a student cannot edit points in the browser console (rules deny the write).
10. **Engagement:** bookmark, mark complete, mark helpful (not on your own topic), and report a topic.
11. **Revisions:** edit a published topic, confirm the live version is unchanged until an admin approves the revision.
12. **Leaderboard:** check competition ranking with tied scores, switch between all-time and weekly, and confirm your own row is highlighted.
13. **Suspension:** suspend a student and confirm they are signed out and blocked. Reinstate them.
14. **Responsive layout:** check the sidebar drawer, tables, and review screen at mobile width. Check dark mode.

---

## 10. Project structure

```
src/
  config/        env.js (Firebase config and mode), appConfig.js (policy, grades, limits)
  lib/           utils (validation, formatting), grades (levels, ranking), pdfAnalysis (PDF.js extraction)
  services/      firebase.js (single init), topicModel.js (shared mapping)
    api/         auth, users, notes, admin: each function picks Firebase or demo backend
    demo/        demoDb.js (localStorage backend for demo mode only)
  context/       AppProviders.jsx (auth/session/profile + toasts)
  hooks/         useAsync.js
  components/
    ui/          design-system primitives (buttons, forms, modal, tabs, badges…)
    layout/      AppShell (student + admin), PublicLayout, RouteGuards
    common/      TopicCard
    topics/      TopicEditor (editing), TopicView (reading)
  pages/
    public/      Landing
    auth/        Login (+ ForgotPassword), Register, AdminLogin
    student/     Dashboard, UploadNotes, PdfReview, Explore, Subjects, SubjectDetail,
                 TopicReader, Library, Leaderboard, Profile
    admin/       AdminDashboard, AdminReviews, AdminUsers, AdminCatalog
functions/       Cloud Functions (submitNote, reviewNote, moderateTopic, markTopicHelpful,
                 generateStudyAids, setUserSuspended), policy.js, grantAdmin.js
firestore.rules  storage.rules  firestore.indexes.json  firebase.json
```
