# capacitor-cli-nx-repro

Minimal reproduction for a `@capacitor/cli` bug where `capacitor:sync:before`/`capacitor:sync:after`
hooks are resolved against the workspace root `package.json` instead of the directory they were
actually meant for (the Capacitor app itself, or a plugin), whenever an `nx.json` file exists
anywhere above that directory — regardless of whether Nx (the tool) is actually installed or used.

Filed as a follow-up to [ionic-team/capacitor#7606](https://github.com/ionic-team/capacitor/issues/7606).

## Layout

```
.
├── nx.json                      # just needs to exist — triggers isNXMonorepo()
├── package.json                 # workspace root — defines its OWN capacitor:sync:before (should never run)
├── apps/
│   └── mobile/                  # the Capacitor app
│       ├── package.json         # defines its own capacitor:sync:before (expected to run, doesn't)
│       ├── capacitor.config.json
│       └── www/index.html
└── packages/
    └── example-plugin/          # a local Capacitor plugin (has a "capacitor" manifest field)
        └── package.json         # defines its own capacitor:sync:before (expected to run, doesn't)
```

Each of the three `package.json` files defines a `capacitor:sync:before` script that just prints
where it ran from, so it's obvious at a glance which one actually executed.

## Reproduction steps

```sh
npm install
npx nx run mobile:sync
```

(`web` is used so the repro needs no Android/iOS SDKs — see "Why `sync web`" below.)

## Expected output

Two distinct lines, one from the app's own hook and one from the plugin's own hook:

```
✅ APP hook ran (apps/mobile/package.json) — cwd=.../apps/mobile
✅ PLUGIN hook ran (packages/example-plugin/package.json) — cwd=.../packages/example-plugin
✔ copy web in ...
✔ update web in ...
[info] Sync finished in ...
```

## Actual output

```
XX ROOT hook ran (workspace root package.json) -- cwd=/Users/louist/capacitor-cli-nx-repro/apps/mobile
XX ROOT hook ran (workspace root package.json) -- cwd=/Users/louist/capacitor-cli-nx-repro/packages/example-plugin
✔ copy web in 1.42ms
✔ update web in 870.08μs
[info] Sync finished in 0.195s
```

Neither `apps/mobile/package.json`'s hook nor `packages/example-plugin/package.json`'s hook ever
runs. Instead, the workspace root's `capacitor:sync:before` script runs **twice** — once for the
app, once for the plugin — each time with `cwd` set to the app/plugin directory rather than the
root. So the CLI reads the *command string* from the wrong `package.json` (the root's) but still
executes it with the *original, correct* `cwd` (the app's or plugin's own directory) — a mismatch
that, in a real project (see below), crashes outright the moment the root's hook script references
anything relative to its own location, since it's actually being run from a different directory.

## Root cause

In `@capacitor/cli`'s `dist/common.js`:

```js
async function runPlatformHook(config, platformName, platformDir, hook) {
    let pkg;
    if (isNXMonorepo(platformDir)) {
        pkg = await readJSON(join(findNXMonorepoRoot(platformDir), 'package.json'));
    } else {
        pkg = await readJSON(join(platformDir, 'package.json'));
    }
    const cmd = pkg.scripts?.[hook];
    if (!cmd) return;
    return new Promise((resolve, reject) => {
        const p = spawn(cmd, { cwd: platformDir, shell: true, ... });
        ...
    });
}
```

`isNXMonorepo`/`findNXMonorepoRoot` (`dist/util/monorepotools.js`) just walk up from `platformDir`
looking for the nearest `nx.json`. `platformDir` is already correctly scoped by the caller — it's
`config.app.rootDir` for the app's own hook, and each plugin's own install directory
(`p.rootPath`) for plugin hooks (`runHooks` in the same file loops over `getPlugins()` and calls
`runPlatformHook` once per plugin). But as soon as *any* `nx.json` exists above that directory,
the command is read from the workspace root's `package.json` instead — while still executing with
`cwd: platformDir`. The redirect isn't a fallback (e.g. "use root only if the app/plugin itself has
no matching script") — it unconditionally overrides an already-correct, specific directory with a
shared one that has no natural connection to that particular app or plugin.

This means in any Nx-style monorepo (an app in a subdirectory, `nx.json` at the root):

- A plugin that ships its own `capacitor:sync:before`/`capacitor:sync:after` hook has that hook
  silently ignored.
- An app that defines its own such hook (in the app's own `package.json`, right next to its other
  Capacitor scripts) has that hook silently ignored too — the CLI looks at the monorepo root
  instead, a file that in a real workspace is shared across many unrelated projects and has no
  reason to define an app-specific Capacitor hook.
- If the workspace root *happens* to define a script under the same name (for some unrelated
  reason), that unrelated script runs instead, once per plugin, each time with the wrong `cwd`.

## Why `cap sync web`

The bug doesn't depend on any native platform being configured — `runHooks` is called with
`config.app.rootDir` and each plugin's `rootPath` regardless of platform, so targeting `web`
(which needs no Android/iOS SDKs, just a `webDir`) is enough to reproduce it with nothing but
`npm install`.

## Real-world case that surfaced this

We hit this with [`@capgo/capacitor-social-login`](https://www.npmjs.com/package/@capgo/capacitor-social-login),
which ships a real `capacitor:sync:before` hook (`node scripts/configure-dependencies.js`) that
conditionally enables/disables native provider SDKs (Google/Facebook/Apple) in its podspec /
`gradle.properties`, based on `capacitor.config.ts`. In our Nx + pnpm workspace (app in
`apps/mobile`, `nx.json` several directories up at the workspace root) this hook never fires
during `cap sync`, so the plugin always ships every provider SDK regardless of config.
