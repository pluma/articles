# A Primer on Node.js Dependency Types

If you've spent any time working with Node.js, you're probably well aware of the difference between dependencies and devDependencies. But if you want to develop reusable packages you might want to publish on NPM, it helps to know that there are a few more types of dependencies your package might want to use.

## Production vs Development

You're likely already familiar with `dependencies`, used to declare packages your app or package will need at runtime, and `devDependencies`, used to declare additional packages needed during development or testing.

One minor point of contention is where dependencies should go for frontend apps. People on one side of the argument will tell you that because `dependencies` is for production, all packages your frontend code needs to run should go in there, whereas `devDependencies` is reserved for test runners, build tools and so on. People on the other side will tell you that because your production built will already have all production dependencies included in the bundles you send to the browser, they too should go in `devDependencies`.

There is no universal right or wrong answer here but personally I prefer keeping my production dependencies in `dependencies` for consistency even if the production bundle will never actually include them because it already contains them in the optimized build. This makes it easier to extract parts of the codebase into separate packages later on as those packages will then need to define their production dependencies explicitly in order to behave as expected.

## Peer vs Optional vs Optional Peer

The use of `peerDependencies` has been somewhat controversial in the past because dependencies declared this way were initially not installed automatically. This behavior was changed in npm v7 but this change also led to some confusion and uncertainy, leading many developers to avoid it entirely and sometimes even still requiring use of the `--legacy-peer-deps` option to successfully install conflicting packages.

Peer dependencies are intended to be used for dependencies which your package needs in order to function but which you want all other dependencies installed alongside your package (its "peers") to use the same copy of. The typical example of this is a frontend library like React which is needed by every package that wants to provide React components but that will result in bugs if multiple versions of the library are installed (which typically results in errors or warnings when detected by the library).

On the other hand, `optionalDependencies` sometimes lead to confusion because contrary to their name, npm will still always try to install them. The distinction however is that unlike regular `dependencies`, npm will be okay with them failing to be installed and proceed to install your package anyway. This makes them useful for dependencies wrapping native extensions for platform specific optimizations that may not be supported by all architectures.

A less known concept is optional `peerDependencies`. Unlike `optionalDependencies`, there is no dedicated field for these but instead you need to use `peerDependenciesMeta` and indicate each dependency as `optional: true` individually. Optional peer dependencies will be shared like regular peer dependencies but npm will _not_ install them automatically.

While at first glance optional peer dependencies simply behave a lot like peer dependencies did in older versions of npm, they actually provide a very important use case for reusable packages. Specifically, they allow users to opt into additional functionality or optimizations by installing additional packages alongside your package. For example, you might want to support a range of different drivers without requiring every single driver to be installed and only check for the presence of the driver a user's application might actually want to use by passing explicit options to your package's code.

## Bundle dependencies

Also aliased as `bundledDependencies`, the behavior of `bundleDependencies` can be confusing and has changed slightly throughout the history of the npm registry and client. Because of its poor documentation, confusing behavior and because most of its potential use cases is better served by other solutions, it's best to consider `bundleDependencies` deprecated or at least avoid it unless you know precisely what it is doing and have considered all other alternatives.

According to the official documentation, `bundleDependencies` can be used to specify packages which should be included in the tarball generated when using `npm pack`. This is also the tarball that will be published to NPM when using `npm publish` and downloaded when npm attempts to install the package.

Historically this allowed npm to avoid fetching the bundled dependencies from the registry individually, which could speed up installation for some packages and to also provide a complete installation even when the registry becomes unavailable or when downloading the tarball in advance to allow for offline installation later before the npm client properly supported using the local cache to support offline installations directly.

(CC) 2025 Alan Plum. Some rights reserved. See [LICENSE.md](./LICENSE.md) for details.
