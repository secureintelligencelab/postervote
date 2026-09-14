# 🏆 PosterVote — Campus Edition

A **zero-build** poster session voting system that runs directly from GitHub Pages.
No Node.js, no build step, no server — just two HTML files.

---

## 🆕 What Changed in This Version

| Before | Now |
|---|---|
| Students logged in with a generated 6-char PIN | Students log in with their **Student ID** (e.g. `CIHE2026`), **case-insensitive** |
| Admin had to distribute PINs | Nothing to distribute — students already know their ID |
| A student could rate every project | A student may rate **at most 3 projects** (configurable per campus) |
| Partial ratings could be submitted | Every project you open must be scored on all 3 criteria |
| Straight to a long scrolling form | Confirm identity → pick projects → score → **review** → submit |
| "Teacher code" | "Guest code" (`mode=guest`); still a shared code, unchanged behaviour |
| Campuses were the top level | **Semesters** are the top level; campuses live inside a semester |

Guests / judges are **unchanged**: they still enter a shared access code and can rate every project.

**Backwards compatible**: `vote.html` still accepts `?class=` links and `?mode=teacher`, and rosters
imported under the old system are upgraded automatically the next time an admin opens the campus.

---

## 🚀 Deploy in 5 Minutes

### Step 1 — Fork / Upload to GitHub

1. Create a new GitHub repository (e.g. `poster-vote`)
2. Upload both files: `index.html` and `vote.html`
3. Go to **Settings → Pages → Branch: main → / (root)** → Save
4. Site live at: `https://YOUR_USERNAME.github.io/poster-vote/`

### Step 2 — Create a Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com) → **Add project**
2. Enable **Firestore Database** (test mode for setup)
3. Enable **Authentication → Sign-in method → Email/Password**
4. Create admin user: **Authentication → Users → Add user**
5. **Project Settings → Your Apps → Add web app** → copy config

### Step 3 — Paste Firebase Config

In both `index.html` and `vote.html`, find the `firebaseConfig` block near the top of the `<script>` and paste your values.

---

## 🏫 Campus Workflow

```
Admin (index.html)                       Students / Guests (vote.html?campus=ID)
──────────────────                       ──────────────────────────────────────
0. Create a Semester (e.g. Semester 1 2026)
1. Inside it, create one Campus per location
2. Import CSV (name + id required)  ──►  Only matching campus rows imported
3. Create groups, assign students   ──►  Student cannot rate their own group
4. Set Guest Code (for judges)
5. Set max projects per student (3)
6. Open voting, share links   ──────►    Student: enters Student ID, confirms name,
                                          picks up to 3 posters, scores each on
                                          Design / Novelty / Content ★, reviews, submits
                                          Guest: enters Guest Code, rates any number
7. View Results ◄─────────────────────    Votes saved per campus
   ► Best Poster winner banner
   ► Semester Results = winner per campus
8. Close voting
```

---

## 📅 Semesters → Campuses

The admin panel now has three levels:

```
Semester  (e.g. "Semester 1 2026")
└── Campus  (e.g. "Main Campus")
    ├── Students
    ├── Groups / posters
    └── Voting session + Best Poster winner
```

Each semester keeps its own campuses, so next teaching period you create a new semester
and start fresh without touching last period's rosters, votes or winners.

**Nothing about the voting page changed.** Campuses are still stored as a top-level
Firestore collection and simply carry a `semesterId` field, so every existing
`?campus=ID` link, printed QR code and security rule keeps working exactly as before.

### Working with semesters

- **Create**: *Semesters → + New Semester*. Names must be unique.
- **Add campuses**: open the semester, then *+ New Campus*. A campus always belongs to
  the semester you created it in.
- **Move a campus**: the ⇄ button on a campus card moves it to another semester,
  taking its students, groups and votes with it. Links stay valid.
- **Delete a semester**: only allowed once it is empty. If it still holds campuses the
  panel refuses and tells you how many — this stops one click from wiping a whole
  semester of student data.
- **Semester Results**: a 🏆 overview showing the Best Poster of every campus in the
  semester side by side, with runners-up and vote counts, plus a link into each
  campus's full leaderboard.

### Upgrading existing data

Campuses created before this change have no `semesterId`. They appear together in an
**"Unassigned campuses"** card on the Semesters screen and keep working normally.
Open it and use ⇄ on each campus to file them into a semester. Once the last one is
moved, the card disappears on its own.

---

## 🔑 Student Login

Students sign in with the **Student ID** stored on their roster entry.

- **Case-insensitive**: `CIHE2026`, `cihe2026` and `CiHe2026` all work.
- Whitespace is ignored.
- After entering the ID, the student sees **"Is this you?"** with their name and group
  before voting, so a mistyped ID can't silently cast someone else's vote.
- Each Student ID can submit **once** (`hasVoted` is set on submission).

