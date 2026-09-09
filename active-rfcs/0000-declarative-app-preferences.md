- Start Date: 2026-09-08
- Target Major Version: 9.x
- Reference Issues: https://github.com/NativeScript/NativeScript/issues/4248
- Implementation PR: (leave this empty)

# Summary

Adopt a declarative preferences package into the NativeScript org as `@nativescript/preferences`.

An app describes its settings once, in `app/app.preferences.ts`, with `definePreferences({ items })`. The keys and value types are inferred from that literal, and the same file is the typed runtime instance. A `before-prepare` hook evaluates the file under Node, the way the CLI reads `nativescript.config.ts`, and generates the iOS `Settings.bundle` and the Android `PreferenceScreen` XML from it. No plist or XML is written by hand, no native code, and no generated TypeScript.

A working implementation ships today as [`nativescript-preferences`](https://github.com/sitefinitysteve/nativescript-preferences) (Apache-2.0): 2.1 is the TypeScript definition described here, 2.0 is the JSON form this RFC first proposed, and both are supported. I am offering to transfer it to the org and keep maintaining it.

# Basic example

`app/app.preferences.ts`:

```ts
import { definePreferences } from '@nativescript/preferences';

export default definePreferences({
  items: [
    {
      type: 'group',
      title: 'General',
      items: [
        { key: 'enabled', type: 'toggle', title: 'Enabled', default: true },
        { key: 'theme', type: 'list', title: 'Theme', default: 'system', options: ['system', 'light', 'dark'] },
        { key: 'volume', type: 'slider', title: 'Volume', default: 50, min: 0, max: 100 },
      ],
    },
  ],
});
```

That produces a page in the iOS Settings app, an AndroidX preference screen, and, with no `as const` and no generated file:

```ts
import settings from './app.preferences';

settings.get('theme');            // 'system' | 'light' | 'dark', never undefined
settings.set('volume', 80);       // NSUserDefaults / SharedPreferences
settings.set('enabled', 'yes');   // compile error
settings.definition;              // the literal above
settings.onChange('theme', applyTheme);
await settings.openSettings();
```

The instance is an `Observable`, so a hand-built screen binds to the same values:

```xml
<Switch checked="{{ enabled }}" />
<Slider value="{{ volume }}" minValue="0" maxValue="100" />
```

Package paths and the XML namespace throughout this RFC use the proposed `@nativescript/preferences` name. The shipping package is `nativescript-preferences`; the rename is part of this proposal.

Both screens below come from the demo app's single [`app.preferences.ts`](https://github.com/sitefinitysteve/nativescript-preferences/blob/master/demo/app/app.preferences.ts):

| iOS, in the Settings app | Android, an AndroidX `PreferenceScreen` |
| --- | --- |
| <img src="https://raw.githubusercontent.com/sitefinitysteve/nativescript-preferences/master/images/ios-settings-root.png" width="250" alt="iOS Settings screen for the demo app" /> | <img src="https://raw.githubusercontent.com/sitefinitysteve/nativescript-preferences/master/images/android-settings-root.png" width="250" alt="Android preference screen for the demo app" /> |

`Theme` is one `list` item in the JSON. It renders as a radio group on iOS and as a `DropDownPreference` with an icon on Android, through the per-platform overrides described below. `Compact rows` shows its summary inline on Android; iOS drops item summaries and renders only a group's text, as a footer. The `Allow Waveform to Access` section at the top of the iOS screen is added by iOS itself, which is why the runtime filters system keys out of `keys()` and `getAll()`.

More screens, including nesting two levels deep and the Android multi-select dialog, are in the [README](https://github.com/sitefinitysteve/nativescript-preferences#one-json-file-both-platforms).

# Motivation

**Core's storage API is thin.** `ApplicationSettings` gives you `getString` / `getNumber` / `getBoolean` over an untyped key space. It has no declared keys, no declared defaults, no `string[]` support and no change notification. Every app that outgrows it writes the same wrapper: a constants file, a defaults object, a typed accessor, a hand-rolled event emitter. That wrapper is app-specific and gets rewritten in every project.

**The native settings UI is manual.** iOS wants a `Settings.bundle` of plists, historically assembled in Xcode and copied into `App_Resources`. Android has no OS-hosted settings, so you write `preferences.xml`, the matching string arrays, and a `PreferenceFragmentCompat` host to render them.

**Settings are where a cross-platform app stops behaving like a native one.** An iOS app that ships a `Settings.bundle` appears in the Settings app next to the system's own entries, which is where a lot of users look first. An Android app rendering an AndroidX `PreferenceScreen` gets Material dialogs, spacing, dark mode and accessibility behaviour without implementing any of it. Both are cheap to describe and expensive to hand-build, so the usual outcome is an in-app settings page that approximates each platform and matches neither. NativeScript's argument has always been that its apps are genuinely native, and this is one of the more visible places where that is currently left entirely to the app author.

An app that does both ends up with the same list of settings in three places: the iOS plists, the Android XML, and the defaults in its own code. Nothing checks that the three agree, so they drift. The usual symptoms are a toggle in iOS Settings that does nothing, or a key that reads `undefined` because its default was only ever declared on one side.

**Every app has settings, and there is no sanctioned way to build them.** A settings screen is close to universal in shipped apps. Core covers the storage half with `ApplicationSettings` and stops there, so each team improvises the rest, and most stop short of the native screens for the reasons above. Being in core changes what people reach for. A documented first-party API gets used where a third-party plugin has to be found, evaluated and justified first. Adopting this is about making the correct approach the default one, since the capability itself already exists as a plugin.

[#4248](https://github.com/NativeScript/NativeScript/issues/4248) asked for the iOS half of this in May 2017: configure the settings bundle on prepare, read the values at runtime. It is still open with no comments.

Constraints any solution has to meet, independent of the design below:

- One description of an app's settings, rather than one per platform.
- Types and defaults derived from that description, so a read cannot be `undefined` and a typo cannot compile.
- Defaults registered natively and in code from the same source, so the OS screen and the app agree before the user has touched anything.
- A value changed in the OS settings UI, in app code, or through a binding is visible everywhere else.
- A way to host a preference screen in-app on Android, since the OS provides none.
- No native code, no manual `Info.plist` or manifest edits.
- An override path for anything the generator gets wrong.

# Detailed design

The package has three parts.

## 1. `Preferences<Schema>`, the runtime

A typed wrapper over `NSUserDefaults` (iOS) and `SharedPreferences` (Android). `definePreferences` constructs it from a definition; it can also be constructed directly:

```ts
import { Preferences } from '@nativescript/preferences';

interface Settings { enabled: boolean; theme: 'system' | 'light' | 'dark'; volume: number }

export const settings = new Preferences<Settings>({
  defaults: { enabled: true, theme: 'system', volume: 50 },
  integers: ['volume'],
});
```

| Member | Description |
| --- | --- |
| `static shared` | Untyped instance of the store behind the OS settings UI. |
| `static applicationSettings` | Untyped instance of the store `ApplicationSettings` uses. |
| `get(key, fallback?)` | Stored value, else `fallback`, else the declared default. |
| `getString / getNumber / getBoolean / getStringArray` | Coerced reads, for untyped use. |
| `set(key, value)` | Writes. `null` / `undefined` removes the key. |
| `remove(key)`, `clear()` | Defaults stay in effect afterwards. |
| `has(key)`, `keys()`, `getAll()` | `has` ignores defaults. |
| `onChange(cb)`, `onChange(key, cb)` | Returns an unsubscribe function. Also `on('change')`. |
| `refresh()` | Re-read the native store, raise events for differences. |
| `registerDefaults()` | iOS: register `Settings.bundle` defaults. Android: persist `preferences.xml` defaults. |
| `openSettings(options?)` | Open the OS settings UI. |
| `ios` / `android` | The underlying `NSUserDefaults` / `SharedPreferences`. |
| `dispose()` | Stop observing native changes. |

A typed schema requires a default for every key, which is what allows `get()` to be typed as always returning a value. `definePreferences` derives those defaults from the definition (an omitted `text` default is `''`, `toggle` `false`, `multilist` `[]`, `slider` its `min`; a `list` default is required and must be one of its options), lists every `slider` as an integer key, and keeps the definition reachable as `settings.definition`.

A read resolves to the stored value, else a registered default. On iOS the shared store registers the in-code defaults and then the `Settings.bundle` defaults, so where the two disagree the bundle value wins, which is also what the Settings app displays. On Android there is no registration domain; `preferences.xml` defaults apply only when the app calls `registerDefaults()`, which fills in keys that have no stored value yet. With the generator every default comes from the same JSON, so this only matters for a hand-written bundle.

`suiteName` opens an iOS App Group suite or a named Android `SharedPreferences` file, for extensions and widgets.

The instance extends `Observable`, which is what makes two-way XML bindings work. A key colliding with a class member (`set`, `keys`) stays readable through `get()` but is not bindable; the constructor traces a warning.

`string`, `boolean`, `number` and `string[]` map to `NSString`/`String`, `Bool`/`boolean`, `Integer`-or-`Double`/`int`-`long`-`float` (Android preserves a key's existing Java type), and `NSArray`/`Set<String>`.

**iOS system keys.** iOS writes its own entries into the same defaults domain (`AppleLanguages`, `NSHyphenatesAsLastResort`, and others). Only declared keys come out of the registration domain, and system-shaped keys are filtered from the persistent domain, so `keys()`, `getAll()` and the global change event see app preferences only.

## 2. The definition, the generator and the build hook

`definePreferences<const D extends PreferencesDefinition>(definition: D & ValidatePreferencesDefinition<D>): Preferences<InferPreferences<D>>`. The `const` type parameter keeps the literal types; `InferPreferences` flattens nested groups and screens and maps each stored key to its value type; `ValidatePreferencesDefinition` narrows a `list` default to that item's own options and turns an unknown property such as `titel` into an error. Requires TypeScript 5.3 (verified through 7.0).

The hook reads the same file under Node with `ts.transpileModule`, exactly as the CLI reads `nativescript.config.ts`, and runs the output with a `require` that stubs `@nativescript/preferences` to return the definition untouched. Relative `.ts` / `.js` helpers go through the same loader; any other import is rejected with a message, since the file also runs in the app. `typescript` is resolved from the project when it has the compiler API (5.3 to 6.x), else from the CLI's own copy, since TypeScript 7 no longer ships that API. The plain object then goes through the same validation as JSON did, and every error names the item.

A `preferences.json` with a published JSON Schema is still accepted, and for it the generator also emits the typed module 2.x users have. The item types:

| `type` | Stores | iOS | Android |
| --- | --- | --- | --- |
| `group` | nothing | `PSGroupSpecifier` | `PreferenceCategory` |
| `screen` | nothing | `PSChildPaneSpecifier` | nested `PreferenceScreen` |
| `text` | `string` | `PSTextFieldSpecifier` | `EditTextPreference` |
| `toggle` | `boolean` | `PSToggleSwitchSpecifier` | `SwitchPreferenceCompat` |
| `list` | one option | `PSMultiValueSpecifier` | `ListPreference` |
| `multilist` | `string[]` | needs an override | `MultiSelectListPreference` |
| `slider` | integer | `PSSliderSpecifier` | `SeekBarPreference` |
| `label` | nothing | `PSTitleValueSpecifier` | `Preference` |

Groups and screens nest, and every item except `group` needs a unique `key`. `default` is optional; a type that omits it gets a natural empty value (`''`, `false`, `[]`, or `min`), so every key in the generated module has one. Defaults are validated against the item's type and options, and errors name the failing item.

Outputs:

| Output | Path |
| --- | --- |
| iOS bundle, one plist per screen | `App_Resources/iOS/Settings.bundle/` |
| Android screen and string arrays | `App_Resources/Android/src/main/res/xml/preferences.xml`, `values/preferences_arrays.xml` |

The definition file is the typed module, so nothing else is generated for it. Generated files name the file they came from in their header.

The hook is a standard `before-prepare` entry in `nativescript.config.ts`, so every `ns run`, `ns build` and `ns prepare` regenerates. It finds `app.preferences.ts` in the app folder (`appPath` from the config, else `src`, else `app`) or the project root, then falls back to `preferences.json`; two present at once is an error, so two sources can never disagree. `npx ns-preferences generate` runs it directly, `check` exits 1 on stale generated output for CI, and `init` scaffolds the definition and registers the hook.

**Per-platform overrides.** Any item takes an `ios` or `android` object. `false` hides the item on that platform. `widget` swaps the control. Anything else is written verbatim as a plist key or an XML attribute, and `null` removes one:

```ts
{
  key: 'theme',
  type: 'list',
  default: 'system',
  options: ['system', 'light', 'dark'],
  ios: { widget: 'PSRadioGroupSpecifier' },
  android: { widget: 'DropDownPreference', 'android:icon': '@drawable/ic_theme' },
}
```

The generator accepts any non-empty `widget` string without checking it against a list, so fully qualified Android classes work. Picking a control that stores a different shape of data than the item type is the author's responsibility.

Two iOS layout rules are applied automatically. A `PSRadioGroupSpecifier` is emitted last within its group, because iOS renders it as its own section and would otherwise reorder the surrounding rows. A `screen` sharing a group with other rows gets its own card instead of inheriting their footer text. `generate` prints a note when it moves something.

**Ownership and escape hatches.** Generated files carry a "Do not edit" header. Strip the header and that file is never touched again; `generate --force` reclaims it. `output: { android: false }` turns off an output entirely. `NS_PREFERENCES_SKIP=1` skips one build, and removing the hook entry stops it permanently. Pre-existing hand-written `Root.plist` or `preferences.xml` files are left alone on first run. Only changed files are written.

## 3. `PreferencesView`, the Android host

Android has no OS-hosted settings, so `openSettings()` navigates the topmost `Frame` to a page rendering `preferences.xml` through `PreferenceFragmentCompat`. Nested screens push as pages and back works.

The same screen embeds in a tab, a modal, or part of a larger page:

```xml
<prefs:PreferencesView resource="preferences" xmlns:prefs="@nativescript/preferences" />
```

It renders nothing on iOS, which `PreferencesView.isSupported` reports. Handling `navigateToScreen` and setting `args.handled = true` lets the app present nested screens itself. The namespace registers with the XML builder on import, so webpack and Vite both work.

## Relationship to `ApplicationSettings`

`ApplicationSettings` stays as it is. `Preferences` covers the same reads and writes and adds typed keys, declared defaults, `string[]`, change events and the native UI.

On iOS both read the same `NSUserDefaults`, so an existing app's values are already there. On Android `ApplicationSettings` uses its own `prefs.db` file, so values written through it are not visible to a default `Preferences` instance. `Preferences.applicationSettings` reads that store for migration, or an app can move the keys across once at startup.

# Drawbacks

**It overlaps `ApplicationSettings`.** Two supported ways to store a setting costs something in docs and in support. The split I would document is `ApplicationSettings` for a handful of ad-hoc keys and `Preferences` when settings are a declared part of the app.

**A build hook writes into `App_Resources`.** Generated files land in source control, and a hook that rewrites tracked files is surprising the first time it happens. The generated-file headers, the changed-files-only writes, the `check` command and the skip flag all exist to limit this, and it is still the part of the design most likely to annoy people.

**iOS caches `Settings.bundle` per install.** After editing `preferences.json`, a plain `ns run ios` can leave the previous Settings screen in place until the app is deleted and reinstalled. This is documented, and it will still get reported as a bug.

**Android gains a dependency.** `androidx.preference` is added via `include.gradle`.

**The schema is a common denominator.** iOS has no multi-select control, and Android's `SeekBarPreference` differs from an iOS slider in look and behaviour. Per-platform overrides cover these cases, at the cost of putting platform detail back into the JSON.

**The definition runs in two runtimes.** The hook evaluates `app.preferences.ts` under Node and the app evaluates it again on device. That is why it has to stay self-contained and deterministic: no app imports, no `@nativescript/*`, no environment checks. The loader enforces the imports and the docs state the rest, but a definition that computes its items differently in the two places would generate one screen and run another.

**It raises the TypeScript floor.** The typings use `const` type parameters, so consumers need TypeScript 5.3 or newer, `preferences.json` users included. Reading the file at build time also needs the compiler API that TypeScript 7 dropped, which the hook works around by using the CLI's own `typescript`.

**It is another concept to teach.** A new file and a new API next to one that already exists.

**It does not have to be in the org.** Everything here works as a community plugin today, and the team could reasonably decline on that basis. Adopting it would change which path developers default to. It would not add a capability that is missing today. Whether that trade is worth an org-maintained package is a judgement the core team is better placed to make than I am.

# Alternatives

**Leave it as a community plugin.** The status quo, and a reasonable outcome. It costs the org nothing, and apps keep hand-writing plists, keep writing their own typed wrapper, keep shipping in-app settings pages in place of the platform's own, and #4248 stays open.

**Types and defaults only, in core.** Ship a typed, defaults-backed store in `@nativescript/core` and skip the generator. Cheaper, and it closes the API gap, but it leaves the duplication between the two platform files and the app's own defaults untouched.

**Generator only, in the CLI.** `ns preferences generate`, in the shape of the landed `ns fonts` RFC, with no runtime piece. Most of the generator's value is in the TypeScript module it emits alongside the native files; without that, the JSON is one more copy of the same list.

**Describe settings in JSON.** The first version of this RFC proposed `preferences.json` with a JSON Schema for completion, generating a typed module alongside the native files. It avoids evaluating user code at build time. It was dropped as the primary form on review: the CLI already evaluates `nativescript.config.ts` the same way, and JSON leaves the types one generation step away from the source, so a `list` default outside its options is a build error rather than a compile error. JSON is still accepted for existing projects.

**Document the manual path properly.** Write a guide on hand-writing both platforms. This is the cheapest option and it is what exists today.

# Adoption strategy

This is additive. Nothing in core changes, no existing API is deprecated, and nothing breaks.

**Existing apps.** `ns plugin add @nativescript/preferences`, then `npx ns-preferences init`. Requires TypeScript 5.3 or newer. On iOS, keys already in `NSUserDefaults` are picked up as they are. On Android, values written through `ApplicationSettings` live in a different file and need either `Preferences.applicationSettings` or a one-time migration. That needs to be prominent in the docs, because it is the one case where an app can silently read a stale value.

Apps with a hand-written `Settings.bundle` or `preferences.xml` keep them, since the first run leaves existing files alone. Migration can be done one screen at a time.

**Naming.** `nativescript-preferences` becomes `@nativescript/preferences`. The old name gets a final version that re-exports the new package, marked with `npm deprecate`, so nothing installed today breaks.

**Docs.** A settings guide covering the JSON format, the API, and the Android migration note above. The package ships `llms.txt` and a `SKILL.md`, so assistants working in a project have a reference without scraping docs.

**Ecosystem.** No effect on other plugins. The build hook uses the standard `before-prepare` mechanism.

**My commitment.** I will transfer the repo to the org and keep maintaining it. I would welcome co-maintainers, particularly on the Android side.

# Unresolved questions

- **Where should it live?** A standalone repo under the org, or the plugins monorepo.
- **Does the runtime belong in core?** `Preferences` could live in `@nativescript/core` as the successor to `ApplicationSettings`, leaving the generator and `PreferencesView` in the plugin. That splits one API across two packages, but keeps `androidx.preference` and the generator out of core. My preference is to keep all of it in the plugin, without a strong argument either way.
- **CLI surface.** Whether this should be `ns preferences generate` instead of `npx ns-preferences generate`, which would need a CLI change.
- **A shared loader for TypeScript config files.** The CLI reads `nativescript.config.ts` with `ts.transpileModule`, and this hook does the same for `app.preferences.ts`. TypeScript 7 removed that API. One CLI-owned loader, exposed to hooks, would keep both working with one fix instead of two.
- **Android entry point.** `openSettings()` navigating a `Frame` is convenient but makes assumptions about navigation. `PreferencesView` is the more composable primitive. Which one the docs should lead with is open.
- **Multiple suites.** One config file currently describes one store. Apps with a widget or an extension may want several suites described together.
- **iOS bundle caching.** Whether the CLI could detect a changed `Settings.bundle` and prompt for a reinstall, instead of leaving it to the README.
