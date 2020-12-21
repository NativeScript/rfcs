- Start Date: 2020-12-21
- Target Major Version: 8.x
- Reference Issues: 
- Implementation PR: 

# Summary

A sore spot for quite a long time is trying to get the font-family combination correct for iOS/Android.
For a custom font such as barlow-bold.ttf the css needs to be:

```
font-family: Barlow, barlow-bold;
```
This feature would list all the available custom fonts with their correct form to use it in css.

# Basic example

Add a convenient cli command which prints out exactly what css is needed for any given font:
```
$ ns fonts

Found Custom Fonts:
-------------------
barlow-bold.ttf    ->    font-family: Barlow, barlow-bold;
fontawesome.ttf    ->    font-family: FontAwesome, fontawesome;
```

# Motivation

Getting this correct is sometimes incredibly frustrating for custom fonts.

This feature would save ton of time debugging why custom fonts aren't working.

# Detailed design

Add a new CLI command `ns fonts` that will find a `fonts` folder in all the possible locations (`<projectRoot>/fonts`, `<projectRoot>/<appPath>/fonts` - where `<projectRoot>` is the root of the project, and `<appPath>` is the value for `appPath` in the `nativescript.config.ts` - falling back to the default value of `app` & `src`).

For each font we get the metadata with a package like [font-finder](https://www.npmjs.com/package/font-finder) and then output a css string with the correct values that can be copy-pasted right into any css file from the output of the `ns fonts` command (no manualy modifications required).
