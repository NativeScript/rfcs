- Start Date: 2022-12-26
- Target Major Version: 9.x

# Summary

Historically NativeScript has always provided platform API types on the global scope, for example:

```
// available anywhere you type
NSString. 
java.
```

On the surface it sounds adverse to common JavaScript standards. In practice it's actually not as bad as it sounds mainly because NativeScript is it's own JavaScript runtime so it's rules our customizable and controllable. The problem often comes down to TypeScript types and the fact that you really don't need all the platform API types available in the global scope. NativeScript 8.2 helped this be confining the types and clarifying how you can include more if you want, see [this post](https://blog.nativescript.org/where-did-my-types-go/).

It's long been debated and even attacked as to whether platform API types should be on the global scope.

# Basic example

Progressive ideas around handling platform API types have emerged in 2022 among TSC discussions.

Here are the biggest three options in consideration.

## A. import from runtime packages

```
import { NSString } from '@nativescript/ios'

import { java } from '@nativescript/android'
```

The types could ship with the runtimes themselves either directly or via proxy to `@nativescript/types`.

## B. import from existing types packages explicitly

```
import { NSString } from '@nativescript/types-ios'

import { java } from '@nativescript/types-android'
```

## B. import specifiers

[See how Node is handling here](https://nodejs.org/api/fs.html#file-system), eg `... from 'node:fs'`

Prior Art: https://github.com/nodejs/node/issues/43413

```
// esm:
import { NSString } from 'native:Foundation';
// commonjs:
const NSString = require('native:Foundation').NSString;
```

For third party vendors:

```
// Objective C/Swift
import { LOTAnimationView } from 'native:Lottie';

// Java/Kotlin
import { LottieAnimationView } from 'native:com.airbnb.lottie';
```

# Motivation

This would improve developer experience and help the TSC optimize various compiler behaviors.

# Drawbacks

Why should we *not* do this? Please consider:

- breaking change
- community migration
- is it more clear or less clear

# Alternatives

What [was done in 8.2](https://blog.nativescript.org/where-did-my-types-go/). Pairing down of types to give the developer more control over which they need at anytime.

# Adoption strategy

This approach could be introduced with NativeScript 9.0 purely as an option. Could be a flag enabled in `nativescript.config` which would enable the bundler to be aware of one way or the other. It may not need a flag as it's possible you could use both approaches for awhile.

This would be a slow migration between version 9 to 12 likely.

TSC could introduce several migrations for plugin vendors and projects until eventually all that exists is the scoped types.

# Unresolved questions

Other approaches are likely possible.
