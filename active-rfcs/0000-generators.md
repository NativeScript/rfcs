- Start Date: 2023-12-30
- Target Major Version: 9.0

# Summary

Right now, there are no built-in generators for building apps. Everything from component creation, to generating types and **auto configuring the project for their usage**, to creating widgets (iOS and Android boilerplate setup like the folder where it goes and supporting files like extension.json).

Generators help speed up development but can also help encourage best practices from naming things to even initial file setup.

@nativescript/nx and xplat come with generators to help project setup/creation/development however they come with other aspects that make them not as easy to pick up and just use right after setting up a NativeScript development machine.

# Basic example

Swift/Kotlin generators for when wanting to drop in either and use with {N} apps, eg:
```bash
ns g swift MyFile

ns g kotlin MyFile
```

This would do several manual steps in one go:
- create a file with all the structure in place and into right place for {N} app to use right away 
- prompts could occur for those which need it, for example a Kotlin file may ask what package scope it should be in (eg: `org.nativescript.custom-kotlin`)
- could also work hand/hand to auto create TypeScript declarations based off any changes in those files and auto update project ref's so TS intellisense in the project is always up to date

Integration generators, eg:
```bash
// auto integrate flutter screen  
// sets up the podfile/app.gradle to use a folder of Dart code
// registers the flutter screen, creates boilerplate with the vm entry already named and ready for use
ns g flutter MyScreen 

// auto integrate RN modules
// install open-native, configure bundler for all named modules
ns g react-native module,module2,module3

// auto integrate Ionic Portal
// creating the build settings for it too
ns g capacitor MyPortal

// component generators
// would work for all flavors
// create start boilerplate component ready to use
// default to auto detect flavor in  use by introspecting project
ns g component MyComponent --flavor=solid

// create web deploy target
// add a web project alongside {N} auto setup to share code with the {N} project - create with any flavor using preferred flavor project creators (prompting to drop in workspace if needed)
ns g web MySite --flavor=angular
```

# Motivation

Improve developer experience and speed up project development.

# Drawbacks

Depending on implementation direction chosen, may introduce some maintainance. However there are steps which could be taken to minimize the need for maintainence like keeping the generation to be as minimal as possible reducing surface area of potential external environment changes (like if Solid syntax changed somewhere in a component generated).

The generators could become plug/play as well by other communities to allow easier self sustaining generation by interested communities to allow them to self govern what's generated.

# Adoption strategy

Just documentation.

# Unresolved questions

Whether various tools out there like builder.io Mitosis could/should be used for component interop with the generator.
