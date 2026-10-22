- Start Date: 2026-10-22
- Target Major Version: 9.x
- Reference Issues: https://github.com/NativeScript/NativeScript/issues/254
- Implementation PRs:
  - Runtime: https://github.com/NativeScript/windows (published as `@nativescript/windows`), https://github.com/NativeScript/windows/pull/32
  - Core: https://github.com/NativeScript/NativeScript/pull/11272, https://github.com/NativeScript/NativeScript/pull/11468, https://github.com/NativeScript/NativeScript/pull/11515, https://github.com/NativeScript/NativeScript/pull/11527
  - CLI: https://github.com/NativeScript/nativescript-cli/pull/6065, https://github.com/NativeScript/nativescript-cli/pull/6167, https://github.com/NativeScript/nativescript-cli/pull/6173, https://github.com/NativeScript/nativescript-cli/pull/6174
  - Templates: https://github.com/NativeScript/nativescript-app-templates/pull/244

# Summary

Add Windows as a first-class NativeScript platform, next to Android, iOS and visionOS.

A NativeScript Windows app is a WinUI 3 desktop app built on the Windows App SDK. JavaScript gets
direct access to every Windows Runtime (WinRT) API, every WinUI API and any .NET type, the same way
it reaches Java on Android and Objective-C on iOS. `@nativescript/core` renders its UI with native
WinUI controls, so an existing NativeScript app runs on Windows with no code changes.

The work spans four projects:

- **Runtime** (`@nativescript/windows`): a new runtime, in https://github.com/NativeScript/windows.
- **CLI**: a `windows` platform for `ns platform add`, `prepare`, `build`, `run`, `debug`,
  `typings` and `doctor`, with MSIX packaging.
- **Core**: a `*.windows.ts` implementation of every core module, plus `isWindows`, `__WINDOWS__`
  and a `windows` getter wherever `ios` and `android` getters exist.
- **Templates**: an `App_Resources/Windows` folder in every app template.

# Basic example

Adding the platform and running an app works the same way it does on Android and iOS:

```bash
ns platform add windows    # installs @nativescript/windows
ns run windows             # builds, registers and launches on the local machine
ns debug windows           # same, with the Chrome DevTools inspector attached
ns build windows --release --certificate ./signing.pfx --certificate-password ****
```

Platform code works as it does on the other platforms. Platform-specific files use the `.windows`
suffix, and `isWindows` / `__WINDOWS__` guard code that only runs on Windows:

```ts
// app/utils/clipboard.windows.ts
export function copyToClipboard(text: string) {
	const { DataPackage, Clipboard } = Windows.ApplicationModel.DataTransfer;
	const data = new DataPackage();
	data.SetText(text);
	Clipboard.SetContent(data);
}
```

```ts
import { Button, isWindows } from '@nativescript/core';

export function onLoaded(args) {
	const button = args.object as Button;
	if (isWindows) {
		// The underlying Microsoft.UI.Xaml.Controls.Button.
		button.windows.CornerRadius = { TopLeft: 8, TopRight: 8, BottomRight: 8, BottomLeft: 8 };
	}
}
```

WinRT async operations convert to promises with `NSWinRT.toPromise`, and events are assigned like
DOM `on*` handlers:

```ts
const folder = Windows.Storage.ApplicationData.Current.LocalFolder;
const file = await NSWinRT.toPromise(folder.CreateFileAsync('notes.txt', Windows.Storage.CreationCollisionOption.ReplaceExisting));
await NSWinRT.toPromise(Windows.Storage.FileIO.WriteTextAsync(file, 'Hello from NativeScript'));

const panel = new Microsoft.UI.Xaml.Controls.StackPanel();
panel.SizeChanged = (sender, e) => console.log(e.NewSize.Width); // assigning subscribes, null unsubscribes
```

C# sources in `App_Resources/Windows` (or a plugin's `platforms/windows`) are compiled into the app
and usable from JavaScript by namespace, with no registration step:

