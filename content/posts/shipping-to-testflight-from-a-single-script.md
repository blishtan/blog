+++
title = "Shipping to TestFlight from a Single Script"
description = "One command — archive, bump the build, upload, and write its own release notes from the user stories that shipped."
type = ["posts","post"]
tags = [
    "ios",
    "testflight",
    "xcode",
    "automation",
    "ci",
]
date = "2026-06-10"
draft = true
categories = [
    "Development",
    "iOS",
]
[ author ]
  name = "Blishtan"
+++

Finishing a feature is the easy part. Getting it in front of testers means a trip through Xcode's Organizer — Archive, Distribute App, click through the wizard, remember to bump the build number, then hand-write release notes from memory. Every release.
I replaced the whole ritual with one command.

---

## The problem

The Organizer flow is fine the first time and tedious the fiftieth. Three things in particular kept biting me:

1. **The build-number trap.** TestFlight rejects any build whose `CFBundleVersion` already exists. Forget to bump it and you don't find out until *after* the slow archive-and-upload — never before.
2. **Release notes are a separate chore.** They live in your head at release time, get written in a hurry, and are the first thing to be skipped.
3. **No paper trail.** Which commit was build 34? You end up squinting at App Store Connect trying to reconstruct it.

None of this is hard. It's just manual, repetitive, and exactly the kind of thing a script never forgets to do.

---

## One command

```bash
./Tools/upload_testflight.sh
```

That's the happy path: **bump → archive → export → upload → release notes → bookkeeping**. A few flags cover the rest:

```text
--archive-only   Archive and export, skip the upload
--upload-only    Upload an IPA that's already exported
--clean          Delete build artifacts when done
--help           Usage
```

Credentials come from a sourced `.env` and the rest is hard-wired to the project. Let's walk through what each stage actually does.

---

## Bumping the build number

This is the very first thing the script does, because it's the thing I most often forgot to do by hand. The build number is `CURRENT_PROJECT_VERSION` in `Config.xcconfig` — one source of truth — so the bump is a three-line `sed`:

```bash
CURRENT_BUILD=$(grep -E '^CURRENT_PROJECT_VERSION' "$CONFIG_FILE" \
    | sed -E 's/^CURRENT_PROJECT_VERSION = ([0-9]+).*/\1/')
NEXT_BUILD=$((CURRENT_BUILD + 1))
sed -i '' -E "s#^CURRENT_PROJECT_VERSION = .*#CURRENT_PROJECT_VERSION = $NEXT_BUILD#" "$CONFIG_FILE"
```

Now every upload gets a unique build number for free. The `build already exists` rejection — the one that wastes a full archive cycle — simply can't happen.

---

## Archive and export

