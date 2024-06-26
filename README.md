# backstage-getting-started

https://backstage.io/docs/getting-started/

## Notes

### Installing the app

The instructions say to create a backstage app using the following command:

```bash
npx @backstage/create-app@latest
```

This command requires node version **18** or **20** when attempting to install
**yarn** (which I already had installed). I used the following command to install
node 20 via nvm:

```bash
# Install the latest supported version of node for @backstage
nvm install 20
# Install yarn specifically for this version of node
npm install --global yarn
yarn --version
# remove the directory created by the npx command
rm -rf backstage
```

Using node 20 worked better as there is a module that is deprecated in node 22.
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
yarn run v1.22.22
```

### GitHub

For authenticating via GitHub, [the documentation](https://backstage.io/docs/getting-started/config/authentication/) recommends creating an OAuth app by going to
[this page](https://github.com/settings/applications/new). There is additional
documentation specifically about [the GitHub Authentication Provider](https://backstage.io/docs/auth/github/provider).

> When creating the GitHub OAuth app, I checked the **Enable Device Flow** option.

Once you create the GitHub OAuth app, they provide the `Client ID` but you have to
generate a `client secret` for that client ID. You will need both for configuring this
Backstage application (`app-config.yaml`).

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

When I resume work on this project, the next step is to figure out how to specify the Backstage user.
