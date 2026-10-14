# Packaging IELTS Prep Studio as an Android app

This turns the `www/` folder into a real, installable Android app using
Capacitor. Everything in this guide runs on **your own computer** — it
needs Android Studio installed, which isn't something that can be done
remotely.

Do this *before* building out your real test/lesson content. The goal
right now is just to prove the whole pipeline works, using the built-in
seed test. Once it installs and runs on a real phone, you build content
separately and get it onto the app later via Export/Import — no repackaging
needed for that part.

---

## 0. Prerequisites (one-time setup)

- **Node.js** installed (so you have `npm`/`npx`)
- **Android Studio** installed, with the Android SDK (Android Studio installs
  this for you during its own setup wizard)
- A physical Android phone is strongly recommended for the first test run —
  emulators can behave differently for things like PDF rendering, which is
  exactly the thing we need to check for real.
  - On the phone: Settings → About phone → tap "Build number" 7 times to
    enable Developer Options, then enable "USB debugging" inside Developer
    Options, then connect it to your computer via USB.

---

## 1. Set your app's identity

Open `capacitor.config.json` and change `appId` to something unique to you —
it must look like a reverse domain name, lowercase, no spaces, e.g.
`com.yourname.ieltsprep`. This is a permanent identifier; changing it later
means Android treats it as a totally different app, not an update.

```json
{
  "appId": "com.yourname.ieltsprep",
  "appName": "IELTS Prep Studio",
  "webDir": "www"
}
```

You can also change `appName` to whatever you'd like students to see under
the app icon.

---

## 2. Install dependencies and add the Android platform

From inside this project folder:

```bash
npm install
npx cap add android
```

This creates an `android/` folder — the actual native Android project.
It's normal for this to take a minute and pull in a lot of files.

---

## 3. Sync your web files into the native project

Every time anything in `www/` changes, run:

```bash
npx cap sync android
```

Run it now for the first time.

---

## 4. Open it in Android Studio

```bash
npx cap open android
```

This launches Android Studio with the project loaded. Let it finish
indexing/Gradle syncing (the progress bar at the bottom) before doing
anything else — this can take a few minutes the first time.

---

## 5. Run it on your phone and test the PDF viewer specifically

With your phone connected (and showing up in the device dropdown at the top
of Android Studio), click the green **Run** button. This installs a debug
build directly onto your phone.

**Once it's running, specifically go test:**
- Open the seed test, click through Listening/Reading/Writing/Speaking
- Go to Lessons → open the one with no PDF attached yet (expected: empty state)
- This is also the moment to check the PDF iframe concern flagged in the code
  (`index.html` has a `TODO(pdf-viewer)` comment about this) — if you upload
  a PDF via `admin.html` (run locally in a desktop browser, not on the phone)
  and export/import it onto the phone to test, see if it renders inline or
  shows blank. If it's blank, that's the moment to come back and swap in a
  plain "open PDF" link or a bundled PDF renderer before going further.

---

## 6. Generate a signed release build

A debug build (step 5) is fine for your own testing but isn't meant for
distribution. For a real release:

1. In Android Studio: **Build → Generate Signed Bundle / APK**
2. Choose **APK**
3. Click **Create new...** under Key store path to make a new keystore:
   - Choose a file location *outside* this project folder
   - Set a keystore password and a key password — write these down somewhere
     safe (a password manager, etc.)
   - Fill in the certificate fields (name, organization — can be minimal,
     doesn't need to be a real company)
4. **Back up the resulting `.keystore` file and both passwords immediately.**
   If you lose this, you can never publish an "update" to this same app —
   only a brand new install for everyone, under a new identity.
5. Choose the **release** build variant, finish the wizard.
6. Android Studio will show a notification with a link to the output folder
   once it's done — the file will be named something like
   `app-release.apk`.

---

## 7. Put it on GitHub

```bash
git init
git add .
git commit -m "Initial Capacitor Android project"
git remote add origin <your-repo-url>
git push -u origin main
```

The `.gitignore` already included keeps build output, `node_modules`, and
(importantly) any keystore file out of the repo.

**Don't commit the APK itself as a regular file.** Instead:

1. On GitHub, go to your repo → **Releases** → **Draft a new release**
2. Give it a tag (e.g. `v1.0.0`) and a title
3. Drag the signed `app-release.apk` into the release's attachment area
4. Publish

Students get a direct download link from that release page.

---

## 8. What to tell students

Something like:

> Download the APK from [this link]. Android will warn that it's from an
> "unknown source" — that's expected for apps outside the Play Store, not
> a sign anything's wrong. Tap "Install anyway" when prompted, then open
> the app normally once it's installed.

---

## Optional: auto-update content from this same GitHub repo

The app can quietly check this repo for new content every time it's opened
with internet, and auto-import it — no "Import update" tap needed at all.
It fails silently with no internet, so this never gets in the way offline.

To turn it on:

1. In this repo, create a file at `content/latest.json`.
2. Whenever you want to push new tests/lessons: build them in `admin.html`
   as usual, click **Export all data**, and commit/overwrite that exported
   file at `content/latest.json` in the repo.
3. In `www/index.html`, find the `CONTENT_UPDATE_URL` constant near the top
   of the `<script>` block and replace the placeholder with your repo's raw
   file URL:
   ```
   https://raw.githubusercontent.com/<your-username>/<your-repo>/main/content/latest.json
   ```
4. Run `npx cap sync android` again so the updated `index.html` is copied
   into the native project, then rebuild (step 6 above) — this one change
   does need a new APK release, since it's a code change, not a content
   change. Every content update after that is just updating the JSON file
   in the repo — no new APK required.

One thing worth knowing: this file is public, same as everything else in
a public GitHub repo — fine for practice content, just not a place to put
anything you'd want to keep private.

---

## What doesn't need repeating later

- **New tests/lessons** → build them in `admin.html` as usual, Export, hand
  students the file, they use "Import update" inside the app. No new APK.
- **Code/bug fixes** → repeat steps 3–7 (sync → build signed APK → new
  GitHub release). Students reinstall.