Two `xcodebuild` invocations, piped through [`xcbeautify`](https://github.com/cpisciotta/xcbeautify) so the log stays readable:

```bash
xcodebuild archive \
    -project "$PROJECT" \
    -scheme "Heimdall" \
    -destination "generic/platform=iOS" \
    -archivePath "$ARCHIVE_PATH" \
    -allowProvisioningUpdates | xcbeautify

xcodebuild -exportArchive \
    -archivePath "$ARCHIVE_PATH" \
    -exportPath "$EXPORT_PATH" \
    -exportOptionsPlist "$EXPORT_OPTIONS" \
    -allowProvisioningUpdates | xcbeautify
```

The export is steered entirely by `ExportOptions.plist`. The parts that matter:

```xml
<key>method</key>          <string>app-store</string>
<key>signingStyle</key>    <string>manual</string>
<key>provisioningProfiles</key>
<dict>
    <key>io.example.heimdall</key>
    <string>Heimdall App Store</string>
</dict>
<key>uploadSymbols</key>   <true/>
```

`method = app-store` produces an IPA destined for App Store Connect. `signingStyle = manual` with an explicit profile keeps signing deterministic — no surprise from whatever Xcode last decided automatic signing meant. And `uploadSymbols` ships the dSYMs so crash reports come back symbolicated instead of as raw addresses.

---

## The upload — no password, no 2FA

Uploading is one call to `altool`:

```bash
xcrun altool --upload-app \
    -f "$IPA_FILE" \
    -t ios \
    --apiKey "$API_KEY_ID" \
    --apiIssuer "$API_ISSUER_ID"
```

The detail worth stealing here is **App Store Connect API key** auth instead of Apple ID + app-specific password. You generate a key once in App Store Connect, drop the `.p8` in `~/.private_keys/`, and pass its key ID and issuer ID. No interactive password, no two-factor prompt, nothing that stalls waiting for a tap on your phone. It works unattended — which is the whole point of putting it in a script.

---

## Release notes that write themselves

This is the part I'm actually fond of. The build *already knows* what shipped — it's encoded in the user stories that reached "Ready for Acceptance" since the last release. So the script reads them instead of asking me to remember.

It scans the `Requirements/` directory for stories in that state:

```bash
candidate_stories=$(grep -rlE '\*\*Status:\*\* Ready for Acceptance' \
    "$requirements_dir" --include="US-*.md")
```

The catch is deduplication: a story that was already "Ready for Acceptance" at the *previous* TestFlight build shouldn't show up again. The previous build is marked by a git tag (more on that below), and because requirements live in a submodule, the script resolves the submodule's commit *at that tag* and checks the story's status back then:

```bash
previous_tag=$(git tag -l 'testflight/*' --sort=-version:refname | head -1)
submodule_commit=$(git ls-tree "$previous_tag" Requirements | awk '{print $3}')
old_status=$(git -C "$requirements_dir" show "$submodule_commit:$relative_path" \
    | grep -oE '\*\*Status:\*\* Ready for Acceptance')
# Already accepted at the last tag? Skip it.
```

What survives gets formatted into bullets — US id from the folder name, title from the story's first line:

```text
New in this build:
• US-009: Offline map tiles
• US-014: Heading-up compass mode
```

Then those notes go up as the TestFlight "What to Test" text. `altool` takes this through a small directory of localized text files rather than a flag:

```bash
xcrun altool --beta-app-store-text "$BETA_TEXT_DIR" --upload \
    --apple-id "$APPLE_ID" \
    --bundle-version "$BUILD_NUMBER" \
    --bundle-short-version-string "$APP_VERSION" \
    --platform ios \
    --apiKey "$API_KEY_ID" --apiIssuer "$API_ISSUER_ID"
```

The result: testers open TestFlight and see, in plain language, exactly what's new — and I never wrote a word of it.

---

## Leaving a trail

After a successful upload the script records what just happened, so future-me never has to reconstruct it. It reads the real version and build straight out of the archive with `PlistBuddy`, pairs them with the current branch and commit, and writes a line into `RELEASE_STATE.md`:

```bash
APP_VERSION=$(/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" "$INFO_PLIST")
BUILD_NUMBER=$(/usr/libexec/PlistBuddy -c "Print :CFBundleVersion" "$INFO_PLIST")

echo "- **$APP_VERSION ($BUILD_NUMBER)** — $UPLOAD_DATE — branch: \`$BRANCH\` — commit: \`$COMMIT\`" \
    >> "$RELEASE_STATE"

git tag "testflight/$APP_VERSION/$BUILD_NUMBER"
```

That `testflight/<version>/<build>` tag does double duty. It's a human-readable marker — `git show testflight/1.0/38` tells you exactly what went out — and it's the **anchor for the next run's release notes**. The bookkeeping and the automation feed each other: today's tag is the line tomorrow's diff is measured against.

`RELEASE_STATE.md` itself becomes the file you check instead of App Store Connect — latest build at the top, full upload history below — and it's plain enough that a small dashboard parses it straight into a status view.

---

## Takeaways

The script is a couple hundred lines of `bash`, but the ideas in it outlast the language:

1. **Automate the step you forget, first.** The build-number bump runs before anything else precisely because it's the mistake that costs a whole archive cycle. Put the easy-to-forget step where it's impossible to skip.
2. **Use the App Store Connect API key, not your Apple ID.** A `.p8` and two IDs give you uploads with no password and no 2FA — the difference between a script that runs and a script that waits for your phone.
3. **Your release notes are already written.** They're in your tracker, your user stories, your accepted work. Derive them; don't retype them.
4. **Make every release leave a trail.** A git tag per build is free, and it pays for itself the moment it becomes the anchor that tells the *next* release what changed.
