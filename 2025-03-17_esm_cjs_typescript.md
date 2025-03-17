# Supporting ESM and CommonJS in TypeScript packages

The future of Node.js module systems is here and it's called ECMAScript Modules (ESM). It doesn't even feel right to call it the future anymore given that ESM support was first added to Node.js in v8.5 all the way back in 2017 and became non-experimental in v12.20 in 2020. But even eight years after its initial availability many developers are still loading modules the conventional Node.js way (CommonJS or CJS) and many library maintainers find themselves having the choice between holding out and being seen as old-fashioned or fully committing to ESM and being branded ruthless activists.

Trying to support both ESM and CJS can often seem like a daunting task and many tutorials written in the past introduce various tooling or omit important niceties like supporting multiple export paths. Luckily for projects already using TypeScript, the solution can be pretty simple even if finding that solution can require hours and days of experimentation and trial and error.

## But first a history lesson

What's so common about CommonJS anyway? Record scratch. Freeze frame. You're probably wondering how we got into this situation. Before diving into the solution, let's quickly recap the events of the past two decades in case you weren't around or weren't interested in the nitty gritty at the time.

Although nowadays the server-side JavaScript ecosystem has largely consolidated around Node.js, Node.js was not the first server-side JavaScript environment by far. Prior to the arrival of Node.js in 2009 and its meteoric rise shortly afterwards, server-side JavaScript was an extremely fragmented space with many different runtimes often largely incompatible with each other.

Development of the JavaScript language and its standard library had largely been stagnant after Microsoft mostly stopped collaborating and disagreements between Macromedia (later acquired by Adobe), Opera, Google and Mozilla led to the collapse of ECMAScript 4, meaning that for the time being, the most recent JavaScript spec dated back to 1999.

This environment led to collaboration outside the ECMA standards body and the TC39 steering committee forming a group first named SystemJS and later CommonJS to create common standards that would allow better interoperability between the different JavaScript environments. The most noteworthy results of this work were the specifications for Packages, Modules and Promises.

Although Node's traditional module system is generally referred to as "CommonJS", it was actually not fully compliant with the CommonJS specifications but due to its dominance in the server-side JavaScript ecosystem, the original CommonJS specifications have largely become irrelevant today.

The failure of the ECMAScript 4 process meanwhile was finally resolved in 2009, leading to the release of ECMAScript 5 and the ECMAScript Harmony project, reevaluating many of the language features originally planned for ECMAScript 4 but following a well-defined process from proposal to specification for each individual feature and introducing a yearly snapshot release process with the first release being ECMAScript 6 in 2015.

ECMAScript 6 included promises (largely equivalent to the original CommonJS spec) and a new language syntax for modules but left a lot of the implementation details of the module system unspecified. Developers would often use tools like 6to5, esnext, traceur or later Babel to translate ES6 code to code that could be run in Node.js or would be understood by bundlers for frontend code, typically expecting Node.js-style CommonJS modules. Nevertheless ES6 modules would provide the basis for future development and eventual native support in browser runtimes and Node.js.

TypeScript of course also predates the finalization of ECMAScript Modules (ESM) and thus has its own historical baggage when it comes to module loading. But with TypeScript closely following the evolution of the underlying JavaScript language, it is becoming increasingly possible to ignore those peculiarities as long as you only target ESM environments.

## Back to the future

With all that history out of the way, let's dive into the solution.

Let's start with a minimal `package.json` file for the package we want to end up publishing. We'll also explicitly declare our package to be of `type: module` so Node.js will interpret all JavaScript files as ESM by default. This is also checked by the TypeScript compiler when deciding whether to emit ESM code, so omitting it can cause some headaches down the road:

```json
{
  "type": "module",
  "name": "@example/hello-world",
  "version": "1.0.0",
  "description": "A package that says hello",
  "license": "MIT",
  "files": ["dist", "README.md", "LICENSE"],
  "scripts": {}
}
```

Our TypeScript code will go in the `src` directory and we'll compile it to the `dist` directory. If you want to publish your source code, you can add `src` to the `files` list but we'll already publish type definitions so this is not strictly necessary.

We'll be using the latest version of TypeScript (v5.8 at the time of this writing) with the config preset for the latest Node.js LTS release (v22 at the time of this writing). Make sure to check the [Node.js release schedule](https://nodejs.org/en/about/releases/) if you're ever unsure about which version to use - for most packages the active LTS version is a good choice but in general you want to avoid anything that's near end-of-life or has already entered maintenance mode.

If you're going to use node-specific APIs, you'll also need to install `@types/node` with the apropriate version (e.g. `@types/node@22` for Node.js v22).

```sh
npm install --save-dev typescript @tsconfig/node22
```

Thanks to the preset, we can keep our `tsconfig.json` base file fairly terse:

```json
{
  "extends": "@tsconfig/node22/tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true
  }
}
```

You can omit `declarationMap` but it's useful if you do plan to publish your source code because it allows tools to map code points between the original source and the type declarations.

Next we can extend this file a little bit to tell TypeScript how to generate our ESM output:

```json
{
  "extends": "./tsconfig.json",
  "include": ["src"],
  "compilerOptions": {
    "outDir": "./dist/esm/",
    "rootDir": "./src/"
  }
}
```

