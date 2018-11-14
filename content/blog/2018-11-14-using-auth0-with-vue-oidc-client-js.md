---
title: "Using Auth0 with Vue and oidc-client-js"
description:
  Shows how you can make use of the oidc-client JavaScript library to enable users to log in to your Vue application with Auth0.
tags:
- auth0
- vue
- oidc
---

## Introduction

Auth0 has a [Vue Quickstart](https://auth0.com/docs/quickstart/spa/vuejs) that demonstrates how to use Auth0 with Vue using the [auth0.js library](https://github.com/auth0/auth0.js/). This works fine but it requires you to do some manual work for storing the token, and it also does not demonstrate how to handle token renewal which many of their other SPA Quickstarts show.

A while back I wanted to use Auth0 in a Vue side project, and I decided to use the [oidc-client](https://github.com/IdentityModel/oidc-client-js) JavaScript library to handle the Auth0 integration since this already handles both token storage and token renewal out of the box.

In this blog post, I will demonstrate how you can use **oidc-client** in your own Vue applications, rather than Auth0's **auth0.js** library. This blog post assumes familiarity with both Auth0 and Vue.

## Creating a new project

Make sure that you have the [Vue CLI 3.x installed](https://cli.vuejs.org/guide/installation.html), then run the following command to create a new Vue application:

```text
vue create auth0-vue-oidc-client
```

We'll be creating this sample using TypeScript, so when prompted by the Vue CLI for the preset to use, choose to **Manually select features** and then select **TypeScript** and **Router**:

![Use TypeScript for application](/images/blog/2018-11-15-using-auth0-with-vue-oidc-client-js/vue-cli-features.png)

Feel free to select whatever features you prefer and then continue with the rest of the prompts to create your project. Here are all the settings I used:

![Settings used for application](/images/blog/2018-11-15-using-auth0-with-vue-oidc-client-js/vue-cli-settings.png)

When finished, remember to change directory into the new project you created:

```text
cd auth0-vue-oidc-client
```

We'll also need to add [oidc-client](https://github.com/IdentityModel/oidc-client-js) as a dependency and the [Copy Webpack Plugin](https://github.com/webpack-contrib/copy-webpack-plugin) as a development dependency, so go ahead and install those:

```text
npm install oidc-client
npm install copy-webpack-plugin --save-dev
```

## Creating an Auth0 application

You will need to register and configure your application in the [Auth0 Dashboard](https://manage.auth0.com/#/applications). If you are not familiar with this concept and process, I suggest you read the Auth0 [documentation on Applications](https://auth0.com/docs/applications) and specifically the part about [Single Page Web Applications](https://auth0.com/docs/applications/spa).

Go ahead and create a new application. Specify a name for the application, select the **Single Page Web App** option and then click **Create**.

![](/images/blog/2018-11-15-using-auth0-with-vue-oidc-client-js/auth0-create-application.png)

Once the application is created, go to the _Settings_ tab and make set the **Allowed Callback URLs** to `http://localhost:8080/callback.html`. Also, set **Allowed Logout URLs** to `http://localhost:8080/` and be sure to save the settings.

## Creating an AuthService

We'll create an `AuthService` class to encapsulate some of the **oidc-client** functionality. Back in your Vue application, create a directory called `src/services` and inside that directory, create a new file called `AuthService.ts`. You can see the source code for the `AuthService` class below:

```ts
import { UserManager, WebStorageStateStore, User } from "oidc-client";

export default class AuthService {
    private userManager: UserManager;

    constructor() {
        const AUTH0_DOMAIN: string = "YOUR_AUTH0_DOMAIN"; // e.g. https://jerrie.auth0.com

        const settings: any = {
            userStore: new WebStorageStateStore({ store: window.localStorage }),
            authority: AUTH0_DOMAIN,
            client_id: "YOUR_AUTH0_CLIENT_ID",
            redirect_uri: "http://localhost:8080/callback.html",
            response_type: "id_token token",
            scope: "openid profile",
            post_logout_redirect_uri: "http://localhost:8080/",
            filterProtocolClaims: true,
            metadata: {
                issuer: AUTH0_DOMAIN + "/",
                authorization_endpoint: AUTH0_DOMAIN + "/authorize",
                userinfo_endpoint: AUTH0_DOMAIN + "/userinfo",
                end_session_endpoint: AUTH0_DOMAIN + "/v2/logout",
                jwks_uri: AUTH0_DOMAIN + "/.well-known/jwks.json",
            }
        };

        this.userManager = new UserManager(settings);
    }

    public getUser(): Promise<User> {
        return this.userManager.getUser();
    }

    public login(): Promise<void> {
        return this.userManager.signinRedirect();
    }

    public logout(): Promise<void> {
        return this.userManager.signinRedirect();
    }
}
```

The `AuthService` class instantiates a new instance of the `UserManager` class and then basically provides thin wrappers around the `signinRedirect()` and `signinRedirect()` functions of the `UserManager` class via the `login()` and `logout()` functions.

Under normal circumstances, **oidc-client** will automatically determine all the relevant endpoints and other information for an OIDC-enabled Identity Provider such as Auth0 through the [OIDC discovery document](https://openid.net/specs/openid-connect-discovery-1_0.html). Unfortunately, Auth0 does not specify a logout endpoint (`end_session_endpoint`) in the discovery document, meaning that it has to be supplied manually.

**oidc-client** allows for manually specifying information typically supplied in the OIDC Discovery Document by passing a `meta` setting atrtribute, but it does not allow you to override just a single value - it is an all-or-nothing situation. You can see in the sample code above that I supplied all the relevant values in the `meta` attribute of the `settings`.

## Create the callback URL

In the configuration above, you will notice that the `redirect_uri` is specified as `http://localhost:8080/callback.html`. This is where Auth0 will redirect the user back to your application after they have authenticated. This is the same value we specified earlier for **Allowed Callback URLs** when registering the application in the Auth0 Dashboard.

In the `/public` directory of your Vue application, create a file named `callback.html` with the following content:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Waiting...</title>
</head>
<body>
  <script src="js/oidc-client.min.js"></script>
  <script>
    var mgr = new Oidc.UserManager({userStore: new Oidc.WebStorageStateStore()});

    mgr.signinRedirectCallback().then(function (user) {
      window.location.href = '../';
    }).catch(function (err) {
      console.log(err);
    });
  </script>
</body>
</html>
```

The code above will complete the authentication process when Auth0 redirects back to your application and then redirect the user to the root URL (`/`) of the website to load the Vue application again.

The `signinRedirectCallback()` will take care of storing the `id_token` and `access_token` received from Auth0 in the local storage for your Vue application to access.

One thing we still need to do is to ensure that the `callback.html` file can reference the **oidc-client** JavaScript file which I linked to in the file above (at `js/oidc-client.min.js`). For the Vue application itself, Webpack will take care of including **oidc-client** in the generated bundle, but for the `callback.html`, we need to copy that `oidc-client.min.js` file manually.

When the Webpack build is executed, it will copy `callback.html` along with the other files in the `public` directory to the `dist` directory. We want to copy the `oidc-client.min.js` file from the `node_modules` directory to the `dist/js` directory.

To do this, we will make use of the [copy-webpack-plugin](https://github.com/webpack-contrib/copy-webpack-plugin) we installed earlier, along with the Vue CLI's ability to [tweak the Webpack configuration](https://cli.vuejs.org/guide/webpack.html#simple-configuration).

Inside the root directory of your project, create a file named `vue.config.js` with the following content:

```js
const CopyWebpackPlugin = require('copy-webpack-plugin')

module.exports = {
    configureWebpack: {
      plugins: [
        new CopyWebpackPlugin([
            { from: 'node_modules/oidc-client/dist/oidc-client.min.js', to: 'js' }
        ])
      ]
    }
  }
```

The snippet above will configure the **copy-webpack-plugin** to copy the `oidc-client.min.js` file from the **oidc-client** module to the `js` directory.

## Updating the Home view

One remaining thing is to add login and logout functionality to the `Home` view that was generated by the Vue CLI. Open the `/src/views/Home.vue` file and update its contents as follows:

```html
<template>
  <div class="home">
    <p v-if="isLoggedIn">User: {{ username }}</p>
    <button @click="login" v-if="!isLoggedIn">Login</button>
    <button @click="logout" v-if="isLoggedIn">Logout</button>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import AuthService from '@/services/AuthService';

const auth = new AuthService();

@Component({
  components: {
  },
})
export default class Home extends Vue {  
  public currentUser: string = '';
  public accessTokenExpired: boolean | undefined = false;
  public isLoggedIn: boolean = false;

  get username(): string {
    return this.currentUser;
  }

  public login() {
    auth.login();
  }

  public logout() {
    auth.logout();
  }

  public mounted() {
    auth.getUser().then((user) => {
      this.currentUser = user.profile.name;
      this.accessTokenExpired = user.expired;

      this.isLoggedIn = (user !== null && !user.expired);
    });
  }
}
</script>
```

You'll notice that we create an instance of the `AuthService` and then call the `login` and `logout` functions from namesake functions inside the `Home` component. These functions are bound to two buttons we display in the view. Also, in the `mounted` function, we call the `getUser` function and then store the user's information to be displayed in the view.

## Testing it out

With that in place, we can now run the application. When starting up we'll be greeted with by a page with a login button.

![Home page displaying login button](/images/blog/2018-11-15-using-auth0-with-vue-oidc-client-js/home-with-login.png)

Clicking on the **Login** button will send us off to the Auth0 login screen where we can log in with our our email address and password:

![Auth0 login screen](/images/blog/2018-11-15-using-auth0-with-vue-oidc-client-js/auth0-login-screen.png)

Clicking on the **Log In** button sends us back to our Vue application where we can see the user's information:

![Home page with the user logged in](/images/blog/2018-11-15-using-auth0-with-vue-oidc-client-js/home-with-user-logged-in.png)

## Conclusion

This blog post demonstrated how you could make use of the **oidc-client** library to enable users to log in to your Vue application with Auth0. Source code is available at [https://github.com/jerriepelser-blog/auth0-vue-oidc-client](https://github.com/jerriepelser-blog/auth0-vue-oidc-client)

Next time, we'll look at displaying a user profile page and protecting routes so only logged in user's can access them.