```csharp
// App_Resources/Windows/src/Greeter.cs
namespace MyApp.Native;

public class Greeter
{
    public string Greet(string name) => $"Hello, {name}!";
    public static async Task<int> AddLaterAsync(int a, int b, int ms) { await Task.Delay(ms); return a + b; }
}
```

```ts
const greeter = new MyApp.Native.Greeter();
greeter.Greet('Windows'); // "Hello, Windows!"
await MyApp.Native.Greeter.AddLaterAsync(20, 22, 10); // 42; a Task is awaitable
```

WinRT and .NET classes can be extended from JavaScript, as Java and Objective-C classes can:

```ts
@NativeClass()
class FixedPanel extends Microsoft.UI.Xaml.Controls.Panel {
	MeasureOverride(available: Windows.Foundation.Size) {
		return { Width: 120, Height: 40 };
	}
	ArrangeOverride(finalSize: Windows.Foundation.Size) {
		return finalSize;
	}
}
```

# Motivation

NativeScript has supported the two mobile platforms since 2015, and visionOS since 8.6. Windows
support was first requested in 2015 (NativeScript/NativeScript#254, then targeting UWP) and the
issue is still open. Windows is the most common desktop OS, and teams that ship a NativeScript
mobile app today have to build their Windows app with a second stack (Electron, .NET MAUI, React
Native for Windows, or plain WinUI).

The goals are the same ones that define NativeScript on the other platforms:

- **All of the platform, from JavaScript.** Every WinRT, WinUI and Windows App SDK API, any
  third-party `.winmd` component, and any .NET assembly must be callable without writing bindings.
  New Windows SDK APIs must be usable the day they ship.
- **Native UI.** Core's views must be real WinUI controls, so apps get native input, accessibility,
  text rendering, theming (light, dark, high contrast) and the Fluent look.
- **Existing apps and plugins keep working.** An app written against `@nativescript/core` must run
  on Windows with no code changes, and a JS-only plugin must need nothing to support Windows.
- **The same workflow.** `ns run`, `ns debug`, LiveSync, HMR, `App_Resources`, `.windows.ts` files,
  `native-api-usage.json` and plugin `platforms/<name>` folders must behave the way they do on
  Android and iOS.
- **Shippable apps.** Release builds must produce signed MSIX packages that can be sideloaded or
  submitted to the Microsoft Store.

Desktop adds requirements mobile does not have, and they shape parts of the design below: windows
resize continuously, an app can have several windows, a window can be closed while the app keeps
running, and there is a mouse, a keyboard and multiple displays with different scale factors.

# Detailed design

## Packages and repositories

| Package | Repository | Contents |
|---|---|---|
| `@nativescript/windows` | NativeScript/windows | The runtime (`nativescript.dll`, x64 and arm64), the WinUI 3 host project template, the .NET bridge and the build tools (typings generator, static binding generator, manifest merger, bundle packer). |
| `@nativescript/windows-v8`, `-hermes`, `-quickjs`, `-jsc` | NativeScript/windows (`packages/`) | The same framework with a Node-API-based runtime on another engine. Drop-in replacements for `@nativescript/windows`, as `@nativescript/android-jsc` is for Android. |
| `@nativescript/core` | NativeScript/NativeScript | `*.windows.ts` modules, the `NativeScript.Widgets` native component, Windows typings in `types-minimal`, webpack and Vite support. |
| `nativescript` | NativeScript/nativescript-cli | The `windows` platform. |
| App templates | NativeScript/nativescript-app-templates | `App_Resources/Windows`. |

The runtime is versioned on its own, like `@nativescript/android` and `@nativescript/ios`. It is in
public beta today (`1.0.0-beta.13` on npm). Core declares the minimum runtime it needs in
`package.json`, as it does for the other platforms (`"windows": "9.1.0"` in
`nativescript.platforms`).

Requirements: Windows 10 version 1809 (build 17763) or later, x64 or arm64, the .NET 10 SDK to
build, and Developer Mode to run debug builds. Visual Studio is not required: the CLI builds with
`dotnet build`. Windows apps can only be built on Windows.

## Runtime

### Host

The generated platform project is a C# WinUI 3 app (`net10.0-windows10.0.26100.0`, Windows App
SDK 1.6). Its `RuntimeHost` P/Invokes a small C ABI exported by `nativescript.dll`
(`runtime_init`, `runtime_runscript`, `runtime_pump_timers`, `runtime_notify_app_event`, ...). The
runtime itself is written in Rust. The default build embeds V8; the engine variants implement the
same ABI over Node-API, so the host project is identical for every engine.

