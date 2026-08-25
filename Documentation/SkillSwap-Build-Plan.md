# SkillSwap — 3 Week Build Plan

A peer-to-peer skill exchange app. Users list what they can teach and what they want to learn, send swap requests to each other, complete swaps, and leave reviews.

This plan is broken into phases. Each phase has:
- **What you're building** (in plain English)
- **Why it exists** (so it doesn't feel like busywork)
- **Concepts you'll learn** (so you know what to Google when stuck, and what to say in interviews)

Work through phases in order. Don't skip to frontend before the backend phase is working — testing with Postman/Thunder Client first will save you from debugging two things at once later.

---

## PHASE 1 — Project Setup & the User System (Days 1–2)

**What you're building:**
The skeleton of the backend, plus the ability for someone to register, log in, and get recognized as "logged in" on future requests.

**Why it exists:**
Every other feature depends on knowing *who* is making the request. You can't send a swap request, review someone, or see a dashboard without a logged-in user. This is the foundation everything sits on.

**Steps:**
1. Set up a Node + Express server, connect it to MongoDB (use MongoDB Atlas, not local — you'll need it live for deployment anyway).
2. Create a `User` model: name, email, password, bio, profile picture, average rating.
3. Build a `/register` route: take the user's details, **hash the password before saving** (never save plain text passwords).
4. Build a `/login` route: check the email exists, compare the password, and if it matches, generate a **JWT (JSON Web Token)** and send it back.
5. Build an `authMiddleware` — a function that runs before certain routes, reads the JWT from the request, and figures out who's making the request. If there's no valid token, it blocks the request.

**Concepts you'll learn:**
- **Express routing & controllers** — separating "what URL does this" from "what logic runs"
- **MongoDB schemas with Mongoose** — defining the shape of your data
- **Password hashing (bcrypt)** — why you never store plain passwords, and how hashing is one-way
- **JWT authentication** — what a token actually is (a signed piece of data, not a session stored on the server), how it proves identity on every request without needing to re-login
- **Middleware** — the idea of code that runs "in between" a request arriving and your controller handling it (this is a concept beginners struggle with the most, and interviewers love asking about it)
- **Environment variables (.env)** — keeping secrets like your JWT secret and DB connection string out of your code

**You'll know you're done when:** you can register a user in Postman, log in, get a token back, and use that token to access a protected test route.

---

## PHASE 2 — Skills & Relationships (Days 3–4)

**What you're building:**
The ability for a user to add skills they can teach and skills they want to learn.

**Why it exists:**
This is where you move from "one simple model" to "models that relate to each other" — which is the single biggest jump in difficulty (and value) from a todo app.

**Steps:**
1. Create a `Skill` model: name, category, and — importantly — a reference to *which user* it belongs to, and *whether it's a "teach" or "learn" skill*.
2. Build CRUD routes for skills: add a skill, edit it, delete it, list all skills for a user.
3. Protect these routes with your `authMiddleware` — only a logged-in user can add skills, and only to *their own* profile.
4. Link skills back to the user model (or query them by user ID — you'll learn both approaches and when each makes sense).

**Concepts you'll learn:**
- **One-to-many relationships in MongoDB** — one user has many skills, and how to reference documents instead of duplicating data
- **Referencing vs embedding** — a core Mongo design decision (should skills live *inside* the User document, or as a separate collection referenced by ID?). You'll actually make this decision yourself and learn the tradeoff.
- **Ownership-based authorization** — not just "are you logged in" but "is this *your* data" (e.g., you shouldn't be able to delete someone else's skill just because you're logged in)
- **CRUD route design conventions** — REST-style routes (`POST /skills`, `GET /skills/:userId`, `PUT /skills/:id`, `DELETE /skills/:id`)

**You'll know you're done when:** a logged-in user can add multiple teach/learn skills, and you can fetch all skills belonging to a specific user.

---

## PHASE 3 — Browse, Search & Filter (Day 5)

**What you're building:**
A way to browse other users and search/filter by skill.

**Why it exists:**
Without this, users can't actually *find* each other, which is the whole point of the app. It also forces you to write queries beyond simple "get all" or "get by ID."

**Steps:**
1. Build a route that returns all users who teach a given skill (e.g., search "guitar").
2. Add basic filtering (by category) and pagination (don't return 500 users at once).
3. Exclude the logged-in user's own profile from their own browse results.

**Concepts you'll learn:**
- **Query parameters** — reading things like `?skill=guitar&page=2` from a URL
- **MongoDB querying** — using operators like `$regex` for partial text search, `$ne` to exclude something
- **Pagination** — `skip()` and `limit()` in Mongo, and why loading everything at once doesn't scale

**You'll know you're done when:** you can search "excel" and get back a paginated list of users who teach it.

---

## PHASE 4 — The Swap Request System (Days 6–7)

**What you're building:**
The core feature: sending a request to swap skills with someone, and that request moving through a lifecycle (pending → accepted/rejected → completed).

**Why it exists:**
This is not a simple CRUD object — it's a **state machine**. This is the single most "interview-impressive" part of your project because it forces real logic, not just saving/fetching data.

**Steps:**
1. Create a `Request` model: sender, receiver, skill offered, skill wanted, status (`pending`, `accepted`, `rejected`, `completed`), timestamps.
2. Build a route to send a request (only logged-in users, can't request yourself).
3. Build a route to accept/reject a request — but **only the receiver** should be allowed to do this, not the sender.
4. Build a route to mark a request as completed — ideally requiring **both users** to confirm.
5. Add validation: you shouldn't be able to accept a request that's already rejected, or complete one that isn't accepted yet.

**Concepts you'll learn:**
- **State machines** — data that moves through defined stages, and enforcing valid transitions
- **Role-based / relationship-based permissions** — "only the receiver can do X" is a different kind of authorization than "only the owner can do X" from Phase 2
- **Data integrity / validation logic** — preventing impossible states, which is what separates a fragile app from a solid one
- **Referencing multiple users in one document** — a Request references *two* different users (sender and receiver), which is slightly trickier than the one-user references from Phase 2

**You'll know you're done when:** you can send a request as User A, accept it as User B, and mark it complete — and the API correctly blocks wrong actions (e.g., User A trying to accept their own sent request).

---

## PHASE 5 — Reviews (Day 8, morning)

**What you're building:**
After a swap is marked completed, both users can leave a rating + short review for each other.

**Why it exists:**
This teaches you how to attach data to a *specific transaction* rather than just to a user in general — and how to compute aggregate data (average rating) from many small pieces.

**Steps:**
1. Create a `Review` model: reviewer, reviewee, the request it's tied to, rating (1–5), comment.
2. Build a route to submit a review — only allowed if the related request is `completed`, and only once per user per request.
3. After a review is saved, recalculate and update the reviewee's average rating on their User document.

**Concepts you'll learn:**
- **Aggregation** — calculating an average from multiple documents (you can do this with a simple loop first, then learn MongoDB's `aggregate()` pipeline as a stretch goal)
- **Preventing duplicate actions** — stopping a user from reviewing the same swap twice
- **Denormalization** — why you might store a "cached" average rating on the User document instead of recalculating it every time you display a profile (a real performance tradeoff used in production systems)

**You'll know you're done when:** completing a swap unlocks a review option, and submitting a review updates the other user's average rating.

---

## PHASE 6 — Dashboard API (Day 8, afternoon)

**What you're building:**
One route that gathers everything a logged-in user needs to see: incoming requests, outgoing requests, active swaps, completed swaps, and their rating.

**Why it exists:**
This ties every model together in one response and teaches you `.populate()` — fetching related data across collections in a single query, which almost every real backend needs.

**Steps:**
1. Build a `/dashboard` route that queries the Request model for anything involving the logged-in user.
2. Use `.populate()` to pull in the other user's name/photo instead of just their ID.
3. Group the results by status (pending, accepted, completed) before sending the response.

**Concepts you'll learn:**
- **Populate / joins in MongoDB** — Mongo doesn't have SQL-style joins, but `.populate()` does the equivalent for referenced documents
- **Shaping API responses** — sending the frontend exactly the structure it needs, instead of raw database dumps

**You'll know you're done when:** hitting `/dashboard` as a logged-in user returns a clean object with all their request categories and the other person's basic info already attached.

---

## PHASE 7 — Frontend Setup & Auth (Days 9–10)

**What you're building:**
The React app skeleton, plus login/register pages and a way for the whole app to know if someone is logged in.

**Why it exists:**
Your backend is useless without a UI. This phase also teaches you how frontend auth actually works — it's not magic, it's just storing a token and attaching it to requests.

**Steps:**
1. Set up React (Vite is faster than Create React App), install React Router and Axios.
2. Build Login and Register pages with forms.
3. Create an `AuthContext` that stores the logged-in user and token, and makes it available to the whole app.
4. Set up an Axios instance that automatically attaches the JWT to every request, and handles 401 errors (e.g., redirect to login if the token is invalid/expired).
5. Build a `ProtectedRoute` component that blocks access to pages unless the user is logged in.

**Concepts you'll learn:**
- **React Router** — public vs protected routes
- **Context API** — sharing state (like "who is logged in") across your whole app without passing props everywhere
- **Axios interceptors** — automatically attaching tokens and catching auth errors in one place instead of repeating code
- **Persisting login state** — storing the token (localStorage) so refreshing the page doesn't log the user out

**You'll know you're done when:** you can register/login through the actual UI, and get redirected away from protected pages if you're not logged in.

---

## PHASE 8 — Core Frontend Pages (Days 11–13)

**What you're building:**
Profile page (add/edit skills), Browse page (search other users), and the Request flow (send/accept/reject) in the UI.

**Why it exists:**
This is where your backend work from Phases 2–4 becomes something a user can actually click through.

**Steps:**
1. Profile page: form to add teach/learn skills, list of your current skills with delete buttons.
2. Browse page: search bar + filtered list of users, each showing their teach skills and a "Request Swap" button.
3. Requests page: two tabs (incoming/outgoing), with accept/reject buttons on incoming ones.
4. Wire every page to your backend routes using Axios.

**Concepts you'll learn:**
- **Component composition** — building small reusable pieces (a `SkillCard`, a `UserCard`, a `RequestCard`) instead of one giant page file
- **Controlled forms** — managing form input state in React
- **Conditional rendering** — showing different buttons/UI depending on data (e.g., only show "Accept/Reject" if you're the receiver)
- **Lifting state / data fetching patterns** — deciding where API calls live and how data flows down to components

**You'll know you're done when:** the full loop works in the browser — add a skill, browse another test user, send them a request, and see it appear on their incoming requests.

---

## PHASE 9 — Dashboard, Reviews & Polish (Days 14–16)

**What you're building:**
The dashboard page (pulling from your Phase 6 API), the review submission UI, and general UI polish.

**Steps:**
1. Dashboard page: show request categories and average rating using your `/dashboard` API.
2. Add a "Leave a Review" button that appears only on completed swaps, opening a small form (star rating + comment).
3. Add loading states and error messages (don't leave blank screens while waiting on API calls).
4. Basic responsive styling — doesn't need to be fancy, but it shouldn't look broken on a laptop screen.

**Concepts you'll learn:**
- **Custom hooks** — extracting repeated logic (like `useFetch` or `useAuth`) into reusable functions
- **Loading/error UI states** — handling the in-between moments of an API call, not just the success case
- **Basic UX polish** — small things (disabled buttons while submitting, confirmation before destructive actions) that make a project feel finished, not thrown together

---

## PHASE 10 — Deployment (Days 17–18)

**What you're building:**
Getting the app live on the internet with a real URL.

**Steps:**
1. Deploy the backend (Render or Railway — both have free tiers).
2. Deploy the frontend (Vercel or Netlify).
3. Set environment variables on both platforms (don't hardcode your DB string or JWT secret).
4. Update your Axios base URL to point to the deployed backend instead of localhost.
5. Test the full flow live, not just locally.

**Concepts you'll learn:**
- **Environment-specific config** — how dev vs production URLs/secrets are managed
- **CORS** — why your deployed frontend and backend (different domains) need explicit permission to talk to each other, and how to configure it
- **Basic deployment troubleshooting** — reading logs when something works locally but not in production (a genuinely valuable real-world skill)

---

## PHASE 11 — Buffer, README & Interview Prep (Days 19–21)

**What you're doing:**
Not new features — protection time. Bugs always take longer than expected, and this phase exists so a slipping schedule doesn't wreck your finish date.

**Steps:**
1. Fix whatever broke during deployment.
2. Write a solid README: what the app does, tech stack, how to run it locally, and 2–3 screenshots.
3. Seed the database with a handful of realistic fake users so your live demo doesn't look empty.
4. Prepare to explain out loud: why you chose to reference vs embed data, how your JWT auth works, why the request status is a state machine, and one bug you hit and how you fixed it. Interviewers care more about *how you think* than the final polish.

---

## Quick Concept Checklist (for revision before interviews)

- [ ] Password hashing & why plain text passwords are dangerous
- [ ] JWT — what it is, how it's verified, stateless vs session auth
- [ ] Middleware — what it is and why it exists
- [ ] Mongoose schemas & relationships (referencing vs embedding)
- [ ] Authorization vs authentication (logged in vs "is this yours")
- [ ] REST API conventions
- [ ] State machines (your Request status flow)
- [ ] Populate / joining data across MongoDB collections
- [ ] Aggregation (average rating)
- [ ] React Context API
- [ ] Protected routes in React Router
- [ ] Axios interceptors
- [ ] CORS and environment variables in deployment

If you can explain every item on this list using your own project as the example, you're in a strong position for placements — because you're not repeating textbook definitions, you're describing decisions you actually made.
