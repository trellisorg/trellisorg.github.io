---
layout: post
title: Converting a Legacy Codebase to Nx Incremental Builds
author: jay
categories: [ angular, nx, compiler ]
tags: [ angular, nx, compiler]
featured: true
image: assets/images/11.jpg
---

We recently converted part of our legacy codebase over to use
Nx's <a href="https://nx.dev/latest/angular/ci/incremental-builds">incremental compilation<a/>
and ran into some issues along the way, hopefully this checklist of things to look for and how to fix common errors
helps someone else.

For the purpose of this article we will assume we have the follow library (ours are Angular and Nest but this should be
able to be applied to any Nx lib).

This has yet to be converted to a buildable library.

```
libs/
--my-lib/
----src/
------index.ts
------test-setup.ts
------lib/
--------my-lib.module.ts
--------my-lib-routing.module.ts
--------my-lib.component.ts
--------my-lib.component.scss
--------my-lib.component.html
----tsconfig.json
----tsconfig.lib.json
----tsconfig.spec.json
```

### 1. Converting the library to be buildable

When you first go to build your app using a command like `nx build app1 --with-deps` (hint: use the `--parallel`
and `--maxParallel` flags to speed it up). You may run into an error that looks something like:

```bash
libs/my-lib/src/index.ts:1:15 - error TS6059: File '/path/to/repo/libs/my-lib/src/lib/my-lib.component.ts' is not under 'rootDir' 'libs/some-other-lib'. 'rootDir' is expected to contain all source files.

1 export * from './lib/my-lib.component';
```

There will be some variation in this, but the general structure is telling you that you have a buildable
lib `some-other-lib` that depends on a non-buildable lib
`my-lib`.

This is because you are trying to build one library that depends on another library that is not buildable. Buildable
libraries can only depend on other buildable libraries, not the other way around.

To be buildable a library needs the following this:

1. a `package.json`
2. a `build` target in your `angular.json` or `workspace.json`
3. if it is an Angular library then it also needs an `ng-package.json`
4. by default Nx will generate new buildable libs with a `tsconfig.lib.prod.json` to be used instead of
   `tsconfig.lib.json` with `--prod`, not sure if this is completely required or not to be honest.

We built a small script to help us with converting libraries, but it is not quite perfect and has some gotchas.

You can find the Nx plugin here: https://www.npmjs.com/package/@trellisorg/make-buildable

once installed run `nx g @trellisorg/make-buildable:migrate` and enter your libraries name, in this case `my-lib`
it will ask you what framework, `node | nest | angular`, we don't have any `react` code so that was never implemented.
When asked about configurations you can leave it blank (we will fix this later).

This will create/update each of the things in the list above.

Our folder structure should now look like:

```
libs/
--my-lib/
----src/
------index.ts
------test-setup.ts
------lib/
--------my-lib.module.ts
--------my-lib-routing.module.ts
--------my-lib.component.ts
--------my-lib.component.scss
--------my-lib.component.html
----package.json
----ng-package.json <-- only for angular libraries
----tsconfig.json
----tsconfig.lib.json
----tsconfig.lib.prod.json
----tsconfig.spec.json
```

Things that need to be fixed after:

1. If your library has any dashes in the name like `my-lib` compared to if you had nested in folders like `my/lib` then
   in the `package.json` update the `name` field to be `<scope>/my-lib` instead of `<scope>/my/lib` (where scope is your
   repos scope), the script will not update the name using the correct dashes, it should match the folder path within
   libs. As a contrived example if your libs path was
   `libs/my-lib/nested/nested-two/some-other-folder/actual-lib` then the script would set the name field to something
   like
   `@<scope>/my/lib/nested/nested/two/some/other/folder/actual/lib` where it should
   be `@<scope>/my-lib/nested/nested-two/some-other-folder/actual-lib`. If you are ever unsure of what the `name` should
   be then it will typically be `@<scope>/<root>` where `root` is the value from the `root` property in your
   libs `(angular|workspace).json` configuration.

2. The same is true for the `ng-package.json`. Except we are fixing the `dest` field instead of `name` (This is only
   relevant to angular libraries). In the `package.json` it was `@<scope>/<root>` in `ng-package.json` it
   is `(../)*dist/<root>` where `(../)*` points to the root of the repo. Relative part of the path shouldn't need to be
   changed, just what is after `dist/`.

If the `name` or `dest` paths are wrong then you will get an errors like:

```bash
Some of the project app1's dependencies have not been built yet. Please build these libraries before:
- my-lib

Try: nx run app1:build --with-deps

———————————————————————————————————————————————

>  NX   ERROR  Running target "app1:build" failed

  Failed tasks:

  - app1:build
```

What is happening here is that the library is being built, but it is being built into the wrong folder so when `app1`
goes be compiled by Nx, it cannot find `my-lib`'s artifacts where it is expecting to find them.

### 2. Class is not visible error

Once you start converting libraries to be buildable you may run into this if you are not exporting all the necessary
pieces from your `index.ts` file:

