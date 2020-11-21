- Start Date: 2020-11-13
- Target Major Version: 7.x
- Reference Issues: https://github.com/NativeScript/rfcs/issues/1
- Implementation PR: https://github.com/NativeScript/NativeScript/pull/9038

# Summary

Improve usage and maintenance overhead of @nativescript/webpack

<!-- # Basic example

If the proposal involves a new or changed API, include a basic code example.
Omit this section if it's not applicable. -->

# Motivation

Webpack configs are currently handled by having separate config templates for different project flavors. These templates are copied in the project's root folder in an npm `postinstall` hook (unless a config already exists). The correct config is determined based on installed dependencies.

Making changes to the official webpack configs is difficult as the changes usually have to be repeated for each template. After an update, users need to manually update their configs, or alternatively delete their existing ones and install `@nativescript/webpack` again in order to re-generate the config. This is generally not an issue, but many times projects need to customize the configs, making the update process even more difficult.

Plugins that require modifications to the webpack config require the user to manually make the right changes to their webpack configs - these changes get lost if the config is deleted and re-generated with the `postinstall` hook.


# Detailed design

**For NativeScript Projects:**

A new and simpler webpack config file:
```js
const webpack = require('@nativescript/webpack')

module.exports = env => {
    // initialize @nativescript/webpack with the environment
    webpack.init(env)

    // Optional: specify base config - defaults to auto-discovery
    webpack.useConfig('vue')

    // Optional: apply changes to the internal config using webpack-chain
    webpack.chainWebpack((config, env) => {
        config.mode('production') // for example: force production mode
    })

    // Optional: merge options into the internal config
    webpack.mergeWebpack({
        something: true
    })
    // or pass a function that returns an object
    webpack.mergeWebpack((env) => {
        return {
            something: true
        }
    })

    // Optional: resolve a new instance of the chainable config
    // Note: this always returns a new instance of the config
    const config = webpack.resolveChainableConfig()

    // resolve a config object that webpack understands
    // Optionally pass in a chainable config 
    // (from `webpack.resolveChainableConfig()` for example)
    // to be used, rather than creating a new instance
    return webpack.resolveConfig()
}
```

**For 3rd party packages:**

Packages can publish a `nativescript.webpack.js` file in the root directory of the package.

```js
/**
 * This optionally provides typehints
 * this requires "@nativescript/webpack" to be a dependency (dev)
 * 
 * @param {typeof import("@nativescript/webpack")} webpack
 */
module.exports = (webpack) => {
    // same API as the user configs
    // for example make changes to the internal config with webpack-chain
    webpack.chainWebpack((config, env) => {
        // as an example - add a new rule for svg files
        config.module
            .rule('svg')
            .test(/\.svg$/)
            .use('svg-loader')
            .loader('svg-loader')
    })
}
```

When resolving the config, `@nativescript/webpack` will scan all dependencies defined in `package.json` for the `nativescript.webpack.js` file in the root directory of the package, and if found - apply it.

It's only scanning packages that are defined in `package.json` since `node_modules` could potentially have thousands of dependencies that we would otherwise be scanning for each build.

This has a drawback for packages that rely on transitive dependencies that provide a `nativescript.webpack.js` as those would not be discovered. The suggested workaround is for packages that rely on transitive packages, to inclue a `nativescript.webpack.js` of their own, and explicitly `require` the transitive configs:
```js
// package-a/nativescript.webpack.js
const packageB = require('package-b/nativescript.webpack.js')

module.exports = (webpack) => {
    // explicitly apply packageB's configuration
    packageB(webpack)
    // ...
}
```

```js
// package-b/nativescript.webpack.js

module.exports = (webpack) => {
    // ...
}
```

**Internally:**

All configuration to be managed with `webpack-chain` internally.

The bulk of the configuration to be managed in a `base` configuration, and only configure flavor specific additions in the flavor specific configurations: `angular`, `react`, `vue`, `svelte`, `typescript` (as flavor), `javascript` (as flavor).


# Drawbacks

Configurations are managed using [webpack-chain](https://github.com/neutrinojs/webpack-chain) which has a slight learning curve, however this only applies to maintainers.

# Alternatives

Continuning with the templated approach, with refactors to be able to share base configurations. 

Merging and managing pure object-based webpack configs is error-prone and messy.

# Adoption strategy

A new major version of `@nativescript/webpack` would prevent existing projects from accidentally breaking (most projects have it pinned to `^3.0.0` or `~3.0.0`).

To switch to the new version, developers would have to delete their `webpack.config.js` configurations (or their custom configs if not using the defaults).
To generate the new config - a binary is to be provided with the new package - developers would invoke the binary with `npx @nativescript/webpack init`.

BREAKING CHANGES:
 - `package.json` main should now use a relative path to the package.json instead of the app directory
   
   For example (given we have a `src` directory where our app is):
   
   `"main": "app.js"` becomes `"main": "src/app.js"`
   
   This simplifies things, and will allow ctrl/cmd + clicking on the filename in some editors.
 
 - `postinstall` scripts have been removed.
 
   The configuration will not need to change in the user projects between updates.
 
   For existing projects we will provide an easy upgrade path, through `ns migrate` and a binary in the package.
   
   For new projects `ns create` should create the config file by invoking a binary in the package. 

 - removed resolutions for short imports - use full imports instead.
 
   For example:
   ```
   import http from 'http'
   // becomes
   import { http } from '@nativescript/core'
   ```

# Unresolved questions

 - What should be the name for plugin-provided configurations? We are leaning towards `nativescript.webpack.js` in the root of the package.