JavaScript runs on the UI thread, driven by that thread's `DispatcherQueue`. Timers, promises and
microtasks drain on the same queue. Workers are supported, each on its own thread with its own
dispatcher. WinRT and .NET callbacks raised on a background thread are delivered on the JS thread
that registered them, and their return values and exceptions flow back to the native caller.

### Metadata

Android and iOS generate metadata at build time. Windows does not need to: every WinRT API is
described by `.winmd` files that the OS and the Windows App SDK already ship. The runtime resolves
types from them lazily, the first time JavaScript touches a namespace or a class, and caches the
result. A plugin's or app's own `.winmd` files are registered at startup, so third-party WinRT
components (WebView2, for example) work the same way.

`App_Resources/Windows/native-api-usage.json` limits the API surface as it does on Android and iOS,
with the same pattern syntax and the same whitelist/blacklist rules. Because Windows resolves types
at run time, the filter is enforced where JavaScript reaches a type by name: a filtered type is
`undefined`, while objects of filtered types returned by allowed APIs still work.

### JavaScript surface

- **Namespaces.** `Windows`, `Microsoft` and `System` are globals, and so is the root namespace of
  any app or plugin C# source or referenced assembly. `Windows.Storage.StorageFile` is the
  `StorageFile` runtime class.
- **Members** keep their WinRT/.NET names and casing (`file.Path`, `FileIO.ReadTextAsync`), as iOS
  keeps Objective-C selectors. Overloads are selected by argument count.
- **Values.** Numbers, strings and booleans map to JS primitives. Structs (`Size`, `Color`,
  `Thickness`) are plain objects with their field names. Enums are numbers, available as
  `Namespace.EnumName.Member`. `IReference<T>` accepts and returns `null` and plain numbers. Values
  boxed in `Object` members (`Tag`, `PropertySet`) read back as JS primitives.
- **Async.** `NSWinRT.toPromise(op)` turns an `IAsyncAction`, `IAsyncOperation<T>` or one of
  their `WithProgress` variants into a promise that resolves to the operation's result and rejects
  when it fails or is cancelled. The conversion is explicit on purpose: operations are not
  thenable, so one can be stored, returned from an `async` function or passed back to a native API
  as an operation, with its own members (`Status`, `Cancel()`, `Progress`) intact. .NET `Task`
  results are awaitable directly.
- **Collections** keep their WinRT methods (`Append`, `GetAt`, `Size`), and an `IBuffer` can be
  viewed as an `ArrayBuffer` without a copy.
- **Events.** Assigning a function subscribes (`el.Loaded = handler`), assigning `null` unsubscribes
  and reading returns the current handler. Static events use the same syntax.
- **Delegates.** JS functions are accepted wherever a WinRT delegate, `Func<>` or `Action<>` is
  expected.
- **Extending.** WinRT classes and .NET classes can be extended with `class ... extends`,
  `@NativeClass`, `Class.extend()` or TypeScript's ES5 output; interfaces are implemented with
  `new Interface({...})` or `@Interfaces`. XAML calls JS overrides (`MeasureOverride`,
  `OnApplyTemplate`) like any other subclass. Extending .NET types relies on a static binding
  generator that emits C# proxies at build time, as Android's SBG does for Java.
- **Lifetime.** JS subclass instances that nothing references are collected on both sides. An
  instance only native code holds keeps working, and UI subclasses in the visual tree keep their
  JS fields.

### .NET

