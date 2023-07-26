# Add external plug-ins to Tanzu Developer Portal

This topic tells you how to add external plug-ins to Tanzu Developer Portal
(formerly Tanzu Application Platform GUI).

## <a id="prereqs"></a> External plug-in prerequisite

As mentioned in [Build your Customized Tanzu Developer Portal with Configurator](building.hbs.md#prereqs),
to use any type of external plug-in, ensure that it is in an npm registry. This registry can be your
own private registry or a plug-in registry if you intend to use a third-party or community plug-in.
# How to Add a plugin to your Tanzu Developer Portal

In this section, we will walk you through the steps required to add a custom or community Backstage plugin to your Tanzu Developer Portal. As an example, we are using Backstage's TechInsights plugin.

The steps include:
1. Extracting the contents of the configurator image
1. Running a local verdaccio server for inner loop development
1. Creating your plugin wrapper
1. Adding your plugin wrapper to your portal configuration file
1. Issuing your portal configuration file to your Tanzu Application Platform cluster
> **Important** Tanzu Application Platform plug-ins cannot be removed from customized portals.
> However, if you decide you want to hide them, you can use the
> [runtime configuration](concepts.hbs.md#runtime) options in your `tap-values.yaml` file.

## Extracting the contents of the configurator image

Find the location of the configurator image by querying your TAP cluster

```shell
$ kubectl -n tap-install get packages tpb.tanzu.vmware.com.0.1.2 -o jsonpath={.spec.template.spec.fetch[0].imgpkgBundle.image}
```

```
REGISTRY/REPOSITORY@sha256:458d9727b6f77ea2fadd2ee597885419e555e7b74057bd654f7d4eee8f006641
```

This is a carvel bundle containing the configurator image, in order to get the
address of the configurator image you must download the carvel bundle and look at the
image lock file within.

```shell
$ imgpkg pull -b REGISTRY/REPOSITORY@sha256:458d9727b6f77ea2fadd2ee597885419e555e7b74057bd654f7d4eee8f006641 -o bundle
$ cat bundle/.imgpkg/images.yml
```

```
---
apiVersion: imgpkg.carvel.dev/v1alpha1
images:
- annotations:
    kbld.carvel.dev/id: harbor-repo.vmware.com/esback/tdp-configurator@sha256:0b03767e0526803c055d080a7d378df9adaeaa7cff0ff5af23a2debe668f20e3
  image: REGISTRY/REPOSITORY@sha256:0b03767e0526803c055d080a7d378df9adaeaa7cff0ff5af23a2debe668f20e3
kind: ImagesLock
```

We want to download the image specified by the `image` key.

```shell
$ imgpkg pull -i REGISTRY/REPOSITORY@sha256:0b03767e0526803c055d080a7d378df9adaeaa7cff0ff5af23a2debe668f20e3 -o configurator
```

Will output the filesystem of the configurator image to a directory called `configurator` on your local machine.

## Running a local verdaccio server for inner loop development

Using your extracted configurator from the previous step you can run a
[verdaccio](https://verdaccio.org/) server on your machine. This will enable you
to depend on typescript libraries that the Tanzu Developer Portal has developed
and are necessary for adding plugins to your Tanzu Developer Portal instance.

note: as of writing this doc a workaround is needed to adjust the parameters
of the verdaccio `offline_config.yaml`. In order to fetch recently published
packages you can adjust the `maxage` parameter under the `npmjs` `uplinks`
section

```diff
--- old_offline_config.yaml	2023-07-21 14:14:16.182418162 -0400
+++ offline_config.yaml	2023-07-21 14:12:07.660945189 -0400
@@ -3,7 +3,7 @@
 uplinks:
   npmjs:
     url: https://registry.npmjs.org/
-    maxage: 2y
+    maxage: 2m
     cache: true
 packages:
   '@tpb/*':
```

This will enable you to fetch recent metadata and versions from npmjs.com which
will prevent errors in future steps; such as when installing lerna.

```shell
$ cd configurator/builder/registry/
$ ./start_offline.sh
```

This has started a verdaccio server using the utility `forever` which runs
background processes and periodically monitors them.

If at any point you wish to deactivate the running verdaccio server you can run
```shell
$ ./node_modules/forever/bin/forever stop node_modules/verdaccio/bin/verdaccio
```

If you prefer to see the log lines verdaccio generates then you can run it
directly without forever.

```shell
$ cd configurator/builder/registry/
$ ./node_modules/verdaccio/bin/verdaccio --config offline_config.yaml
```

## Creating your plugin wrappers

With your verdaccio server running, you should have access to the requisite
libraries of the Tanzu Developer Portal. The only requirement is
that you will have to tell your local machine to fetch its dependencies through
verdaccio.

We are going to use Backstage's TechInsights plugin as an example. This plugin is broken down into the frontend and backend, so we will be creating two plugin wrappers: one for
[`tech-insights`](https://github.com/backstage/backstage/tree/master/plugins/tech-insights)
and one for
[`tech-insights-backend`](https://github.com/backstage/backstage/tree/master/plugins/tech-insights-backend).
As such we will be creating a monorepo that will contain all of our plugin
wrappers. We will be using some backstage tooling in order to manage our
monorepo. Namely `@backstage/create-app` and the `backstage-cli`. The backstage
tooling gives great convenience functions for managing multiple packages however
we will not be developing a classical backstage app. We will also be using
[`yarn` version 1](https://classic.yarnpkg.com/lang/en/) for
javascript/typescript package management.

<!-- TODO: what do we want to call "plugin wrappers" in the official docs -->
Start by creating a directory for your plugin wrappers. The cli prompts you for
a name for your project. In this example we'll be using `plugin-wrappers`.

```shell
$ npx @backstage/create-app@latest --skip-install
```

```
? Enter a name for the app [required] plugin-wrappers

Creating the app...

 Checking if the directory is available:
  checking      plugin-wrappers ✔

 Creating a temporary app directory:
  creating      temporary directory ✔

 Preparing files:
  templating    .eslintrc.js.hbs ✔
  copying       .dockerignore ✔
  templating    .gitignore.hbs ✔
  copying       .prettierignore ✔
  copying       README.md ✔
  copying       app-config.local.yaml ✔
  copying       app-config.production.yaml ✔
  templating    app-config.yaml.hbs ✔
  templating    backstage.json.hbs ✔
  templating    catalog-info.yaml.hbs ✔
  copying       lerna.json ✔
  templating    package.json.hbs ✔
  copying       tsconfig.json ✔
  copying       yarn.lock ✔
  copying       README.md ✔
  copying       org.yaml ✔
  copying       entities.yaml ✔
  copying       template.yaml ✔
  copying       catalog-info.yaml ✔
  copying       index.js ✔
  copying       package.json ✔
  copying       README.md ✔
  templating    .eslintrc.js.hbs ✔
  copying       Dockerfile ✔
  copying       README.md ✔
  templating    package.json.hbs ✔
  copying       index.test.ts ✔
  copying       index.ts ✔
  copying       types.ts ✔
  copying       app.ts ✔
  copying       auth.ts ✔
  copying       catalog.ts ✔
  copying       proxy.ts ✔
  copying       scaffolder.ts ✔
  templating    search.ts.hbs ✔
  copying       techdocs.ts ✔
  copying       .eslintignore ✔
  templating    .eslintrc.js.hbs ✔
  copying       cypress.json ✔
  templating    package.json.hbs ✔
  copying       android-chrome-192x192.png ✔
  copying       apple-touch-icon.png ✔
  copying       favicon-16x16.png ✔
  copying       favicon-32x32.png ✔
  copying       favicon.ico ✔
  copying       index.html ✔
  copying       manifest.json ✔
  copying       robots.txt ✔
  copying       safari-pinned-tab.svg ✔
  copying       .eslintrc.json ✔
  copying       app.js ✔
  copying       App.test.tsx ✔
  copying       App.tsx ✔
  copying       apis.ts ✔
  copying       index.tsx ✔
  copying       setupTests.ts ✔
  copying       LogoFull.tsx ✔
  copying       LogoIcon.tsx ✔
  copying       Root.tsx ✔
  copying       index.ts ✔
  copying       SearchPage.tsx ✔
  copying       EntityPage.tsx ✔

 Moving to final location:
  moving        plugin-wrappers ✔
  init          git repository ◜
🥇  Successfully created plugin-wrappers


 All set! Now you might want to:
  Run the app: cd plugin-wrappers && yarn dev
  Set up the software catalog: https://backstage.io/docs/features/software-catalog/configuration
  Add authentication: https://backstage.io/docs/auth/
```

This command scaffolds a backstage project structure.

We'll start by putting into place our `.yarnrc` which will tell yarn to point to
our verdaccio server.

```shell
$ cd plugin-wrappers
$ echo 'registry "http://localhost:4873"' > .yarnrc
```

Before installing our dependencies we will remove the packages directory, which
contains a scaffolded backstage `app` and `backend`. This is because we are only
making a plugin repository and we don't need to download the dependencies of
`app` and `backend`.

```shell
$ rm -rf packages/
```

And you need to remove the packages directory from the yarn workspaces.
```diff
diff --git a/package.json b/package.json
index 00d64c9..77f38f3 100644
--- a/package.json
+++ b/package.json
@@ -24,7 +24,6 @@
   },
   "workspaces": {
     "packages": [
-      "packages/*",
       "plugins/*"
     ]
   },
```

```shell
$ yarn install
```

Which will install the backstage cli and a few other dependencies.

### Tech Insights frontend wrapper

Next you can use the backstage cli to create your plugin wrapper for `tech-insights`
frontend. For this example, we'll be using the command

```shell
$ yarn backstage-cli new
```

When asked "What do you want to create?" select "plugin - A new frontend plugin".

When asked for "... the ID of the plugin [required]" enter "tech-insights-wrapper"

Which should yield output similar to

```
Creating frontend plugin backstage-plugin-tech-insights-wrapper

 Checking Prerequisites:
  availability  plugins/tech-insights-wrapper ✔
  creating      temp dir ✔

 Executing Template:
  copying       .eslintrc.js ✔
  templating    README.md.hbs ✔
  templating    package.json.hbs ✔
  templating    index.tsx.hbs ✔
  templating    index.ts.hbs ✔
  templating    plugin.test.ts.hbs ✔
  templating    plugin.ts.hbs ✔
  templating    routes.ts.hbs ✔
  copying       setupTests.ts ✔
  templating    ExampleFetchComponent.test.tsx.hbs ✔
  templating    ExampleFetchComponent.tsx.hbs ✔
  copying       index.ts ✔
  templating    ExampleComponent.test.tsx.hbs ✔
  templating    ExampleComponent.tsx.hbs ✔
  copying       index.ts ✔

 Installing:
  moving        plugins/tech-insights-wrapper ✔
  executing     yarn install ✔
  executing     yarn lint --fix ✔

🎉  Successfully created plugin

Done in 45.38s.
```

Which creates a directory `./plugins/tech-insights-wrapper` where our source code will reside.

```shell
$ cd plugins/tech-insights-wrapper
```

Because we will be publishing this plugin to an NPM compliant registry, we'll
update the name of the package.

```diff
diff --git a/plugins/tech-insights-wrapper/package.json b/plugins/tech-insights-wrapper/package.json
index fa7c7e0..8544cd7 100644
--- a/plugins/tech-insights-wrapper/package.json
+++ b/plugins/tech-insights-wrapper/package.json
@@ -1,5 +1,5 @@
 {
-  "name": "backstage-plugin-tech-insights-wrapper",
+  "name": "@mycompany/tech-insights-wrapper",
   "version": "0.1.0",
   "main": "src/index.ts",
   "types": "src/index.ts",
```

We also want to update the list of dependencies our package uses, to be the ones
that our source will actually use. Here's the final state of the file since the
versions you generate may not match the example

```json
{
  "name": "@mycompany/tech-insights-wrapper",
  "version": "0.1.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "license": "Apache-2.0",
  "publishConfig": {
    "access": "public",
    "main": "dist/index.esm.js",
    "types": "dist/index.d.ts"
  },
  "backstage": {
    "role": "frontend-plugin"
  },
  "scripts": {
    "start": "backstage-cli package start",
    "build": "backstage-cli package build",
    "lint": "backstage-cli package lint",
    "test": "backstage-cli package test",
    "clean": "backstage-cli package clean",
    "prepack": "backstage-cli package prepack",
    "postpack": "backstage-cli package postpack"
  },
  "dependencies": {
    "@backstage/plugin-catalog": "1.11.2",
    "@backstage/plugin-tech-insights": "0.3.11",
    "@tpb/core-common": "1.6.0-release-1.6.x.1",
    "@tpb/core-frontend": "1.6.0-release-1.6.x.1",
    "@tpb/plugin-catalog": "1.6.0-release-1.6.x.1"
  },
  "peerDependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2",
    "react-router": "6.0.0-beta.0",
    "react-router-dom": "6.0.0-beta.0"
  },
  "devDependencies": {
    "@backstage/cli": "^0.22.6",
    "@types/react": "^16.14.0",
    "@types/react-dom": "^16.9.16",
    "eslint": "^8.16.0",
    "typescript": "~4.6.4"
  },
  "files": [
    "dist"
  ]
}
```

For completeness here are the differences

```diff
diff --git a/plugins/tech-insights-wrapper/package.json b/plugins/tech-insights-wrapper/package.json
index fa7c7e0..e744978 100644
--- a/plugins/tech-insights-wrapper/package.json
+++ b/plugins/tech-insights-wrapper/package.json
@@ -1,10 +1,9 @@
 {
-  "name": "backstage-plugin-tech-insights-wrapper",
+  "name": "@mycompany/tech-insights-wrapper",
   "version": "0.1.0",
   "main": "src/index.ts",
   "types": "src/index.ts",
   "license": "Apache-2.0",
-  "private": true,
   "publishConfig": {
     "access": "public",
     "main": "dist/index.esm.js",
@@ -23,27 +22,24 @@
     "postpack": "backstage-cli package postpack"
   },
   "dependencies": {
-    "@backstage/core-components": "^0.13.3",
-    "@backstage/core-plugin-api": "^1.5.3",
-    "@backstage/theme": "^0.4.1",
-    "@material-ui/core": "^4.12.2",
-    "@material-ui/icons": "^4.9.1",
-    "@material-ui/lab": "4.0.0-alpha.61",
-    "react-use": "^17.2.4"
+    "@backstage/plugin-catalog": "1.11.2",
+    "@backstage/plugin-tech-insights": "0.3.11",
+    "@tpb/core-common": "1.6.0-release-1.6.x.1",
+    "@tpb/core-frontend": "1.6.0-release-1.6.x.1",
+    "@tpb/plugin-catalog": "1.6.0-release-1.6.x.1"
   },
   "peerDependencies": {
-    "react": "^16.13.1 || ^17.0.0"
+    "react": "^17.0.2",
+    "react-dom": "^17.0.2",
+    "react-router": "6.0.0-beta.0",
+    "react-router-dom": "6.0.0-beta.0"
   },
   "devDependencies": {
-    "@backstage/cli": "^0.22.9",
-    "@backstage/core-app-api": "^1.9.0",
-    "@backstage/dev-utils": "^1.0.17",
-    "@backstage/test-utils": "^1.4.1",
-    "@testing-library/jest-dom": "^5.10.1",
-    "@testing-library/react": "^12.1.3",
-    "@testing-library/user-event": "^14.0.0",
-    "@types/node": "*",
-    "msw": "^1.0.0"
+    "@backstage/cli": "^0.22.6",
+    "@types/react": "^16.14.0",
+    "@types/react-dom": "^16.9.16",
+    "eslint": "^8.16.0",
+    "typescript": "~4.6.4"
   },
   "files": [
     "dist"
```

We don't want what backstage scaffolded for the src directory, so we're going to
delete it and recreate it.

```shell
$ rm -rf dev/ src/ && mkdir src
```

Then within the `src/` directory we'll create a file called `index.ts`

```shell
$ cd src/
$ touch index.ts TechInsightsFrontendPlugin.tsx
```

And we'll add to `TechInsightsFrontendPlugin.tsx` the following content

```tsx
import { EntityLayout } from '@backstage/plugin-catalog';
import { EntityTechInsightsScorecardContent } from '@backstage/plugin-tech-insights';
import { AppPluginInterface, AppRouteSurface } from '@tpb/core-frontend';
import { SurfaceStoreInterface } from '@tpb/core-common';
import { EntityPageSurface } from '@tpb/plugin-catalog';
import React from 'react';

export const TechInsightsFrontendPlugin: AppPluginInterface =
  () => (context: SurfaceStoreInterface) => {
    context.applyWithDependency(
      AppRouteSurface,
      EntityPageSurface,
      (_appRouteSurface, entityPageSurface) => {
        entityPageSurface.servicePage.addTab(
          <EntityLayout.Route path="/techinsights" title="TechInsights">
            <EntityTechInsightsScorecardContent
              title="TechInsights Scorecard."
              description="TechInsight's default fact-checkers"
            />
          </EntityLayout.Route>,
        );
      },
    );
  };
```

And to index.ts we'll add
```ts
export { TechInsightsFrontendPlugin as plugin } from './TechInsightsFrontendPlugin';
```

The reason we alias `TechInsightsFrontendPlugin` to `plugin` is because the
Tanzu Developer Portal Configurator expects compatible plugins to export a
symbol with the name `plugin`.

```shell
$ yarn install && yarn tsc && yarn build
```

### Tech Insights backend wrapper

<!-- skipping for now because it's getting late on a friday and we're getting tired -->

## Publishing your plugin wrapper

For this example we'll publish to our local verdaccio server which requires modifying the verdaccio config

Here is the final config we'll use

```yaml
storage: ./storage
auth:
  htpasswd:
    file: ./htpasswd
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    maxage: 2m
    cache: true
packages:
  '@mycompany/*':
    access: $all
    publish: $all
  '@tpb/*':
    access: $all
  '**':
    access: $all
    proxy: npmjs
log: { type: stdout, format: pretty, level: http }
```

Changes:

```yaml
auth:
  htpasswd:
    file: ./htpasswd
```

By default verdaccio has no auth mechanism set up. The simplest, and the one
that is built in, is `htpasswd`.

```yaml
packages:
  '@mycompany/*':
    access: $all
    publish: $all
```

We've added the npm organization `@mycompany/*` to our config with publish access.
This is because of the names in our package.json files we chose earlier:
`@mycompany/tech-insights-wrapper` `@mycompany/tech-insights-backend-wrapper`.

Now that authentication has been set up for your verdaccio server you can add your user with `yarn login`
```shell
$ yarn login
yarn login v1.22.19
question npm username: my-user # can be anything
question npm email: my.email@example.com # can be anything
Done in 10.06s.
```

Restart your verdaccio server and you should be able to run, from the
`plugins/tech-insights-wrapper` and the `plugins/tech-insights-backend-wrapper`
directories respectively

```
$ yarn publish
yarn publish v1.22.19
[1/4] Bumping version...
info Current version: 0.1.0
question New version: 1.0.0
[2/4] Logging in...
[3/4] Publishing...
$ backstage-cli package prepack
$ backstage-cli package postpack
success Published.
[4/4] Revoking token...
info Not revoking login token, specified via config file.
Done in 2.76s.
```

## Adding your plugin wrapper to your portal configuration file

```yaml
app:
  plugins:
    - name: '@tpb/plugin-hello-world'
      version: '^1.6.0-release-1.6.x.1'
    - name: '@mycompany/plugin-techinsights'
      version: '1.0.0'
backend:
  plugins:
    - name: '@tpb/plugin-hello-world-backend'
      version: '^1.6.0-release-1.6.x.1'
```

## Issuing your portal configuration file to your Tanzu Application Platform cluster for building

If you're using verdaccio for your publishing you'll need to make sure your
cluster can access your verdaccio server. This tutorial will not go in to how to
do that.
