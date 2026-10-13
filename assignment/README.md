# StudentForum — team implementation plan

HCMUT Database Lab, Team 4 (L10). This file is the working plan for building the StudentForum web app. Figma is the **visual** source of truth. The EERD and semantic constraints in the assignment report are the **data** source of truth.

- UI concept (read-only): [StudentForum — UI Concept](https://www.figma.com/design/jwt1OdHXU7yBkmSR1lH4zo/StudentForum-%E2%80%94-UI-Concept?node-id=31-4)
- Domain write-up: `database_lab_assignment_report/` (`main.tex` → `assignment1/analyse_description.tex`, `design_eerd.tex`)

Do not treat this README as an invitation to change the Figma file or the LaTeX report. Implement the app under `assignment/studentforum/` as described below.

---

## 1. Product idea

StudentForum is a student Q&A / community forum inspired by **Reddit** (communities, nested discussion, voting) and **Stack Overflow** (tags, accepted-quality ranking via votes, edit history), adapted to universities.

Each participating university has a **community**. Inside a community, students join, open **forums** (topic spaces), create **posts**, attach **tags**, write nested **comments**, **vote**, **subscribe** for notifications, and (after approval) may become **forum admins**. Community admins review forum proposals. A system admin handles universities at platform level.

### 1.1 Roles

| Role | How it is represented | What they do |
| --- | --- | --- |
| USER | Superclass of student / community admin | Account: username, hashed password, email, phones, gender, DOB, created date |
| STUDENT | Subtype of USER (`program`; studies at exactly one university) | Join communities, post, comment, vote, subscribe, propose forums |
| COMMUNITY_ADMIN | Subtype of USER (`privileges`); overlapping with STUDENT is allowed | Controls exactly one community; reviews forum-creation requests |
| FORUM_ADMIN | Role via `MANAGE` (a student manages one or more forums) | Moderates a forum; need not belong to that university |
| SYSTEM_ADMIN | Separate entity | Handles universities / platform-wide work, not community content |

### 1.2 Spaces and discussion objects

```
UNIVERSITY ──correspond──> COMMUNITY ──has──> FORUM ──contains──> POST ──has──> COMMENT
                                │                                      │
                                └── member_of / control                └── tags (M:N, at least one)
```

- **DISCUSSION_ITEM** is a category over FORUM, POST, and COMMENT. Votes, subscriptions, and “notification about” attach to this category, not to three duplicated relationship types.
- **POST EDIT** and **COMMENT EDIT** are weak entities (partial key `No.`, old content, edit time). Creator-only edits; **at most 5 edits**.
- Comments: recursive parent; parent and child must be on the **same post**; a comment cannot reply to itself.
- Votes: at most one current vote per (student, discussion item); change vote updates the row; remove vote deletes the row.
- Guest / non-member: community **preview is limited to 3 posts** until they join.

### 1.3 Semantic constraints to enforce in PostgreSQL + API

1. Students log in with a **verified educational email** whose domain matches a participating university (`universities.email_domain`).
2. Membership is required before full community access (join requirements).
3. Post `created_time` cannot be earlier than the parent forum `creation_date`.
4. Only a **member** of a community may request to create a forum there.
5. `comment_id ≠ parent_comment_id`.
6. Parent and child comments share the same `post_id`.
7. At most one current vote per student–item; update or delete rather than inserting a history of votes.
8. A post or comment may be edited **at most 5 times** by its creator.
9. Forum name is unique **within a community**.
10. A student may edit only their own posts/comments.

Also implement mapping-level rules: unique tag names; a post must have **≥ 1 tag**; CASCADE deletes for post–tag links and user notifications as ordinary FK behavior.

**Report gap:** `assignment1/mapping.tex` is still empty. The first database task is finish EERD → relational mapping, then write `schema.sql`. Do not invent tables that contradict the EERD.

### 1.4 Core tables (target after mapping)

`users`, `user_phones`, `students`, `community_admins`, `system_admins`, `universities`, `communities`, `member_of`, `forums`, `manage`, `discussion_items`, `posts`, `comments`, `tags`, `post_tags`, `votes`, `subscriptions`, `notifications`, `post_edits`, `comment_edits`.

Plus supporting columns/tables for forum-proposal status if needed so screens 05–06 can be stored without breaking the EERD (keep it as attributes/status on `forums` or a small request table documented in `docs/api.md`).

---

## 2. Is React + Express + PostgreSQL OK?

**Yes.** That stack fits this assignment:

| Layer | Choice | Why it fits StudentForum |
| --- | --- | --- |
| Database | **PostgreSQL** | Keys, M:N tables, weak entities, CHECK / UNIQUE / TRIGGER, `edu` domain checks |
| API | **Node.js / Express** | Thin REST over SQL; easy to quote in the lab report |
| UI | **React** | Matches the Figma high-fidelity screens and nested comment trees |

**Team defaults** (change only if the whole team agrees in Phase 0):

- Frontend: **Vite + React + JavaScript** (TypeScript is optional, not required)
- Auth: **JWT** + **bcrypt** password hashes (never store plain passwords)
- SQL in `database/schema.sql` + `database/seed.sql` (migrations folder ready for later)
- One shared API contract (`docs/api.md`) **before** splitting work so five people can code in parallel
- Local Postgres (or Docker Postgres) via `.env`; never commit secrets

---

## 3. Figma screens to implement

Source: Figma page **High-Fidelity Prototype** (file *StudentForum — UI Concept*). Implement these eight screens plus a shared chrome.

| # | Screen | Main behaviors |
| --- | --- | --- |
| 01 | Registration and email verification | Sign up / login; edu-email check; verification state |
| 02 | Community preview · three-post limit | Guest or non-member sees a short feed, lock overlay, **Join community** |
| 03 | Post detail · comments and notifications | Title, tags, edit history, upvote/downvote, nested comments, reply, sort |
| 04 | Create post | Content, attachments, **≥ 1 tag**, forum context |
| 05 | Propose forum | Member proposes a forum; waits for community-admin approval |
| 06 | Community admin review | Approve / reject forum requests; community controls |
| 07 | Forum admin moderation | Moderate posts/comments in assigned forums |
| 08 | Notifications inbox | List, read/unread, jump to the related discussion item |

**Shared sidebar (logged-in):** My communities, Saved posts, My subscriptions, My activity, **Your communities** (seed: HCMUT, HUTECH, UIT), “Have an idea for a forum?”, **Administration dashboard** (Assigned Community / Assigned Forum — show only if the user has those roles).

Visual target: light layout, blue primary buttons (`Join community`, `Join now`, `Upvote` / `Downvote`), tag chips, nested reply cards. Match Figma spacing and labels; do not restyle the product independently.

---

## 4. Project skeleton

Create the app here (this README stays at `assignment/README.md`):

```
assignment/
  README.md                 ← this plan
  studentforum/
    client/                 # Vite + React
      package.json
      index.html
      src/
        main.jsx
        App.jsx
        routes.jsx
        api/                # fetch wrappers; one file per resource
        components/         # Layout, Sidebar, VoteBar, TagChip, CommentThread
        pages/              # Screen01Register.jsx … Screen08Notifications.jsx
        styles/
    server/                 # Express
      package.json
      src/
        index.js            # listen, mount routes
        config/             # env, db pool
        middleware/         # auth JWT, roles, errors
        routes/
        controllers/
        services/           # constraint checks (edit cap, membership, votes)
        db/                 # query helpers
    database/
      schema.sql            # from finished EERD mapping
      seed.sql              # HCMUT / HUTECH / UIT + demo users
      migrations/           # optional follow-up ALTERs
    docs/
      api.md                # contract: routes, JSON, error codes
    .env.example
```

### 4.1 Frontend pages ↔ Figma

| File (suggested) | Figma screen |
| --- | --- |
| `pages/RegisterPage.jsx` | 01 |
| `pages/CommunityPreviewPage.jsx` | 02 |
| `pages/PostDetailPage.jsx` | 03 |
| `pages/CreatePostPage.jsx` | 04 |
| `pages/ProposeForumPage.jsx` | 05 |
| `pages/CommunityAdminPage.jsx` | 06 |
| `pages/ForumAdminPage.jsx` | 07 |
| `pages/NotificationsPage.jsx` | 08 |

### 4.2 Backend route groups (to lock in `docs/api.md`)

| Prefix | Owner phase | Examples |
| --- | --- | --- |
| `/api/auth` | Phase 1 | register, login, verify-email, me |
| `/api/universities`, `/api/communities` | Phase 1 | list, get, join, preview feed |
| `/api/forums` | Phase 2–3 | list, propose, get |
| `/api/posts`, `/api/tags` | Phase 2 | CRUD, attach tags |
| `/api/comments` | Phase 2 | create, reply, sort |
| `/api/votes`, `/api/subscriptions` | Phase 2 | upsert/delete vote, subscribe |
| `/api/edits` | Phase 2 | post/comment history, 5-edit cap |
| `/api/admin/community`, `/api/admin/forum` | Phase 3 | review, moderate |
| `/api/notifications` | Phase 3 | list, mark read |

Agree request/response JSON **in Phase 0**. Frontend may mock these shapes until the matching route exists.

---

## 5. Team (5 people)

| Student ID | Name | Short name |
| --- | --- | --- |
| 2411545 | Trần Nguyễn Khánh | Khánh |
| 2410148 | Trần Hồng Anh | Hồng Anh |
| 2411997 | Đặng Thị Hoàng Mai | Mai |
| 2412166 | Lê Mai Nam | Nam |
| 2411258 | Phạm Gia Huy | Huy |

### Role matrix (folders / screens)

| Person | Primary folders | Screens / features |
| --- | --- | --- |
| **Khánh** | repo, `docs/api.md`, `server/src/middleware`, `client/src/api` | 01 auth UI, later wire all `api/` clients, role-aware sidebar |
| **Hồng Anh** | `database/schema.sql`, mapping, edit-history SQL | constraints; 06 community admin review |
| **Mai** | membership, comments, votes UI+API | 02 preview lock; nested comments + sort; 05 propose forum |
| **Nam** | `client/src/components`, `client/src/pages` | shared Layout/Sidebar (Figma); 03–04 posts; 07 forum admin UI |
| **Huy** | `discussion_items`, votes/subscribe services, notifications | vote upsert/delete; 08 inbox; trigger-style constraints |

Everyone reviews `docs/api.md` before writing code that another person must consume.

---

## 6. Implementation roadmap

Work **API contract first**, then vertical slices. Do not wait for a finished monolith.

```
Phase 0  →  Phase 1  →  Phase 2  →  Phase 3  →  Phase 4
contract     DB+auth     forum core   admin+notifs  demo
```

### Phase 0 — together (day 1–2)

Create the folder tree above, a running empty Vite app and empty Express server, Postgres connection, `.env.example`.

Khánh owns the repo layout and keeps this README in sync.

**Must produce before Phase 1 splits:**

1. Finished EERD → relation list (fill mapping in the report later; draft tables in `schema.sql` comments is OK)
2. `docs/api.md` with method, path, body, success JSON, error codes
3. Seed university list: HCMUT, HUTECH, UIT (email domains)

### Phase 1 — foundation (parallel)

| Person | Do this |
| --- | --- |
| Huy + Hồng Anh | `schema.sql` + constraints (edu email vs `email_domain`, unique forum name per community, parent comment same post, max 5 edits, one vote per student–item) |
| Khánh | Auth API + React register/login (screen 01) |
| Mai | Membership + community preview 3-post lock (screen 02) |
| Nam | Shared layout/sidebar matching Figma, using mock data until APIs exist |

**Exit criteria:** a student with an `@hcmut.edu.vn`-style address can register, log in, and see a locked community preview.

### Phase 2 — discussion core (parallel)

| Person | Do this |
| --- | --- |
| Nam | Posts, tags, create post (04), post detail shell (03) |
| Mai | Nested comments, sort by time / upvotes / downvotes (tie-break: newer first), votes on comments |
| Huy | `discussion_items`, vote update-or-delete, subscribe |
| Hồng Anh | Edit history + 5-edit cap (API + DB) |
| Khánh | Replace mocks in `client/src/api` with real endpoints |

**Exit criteria:** member can create a tagged post, comment/reply, vote, edit (≤ 5), see history.

### Phase 3 — admin and notifications (parallel)

| Person | Do this |
| --- | --- |
| Mai | Propose forum (05) |
| Hồng Anh | Community admin review (06) |
| Nam | Forum admin moderation (07) |
| Huy | Notification create / list / read (08) |
| Khánh | Sidebar: Assigned Community / Assigned Forum only when the JWT user has those roles |

**Exit criteria:** propose → admin approve → proposer can be assigned as forum admin; subscribers get notifications.

### Phase 4 — integrate and demo

- Seed HCMUT / HUTECH / UIT communities and forums
- Demo accounts: student, community admin, forum admin, system admin
- Constraint test list (try 6th edit, duplicate forum name, non-edu email, vote flip)
- Visual pass against Figma screens 01–08
- Short demo script for the lab session

---

## 7. Minimum demo path

Use this sequence in the oral demo (happy path):

1. **Register** with a university edu email → verify → log in (01)
2. Open a community as guest → see **3-post preview** and lock → **Join** (02)
3. **Create a post** with at least one tag (04)
4. Open **post detail** → comment → nested reply → upvote/downvote → view edit history (03)
5. **Propose a forum** (05)
6. Switch to **community admin** → approve (06)
7. **Forum admin** moderates the new forum (07)
8. Open **notifications inbox** for a subscriber (08)

---

## 8. Working rules

1. **Contract first.** If the JSON shape is not in `docs/api.md`, do not invent a private format.
2. **Constraints live in the database**, then the API returns a clear error. UI only mirrors those errors.
3. **Do not block** on another person’s unfinished screen: mock the agreed JSON.
4. **No Figma edits** as part of implementation; screenshot/compare instead.
5. Passwords hashed; `.env` not committed; JWT on protected routes; role checks on admin routes.

When Phase 0 is done, start implementation in `assignment/studentforum/` following this file.