The host already runs on .NET, so .NET types are reachable directly, through a bridge
(`DotNetBridge`) that the runtime loads in-process. Apps are framework-dependent: the .NET Desktop
Runtime is a prerequisite on the user's machine, like the Windows App Runtime, and is not bundled
into the package. An app that wants to ship its own copy can publish self-contained through
`App_Resources/Windows/app.csproj`. C# files in `App_Resources/Windows` and in each
plugin's `platforms/windows` are compiled into the app, and NuGet packages can be referenced from
`App_Resources/Windows/app.csproj` or a plugin's `plugin.props`. This is the Windows counterpart of
Java/Kotlin sources and Gradle dependencies on Android, or Swift sources and CocoaPods on iOS.

### Engines

`@nativescript/windows` uses V8. The `@nativescript/windows-<engine>` packages provide the same
framework on V8 over Node-API, Hermes, QuickJS and JavaScriptCore. An app switches by changing
the dependency and rebuilding. The CLI recognises all five package names as the Windows runtime.
The engine variants exist for size (QuickJS and Hermes builds are a fraction of V8's) and for teams
that standardise on one engine across platforms. This RFC does not propose a default other than
V8.

### Debugging

Debug builds include the Chrome DevTools inspector. `ns debug windows` starts the app with the
inspector on `127.0.0.1:43000` (or the next free port), and supports `--debug-brk` and `--start`
as on the other platforms. Console output, uncaught JS errors and native crash diagnostics are
written to the app's `LocalState` folder, which the CLI tails.

### Source protection

Windows apps ship JavaScript to a desktop file system where it is easy to read. Release builds can
seal the bundler output into an encrypted `app.nsbundle` instead of plain files, opted into with
`windows.sourceProtect` in `nativescript.config.ts` or `--source-protect` on the command line. It
raises the bar for casual inspection; it is not DRM, since the key material ships in the app.

## CLI

### Platform project

`ns platform add windows` installs the runtime package and copies its `framework/` into
`platforms/windows`, naming the project folder after the app:

```
platforms/windows/
  <ProjectName>/            WinUI 3 host project (csproj, App.xaml, Package.appxmanifest)
    app/                    bundler output
    App_Resources/Windows/  staged app resources
    Assets/                 staged App_Resources/Windows/Assets
    plugins/                staged plugin platforms/windows folders, Plugins.props/.targets
  dotnet-bridge/
  libs/<arch>/nativescript.dll
  tools/                    typings generator, binding generator, manifest merger, packer
```

### Prepare

`ns prepare windows` runs the bundler (`platforms/windows/<ProjectName>/app` is the output path for
both webpack and Vite), then:

- Creates `App_Resources/<Platform>` when the app has none, by copying it from the default
  template, so existing apps can add Windows without hand-copying files. The CLI used to do this
  only when the whole `App_Resources` folder was missing; it now does it per platform, for every
  platform.
- Copies `App_Resources/Windows` into the platform project and its `Assets` into the project's
  `Assets`, honouring `appResourcesPath`.
- Interpolates `App_Resources/Windows/Package.appxmanifest`: the package identity comes from `id`
  (or `windows.id`), placeholder `PhoneProductId`/`PhonePublisherId` GUIDs are replaced with generated
  values, and missing required attributes are filled in. The app's manifest is merged into the
  template's at build time. `--phone-product-id`, `--phone-publisher-id`, `--regen-guid` and `--no-auto-guid`
  control the GUIDs.
- Stages each plugin's `platforms/windows` folder into `plugins/<name>`, rebuilt from scratch on
  every prepare so deleted files and uninstalled plugins disappear, and writes aggregate
  `Plugins.props`/`Plugins.targets` that the host project imports.
- Writes `native-api-usage.json` out as `whitelist.mdg`/`blacklist.mdg`, as it does for the other
  platforms; the build copies them next to the exe.

### Build and deploy

The CLI builds with `dotnet build`. There is no emulator: the local machine is the only device, and
`ns devices` lists it.

- **Debug** builds run from the build output. The CLI registers the loose layout with
  `Add-AppxPackage -Register` (which needs Developer Mode) and launches it through its package
  family name.
- **Release** builds produce an MSIX package in `platforms/windows/<ProjectName>/AppPackages`:
  - `--certificate <pfx>` with `--certificate-password`, or `--certificate-thumbprint`, signs a
    sideloadable `.msix`;
  - `--store-upload` produces an unsigned `.msixupload` for Partner Center, which re-signs it;
  - `--msixbundle` produces a `.msixbundle`;
  - with no signing options the CLI warns and produces an unsigned package.
- `--arch x64|arm64` selects the architecture. The default is `x64` (64-bit x86); `arm64` is opt-in.
  `--copy-to` works as on the other platforms.

### LiveSync and HMR

HMR works with both bundlers, on every engine.

- **Webpack.** The CLI copies webpack's hot-update files into the running app's `bin\app` and
  then writes an `ns-livesync` marker into the app's LocalState. Debug hosts watch for it and call
  `global.__onLiveSync()` on the UI thread, which is the hook the Android and iOS runtimes call
  after a LiveSync, so webpack's HMR runtime applies the update in place. When a hot update fails
  (an entry module that is not hot-acceptable, for example), the CLI restarts the app, as on the
  other platforms. A host writes `ns-livesync-ready` while it watches, so a CLI paired with an
  older host falls back to restarting.
- **Vite.** A running app pulls changed modules from the dev server over its WebSocket.

### Other commands

- `ns doctor` checks for Windows, the .NET SDK version and Developer Mode.
- `ns typings windows` generates TypeScript declarations into `typings/windows` from the Windows
  SDK, the Windows App SDK, referenced `.winmd` files and .NET assemblies (`--lib`, `--libs`,
  `--input`, `--root`, `--roots` select extra inputs).
- `ns build|run|debug windows` are Windows-only; their help says so on macOS and Linux.

## Core

### Platform detection

| Android / iOS | Windows |
|---|---|
| `__ANDROID__` / `__IOS__` | `__WINDOWS__` |
| `isAndroid` / `isIOS` | `isWindows` |
| `platformNames.android` | `platformNames.windows` (`'Windows'`) |
| `view.android` / `view.ios` | `view.windows`, the native `Microsoft.UI.Xaml` element |
| `Application.android` / `Application.ios` | `Application.windows` (`WindowsApplication`) |
| `<android>` / `<ios>` in XML | `<windows>` |
| `*.android.ts` / `*.ios.ts` | `*.windows.ts` |
| `config.android` | `config.windows` |

`isWindows` reads `typeof __WINDOWS__ !== 'undefined'`, so code compiled by a bundler that predates
this RFC still evaluates it to `false`.

### UI

Every core module has a `*.windows.ts` implementation: application, file system, HTTP, connectivity,
application settings, image source and cache, timers, fonts, color, dialogs, gestures, animations,
transitions and all views and layouts. Views map to WinUI controls (`Button` to `Button`, `TextField`
to `TextBox`, `ListView` to `ListView`, `DatePicker` to `DatePicker`, and so on).

Layout differs from the other platforms. On Android and iOS core measures and positions every view
in JavaScript. On Windows each layout container is a XAML panel, and XAML's measure and arrange pass
positions the children: `GridLayout` is a XAML `Grid`, `StackLayout` and `FlexboxLayout` are native
C++/WinRT panels, and `AbsoluteLayout`, `DockLayout` and `WrapLayout` place children on a `Canvas`.
Core still implements `measure`/`layout`/`onMeasure` so custom views and plugins that call them
work, but the common containers lay out in native code, which matters for resizable desktop windows. A parent with its own
layout engine can take over its children's percentage sizes through
`View._setChildPercentSize(child, horizontal, fraction)` (Windows only, NativeScript/NativeScript#11468).

The panels and helpers XAML lacks (flexbox, CSS-style borders and shadows, clipping, tiled
backgrounds, text shadows) live in `NativeScript.Widgets`, a small C++/WinRT component in
`packages/ui-mobile-base/windows`. It plays the role `widgets-release.aar` plays on Android, and
ships the same way: prebuilt for x64 and arm64 in core's `platforms/windows`, as the Android widgets
are in `platforms/android`. Its `plugin.targets` copies the right architecture into the app and
registers its activatable classes in the app manifest. Apps never list them by hand.

CSS is applied through the same style system. Backgrounds, borders (including per-side colours and
radii), `box-shadow`, `clip-path`, gradients and fonts are rendered with WinUI brushes and the
Composition API.

### Desktop behaviour

- **Windows.** Core's cross-platform `NativeWindow` (NativeScript/NativeScript#11181) is backed by
  `Microsoft.UI.Xaml.Window`. An app can open several windows; `Screen` metrics and the app root
  follow the primary window, and the app ends once its last window closes.
- **Lifecycle.** `suspend`/`resume` and `background`/`foreground` fire when the app as a whole
  moves between active and inactive, not when focus moves between its own windows.
- **Resizing.** `@media` queries are evaluated against the window a view is in, not the main
  screen, and re-evaluated as the window resizes. File-name screen qualifiers
  (`main-page.minWH600.xml`, `.land`/`.port`) are re-resolved when the window crosses a threshold,
  and the current page is swapped for the matching variant.
- **Coordinates.** `getLocationInWindow`, `getLocationOnScreen` and `getLocationRelativeTo` return
  DIPs as on the other platforms, accounting for the window's position and scale
  (NativeScript/NativeScript#11515).

### Typings

Windows typings follow the existing split:

- `@nativescript/types-minimal` holds the declarations core compiles against, for the latest SDK
  only. It gains `lib/windows`: the Windows SDK (`windows.d.ts`), WinUI and the Windows App SDK
  (`microsoft.ui.d.ts`), `NativeScript.Widgets` and the runtime's helpers.
- A new `@nativescript/types-windows`, next to `types-ios` and `types-android`, holds the full
  declarations for apps, and `@nativescript/types` depends on it.

All of them are generated by the runtime's typings generator. Apps can generate declarations for
their own `.winmd` and .NET inputs with `ns typings windows`.

### Bundlers

`@nativescript/webpack` gets a `windows` platform that defines `__WINDOWS__` and emits into the
Windows project's `app` folder. `@nativescript/vite` supports the `windows` platform in build mode
and in HMR mode, including `.windows` file resolution and the per-platform global defines.

### Changes to shared code

A few changes in the core PR touch code all platforms run, and are worth reviewing on their own:

- `checkIfMediaQueryMatches` and `matchMediaQueryString` take an optional window context, and the
  selector scope passes the node's window. Without one they behave as today.
- `var()` fallbacks may contain commas and nested parentheses, custom properties may refer to ones
  declared after them, and a declaration that fails to apply resets the property instead of leaving
  the previous value in place (CSS's "invalid at computed-value time" rule). These are under
  review on their own in NativeScript/NativeScript#11527.
- A stylesheet whose resolved path does not exist is skipped instead of being read.

## Templates

Every template gets `App_Resources/Windows`, through the shared `shared-mobile` resources:

- `Package.appxmanifest` with `__APP_IDENTIFIER__` and `__PROJECT_NAME__` placeholders, the Windows
  App Runtime dependency, and placeholder phone identity GUIDs the CLI fills in;
- `app.csproj`, the app's MSBuild hook (NuGet references, properties), imported by the host project;
- `app.manifest` (per-monitor DPI awareness, supported OS);
- tile, splash and store logos at every scale.

Templates do not add `@nativescript/windows` to `package.json`; `ns platform add windows` or the
first `ns run windows` installs it, as for the other runtimes.

## Plugins

A plugin supports Windows the way it supports the other platforms:

```
my-plugin/
  index.windows.ts
  platforms/windows/
    src/*.cs                 compiled into the app
    *.winmd, *.dll           WinRT components and native libraries
    plugin.props             optional: NuGet PackageReferences, MSBuild properties
    plugin.targets           optional: custom build steps
```

The CLI copies everything except JS, TS, JSON, map and markdown files. When a plugin provides no
`plugin.props`/`plugin.targets` the CLI generates them, copying the plugin's binaries into the app
output. JS-only plugins need nothing. A plugin declares its minimum runtime under
`nativescript.platforms.windows` in `package.json`, as core does.

# Drawbacks

- **A third platform to maintain.** Every new core feature and every bug fix now has a third
  implementation to keep in step. The Windows views are a large body of new code (the core PR adds
  about 16,000 lines of Windows modules and native widgets, outside generated typings) and there
  are no automated specs for the Windows views yet.
- **Different layout model.** Delegating layout to XAML is what makes resizing fast and keeps
  controls native, but it means Windows can disagree with Android and iOS in edge cases that the
  shared JS layout code would have handled identically. Fixes for those land in Windows-specific
  code, not in the shared layout.
- **Larger core package.** The Windows SDK and WinUI declarations add about 166,000 lines to
  `types-minimal`, and core carries prebuilt `NativeScript.Widgets` binaries for two
  architectures, as it already does for the Android widgets.
- **Plugin ecosystem starts empty.** Most popular plugins have native code and need a Windows
  implementation. Until they have one, apps that depend on them cannot target Windows.
- **Windows-only builds.** Like iOS on macOS, Windows apps can only be built on Windows, so CI needs
  Windows runners.
- **.NET at run time.** Apps are framework-dependent, so end users need the .NET Desktop Runtime
  in addition to the Windows App Runtime unless the app opts into a self-contained publish.
- **Maturity.** The runtime is in beta. Automated Windows specs in core and arm64 CI are not done
  yet.

# Alternatives

- **UWP.** The original request targeted UWP. Microsoft has moved new UI work to WinUI 3 and the
  Windows App SDK, and UWP apps are sandboxed in ways that make many desktop use cases impossible.
  WinUI 3 runs as a full-trust desktop app and still uses MSIX for packaging and the Store.
- **Win32 / WPF.** Either would run on older Windows versions, but with an older UI stack, no
  current Fluent controls, and no metadata format the runtime could read the way it reads `.winmd`.
- **A web view (Electron, Tauri, Capacitor).** Already available to anyone today, and the opposite
  of NativeScript's native-UI model.
- **Build-time metadata, as on Android and iOS.** WinRT metadata already ships with the OS and the
  SDKs, so reading it at run time avoids a generation step and makes new SDK APIs available
  immediately. The cost is a lazy lookup on first use of each type, which is cached.
- **JS-driven layout on Windows too.** Positioning every XAML element from JavaScript would make
  layout identical across platforms, but every window resize would run the whole layout pass in JS
  while working against XAML's own layout system.
- **Only a Node-API runtime.** Every engine variant already shares one Node-API implementation, so
  the runtime could drop the classic V8 build. The classic build is the most mature and is what
  apps have been tested on, so this RFC keeps it as the default and leaves the variants opt-in.
  The same code also builds as a Node-API addon (`@nativescript/windows-napi`) that exposes WinRT
  to Node, Bun and Deno; that is a separate use case and not part of this RFC.
- **Not doing this.** NativeScript remains mobile plus visionOS, and teams keep a second stack for
  Windows.

# Adoption strategy

This is additive and not a breaking change. Apps and plugins that do nothing are unaffected.

1. Core, the CLI and the templates merge their Windows branches. Core's Windows modules are inert
   on Android and iOS, since bundlers only include `.windows.ts` files for the Windows platform.
2. `@nativescript/windows` stays in beta and is published from the runtime repo's CI
   (`npm_release.yml`), with the engine variants alongside it.
3. Existing apps opt in with `ns platform add windows`; the CLI adds `App_Resources/Windows` from
   the default template if the app has none. This needs the templates from
   NativeScript/nativescript-app-templates#244 to be published first.
4. Plugin authors add `index.windows.ts` and `platforms/windows`. Porting the plugins in
   `NativeScript/plugins` early would give authors working examples.
5. Docs gain a Windows section: setup, `.windows.ts`, the WinRT/.NET marshaling rules,
   `App_Resources/Windows`, signing and Store submission.
6. 1.0 is declared once core has automated Windows specs on CI and arm64 is verified on CI.

# Unresolved questions

None at the time of writing.
