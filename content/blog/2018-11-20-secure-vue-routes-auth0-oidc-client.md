---
title: "Securing Vue routes when using Auth0 and odic-client.js"
description:
  Demonstrates how to secure routes in Vue with navigation guards when authentication users with Auth0 and oidc-client.js
tags:
- auth0
- vue
- oidc
---

## Introduction

In the [previous blog post](/blog/using-auth0-with-vue-oidc-client-js/) we looked at allowing users to log in to a Vue application with Auth0 using the [oidc-client](https://github.com/IdentityModel/oidc-client-js) JavaScript library. In this blog post, we'll create a user profile page and then secure that page to ensure that only authenticated users can view it.

## Updating the scopes

For the user's profile page, we want to display the user's name, email address and profile picture. I did not pay too much attention to it in the previous blog post, but you may remember that we requested the `openid profile` scope when the user authenticated with Auth0. This will ensure that Auth0 returns the user's name and picture as part of the profile information, but it will not return the user's email address.

To get the user's email address, we will need to request the `email` scope in addition to the scopes we already request. If you are unfamiliar with this, I suggest you read the Auth0 documentation on [OpenID Connect Scopes](https://auth0.com/docs/scopes/current/oidc-scopes).

Go ahead and update the `AuthService` class we created last time also to request the `email` scope:

```ts
export default class AuthService {
    // Code omitted for brevity. Refer to the GH repo for the complete source code
    constructor() {
        const settings: any = {
            scope: "openid profile email",
        };
    }
}
```

## Creating the profile view

To display the user's profile information, we'll add a new view under the `views` folder called `Profile.vue`. The code for this view can is found below:

```html
<template>
  <div class="user-profile">
    <div class="profile-card">
      <h1 class="profile-card__title">{{ userName }}</h1>
      <p class="profile-card__subtitle">{{ userEmail }}</p>
      <img class="profile-card__avatar" :src="userPhoto" />
    </div>
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
  public userName: string = '';
  public userEmail: string = '';
  public userPhoto: string = '';

  public mounted() {
    auth.getUser().then((user) => {
      console.log(user);

      this.userName = user.profile.name;
      this.userEmail = user.profile.email;
      this.userPhoto = user.profile.picture;
    });
  }
}
</script>

<style>
.profile-card {
  background-color: #fff;
  color: #000;
  padding: 15vmin 20vmin;
  text-align: center;
}
.profile-card__title {
  margin: 0;
  font-size: 36px;
}
.profile-card__subtitle {
  margin: 0;
  color: #c0c0c0;
  font-size: 18px;
  margin-bottom: 20px;
}
.profile-card__avatar {
  display: inline-block;
  margin: 0;
  border-radius: 50%;
  border: 4px solid #fff;
  width: 200px;
  height: 200px;
}
</style>
```

As you can see, we retrieve the user's profile information in the `mounted()` function which is then stored in `userName`, `userEmail` and `userPhoto` properties. The template has a bit of markup which displays the profile information, styling it all nicely with the included CSS styles.

## Adding a route and linking to the profile page

We will need to add a route for the Profile page, and then also link to the profile page. Open the `router.ts` file and import the new `Profile` component. Then, add a `/profile` route which will route the user to this new component.

```ts
import Vue from 'vue'
import Router from 'vue-router'
import Home from './views/Home.vue'
import Profile from './views/Profile.vue'

Vue.use(Router)

export default new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [
    {
      path: '/',
      name: 'home',
      component: Home
    },
    {
      path: '/profile',
      name: 'profile',
      component: Profile
    },
    {
      path: '/about',
      name: 'about',
      // route level code-splitting
      // this generates a separate chunk (about.[hash].js) for this route
      // which is lazy-loaded when the route is visited.
      component: () => import(/* webpackChunkName: "about" */ './views/About.vue')
    }
  ]
})
```

Also, update the template in the `Home` component to link to the new profile page:

```html
<template>
  <div class="home">
    <p v-if="isLoggedIn">User: <router-link to="/profile">{{ username }}</router-link></p>
    <button @click="login" v-if="!isLoggedIn">Login</button>
    <button @click="logout" v-if="isLoggedIn">Logout</button>
  </div>
</template>
```

## Testing it out

Run the application and go to the home page. Click on the **Login** button:

![Home page when logged out](/images/blog/2018-11-20-secure-vue-routes-auth0-oidc-client/home-page.png)

This will take you to the Auth0 Lock screen. I will choose to log in with my Google account:

![Auth0 Lock screen](/images/blog/2018-11-20-secure-vue-routes-auth0-oidc-client/auth0-lock-screen.png)

I am signed in with multiple Google accounts, so Google will prompt me for the account I want to use. I'll go ahead and select my personal account:

![Google account selection screen](/images/blog/2018-11-20-secure-vue-routes-auth0-oidc-client/google-account-selection-screen.png)

This redirects me back to the application's home page where I can now see that I am logged in:

![Home page when logged in](/images/blog/2018-11-20-secure-vue-routes-auth0-oidc-client/home-page-logged-in.png)

Let's go ahead and click on the hyperlink with my name. This takes me to the new profile page:

![User profile page](/images/blog/2018-11-20-secure-vue-routes-auth0-oidc-client/profile-logged-in.png)

## Securing the profile route

There is just one minor problem. Even if I am not signed in, I can still go to the profile page by typing the `/profile` URL in the browser's address bar. Currently, this will take me to the profile page, but when I am not signed in, it will just display a blank screen since none of the user's information could be retrieved:

![Profile page when not logged in](/images/blog/2018-11-20-secure-vue-routes-auth0-oidc-client/profile-logged-out.png)

To prevent this from happening we'll be implementing a [Navigation Guard](https://router.vuejs.org/guide/advanced/navigation-guards.html).

Before we get to that though, we will need to update the `AuthService` to add a function that will tell us whether the user is logged in or not. Update the `AuthService` class and add the `isLoggedIn` function:

```ts
export default class AuthService {
    // Code omitted for brevity. Refer to the GH repo for the complete source code

    public async isLoggedIn(): Promise<boolean> {
        const user: User = await this.getUser();

        return (user !== null && !user.expired);
    }
}
```

Head back to `router.ts` and import the `AuthService`:

```ts
import Vue from 'vue'
import Router from 'vue-router'
import Home from './views/Home.vue'
import Profile from './views/Profile.vue'
import AuthService from './services/AuthService';

Vue.use(Router)

// Rest of the code omitted for brevity
```

Change the creation of the routes assign the route definitions to a `router` constant rather than exporting it directly. We will make use of [Route Meta Fields](https://router.vuejs.org/guide/advanced/meta.html) to indicate whether a route is secure, so add an `isSecure` meta field to the `/profile` route and set its value to `true`:

```ts
// Code omitted for brevity...

const router =  new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [
    {
      path: '/',
      name: 'home',
      component: Home
    },
    {
      path: '/profile',
      name: 'profile',
      component: Profile,
      meta: {
        isSecure: true,
      }
    },
    {
      path: '/about',
      name: 'about',
      // route level code-splitting
      // this generates a separate chunk (about.[hash].js) for this route
      // which is lazy-loaded when the route is visited.
      component: () => import(/* webpackChunkName: "about" */ './views/About.vue')
    }
  ]
});

export default router;
```

The final part is to add a `beforeEach` global guard with some code to check whether the requested route is secure. If the route is secure and the user is not logged in, we redirect the user back to the home page. Otherwise, we allow the user to visit the route. Also, remember to export the `router`:

```ts
// Code omitted for brevity...

router.beforeEach((to, from, next) => {
  const authService: AuthService = new AuthService();

  if (to.matched.some(record => record.meta.isSecure)) {
    // this route requires auth, check if logged in
    // if not, redirect to login page.
    authService.isLoggedIn().then((isLoggedIn: boolean) => {
      if (isLoggedIn) {
        next();
      } else {
        next({
          path: '/',
          query: { redirect: to.fullPath }
        });
      }
    });
  } else {
    next();
  }
});

export default router;
```

With that in place, the `/profile` route should now be guarded to ensure that only logged in users can access it. Be sure to confirm this for yourself.

## Conclusion

In this blog post, we looked at how to display a user's profile information in a Profile view. We also added a route guard to ensure that only logged in users can access that view.

Next time we'll conclude this series by looking at how you can do silent renewal of tokens.

Source code is available at [https://github.com/jerriepelser-blog/auth0-vue-oidc-client](https://github.com/jerriepelser-blog/auth0-vue-oidc-client)