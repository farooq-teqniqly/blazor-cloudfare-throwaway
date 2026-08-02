# Deploying a Blazor WASM PWA to Cloudflare, Secured with Access

A hands-on walkthrough. The goal is a private, installable web app at a URL only you can open — built with a throwaway Blazor project so you learn the deployment mechanics before the real app exists.

**Time:** 45–60 minutes for a first run
**Cost:** $0
**Target:** .NET 10 (current LTS), Cloudflare Workers with static assets

---

## Correction to the earlier proposal

Two things changed since the proposal document:

**1. Workers, not Pages.** Cloudflare now recommends Workers with static assets for new projects. Workers reached feature parity for static hosting and custom domains, and new platform features ship to Workers first. Pages still works and isn't being shut down, but `wrangler pages` commands now nudge you toward `wrangler deploy`. For a new project there's no reason to start on the older path.

**2. No domain purchase needed.** Cloudflare Access can now protect a `workers.dev` URL directly with a one-click toggle in the Worker's settings. The proposal assumed you might need a custom domain for this. You don't.

Everything else in the proposal stands.

---

## Prerequisites

- **.NET 10 SDK** — verify with `dotnet --version` (expect `10.x`)
- **Node.js** — only for Wrangler, Cloudflare's CLI. Not used by the app itself
- **A Cloudflare account** — free tier
- A working email address you control (Access sends login codes to it)

---

## Part 1 — Create the Blazor PWA

### Step 1.1: Scaffold the project

```bash
dotnet new blazorwasm --pwa -o WorkoutSpike
cd WorkoutSpike
```

If that template name errors, run `dotnet new list blazor` to see what's registered on your machine. Template short-names have shifted across .NET versions.

The `--pwa` flag is doing real work: it adds `manifest.json`, a service worker, and icon assets. Retrofitting these later is annoying, so it's worth having from the start even on a spike.

### Step 1.2: Run it locally

```bash
dotnet run
```

Open the URL it prints. You should see the default counter app. Confirm this works before deploying — you want to isolate "app is broken" from "deployment is broken."

### Step 1.3: Add a visible marker

Edit `Pages/Home.razor` and add something obvious like a version string or timestamp. When you deploy, this tells you at a glance whether you're seeing fresh content or a cached copy. Service workers make this question come up constantly.

### Step 1.4: Add the SPA routing file

Create `wwwroot/_redirects` with exactly this line:

```
/* /index.html 200
```

Without it, navigating directly to `/counter` or refreshing on that route returns a 404, because the server looks for a physical file at that path and finds nothing. The rule tells Cloudflare to serve `index.html` for any unmatched path and let Blazor's router handle it client-side.

`_redirects` and `_headers` are supported natively by Workers static assets, same as they were on Pages. The file must live inside the published static directory — putting it in `wwwroot` accomplishes that, since `wwwroot` is what gets published.

### Step 1.5: Publish

```bash
dotnet publish -c Release
```

Output lands in `bin/Release/net10.0/publish/wwwroot`. Look inside — you'll see `_framework/` containing the .NET runtime and your compiled assemblies. That whole folder is what gets uploaded.

> **.NET 10 note:** `blazor.boot.json` no longer exists. Boot configuration is now embedded directly in `dotnet.js`. If you're following an older tutorial that tells you to configure MIME types or caching for `blazor.boot.json`, that step is obsolete.

---

## Part 2 — Deploy to Cloudflare

### Step 2.1: Install Wrangler

```bash
npm install -g wrangler
wrangler login
```

This opens a browser for OAuth. If the account list looks wrong afterward, `wrangler whoami` confirms which account you're pointed at.

> **First-time accounts:** if deploy fails with *"You need a workers.dev subdomain in order to proceed,"* log into the Cloudflare dashboard and open the Workers & Pages section once. Visiting that page provisions the subdomain. It's a known rough edge in the onboarding flow.

### Step 2.2: Create the Wrangler config

Create `wrangler.jsonc` in the **project root** (next to the `.csproj`, not inside `wwwroot`):

```jsonc
{
  "name": "workout-spike",
  "compatibility_date": "2026-08-01",
  "assets": {
    "directory": "./bin/Release/net10.0/publish/wwwroot",
    "not_found_handling": "single-page-application"
  }
}
```

Notes on each field:

- **`name`** becomes your subdomain: `workout-spike.<your-subdomain>.workers.dev`
- **`compatibility_date`** pins runtime behavior. Set it to today's date and leave it alone. Bumping it later opts into behavior changes, which is a deliberate act, not a routine update
- **`assets.directory`** replaces what Pages called the "build output directory"
- **`not_found_handling`** is the Workers-native equivalent of the `_redirects` rule. Having both is belt-and-suspenders — harmless, and it means the app still routes correctly if you ever move it to a different static host

There's no `main` field. That's intentional: without one, this is a pure static asset deployment with no Worker script. You'd add `main` only if you later wanted server-side logic, which this app doesn't need.

### Step 2.3: Deploy

```bash
wrangler deploy
```

Wrangler uploads the folder and prints your URL. Open it. You should see your app, marker string and all.

At this point the site is **public**. Anyone with the URL can load it. That's the next part.

### Step 2.4: Make this repeatable

Add to your `.csproj` so publish-then-deploy is one command:

