## 04/09-2026 Progress:
### Project foundation

- Scaffolded the full Android app: Kotlin + Jetpack Compose, `minSdk` 26, package `za.org.team25310.ftcequipmenttracker`
- Set up Firebase (Auth + Firestore) with a Gradle version catalog and Gradle wrapper
- Renamed app to **"FTC Equipment Tracker"** (from "FTC Parts")
- Added a proper `.gitignore` for build outputs
- Chose **MIT** license for GitHub

### Core features built

- Loan creation: catalog part _or_ custom/3D-printed part, loaner/borrower teams, loan/due dates, location, notes, and camera photo
- Loans list with **item thumbnails** and status badges
- Loan detail screen: coach confirmations, captain signature pads, "mark returned"
- Real-time Firestore sync of the loan log
- Photos stored as compressed base64 in Firestore (avoids the paid Blaze plan)

### Trust model & team scoping (the big feature)

- `users` collection with roles: **coach / captain / member**
- **Invite-code verification** — new accounts select their team + role and enter a team invite code (validated server-side via Firestore rules)
- **Team-scoped loans** — each user only sees loans their team is part of (`teamIds` + `arrayContains`)
- Loaner side fixed to the current user's team; `teamIds` auto-populated
- Confirm/sign actions gated to a coach/captain of the matching team
- Status auto-advances `PENDING_CONFIRMATION → ACTIVE` when both sides confirm
- **Email verification** — sign-up sends a confirmation email; access is blocked until verified (with resend + "I've verified" flow)

### Bug fixes

- Camera crash → added runtime `CAMERA` permission request
- `startCamera` unresolved reference → fixed declaration order
- Signature flow → lift-finger multi-stroke, "Confirm signature" button, removed "Sign again"
- Crash after account setup → removed the composite-index requirement and caught the Firestore flow error
- Blank loaner field + disabled Save button → hoisted shared ViewModels to activity scope
- Blank photo preview → only set preview URI after successful capture
- Smart-cast compile error on `user` → captured into a local val
- `PERMISSION_DENIED` on onboarding → forced ID-token refresh after email verification

### Planning & documentation

- Read the engineering scoping document and produced a feature checklist + phased implementation plan
- Locked decisions: roles-first build, two-person sign-off, keep Firestore-base64 photos, single-part loans
- Delivered Firestore security rules (roles + email verification + invite codes) and Firebase setup walkthrough