```bash
ERROR: libs/my-lib/src/lib/my-lib.component.ts:8:14 - error NG3001: Unsupported private class
 MyLibComponent. This class is visible to consumers via MyLibModule -> MyLibComponent, but is not exported from the
 top-level library entrypoint.

8 export class MyLibComponent implements OnInit {
               ~~~~~~~~~~~~~~~~~~

libs/my-lib/src/lib/my-lib.component.ts:8:14 - error NG3001: Unsupported private class
MyLibComponent. This class is visible to consumers via MyLibModule -> MyLibComponent, but is not exported from the
top-level library entrypoint.

8 export class MyLibComponent implements OnInit {
               ~~~~~~~~~~~~~~~~~~
```

What this is telling you is that in your `MyLibModule` you have something like this:

```typescript
@NgModule({
    declarations: [MyLibComponent],
    imports: [CommonModule, MyLibRoutingModule],
    exports: [MyLibComponent],
})
export class NotFoundModule {
}
```

Notice how you are have `MyLibComponent` in the `exports` array? Well you are telling Angular that that component is
allowed to be used outside this module. Which means it needs to be exported in your `index.ts`.

`index.ts` before:

```typescript
export * from './lib/my-lib.module';
```

`index.ts` after:

```typescript
export * from './lib/my-lib.component';
export * from './lib/my-lib.module';
```

### Building multiple chunks error when using dynamic imports

When building a library that uses dynamic imports, like in the case of Angular Routing modules, you may run into this:

```bash
ERROR: When building multiple chunks, the "output.dir" option must be used, not "output.file". To inline dynamic imports, set the "inlineDynamicImports" option.
When building multiple chunks, the "output.dir" option must be used, not "output.file". To inline dynamic imports, set the "inlineDynamicImports" option.
```

This will only happen if the path within the `import(<path>)` is a relative path to within the same library. Having an
alias in the import will not affect it as those are inherently not a part of the library. This one is very similar to
the error caused by not exporting the component in the `index.ts` that is exported in the `exports` array in `@NgModule`
.

What this is saying is: "you are dynamically providing something within this library but are not exporting that thing"

Imagine we not have this folder structure

```
libs/
--my-lib/
----src/
------index.ts
------test-setup.ts
------lib/
--------child/
----------child.component.ts
----------child.component.scss
----------child.component.html
----------child.module.ts
--------my-lib.module.ts
--------my-lib-routing.module.ts
--------my-lib.component.ts
--------my-lib.component.scss
--------my-lib.component.html
----package.json
----ng-package.json <-- only for angular libraries
----tsconfig.json
----tsconfig.lib.json
----tsconfig.lib.prod.json
----tsconfig.spec.json
```

And in your `MyLibRoutingModule` you had the following route configuration

```typescript
const routes: Routes = [
    {
        path: '',
        component: MyLibComponent,
        children: [
            {
                path: '',
                loadChildren: () => import('./child/child.module.ts').then((m) => m.ChildModule)
            }
        ]
    }
]
```

then in your libraries `index.ts` file you will need to export both `ChildModule` and `ChildComponent` like so:

```typescript
export * from './lib/my-lib.component';
export * from './lib/my-lib.module';
export * from './lib/child/child.module';
export * from './lib/child/child.component';
```

### Compilation errors after all dependencies were built

These will either show an error with your code or an error showing an external dependency. Both are the same fix.

```bash
Error: ./dist/libs/my-lib/fesm2015/my-lib.js 1006:103-111
"export 'default' (imported as 'my_lib_1') was not found in '@<scope>/my-lib'
    at HarmonyImportSpecifierDependency._getErrors (/path/to/repo/node_modules/@angular-devkit/build-angular/node_modules/webpack/lib/dependencies/HarmonyImportSpecifierDependency.js:109:11)
    at HarmonyImportSpecifierDependency.getErrors (/path/to/repo/node_modules/@angular-devkit/build-angular/node_modules/webpack/lib/dependencies/HarmonyImportSpecifierDependency.js:68:16)
    at Compilation.reportDependencyErrorsAndWarnings (/path/to/repo/node_modules/@angular-devkit/build-angular/node_modules/webpack/lib/Compilation.js:1463:22)
    at /path/to/repo/node_modules/@angular-devkit/build-angular/node_modules/webpack/lib/Compilation.js:1258:10
    at _next0 (eval at create (/path/to/repo/node_modules/tapable/lib/HookCodeFactory.js:33:10), <anonymous>:30:1)
    at eval (eval at create (/path/to/repo/node_modules/tapable/lib/HookCodeFactory.js:33:10), <anonymous>:43:1)
    at runMicrotasks (<anonymous>)
    at processTicksAndRejections (internal/process/task_queues.js:97:5)
 @ ./apps/app1/src/app/app-routing.module.ts
 @ ./apps/app1/src/app/app.module.ts
 @ ./apps/app1/src/main.ts
 @ multi ./apps/app1/src/main.ts
```

This is telling you there is a compilation error, it took be a little to track down but by comparing
the `tsconfig.*.json` files with
https://github.com/nrwl/nx-incremental-large-repo I was able to track down that the cause was that some
libraries `tsconfig.lib.json` files had `module` set to `commonjs` rather than inheriting `"module": "esnext"` from
the `tsconfig.base.json` in the root of the project. Just remove the explicit `module` declaration in
your `tsconfig.lib.json` in the library to fix this.