```xml
<Target Name="Deploy" AfterTargets="Publish"
        Condition="'$(Configuration)' == 'Release'">
  <Exec Command="wrangler deploy" />
</Target>
```

Or just keep a shell script. Either beats retyping two commands.

---

## Part 3 — Lock it down with Access

### Step 3.1: Enable Access on the Worker

1. Cloudflare dashboard → **Workers & Pages**
2. Select your Worker
3. **Settings** → **Domains & Routes**
4. Next to the `workers.dev` entry, click **Enable Cloudflare Access**

This creates an Access application in front of the URL, defaulting to your account email. That's often all you need.

### Step 3.2: Review the policy

Click **Manage Cloudflare Access** to open the Zero Trust dashboard, where you can confirm or edit:

- **Action:** Allow
- **Include:** Emails → your address

Add other addresses here if you ever want to share the app.

> Cloudflare changed this in late 2025 so the one-click button creates *reusable* Access policies rather than a duplicate policy per resource. If you protect several Workers, they can share one policy — edit it once, applies everywhere.

### Step 3.3: Set a long session duration

Still in the Zero Trust dashboard, find the application's session duration and set it to something generous — **1 month**.

This matters more than it sounds. The default is short, and re-authenticating by email code while standing in a gym with bad signal is exactly the kind of friction that makes you stop using an app. Long sessions are the right tradeoff for a single-user personal tool.

### Step 3.4: Verify it actually works

Open your URL in a **private/incognito window**. You should get Cloudflare's login screen, not your app. Enter your email, receive a one-time code, enter it, and land on the app.

Then verify the gate covers everything, not just the HTML. In the incognito window before logging in, try loading a framework file directly:

```
https://your-app.<subdomain>.workers.dev/_framework/dotnet.js
```

You should be redirected to login rather than getting the file. Access authenticates at the edge before any asset is served — that's what makes this genuinely private rather than merely obscure. Worth confirming with your own eyes once.

---

## Part 4 — Install as a PWA

### Step 4.1: Install it

**Android/Chrome:** menu → "Add to Home screen" or an install prompt
**iOS/Safari:** Share → "Add to Home Screen"
**Desktop Chrome/Edge:** install icon in the address bar

Launch from the home screen. It opens without browser chrome, looking like a native app.

### Step 4.2: Confirm the Access cookie carried over

The installed PWA shares the browser's cookie store, so you shouldn't be asked to log in again. If you are, the install happened in a different browser context than where you authenticated — log in once inside the installed app and it'll stick.

### Step 4.3: Confirm offline behavior

Enable airplane mode and launch the app. It should load from the service worker cache.

This is the payoff for choosing WASM over Server: a Blazor Server app is a blank screen the moment connectivity drops. Worth experiencing directly, since it's the single strongest argument for the architecture.

---

## Part 5 — The service worker will confuse you

This is the part that wastes people's afternoons, so it's worth knowing before it bites.

**The symptom:** you deploy a change, reload, and see the old version. You reload again. Still old. You start doubting the deploy worked.

**Why:** the service worker serves from cache first and updates in the background. The new version is downloaded but doesn't activate until all tabs of the app are closed.

**What to do:**

- **DevTools → Application → Service Workers → check "Update on reload"** during development. Fixes 90% of it
- Or **"Unregister"** then hard-refresh for a clean slate
- On mobile, fully close the app (not just background it) and reopen
- Your visible marker from Step 1.3 makes this diagnosable in one glance

There are two different service worker files in the project: `service-worker.js` (development, deliberately does nothing) and `service-worker.published.js` (the real caching one, used only in published builds). If you're testing caching behavior, you must test a published build — `dotnet run` won't exercise it.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| 404 on refresh at any route | SPA fallback not configured | Check `_redirects` is in the published output and `not_found_handling` is set |
| "You need a workers.dev subdomain" | Account not fully initialized | Open Workers & Pages in the dashboard once |
| Deploy succeeds, site is stale | Service worker cache | Update on reload, or unregister |
| Deploy uploads almost nothing | `assets.directory` wrong | Must point at `publish/wwwroot`, not `publish` or `wwwroot` |
| Site loads without login prompt | Testing in an already-authenticated browser | Use incognito |
| Very slow first load | Full .NET runtime download | Expected. See optimization below |

---

## Optional: trim the payload

The default publish is several megabytes. If first-load size bothers you:

```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
  <InvariantGlobalization>true</InvariantGlobalization>
</PropertyGroup>
```

Trimming strips unreferenced IL. `InvariantGlobalization` drops ICU data, which is a large chunk — safe if you never format dates or numbers for non-English locales, which for a personal workout app you won't.

Do this *after* everything works. Trimming can break reflection-based code in ways that only surface at runtime, and you don't want to be debugging that simultaneously with deployment issues.

---

## What you should have at the end

- A Blazor WASM PWA live on a `workers.dev` URL
- That URL gated by Access — email verification required, covering every asset
- The app installed on your phone, working offline
- A two-command deploy loop: `dotnet publish -c Release && wrangler deploy`

Once this is working with the throwaway app, the real one is the same pipeline with different content. That's the point of doing it first — when the workout app misbehaves, you'll know it's the app, not the hosting.

---

## Next

Deploy the spike, confirm Access works, then send the workout images. Building against a known-good deployment path removes an entire category of confusion from the actual project.