The admin panel flags any student with a **missing or duplicated** Student ID, because those
students cannot sign in. Use ✏️ in the Students tab to fix them, or **+ Add Student** to add one by hand.

> **Note on security:** Student IDs are more guessable than random PINs. Voting is only
> open during the window the admin chooses, each ID votes once, and students can't rate
> their own group — but if you need stronger guarantees for a high-stakes prize,
> keep the voting window short and watch the results page live.

---

## 🗳 The 3-Project Limit

- A student taps **"Rate this project"** to add a poster to their picks. After 3 picks,
  the remaining cards lock and show *"Limit reached"*.
- Removing a pick frees the slot again.
- A project only counts once **all three criteria** are scored — this prevents partial
  ratings from being averaged in as zeros.
- The limit is configurable per campus: **Voting tab → Maximum projects one student may rate**.
- Guests and judges are **not** limited.

To enforce the cap server-side as well, the Firestore rules below include a `ratings.size()` check.

---

## 📋 CSV Format

```csv
name,id,email,campus
Alice Johnson,CIHE2026,alice@university.edu,Main Campus
Bob Smith,CIHE2027,bob@university.edu,City Campus
Carol Lee,CIHE2028,carol@university.edu,Main Campus
```

- **name** and **id** are required. `id` is the student's login — rows without one are skipped.
- **campus** is optional. If present, only rows matching the campus name (case-insensitive)
  are imported; others are skipped with a count shown. If omitted, all rows import here.
- Rows whose ID is already on the roster are skipped, so **re-importing the same file is safe**.

---

## 🔐 Firestore Security Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // semesters are admin-only; the voting page never reads them
    match /semesters/{semesterId} {
      allow read, write: if request.auth != null;
    }

    match /campuses/{campusId} {
      allow read: if true;
      allow write: if request.auth != null;

      match /students/{studentId} {
        allow read: if true;
        allow write: if request.auth != null;
        // unauthenticated voters may only flip their own hasVoted flag
        allow update: if request.auth == null
          && request.resource.data.diff(resource.data).affectedKeys()
             .hasOnly(['hasVoted']);
      }

      match /groups/{groupId} {
        allow read: if true;
        allow write: if request.auth != null;
      }

      match /votes/{voteId} {
        allow read: if request.auth != null;
        // students may submit at most 3 projects in one vote; guests are unlimited
        allow create: if request.resource.data.voterType != 'student'
                      || request.resource.data.ratings.size() <= 3;
        allow update, delete: if request.auth != null;
      }
    }

    // Legacy class-based rules (keep if upgrading from the old version)
    match /classes/{classId} {
      allow read: if true;
      allow write: if request.auth != null;
      match /students/{s}{ allow read: if true; allow write: if request.auth != null; }
      match /groups/{g}{ allow read: if true; allow write: if request.auth != null; }
      match /votes/{v}{ allow read: if request.auth != null; allow create: if true; allow update,delete: if request.auth != null; }
    }
  }
}
```

If you raise the per-campus limit above 3, raise the `<= 3` in the rule to match.

---

## 🗄 Firestore Structure

```
semesters/{semesterId}
  .name, .description, .createdAt

campuses/{campusId}                       ← still top-level, so links never break
  .semesterId, .semesterName              ← which semester this campus belongs to
  .name, .description, .votingOpen, .guestCode, .teacherCode, .maxVotesPerStudent
  /students/{id}   .name, .studentId, .studentIdLower, .email, .campusId, .groupId, .hasVoted
  /groups/{id}     .name, .projectName, .memberIds, .campusId
  /votes/{id}      .voterType ('student'|'guest'), .voterName, .studentId,
                   .ratings {groupId: {design,novelty,content}}, .ratedCount, .submittedAt
```

`studentIdLower` is the lower-cased copy of `studentId` that makes login case-insensitive.
It is written on import and backfilled automatically for older records when an admin
opens the campus. `vote.html` also falls back to a client-side scan if it is ever missing.

---

## 📁 Files

| File | Purpose |
|------|---------|
| `index.html` | Admin panel — campuses, students, groups, voting control, results |
| `vote.html` | Voting page — Student ID sign-in or guest code + scoring form |
| `Score Calculation Formula.docx` | The weighted scoring maths used on the results page |

---

## 🗺 Application Flow Summary

- **One semester = many campuses.** One campus = one voting session = one Best Poster winner
- Campuses without a `semesterId` show up under **Unassigned campuses** until you move them
- Students sign in with their **Student ID**, confirm who they are, and rate **up to 3** projects
- Students **cannot** rate their own group
- Guests log in with the Guest Code and can rate every project
- Guest/judge ratings are weighted more heavily than student ratings (70/30 base,
  adjusted by how many votes each side cast) — see the formula document
- Results page shows a **🏆 Best Poster** banner per campus, plus a ranked leaderboard
- Auto-refresh every 15 seconds in live mode