We want to output all ESM code to the `dist/esm` directory and use the `src` directory as the only place TypeScript should look for source files. The base config already tells it to also emit type declarations alongside the generated code.

Let's tie this up by adding a `build:esm` script to our `package.json` file:

```json
"scripts": {
  "build": "npm run build:esm",
  "build:esm": "tsc --project ./tsconfig.esm.json"
}
```

## Adding a CommonJS build

For CommonJS things get slightly trickier because TypeScript decides whether it should emit CJS or ESM code for ambiguous filenames (i.e. `index.ts` instead of `index.mts` or `index.cts`) based on the nearest `package.json` file containing a `type` field.

There are three approaches to solving this:

1. Manipulate the `package.json` file before and after `tsc` runs. This requires modifying the file while a script defined in the file is executing and making sure to restore it to its previous state even if the script fails. This way lies madness.
2. Copy the source code to a temporary directory with a modified `package.json` file and compile the code from there. This is a bit of a hack but it works.
3. Use a helper directory containing a minimal `package.json` file declaring that directory's contents as CJS and symlink the source code into this directory. This is the approach we'll take.

Setting up the helper directory is fairly straightforward, again starting in the module root:

```sh
mkdir cjs
echo '{"type": "commonjs"}' > cjs/package.json
ln -s ../src cjs/src
```

Keep in mind that the first argument to the symlink is a path relative to the path at which the link is created.

We can now create a `tsconfig.cjs.json` file that extends the base config for CJS output:

```json
{
  "extends": "./tsconfig.json",
  "include": ["cjs/src"],
  "compilerOptions": {
    "outDir": "./dist/cjs/",
    "rootDir": "./cjs/src"
  }
}
```

Our build script is similar to the ESM one but we also need to copy the minimal `package.json` file to allow Node.js to understand the output as CJS:

```json
"scripts": {
  "build": "npm run build:esm && npm run build:cjs",
  "build:esm": "tsc --project ./tsconfig.esm.json",
  "build:cjs": "tsc --project ./tsconfig.cjs.json && cp cjs/package.json dist/cjs/"
}
```

Because we want to publish the package, we also need to add a few more fields to the `package.json` file in the module root:

```json
"main": "dist/cjs/index.js",
"module": "dist/esm/index.js",
"exports": {
  ".": {
    "require": "./dist/cjs/index.js",
    "import": "./dist/esm/index.js"
  },
  "./*": {
    "require": "./dist/cjs/*.js",
    "import": "./dist/esm/*.js"
  }
}
```

The `main` and `module` fields have been superseded by the `exports` field in more recent versions of Node.js but it's still good practice to keep them around even if it leads to a little bit of duplication. Instead of defining each export individually, we only need to use a default path for the index file and a wildcard path matching everything else. Note that the asterisk in this case also matches subdirectory paths, not just files at the same level.

Because we're using an approach that does not require modifying the `package.json` file, we can also add a tool like `npm-run-all` to run both builds in parallel:

```sh
npm install --save-dev npm-run-all
```

This avoids having to wait for one build to finish before the other can start:

```json
"scripts": {
  "build": "npm-run-all --parallel build:*",
  "build:esm": "tsc --project ./tsconfig.esm.json",
  "build:cjs": "tsc --project ./tsconfig.cjs.json && cp cjs/package.json dist/cjs/"
}
```

## Why you shouldn't do this

In my experience this is a good solution for the problem of having to provide builds for both ESM and CJS in the same published package. **However, an important question to ask yourself is whether that's really the problem you should be solving.**

By providing two builds in one package, you're shipping two versions of the same code to every consumer of your package. This means you can run into the same problems that can also occur when different dependencies use different versions of the same package, except that it's no longer possible to easily detect this issue by looking at the installed packages as the cause will be one package consuming ESM while the other consumes CJS. Additionally, you're also doubling the amount of disk space (and bandwidth for installation) used by your package.

It's possible to import CommonJS modules in ESM code using the native import syntax. Node.js will try its best to identify named exports and even has support for accessing the `module` object of the imported CommonJS module:

```js
import * as example from "@example/commonjs-example";
// "example" is now a Module object for the CommonJS module
```

For the inverse, CommonJS modules can use the `import` function to asynchronously load an ESM module:

```js
const example = await import("@example/esm-example");
```

Starting with Node.js v22, it is also possible to [`require` an ESM module](https://nodejs.org/api/modules.html#loading-ecmascript-modules-using-require) as long as the module you wish to import does not use top-level `await`. While v22 requires using the `--experimental-require-module` flag to opt into this feature, it is already enabled by default in the current version of Node.js v23 and well on track to mature from release candidate to stable just in time for the Node.js v24 LTS release:

```js
const example = require("@example/esm-example");
// "example" is now a Module object for the ESM module
```

Unless you're already maintaining a package that was written for CommonJS and now find yourself wishing to make the move to ESM while continuing to support older codebases, I would strongly recommend going all in on ESM.

With the release of Node.js v24 being just around the corner and it being scheduled to become the next LTS release by the end of this year, compatibility between ESM and CJS in both directions will become a solved problem for most projects.

(CC) 2025 Alan Plum. Some rights reserved. See [LICENSE.md](./LICENSE.md) for details.
