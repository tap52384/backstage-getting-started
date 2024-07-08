# backstage-getting-started

<https://backstage.io/docs/getting-started/>

## Getting Started / Authentication via GitHub

### References - Getting Started / Authentication via GitHub

- [backstage.io - Getting Started](https://backstage.io/docs/getting-started/)
- [backstage.io - Authentication](https://backstage.io/docs/getting-started/config/authentication/)

### Installing the app

The instructions say to create a backstage app using the following command:

```bash
npx @backstage/create-app@latest
```

This command requires node version **18** or **20** when attempting to install
**yarn** (which I already had installed). I used the following command to install
node 18 via nvm:

```bash
# Install the latest supported version of node for @backstage
nvm install 18
# Install yarn specifically for this version of node
npm install --global yarn
yarn --version
yarn install
# remove the directory created by the npx command (only if something's wrong)
# rm -rf backstage
```

Using node 18  worked better as there is a module that is deprecated in node 22.
This message appeared every time I started the `npx` command with node 22:

```bash
zplewis@Zs-MacBook-Air backstage-getting-started % npx @backstage/create-app@latest
(node:7508) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
```

### Running the app

To start the app, use the `yarn dev` command inside the folder you created for
your Backstage app. I used the default name `backstage` for my app name.

On macOS, the terminal (iTerm2) requested access to execute AppleScript.

```bash
zplewis@Zs-MacBook-Air backstage-getting-started % cd backstage
zplewis@Zs-MacBook-Air backstage % yarn dev
```

### GitHub

For authenticating via GitHub, [the documentation](https://backstage.io/docs/getting-started/config/authentication/) recommends creating an OAuth app by going to
[this page](https://github.com/settings/applications/new). There is additional
documentation specifically about [the GitHub Authentication Provider](https://backstage.io/docs/auth/github/provider).

> When creating the GitHub OAuth app, I checked the **Enable Device Flow** option.

Once you create the GitHub OAuth app, they provide the `Client ID` but you have to
generate a `client secret` for that client ID. You will need both for configuring this
Backstage application (`app-config.yaml`).

> It is required (unless you can make environment variables work) to use `app-config.local.yaml`
> to include the client ID and secret from GitHub for authentication to work properly. Otherwise,
> the sensitive values would have to be included in files that are committed (like app-config.yaml).
> Without the GitHub client ID and secret, the popup that appears when attempting to sign in via
> GitHub and authorize our Backstage app with our GitHub OAuth app will fail.

The documentation on the GitHub provider mentions a resolver which seems to
determine how to match the GitHub user to the Backstage user and they offer
three different ways to do so. The default is `usernameMatchingUserEntityName`.
You can specify what resolving methods to use and even use a custom resolver
if necessary. More information on sign-in resolvers [can be found here](https://backstage.io/docs/auth/identity-resolver/#sign-in-resolvers).

### Environment Variables

In setting up the [GitHub Authentication Provider](https://backstage.io/docs/auth/github/provider), environment variables
were needed. Backstage does provide a way using `app-config.local.yaml` and other
[environment-specific configuration files](https://backstage.io/docs/conf/writing/#configuration-files).

I wanted to make `.env.yarn` work, but it never did.

- [StackOverflow - How to edit environment variables in backstage.io](https://stackoverflow.com/a/77950324/1620794)
- [Yarn - Scripting](https://yarnpkg.com/features/scripting)

### Add sign-in option to the frontend

When adding code to `packages/app/src/App.tsx`, this package has already been
included:

```javascript
import { githubAuthApiRef } from '@backstage/core-plugin-api';
// this line is not needed
import { SignInPage } from '@backstage/core-components';
```

In the `app` const (`const app = createApp`), you may already see a `SignInPage`
component with an array of `providers`. Add the GitHub provider to the list.

You also have to [manually enable the GitHub authentication plugin](https://backstage.io/docs/backend-system/building-backends/migrating/#github-1) by modifying
the file `packages/backend/src/index.ts`:

```javascript
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```

This page also confirms that three resolvers are natively supported by this plugin for GitHub:

- `usernameMatchingUserEntityName`
- `emailMatchingUserEntityProfileEmail`
- `emailLocalPartMatchingUserEntityName`

The resolvers are specified in the config.yaml file. Enabling the plugin is not explicitly stated
in the same documentation that explains how to add sign-in to the frontend.

### Attempting to sign-in

Make sure that pop-ups are allowed from localhost in your web browser. The
Backstage app automatically refreshed as I made changes to `App.tsx`, but I
manually refreshed the page any way sometimes to be safe. When attempting to
login with GitHub, I received the following error:

```json
{
  "error": {
    "name": "NotFoundError",
    "message": "Unknown auth provider 'github'",
    "stack": "NotFoundError: Unknown auth provider 'github' at ..."
  }
}
```

> **NOTE:** You must shutdown the Backstage app completely using Control + C in the terminal and
> start it again using `yarn dev` for the plugin and environment variables to be recognized
> properly.

Once the pop-up window allowed me to authorize my Backstage app, the message
**Login failed, user profile does not contain an email** is returned to the application.

According to [a user with the same issue](https://github.com/backstage/backstage/issues/23748#issuecomment-2016989411), the fix is to [make the email address of the GitHub profile public](https://github.com/settings/emails). To do so, uncheck the box beside **Keep my email addresses private**.
After doing so, choose an email address for the **Primary email address** field and click **Save**.

In summary, to resolve these errors, I had to:

- add the github auth provider plugin
- include the client id and client secret values directly into `app-config.local.yaml` as I couldn't get environment variables to work
- completely stop the website using Control + C in the terminal then starting the app again using `yarn dev`
- In the Settings of my GitHub account, not only do you have to uncheck the box keeping your email address private,
but you have to actually select the public email address

### Attempting to sign-in, part 2: resolver error

This error remains which leads me to believe that the OAuth authentication is working but mapping the
GitHub user to the Backstage user is not just yet:

```log
Failed to sign-in, unable to resolve user identity
```

The setup directions create a `guest` user. I found [a YouTube video](https://youtu.be/VCMoLchQRL8?t=1926)
showing how multiple providers can be specified at the same time and made a similar change in the
file `packages/app/src/App.tsx` to allow logging in as **guest** and via GitHub. Logging in as `guest`
takes to the `catalog` where you can find different types of objects like API, Component, Group, etc.
We are interested in type `User`.

You can add a new user by editing `examples/org.yaml` using [the example from the documentation](https://youtu.be/VCMoLchQRL8?t=1926).
For the purposes of authenticating with GitHub, I added an email address to the new user to resolve:

```yaml
---
# https://backstage.io/docs/features/software-catalog/descriptor-format#kind-user
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: tap52384
spec:
  profile:
    displayName: Patrick Lewis
    email: tap52384@gmail.com
    picture: https://avatars.githubusercontent.com/u/647983?v=4
  memberOf: [guests]
```

To have the changes to the guest user reflected in the site, I had to use **Control + C** to stop
the app entirely. I also went to [the GitHub OAuth app](https://github.com/settings/developers) and
revoked the login token (kept the client secret) so that the app could login again.

If you need to log out of the scaffolding test Backstage app, you have two options on the **Settings** page:

- On the **General** tab, in the **Profile** section, click the three dots and click **Sign Out**
- On the **Authentication Providers** tab, in the **Available Providers** section, click **Sign Out**

You can also sign in again from the **Authentication Providers** tab.

If done correctly, signing in via GitHub should work even if you login first as guest and then via GitHub.

## Building a Docker image

### References - Building a Docker image

- [backstage.io - Building a Docker image](https://backstage.io/docs/deployment/docker/)

### Host Build

Running the following command prevents the `yarn.lock` file from being updated when you run
`yarn install`. This makes sense to me as that is how `composer` for PHP works. Updates only occur
when requested via `composer update`.

The directions say that "tsc outputs type definitions to dist-types/ in the repo root", but that is
not true. The command `yarn tsc` only works when run inside the Backstage app folder created by using
the `@backstage/create-app` command. Therefore, the `dist-types` folder is created within that folder
and not necessarily within the root of the repository.

```bash
# Make sure you are inside the Backstage app folder
# In the Dockerfile, this may be referred to as /app
# All of the following commands are executed from the root of the Backstage app folder.
cd backstage

# Make sure you are using the correct version of node
# Based on the Dockerfile included with this app, using version 18.20.3 is the best version
# https://github.com/nodejs/docker-node/blob/main/18/bookworm-slim/Dockerfile
nvm install 18.20.3
nvm use 18.20.3

# Installs packages specified in package.json without updating them
yarn install --frozen-lockfile

# tsc compiles the project defined in tsconfig.json, which also proves that this
# command must be run from within the Backstage app folder
# https://www.typescriptlang.org/docs/handbook/compiler-options.html
# "tsc" stands for "typescript compile, maybe?"; yes: tsc: The TypeScript Compiler
# tsc - compiles the current project (tsconfig.json in the working directory)
yarn tsc

# From the Dockerfile, these commands seem to successfully build the image
# You can see how these commands actually work by viewing package.json
# Build the backend, which bundles it all up into the packages/backend/dist folder.
# The configuration files here should match the one you use inside the Dockerfile below.
yarn build:backend

# The Dockerfile for the "Host Build" step is found here:
# backstage/packages/backend/Dockerfile
# However, we are executing it from the root of the backstage app for the build context (files
# included when the image is built)
# This creates an image called "backstage"
yarn build-image

# To create a container from the "backstage" image and run it
yarn build-image && docker run -it -p 7007:7007 backstage
docker run -it -p 7007:7007 --env-file .env.yarn backstage
```

When starting the container, there is a failure to connect to the database. Based
on the port number in the error message and `app-config.production.yaml`, it appears
that this image is attempting to connect to a PostgresSQL database:

```log
/app/node_modules/@backstage/backend-defaults/dist/database.cjs.js:454
        throw new Error(
              ^

Error: Failed to connect to the database to make sure that 'backstage_plugin_app' exists, Error: connect ECONNREFUSED 127.0.0.1:5432
    at PgConnector.getClient (/app/node_modules/@backstage/backend-defaults/dist/database.cjs.js:454:15)
    at runNextTicks (node:internal/process/task_queues:60:5)
    at listOnTimeout (node:internal/timers:538:9)
    at process.processTimers (node:internal/timers:512:7)
```

For purposes of this test, use the same `database` config from `app-config.yaml` for `app-config.production.yaml`
if you do not have a database available to connect to.

The `guest` auth provider is not allowed in a production environment, so you may have to remove it.
Since we took the time to get the GitHub provider working first, this should work.

One more note: any time that you change one of the `app-config.*.yaml` files, you have to run the
command `yarn build-image` again since the code is part of the image.

Based on the Dockerfile used for the **Host Build** step, two config.yaml files are used:

- app-config.yaml
- app-config.production.yaml

I had to comment out the `guest` auth provider from both config files.
