- Start Date: 2026-09-17
- Target Major Version: 9.x
- Reference Issues: https://github.com/NativeScript/NativeScript/pull/11449
- Implementation PR: (leave this empty)

# Summary

Support the CSS `env()` function in NativeScript stylesheets, inline styles and JS/TS style
writes, with the `safe-area-inset-*` and `safe-area-max-inset-*` environment variables from
[CSS Environment Variables Module Level 1](https://drafts.csswg.org/css-env-1/) populated from
the real window insets on iOS and Android.

Add one cross-platform property, `overflowSafeArea`, that tells the framework to stop insetting
a view on the given edges, so the author can inset it themselves in CSS without the two paddings
stacking.

Make padding that names an edge's inset take that edge over, so `padding-bottom:
env(safe-area-inset-bottom)` never stacks with the framework's own inset on the same view, and
tells the views inside it that the edge is already taken. Add a `safe-area-padding` shorthand
for the common case of "this padding, plus whatever inset is still left".

# Basic example

```css
/* The screen lays out under the navigation bar; the footer keeps itself clear of it. */
.screen {
	overflow-safe-area: bottom;
}

.footer {
	padding-bottom: calc(12dip + env(safe-area-inset-bottom));
}
```

```xml
<GridLayout class="screen" rows="*, auto">
	<ListView row="0" />
	<StackLayout row="1" class="footer" />
</GridLayout>
```

The same values set from TypeScript:

```ts
screen.overflowSafeArea = 'bottom';
footer.style.paddingBottom = 'calc(12dip + env(safe-area-inset-bottom))';

// The `View` alias works too, since padding is a style property.
footer.paddingBottom = 'calc(12dip + env(safe-area-inset-bottom))';
```

The footer needs no switch of its own: its padding names the bottom inset, so the framework
stops insetting its bottom edge. The same footer written with the shorthand, which adds only
the inset an ancestor has not already taken:

```css
.footer {
	safe-area-padding: 0 16dip 12dip;
}
```

And the window insets on their own, when a layout is easier to do in code:

```ts
import { SafeArea } from '@nativescript/core';

SafeArea.insets; // { top: 24, right: 0, bottom: 48, left: 0 } in dip
SafeArea.on('insetsChanged', ({ insets }) => console.log(insets.bottom));
```

# Motivation

Edge-to-edge is no longer optional. Android 15 (SDK 35) removed the ability to opt out, and
iOS has had a non-rectangular safe area since the iPhone X. Every non-trivial NativeScript
app now has to place content relative to system bars, the display cutout and the home
indicator.

Today an app has three ways to do that, and none of them is CSS:

1. `iosOverflowSafeArea` / `iosIgnoreSafeArea`: iOS only, boolean, all edges at once.
2. `androidOverflowEdge`: Android only, per-edge, with its own vocabulary
   (`dont-apply`, `all-but-top`, `cutout`, `ignore`).
3. The `androidOverflowInset` event: Android only, imperative, fires on the layout pass.

That produces three concrete problems.

**Layout values live in the wrong place.** The inset is a length, and every other length in a
NativeScript layout is expressed in CSS. Safe-area padding is the one that has to be computed
in a TypeScript event handler, which means it cannot participate in the cascade, cannot be
overridden by a theme, cannot be varied by a media query, and cannot be shared between
components the way a class can.

**The two mechanisms stack.** Both `androidOverflowEdge` and the iOS shrink-to-safe-area pass
apply padding, and so does the author. When both fire you get the inset twice. There is no
declaration an author can write that means "I am taking this edge, leave it alone", so the
workaround is to disable the framework's handling wholesale and reimplement all four edges by
hand, including the edges that were working fine.

**Web code does not port.** `padding-bottom: env(safe-area-inset-bottom)` is the single most
copied line of layout CSS in mobile web development. It is what every design system, every
Tailwind `pb-safe` utility and every Stack Overflow answer produces. Today it silently does
nothing in NativeScript: `env(...)` reaches `Length.parse`, `parseFloat` returns `NaN`, and the
declaration is dropped. Authors coming from web, and LLMs generating NativeScript CSS from web
training data, both write it and both get a layout that is wrong only on hardware with a
notch.

The constraints this RFC is trying to satisfy, independent of the solution below:

- Safe-area lengths must be usable anywhere a length is usable, including inside `calc()`.
- There must be exactly one declaration that stops the framework from insetting an edge, and
  it must work the same on both platforms.
- The values must update when the insets change (rotation, cutout, bar visibility) without
  the author subscribing to anything.
- CSS written against the web spec should behave the way the web spec says it behaves, so that
  existing knowledge and existing code transfer.

# Detailed design

## The `env()` function

The grammar is the one from css-env-1 § 2:

```
env() = env( <custom-ident> <integer [0,∞]>*, <declaration-value>? )
```

The first argument names an environment variable. Any integers after it are indices into the
dimensions of an array-like variable. Everything after the first top-level comma is the
fallback, which may itself contain `env()`, `var()` and `calc()`.

Substitution follows css-env-1 § 2.1:

- If the name is recognised, the index count matches the variable's dimension count and the
  indices are in range, the function is replaced by the variable's value. **The fallback is
  discarded without being parsed**, exactly as on the web. An invalid fallback next to a
  valid name is not an error.
- Otherwise, if a fallback is present, the function is replaced by the fallback, with any
  nested functions substituted in turn.
- Otherwise the declaration is **invalid at computed-value time** (IACVT). NativeScript already
  has this concept: `var()` resolves to the `unset` sentinel, which the cascade turns into
  `unsetValue` and the property falls back to its inherited or initial value. `env()` reuses it.

Environment variable names are `<custom-ident>`, and so are case-sensitive.
`env(Safe-Area-Inset-Top)` is an unknown variable, not a spelling of a known one.

### Supported environment variables

| Name | Type | Dimensions | Source |
| --- | --- | --- | --- |
| `safe-area-inset-top` | `<length>` | 0 | Window inset, current state |
| `safe-area-inset-right` | `<length>` | 0 | " |
| `safe-area-inset-bottom` | `<length>` | 0 | " |
| `safe-area-inset-left` | `<length>` | 0 | " |
| `safe-area-max-inset-top` | `<length>` | 0 | Largest value seen this session |
| `safe-area-max-inset-right` | `<length>` | 0 | " |
| `safe-area-max-inset-bottom` | `<length>` | 0 | " |
| `safe-area-max-inset-left` | `<length>` | 0 | " |

`viewport-segment-*` and `preferred-text-scale` are parsed correctly (they are unknown names,
so they take their fallback) but are not populated. See [Unresolved questions](#unresolved-questions).

Variables are published through a small registry, so a plugin can add its own:

```ts
registerCssEnvironmentVariable('brand-gutter', 0, () => '16dip');
```

### Scope: the window, not the view

`env(safe-area-inset-bottom)` resolves to the same value for every view in the application,
because on the web it is a property of the viewport rather than of an element. A view nested
ten levels deep inside a page that has already consumed the bottom inset still sees the full
navigation bar height.

This is a deliberate choice and the alternative, resolving to whatever inset an ancestor has
not yet consumed, is discussed under [Alternatives](#alternatives).

### Units

Values substitute as `<n>dip`. NativeScript's CSS length unit is the device-independent pixel,
which is the same thing a browser calls a CSS pixel, so `48dip` is the direct analogue of the
`48px` a browser would substitute.

The existing `calc()` pipeline strips the `dip` suffix before handing the expression to
`@csstools/css-calc`, so `calc(12dip + env(safe-area-inset-bottom))` reduces to `calc(12 + 48)`
and then to `60`. A bare `env(safe-area-inset-bottom)` reaches `Length.parse` as `48dip`, which
parses as 48 dip.

Mixing `dip` and `px` inside one `calc()`, as in `calc(env(safe-area-inset-top) + 2px)`, does not
work. This is pre-existing behaviour of the `dip` stripping and this RFC does not change it.

### Substitution order

A declaration value is resolved in this order:

1. `env()` substitution.
2. `var()` substitution.
3. `env()` substitution again, but only if the value still contains `env(`, which happens
   when a custom property set at runtime through `Style.setUnscopedCssVariable` expanded to
   text containing `env()`.
4. `calc()` reduction.

`env()` runs before `var()` because css-env-1 § 2.1 notes that env substitution has to precede
var substitution, and because the ordering is observable: in

```css
padding-top: env(safe-area-inset-top, var(--fallback-that-does-not-exist));
```

the name resolves, so the fallback must never be looked at. Resolving `var()` first would turn
the whole declaration into `unset` before `env()` ever ran.

Custom property declarations are themselves resolved, so this works and stores `24dip`:

```css
:root { --safe-top: env(safe-area-inset-top); }
.header { padding-top: var(--safe-top); }
```

### Shorthands

`env()` in a shorthand has the same problem `var()` does, and takes the same solution. Core
already cascades a `CssPendingSubstitution` placeholder per longhand when a shorthand value
contains an unresolved expression, and expands the shorthand once the expression has been
resolved against the view. `env()` joins the set of expressions that trigger that path, so

```css
padding: env(safe-area-inset-top) 16dip env(safe-area-inset-bottom) 16dip;
```

expands to four longhands after substitution. A shorthand that turns out to be invalid at
computed-value time unsets every one of its longhands rather than leaving them on whatever the
previous declaration set.

## Setting `env()` from JS/TS

Today `view.style.paddingBottom = 'calc(12dip + env(...))'` cannot work, and neither can
`'calc(12dip + 4dip)'`: `CssProperty`'s local-value setter runs the value converter directly on
the string, so the expression reaches `Length.parse` unevaluated. Expression evaluation is only
wired into the cascade.

This RFC routes local writes through the same evaluator the cascade uses. A string value
containing `env(`, `var(` or `calc(` is resolved against the view before the converter sees it.
As a side effect `calc()` and `var()` start working from JS/TS too, which they do not today.

The raw text is retained alongside the computed value so the write can be re-evaluated when
the insets change. Setting a non-expression value, or resetting the property, drops it.

This applies to the `View` aliases as well, because `paddingBottom` and friends are style
properties projected onto the view.

`CoreTypes.FixedLengthType` gains a `CssFunctionValue` member so this typechecks without a
cast. The type already admitted `` `${number}dip` `` and `` `${number}px} `` string literals,
so this extends an existing pattern rather than widening anything to bare `string`:

```ts
export type CssFunctionValue = `calc(${string})` | `env(${string})` | `var(--${string})`;
```

## Where the values come from

A new module, `@nativescript/core/safe-area`, owns the window insets and is the single source
for the environment variables.

```ts
export interface SafeAreaInsets {
	readonly top: number;    // dip
	readonly right: number;
	readonly bottom: number;
	readonly left: number;
}

export declare class SafeAreaCommon extends Observable {
	readonly insets: SafeAreaInsets;
	readonly maxInsets: SafeAreaInsets;
	refresh(): boolean;
	on(event: 'insetsChanged', cb: (data: SafeAreaInsetsChangedEventData) => void, thisArg?: any): void;
}

export declare const SafeArea: SafeAreaCommon;
```

**iOS** reads `safeAreaInsets` from the key window. The change signal is the Page view
controller's `viewSafeAreaInsetsDidChange`, which is UIKit's own notification that the insets
have settled; there is no `NSNotification` for it.

**Android** reads `androidx.core.view.ViewCompat.getRootWindowInsets(decorView)` and takes
`Type.systemBars() | Type.displayCutout()`. The change signal is an
`OnApplyWindowInsetsListener` installed on the decor view that returns
`ViewCompat.onApplyWindowInsets(v, insets)`, the androidx-documented way to observe insets
without consuming them or replacing the default handling. The listener is re-installed when the
decor view changes, which is what happens when the activity is recreated.

Both are subscribed lazily, on the first read, and seeded at that point; `Application.initRootView`
refreshes as a backstop so a recreated activity re-subscribes.

The IME is deliberately excluded. On the web the keyboard is `env(keyboard-inset-*)` from the
VirtualKeyboard API, not part of the safe area, and folding it in would make
`padding-bottom: env(safe-area-inset-bottom)` jump when the keyboard opens. `androidOverflowEdge`
continues to handle the IME as it does today.

Android needs no change to `widgets-release.aar`; the inset source is entirely TypeScript.

`maxInsets` is the largest value seen for each edge during the session. The spec describes
`safe-area-max-inset-*` as the maximum the inset can take in any state, which is not
something either platform reports; a running maximum converges on it after the first rotation
and is the same approximation browsers make for their dynamic toolbars.

## Re-evaluation

When the insets change, every view holding an `env()`-dependent value re-resolves it.

A `CssState` whose applied declarations contained `env(` registers itself as a dependent while
loaded and deregisters on unload. On an inset change each registered state re-runs its dynamic
update, which re-reads the raw declarations from the matched rulesets and resolves them again;
the existing diff means only the values that actually changed are written to the view. A `Style`
holding local `env()` writes registers itself the same way and replays them from the retained
source text. Dependents are held weakly, so forgetting to deregister cannot leak.

The cost when nothing uses `env()` is one `Set` that stays empty, and one
`value.includes('env(')` per declaration on the paths that already call
`value.includes('var(--')` and `value.includes('calc(')`.

## Nested pages: the consumption chain

Window-scoped `env()` composes as long as an edge is padded at exactly one level. It stops
composing when a view cannot know what its ancestors did:

```
frame
  page 1: no scroll                   pads itself
  page 2: scroll                      pads its scroll content, runs under the bar
  page 3: no scroll
    page 4: nested frame, scrolls     pads its own scroll content
```

Page 3 cannot pad, because it does not know whether page 4 scrolls. Page 4 cannot know
whether page 3 already padded. Putting the padding on the frame breaks pages 2 and 4. There
is no arrangement of window-scoped `env()` that makes all four correct without each page
knowing about the others.

Two additions fix this, both **opt-in** so no existing layout moves:

```ts
SafeArea.trackConsumption = true;   // default false
```

**`View.getRemainingSafeAreaInsets()`**: the window insets minus whatever an ancestor
already took, in dip.

**`env(ns-safe-area-inset-*)`**: the same value in css. A NativeScript extension, so the
spec's `safe-area-inset-*` keep their window scope. Page 4 writes one declaration that is
correct either way:

```css
.footer { padding-bottom: env(ns-safe-area-inset-bottom); }
```

Two sources feed it:

1. **Declared consumption.** An ancestor whose `overflowSafeArea` does not name an edge is
   insetting itself there, and an ancestor whose padding names an edge's inset has padded
   it (see [Smart padding](#smart-padding)), so its subtree sees that edge as taken. This
   is platform agnostic and is what Android uses.
2. **Platform consumption**, when tracking is on. UIKit already reduces `safeAreaInsets`
   down the hierarchy, so a view inside a ScrollView that was inset reports zero. This
   matters because the layouts inset *without declaring it* (`ScrollView.onLayout` and
   `StackLayout.layoutVertical` both subtract `getSafeAreaInsets()`), and the declared
   chain cannot see them. Views expose `_supportsPerViewSafeArea`; Android leaves it false
   and stays on the declared chain, because it distributes through `LayoutBase` and has no
   per-view equivalent.

The two are intersected, so a declaration can only ever take *more*, never hand back an
edge the platform already consumed. The platform value is read before the view's own
`overflowSafeArea` and padding are taken out, so a view asking for the inset it is about to
pad itself gets the real value rather than zero.

Measured on an iPhone 17e (bottom inset 34dip), page 4's css unchanged throughout:

| page 3 declares | tracking off | tracking on |
| --- | --- | --- |
| `bottom` (pass through) | resolves 34dip, gap 68dip, doubled | resolves 0dip, gap 34dip |
| `none` (consume) | resolves 0dip, gap 34dip | resolves 0dip, gap 34dip |

## `overflowSafeArea`

```
overflowSafeArea: 'auto' | 'none' | 'all' | <edge>[ <edge>]*
<edge> = top | right | bottom | left
```

A `CssProperty`, so it can be set from a stylesheet (`overflow-safe-area: bottom;`), from XML
(`overflowSafeArea="bottom"`) and from JS/TS. Edges may be separated by spaces or commas.

The value always names **the edges the view overflows into**, meaning the framework does not
inset it there and the author is responsible for the padding, typically with `env()`. The
framework insets every edge that is *not* named.

- `auto` (the default) leaves each platform's current handling exactly as it is today. It is a
  sentinel, not an empty edge list: on Android nothing insets a view by default, while on iOS
  the layout shrinks to the safe area, so there is no single mask that means "unchanged".
- `none` overflows no edge, so the framework insets all four.
- `all` overflows every edge, so the framework insets none.
- `bottom` overflows the bottom and insets the other three.

This is the NativeScript equivalent of `<meta name="viewport" content="viewport-fit=cover">`.
It is the piece of the web model that has no CSS form on the web, which is why it is a
NativeScript-specific property rather than a standard one.

Per-edge rather than a single boolean because the common layout wants both behaviours at once:
the status bar padded automatically, the list scrolling under the navigation bar.

### Mapping

**Android.** `overflowSafeArea` sets the same `LayoutBase` overflow flags `androidOverflowEdge`
does. `auto` leaves them alone; `none` is `OverflowEdgeNone` (distribute every inset as
padding); each named edge sets the matching `OverflowEdge*` bit, which consumes that edge
without padding; `all` sets all four.

Note this makes a named edge load-bearing on Android in the *other* direction: a view is not
inset by default there, so `overflowSafeArea="bottom"` turns *on* automatic padding for the
top, left and right. That is the point, since it is the same contract as on iOS, but it is a
behaviour change for the view it is set on, which is why `auto` rather than `none` is the
default.

**iOS.** `auto` and `none` both keep the current shrink-to-safe-area pass, which already insets
every edge. A named edge zeroes that edge in `View.getSafeAreaInsets()`, so the layout stops
insetting it: the per-edge form of what `iosOverflowSafeArea` does for all edges at once.

### Relationship to the existing properties

`androidOverflowEdge`, `iosOverflowSafeArea` and `iosIgnoreSafeArea` are unchanged and keep
working. They are lower-level and platform-specific: `androidOverflowEdge` has a vocabulary
(`dont-apply`, `*-dont-consume`, `all-but-*`, `cutout`, `ignore`) that `overflowSafeArea`
deliberately does not reproduce.

When both are set on a view the platform-specific property wins, and a warning is traced under
`Trace.categories.Style`. Both feed a single resolver rather than racing to call
`setOverflowEdge` last, so the outcome does not depend on the order they were applied in. This
keeps existing apps working byte-for-byte and makes the conflict visible rather than silent.

No deprecation is proposed in this RFC.

## Smart padding

On the web, `viewport-fit=cover` hands every edge to the author, and from then on the author's
`padding: env(safe-area-inset-*)` is the only padding there is. Nothing else pads an element,
so there is nothing for it to stack with. NativeScript has a second actor: the layouts inset
themselves by default. `overflowSafeArea` is the switch that turns that off, but requiring it
next to every `env()` padding means writing the same intent twice, and forgetting the switch
is exactly the double padding this RFC exists to remove.

So a padding that names an edge's inset is itself the declaration. A `padding-*` longhand whose
value contains `env(safe-area-inset-<edge>)`, `env(safe-area-max-inset-<edge>)` or
`env(ns-safe-area-inset-<edge>)` for **its own edge** pads that edge from the safe area, and the
view takes the edge over:

- **The framework stops insetting that edge on the view.** On iOS the edge is zeroed in
  `View.getSafeAreaInsets()`, the same as naming it in `overflowSafeArea`. On Android the edge
  is added to the view's `LayoutBase` overflow flags, so the inset pass consumes it without
  padding it again. A view left on `ignore` pads nothing already, and a view whose
  `androidOverflowEdge` uses the richer tokens (`dont-apply`, `*-dont-consume`, `all-but-*`)
  is left as the author wrote it.
- **Descendants see the edge as taken.** The consumption chain treats a padded edge like one
  the ancestor insets, so `env(ns-safe-area-inset-*)` resolves to zero below it.

```css
/* No overflow-safe-area needed: this padding is what takes the bottom edge. */
.footer {
	padding-bottom: calc(12dip + env(safe-area-inset-bottom));
}
```

Only the edge's own variable counts. `padding-top: env(safe-area-inset-bottom)` is a length
like any other, not a claim on the top edge. The `padding` shorthand is checked per edge, so
`padding: env(safe-area-inset-top) 16dip` takes only the top: with two values the bottom gets
`env(safe-area-inset-top)` as well, which is not the bottom's own inset.

The declaration is read before it is resolved, since the resolved value (`48`) no longer says
where it came from. The cascade records it from the matched declarations, a local write records
it from the value written, and each edge follows the same precedence as the padding value
itself: a local write on an edge replaces what the cascade said about that edge, and resetting
the local value hands the edge back to the cascade.

A change to the padded edges re-applies the view's own inset handling and re-evaluates the
`env()` dependents once, on the next microtask, since it usually happens while styles are being
applied.

Padding wins over `overflowSafeArea` for the edges it names. `overflowSafeArea="none"` asks the
framework to inset every edge; a padding that names the bottom inset still takes the bottom,
because that is the more specific statement about that edge.

## `safe-area-padding`

```
safe-area-padding: <length>{1,4}
```

The same one to four values as `padding`, in the same order, with whatever safe area inset is
still left at the view added to each edge. It expands to the four `padding-*` longhands:

```css
.footer { safe-area-padding: 0 16dip 12dip; }

/* is */
.footer {
	padding-top: env(ns-safe-area-inset-top);
	padding-right: calc(16dip + env(ns-safe-area-inset-right));
	padding-bottom: calc(12dip + env(ns-safe-area-inset-bottom));
	padding-left: calc(16dip + env(ns-safe-area-inset-left));
}
```

It uses the per-view `ns-safe-area-inset-*` rather than the window value, so a view nested
inside one that already padded an edge adds only its own length there. And because every
longhand names its own inset, all four edges are smart padding: the view takes them over and
its descendants see them as taken.

Like `padding`, it always sets all four longhands. A value that is not one to four values is
invalid at computed-value time and unsets all four. It accepts `var()` and `calc()` values,
and is settable from XML (`safeAreaPadding="0 16 12"`) and JS/TS (`view.safeAreaPadding =
'0 16dip 12dip'`). Reading it back returns the resolved padding, the same as reading
`padding`.

This is a NativeScript property. The web writes the expansion above by hand, and nothing
stops it from doing so here, but the web also has no layout that insets itself to stack
with, so it rarely needs the remaining-inset form.

## Deviations from css-env-1

Stated explicitly so the conformance tests can encode them:

| Area | Spec | NativeScript | Why |
| --- | --- | --- | --- |
| Length unit | `px` | `dip` | `dip` is NativeScript's CSS pixel |
| Opting into the full viewport | `viewport-fit=cover` meta tag | `overflowSafeArea` property, per view | No document/meta layer exists |
| `safe-area-max-inset-*` | Maximum in any state | Maximum seen this session | Neither platform reports the former |
| `viewport-segment-*` | Populated on foldables | Unknown name, takes fallback | No multi-segment support yet |
| `preferred-text-scale` | Populated | Unknown name, takes fallback | Not yet mapped to the a11y font scale |
| Syntactically invalid `env()` | Invalid at parse time | Invalid at computed-value time | Core has no parse-time drop for this |
| `env()` in media queries | Open issue in the spec | Not supported | Spec has not settled |
| `min()` / `max()` / `clamp()` | Supported alongside `calc()` | `calc()` only | Pre-existing gap, orthogonal to this RFC |
| Per-view remaining inset | No equivalent | `env(ns-safe-area-inset-*)`, opt-in | Nested frames have no other answer |
| Padding with `env()` | Just a length | Also takes the edge over from the framework | The web has no second actor to stop |
| `safe-area-padding` | No equivalent | `padding` plus the remaining inset | Shorthand for the common nested case |

# Drawbacks

**It adds a third way to spell edge-to-edge.** `androidOverflowEdge` and `iosOverflowSafeArea`
already exist, and this RFC adds `overflowSafeArea` next to them without removing either. An
author reading an unfamiliar codebase now has three properties to understand instead of two,
and has to know the precedence rule. The alternative, deprecating the platform properties,
is worse in the short term, because `androidOverflowEdge` can express things
`overflowSafeArea` cannot.

**`overflowSafeArea` does not mean the same thing to each platform's defaults.** `auto` exists
precisely because Android and iOS disagree about what happens when you say nothing, and that
asymmetry leaks: naming an edge on Android switches three other edges from "not inset" to
"inset", while on iOS it switches one edge from "inset" to "not inset". The contract is the
same, the delta from the default is not, and that will surprise people porting a layout.

**Local-value evaluation changes behaviour for existing code.** Today
`view.style.width = 'calc(50% - 10dip)'` throws or produces `NaN`. After this change it
computes. That is the intended improvement, but it is a behaviour change on a code path apps
already exercise, and an app that relied on the throw, or on the string passing through a
custom converter untouched, would see something different. The evaluation only triggers on
strings containing `env(`, `var(` or `calc(`, which bounds the blast radius.

**Re-evaluation is a full dynamic update.** An inset change re-runs `setPropertyValues` for
every registered view rather than recomputing just the `env()`-dependent declarations. The
existing diff keeps the writes minimal, but the match and evaluate work is redundant. Rotation
already triggers a relayout, so this is not a new cost in the common case; a cutout or bar
visibility change that does not otherwise relayout would be.

**Window-scope values make double padding possible.** Because `env()` reports the same value
everywhere, an author who writes `padding-bottom: env(safe-area-inset-bottom)` on a view whose
*ancestor* has already consumed that inset gets it twice. Smart padding only covers the view
the padding is on: it can stop the framework insetting that view, not undo what an ancestor
already did. `overflowSafeArea` on the ancestor, or `env(ns-safe-area-inset-*)` and
`safe-area-padding` on the view, are the tools for this, but nothing enforces their use. The
view-relative alternative below removes the failure mode at a cost discussed there.

**Smart padding makes a length load-bearing.** Two paddings with the same resolved value can
behave differently, depending on whether the source text named an inset. That is the point,
and it matches how authors already read the declaration, but it is invisible in the computed
style and has to be documented.

**`safe-area-max-inset-*` is an approximation** and reports a too-small value until the device
has been in the orientation that produces the maximum. An app that sizes a fixed header from
it will resize once on first rotation.

**This cannot be done in user space.** A plugin can read the insets and expose them, but it
cannot participate in the cascade, cannot make `env()` parse, and cannot re-evaluate a
stylesheet declaration. The `env()` half of this has to live in core.

# Alternatives

**View-relative `env()`: resolve to the inset an ancestor has not yet consumed.** Double
padding becomes structurally impossible and `overflowSafeArea` becomes optional. Rejected for
three reasons. It diverges from the spec, so the web CSS this RFC exists to support would
behave differently in NativeScript than in a browser, and a conformance test suite could not be
written. The value of a declaration would depend on where the view sits in the tree, which is
hard to reason about and hard to test. And the ordering does not work on Android: the
consumption chain is only known after `LayoutBase` has run its inset pass, which happens on the
first layout, after CSS has been applied, so the first frame would resolve against insets
nobody has consumed yet.

**A `safe-area-inset-*` set of read-only CSS custom properties** (`var(--safe-area-inset-top)`)
instead of `env()`. Reuses the whole variable machinery and needs no new grammar. Rejected
because it is not what authors write, not what the web produces, and the leading `--` marks it
as author-defined when it is not. It also inherits `var()`'s fallback semantics, which differ
from `env()`'s in exactly the case that matters: a valid name with an invalid fallback.

**A single boolean `overflowSafeArea`** instead of an edge list. Smaller API, easier to teach.
Rejected because the everyday layout needs mixed behaviour on one view, and the all-or-nothing
form makes the author reimplement the edges that already worked.

**Requiring `overflowSafeArea` next to every `env()` padding**, with no smart padding. Keeps
padding a plain length. Rejected because it asks for the same intent twice, and the failure
mode when the switch is forgotten is the double padding this RFC is meant to remove.

**`safe-area-padding` as an edge list** (`safe-area-padding: bottom`) that pads the named edges
with the remaining inset and sets `overflowSafeArea` for the rest. Rejected because CSS has no
shorthand whose set of longhands depends on its value, which leaves the unnamed edges with no
defined meaning, and because it had to express "consumed for my subtree" and "overflowed for
myself" through the one `overflowSafeArea` value, which means opposite things for the two
jobs. Smart padding keeps those separate.

**`iosOverflowEdge` to mirror `androidOverflowEdge`, with no cross-platform property.** Keeps
the vocabulary uniform and adds no new concept. Rejected because every layout would need both
properties set to the same value, which is the problem this RFC is trying to remove.

**Doing nothing.** Apps keep computing insets in `androidOverflowInset` handlers and iOS
equivalents, safe-area layout stays outside the cascade, and copied web CSS keeps silently
failing. The workaround is real and people are using it, so this is survivable; it just keeps
the cost on every app instead of paying it once in core.

# Adoption strategy

Not a breaking change. `env()` in a stylesheet is currently a dropped declaration, so no
existing app depends on its behaviour. `overflowSafeArea` is new and defaults to `'auto'`,
which is by definition the current behaviour on both platforms. `androidOverflowEdge`,
`iosOverflowSafeArea` and `iosIgnoreSafeArea` are untouched and take precedence where both are
set.

The one behaviour change is that `calc()`, `var()` and `env()` in a JS/TS style write are now
evaluated instead of reaching the converter as raw text. No codemod is needed; an app that
wants the old behaviour can compute the value itself and assign a number.

Adoption is incremental and per-view. An app can convert one screen at a time:

```diff
- <GridLayout androidOverflowEdge="bottom">
+ <GridLayout overflowSafeArea="bottom">
```

```diff
- view.on('androidOverflowInset', ({ inset }) => {
-   footer.paddingBottom = 12 + inset.bottom;
- });
+ .footer { padding-bottom: calc(12dip + env(safe-area-inset-bottom)); }
```

For the ecosystem:

- `@nativescript/theme` gains somewhere to put safe-area utilities that work on both platforms.
- Tailwind-style plugins can implement `pb-safe` and friends as a real class instead of a
  platform shim.
- Plugins that ship layout CSS can drop their `isIOS`/`isAndroid` branches for safe-area padding.

Documentation needs: the `env()` reference page, `overflowSafeArea`, smart padding and
`safe-area-padding` on the View API page, and a migration note on the `androidOverflowEdge` page pointing at the cross-platform property for the
cases it covers.

# Unresolved questions

- **`env(keyboard-inset-*)`.** Android already surfaces the IME inset through
  `LayoutBase.getImeInsets`, and the web has the VirtualKeyboard API's `keyboard-inset-*`
  variables. Wiring them up would let keyboard-aware layout move into CSS as well. Left out of
  this RFC to keep it to one concept, but the registry is built to take more variables.
- **`preferred-text-scale`.** Core already tracks an accessibility font scale as
  `Style.fontScaleInternal`. Exposing it as the spec's `<number>` variable looks close to free
  and would make `font-size: calc(16dip * env(preferred-text-scale))` work. Worth deciding
  whether it belongs in this RFC or a follow-up.
- **`min()`, `max()` and `clamp()`.** `clamp(0dip, env(safe-area-inset-bottom), 32dip)` is the
  natural way to cap an inset, and is what web authors write. Core only recognises `calc(`
  today. Adding the other math functions is independent of this RFC but is the first thing
  people will ask for after it lands.
- **Whether `overflowSafeArea` should eventually subsume `androidOverflowEdge`.** If it does,
  it needs the `dont-apply` and `*-dont-consume` concepts, which would roughly double its
  vocabulary. Deferred until there is usage data on which of those tokens are actually load-bearing.
- **Default for `overflowSafeArea` on `Page`.** `auto` is the safe default for a view, but pages
  are the thing that usually wants to go edge-to-edge. Whether `Page` should default
  differently, or whether that belongs to the theme, is open.
- **`GridLayout` drops the bottom and right insets on iOS.** `onLayout` adds only
  `insets.left` and `insets.top`, so `overflowSafeArea="none"` is invisible in a
  GridLayout while `StackLayout` honours all four. Reserving them in `onMeasure` was tried
  and reserves the inset a second time on top of what the page controller already did
  (measured: a 34dip inset became a 115dip gap), so the current behaviour may be
  deliberate compensation. Needs its own investigation.
- **Layouts that consume without declaring.** `ScrollView` and `StackLayout` inset
  themselves from `getSafeAreaInsets()`. Consumption tracking papers over this on iOS by
  reading UIKit's own propagation, but Android has no per-view equivalent, so the declared
  chain is all it has there. Making those layouts declare what they consumed would let both
  platforms use one mechanism.
- **Foldables.** `viewport-segment-*` is the spec's answer and Android reports hinge geometry
  through `WindowLayoutInfo`. Out of scope here, but the `env()` grammar implemented by this
  RFC includes index support specifically so that the array-like variables can be added without
  a grammar change.

