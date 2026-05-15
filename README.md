# 🏆 PosterVote — GitHub Pages Edition

A **zero-build** poster session voting system that runs directly from GitHub Pages.  
No Node.js, no build step, no server — just two HTML files.

---

## 🚀 Deploy in 5 Minutes

### Step 1 — Fork / Upload to GitHub

1. Create a new GitHub repository (e.g. `poster-vote`)
2. Upload both files:
   - `index.html` — Admin panel
   - `vote.html` — Student voting page
3. Go to **Settings → Pages → Branch: main → / (root)** → Save
4. Your site is live at: `https://YOUR_USERNAME.github.io/poster-vote/`

### Step 2 — Create a Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com) → **Add project**
2. Enable **Firestore Database**
   - Start in **test mode** for initial setup
   - Later, apply the security rules below
3. Enable **Authentication → Sign-in method → Email/Password**
4. Create your admin user: **Authentication → Users → Add user**
   - e.g. `admin@yourschool.edu` with a strong password
5. **Project Settings → Your Apps → Add web app** → copy the config

### Step 3 — First-time Admin Setup

1. Open `https://YOUR_USERNAME.github.io/poster-vote/`
2. You'll see a **Firebase Setup** screen — paste your config values
3. Click **Save & Continue** — config is saved in your browser
4. Log in with your admin email and password

**That's it!** The voting link for students will be:
```
https://YOUR_USERNAME.github.io/poster-vote/vote.html?class=CLASS_ID
```
(The class ID is shown when you click "Voting" tab inside a class.)

---

## 🔐 Firestore Security Rules

In Firebase Console → Firestore → Rules, replace the default rules with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /classes/{classId} {
      allow read: if true;
      allow write: if request.auth != null;

      match /students/{studentId} {
        allow read: if true;
        allow write: if request.auth != null;
        allow update: if request.auth == null
          && request.resource.data.diff(resource.data).affectedKeys()
             .hasOnly(['hasVoted', 'votedForGroupId']);
      }

      match /groups/{groupId} {
        allow read: if true;
        allow write: if request.auth != null;
      }

      match /votes/{voteId} {
        allow read: if request.auth != null;
        allow create: if true;
        allow update, delete: if request.auth != null;
      }
    }
  }
}
```

---

## ⚠️ Important Notes

### Firebase config on student devices
The Firebase config is stored in **your browser's localStorage**. This means:
- Students who open `vote.html` on their own device need the config too.
- **Solution**: Before the voting session, visit `vote.html?class=YOUR_CLASS_ID` on a shared/classroom device where you've already opened the admin panel from the same browser. The config will be present.
- **Better solution**: Add `?config=...` URL support OR host on a custom domain and pre-bake the config. See "Baking config into HTML" below.

### Baking your config into the HTML (recommended for classrooms)

Edit `vote.html` and `index.html`. Find this line near the top of the `<script>` section:

```js
let cfg;
try { cfg = JSON.parse(localStorage.getItem('pv_config')||'null'); } catch{}
```

Replace it with your actual config (hardcoded):

```js
let cfg = {
  apiKey: "AIzaSy...",
  authDomain: "yourproject.firebaseapp.com",
  projectId: "yourproject",
  storageBucket: "yourproject.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123:web:abc"
};
```

This way students don't need to set up anything — just open the link.

---

## 📋 CSV Format

```csv
name,id,email
Alice Johnson,20210001,alice@university.edu
Bob Smith,20210002,bob@university.edu
```

Accepted column names: `name` / `full_name`, `id` / `student_id`, `email` / `email_address`

---

## 🗺 Application Flow

```
Admin (index.html)                   Students (vote.html?class=ID)
──────────────────                   ──────────────────────────────
1. Create class                      
2. Import CSV → PINs generated       
3. Create groups, assign students    
4. Set project/poster names          
5. Open voting, copy link  ───────►  6. Open link, enter PIN
                                     7. Select project to vote for
                                     8. Rate Design / Novelty / Content (★)
                                     9. Submit (once only, can't vote own group)
10. View live results     ◄────────  Vote saved to Firestore
11. Close voting when done
```

---

## 📁 Files

| File | Purpose |
|------|---------|
| `index.html` | Admin panel — classes, students, groups, voting control, results |
| `vote.html` | Student voting page — PIN auth + scoring form |

No build tools, no dependencies to install, no server required.
