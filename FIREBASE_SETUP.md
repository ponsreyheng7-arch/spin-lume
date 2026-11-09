# Firebase setup for spin-lume

The app stores spin records in Cloud Firestore and signs users in anonymously.
No SMS verification is used. Admin management uses a separate Email/Password
Firebase Authentication account and an explicit Firestore admin role.

## One-time Firebase Console setup

1. In project `spin-lume`, create a Cloud Firestore database in Native mode.
   The Spark plan includes Firestore's published free usage limits.
2. In **Authentication → Sign-in method**, enable **Anonymous** and
   **Email/Password**, then click **Save** for each provider. If the app reports
   `auth/admin-restricted-operation`, Anonymous is still disabled (or the
   provider change has not propagated yet). Confirm the Console is showing the
   `spin-lume` project, enable **Anonymous**, save, wait briefly, and retry.
3. In **Authentication → Settings → Authorized domains**, add the domain used
   to host this app, including the Vercel domain such as
   `spin-lume.vercel.app` (enter only the hostname, not `https://`). Add any
   custom domain too. The app must be served over HTTP(S), not opened as
   `file://`.
4. Create the first Admin account under **Authentication → Users → Add user**.
   Keep its password private and copy its Firebase **UID**.
5. In **Firestore Database → Data**, create a collection named `admin`. Add
   one document whose document ID is that Admin UID, with this field:

   ```json
   {
     "active": true
   }
   ```

   The app's Admin sign-in checks this role document. Do not give users write
   access to the `admin` collection.
   The Admin Email/Password form only signs in accounts created in Firebase
   Authentication; entering the old phone number or creating only a Firestore
   document is not enough. If sign-in succeeds but the Admin role is missing,
   the app shows the UID to use for the `admin/{UID}` document.

## Deploy Firestore rules and Hosting

The app is a static HTML app. `vercel.json` rewrites the site root to
`water_festival_lucky_spin_app.html`; Vercel serves the app and `assets/`
without a build step. Import this folder's GitHub repository into Vercel and
deploy with the default static/Other framework settings.

Vercel deploys the website, but does **not** deploy Firestore rules. From this
folder, install/use the Firebase CLI, sign in, and deploy the rules separately:

```powershell
npx firebase-tools login
npx firebase-tools deploy --only firestore:rules --project spin-lume
```

The included `.firebaserc` targets `spin-lume`. Confirm deployment completed
successfully. The published rules allow authenticated users to read one spin
document and create an immutable result; only an Admin can list or delete all
results. The app uses a SHA-256 phone document ID and a Firestore transaction to
avoid duplicate records when devices submit the same phone concurrently.

## Data migration

Old `water_fest_spin_history` data is stored per browser and is not copied
automatically. The Admin page only displays records stored in Firestore.

## Important limitation

Anonymous sign-in identifies an app session, not ownership of a phone number.
Without phone verification or a trusted server, a determined client can
impersonate a phone number, and authenticated clients can attempt direct reads
of individual spin records. The transaction prevents ordinary concurrent
duplicate spins, but this setup is not fraud-proof or suitable for highly
sensitive personal data.
