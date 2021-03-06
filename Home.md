# How to implement vuex-oidc

The library is available as an [npm package](https://www.npmjs.com/package/vuex-oidc). oidc-client is a peer dependency.
To install the package (and it's peer dependency) run:

```bash
npm install oidc-client vuex-oidc --save
```

For reference, there is [an example](https://github.com/perarnborg/vuex-oidc-example) of a working implementation.

There is also [an example](https://github.com/perarnborg/vuex-oidc-example-nuxt) of how to implement vuex-oidc with Nuxt SPA mode. Nuxt Universal mode is currently not supported.

## 1) Create your oidc settings

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/config/oidc.js

export const oidcSettings = {
  authority: 'https://your_oidc_authority',
  clientId: 'your_client_id',
  redirectUri: 'http://localhost:1337/oidc-callback',
  responseType: 'id_token token',
  scope: 'openid profile'
}

```

Check out [documentation for oidc-client](https://github.com/IdentityModel/oidc-client-js/wiki) to see all options. In addition to the options in oidc-client there is also an option to turn off automatical signin on public routes. See more about this in the section about silent signin.

A note on the settings: oidc-client uses snake_case for some options and camelCase for others. With vuex-oidc you can use camelCase for all of the options. Or if you want you can use snake_case for the ones that are snake_cased in oidc-client (e.g. redirect_uri).

## 2) Setup vuex

Import and use vuex module. Is is created with a factory function that takes your oidc config as argument.

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/store.js

import { vuexOidcCreateStoreModule } from 'vuex-oidc'

import { oidcSettings } from './config/oidc'

export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(oidcSettings)
  }
})
```

If you want a namespaced module you can pass storeSetting as a second argument:

```js
export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(oidcSettings, { namespaced: true })
  }
})
```

If you want to have some of your routes public, you can specify those as publicRoutePaths in the storeSetting argument. If you use vue-router you can in stead specify isPublic: true on the routes' meta properties (see step #8).

```js
export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(oidcSettings, { namespaced: true, publicRoutePaths: ['/', '/about-us'] })
  }
})
```

If you use vue-router with a router base other than the default (`/`), you will want to pass your route as routeBase in the storeSetting argument.

```js
export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(oidcSettings, { namespaced: true, routeBase: '/app/' })
  }
})
```

By default this package binds access to if the user has a valid id_token. You can change this with the isAuthenticatedBy property on the storeSetting argument so access is bound to a valid access_token in stead.

```js
export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(oidcSettings, { namespaced: true, isAuthenticatedBy: 'access_token' })
  }
})
```

## 3) Setup route for Open id callback

Create a callback component. The component will be rendered during the time that the callback is made, so feel free to add any loader/spinner if you want.

If you have created your vuex store module as a namespaced module you will have to map the action from the correct namespace.

```vue
<!-- https://github.com/perarnborg/vuex-oidc-example/blob/master/src/views/OidcCallback.vue -->

<template>
  <div>
  </div>
</template>

<script>
import { mapActions } from 'vuex'

export default {
  name: 'OidcCallback',
  methods: {
    ...mapActions([
      'oidcSignInCallback'
    ])
  },
  mounted () {
    this.oidcSignInCallback()
      .then((redirectPath) => {
        this.$router.push(redirectPath)
      })
      .catch((err) => {
        console.error(err)
        this.$router.push('/oidc-callback-error') // Handle errors any way you want
      })
  }
}
</script>
```

Setup the route with your callback component. If you use nuxt-router you do not need to do this.

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/router.js

import OidcCallback from '@/components/OidcCallback'

const routes = [
  ...yourApplicationsRoutes,
  {
    path: '/oidc-callback', // Needs to match redirectUri (redirect_uri if you use snake case) in you oidcSettings
    name: 'oidcCallback',
    component: OidcCallback
  }
]
```

## 4a) Setup vue-router

Create the oidc router middleware with factory function vuexOidcCreateRouterMiddleware that takes your vuex store as argument.

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/router.js

import Router from 'vue-router'
import { vuexOidcCreateRouterMiddleware } from 'vuex-oidc'

import store from '@/store'

const router = new Router({
  mode: 'history',
  routes: routes
})
router.beforeEach(vuexOidcCreateRouterMiddleware(store))
```

vuexOidcCreateRouterMiddleware takes an optional second argument called vuexNamespace. I you have created you oidc vuex module as a namespaced module you should pass the namespace string as this second argument:

```js
router.beforeEach(vuexOidcCreateRouterMiddleware(store, 'oidcStore'))
```

## 4b) Setup nuxt-router

If you use Nuxt and not vue-router, you create the oidc router middleware with factory function vuexOidcCreateNuxtRouterMiddleware that takes your vuex store namespace as argument.

```js
// https://github.com/perarnborg/vuex-oidc-example-nuxt/blob/master/middleware/vuex-oidc-router.js

import { vuexOidcCreateNuxtRouterMiddleware } from 'vuex-oidc'

export default vuexOidcCreateNuxtRouterMiddleware('oidc')
```

NOTE: If you use vue-router with routerMode: 'hash' you need to handle the oidc callback outside of the Vue app. For reference, there is [a branch](https://github.com/perarnborg/vuex-oidc-example/tree/example/hash-mode) of the example project that has vue-router with hash mode, [the diff page](https://github.com/perarnborg/vuex-oidc-example/compare/example/hash-mode) shows exactly the changes that can be made to support hash mode.

## 5) Optional: Control rendering in app layout or common components

The router middleware will ensure that routes that require authentication are not rendered. If you want to control rendering outside of the router-view you can use the vuex getter oidcIsAuthenticated to check authentication.

This can be done in any way you want. Here is an example of how to condition rendering against authentication in a component.

```vue
<template>
  <div v-if="oidcIsAuthenticated">
    Protected content
  </div>
</template>

<script>
import { mapGetters } from 'vuex'

export default {
  name: 'MyCommonComponent',
  computed: {
    ...mapGetters([
      'oidcIsAuthenticated'
    ])
  }
}
</script>
```

If you want to know not only if the user is authenticated, but if they have access to the current route you can do so by checking oidcIsAuthenticated together with meta.isPublic of current route.

```vue
<template>
  <div v-if="hasAccess">
    You have access to this route
  </div>
  <router-view></router-view>
</template>

<script>
import { mapGetters } from 'vuex'

export default {
  name: 'App',
  computed: {
    ...mapGetters([
      'oidcIsAuthenticated'
    ]),
    hasAccess: function() {
      return this.oidcIsAuthenticated || this.$route.meta.isPublic
    }
  }
}
</script>
```

## 6) Optional: use access token in AJAX requests

Get the access_token from the store and set it as Bearer authentication header if you want to make authenticated AJAX requests. There is also a vuex getter that can be used to obtain the access_token (See API bellow).

```js
import store from '@/store'

makeRequest('get', 'https://api.example.com/path', store.state.oidcStore.access_token)
  .then((reponse) => {
    console.log(response)
  })

function makeRequest(method, url, access_token) {
  return new Promise(function (resolve, reject) {
    let xhr = new XMLHttpRequest()
    xhr.open(method, url)
    xhr.withCredentials = true
    xhr.setRequestHeader('Authorization', 'Bearer ' + access_token)
    xhr.onload = function () {
      if (this.status >= 200 && this.status < 300) {
        resolve(xhr.response)
      }
    }
    xhr.onerror = function () {
      ...
    }
    xhr.send()
  })
}
```

## 7) Optional: display signed in user info and show sign out button

Use vuex getter oidcUser to show user info. Use vuex action signOutOidc to sign out user.

```vue
<template>
  <div v-if="oidcUser">
    Signed in as {{ oidcUser.email }}
    <button @click="signOutOidc">Sign out</button>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'

export default {
  name: 'MyProtectedRouteComponent',
  computed: {
    ...mapGetters([
      'oidcUser'
    ])
  },
  methods: {
    ...mapActions([
      'signOutOidc'
    ])
  }
}
</script>
```

If you for some reason want to sign out the user only from the current application, you can use the action `removeOidcUser` in stead of `signOutOidc`.

A third option is to use the action `signOutOidcSilent`, which signs the user out at the provider in a hidden iframe (in stead of redirecting the browser window to the identity provider's server).

## 8) Optional: set specific routes as public

```js
import { PublicRouteComponent } from '@/components/PublicRouteComponent'

const routes = [
  ...yourApplicationsRoutes,
  {
    path: '/public',
    name: 'publicRoute',
    component: PublicRouteComponent,
    meta: {
      isPublic: true
    }
  }
]
```

Routes with meta.isPublic will not require authentication (you may also specify what routes are public when you create your store module, see step #2). If you have setup a silentRedirectUri (silent_redirect_uri if you use snake case) a silent signIn will be made on public routes.

If there are other parameters that you want to consider when deciding if a route is public you can also provide a function that takes the route as an argument when you create your store module:

```js
vuexOidc.vuexOidcCreateStoreModule(oidcConfig, { isPublicRoute: (route) => route.name && route.name.startsWith('public') })
```

## 9) Optional: setup silent renew callback

```js
export const oidcSettings = {
  ...youOidcOtherSettings,
  silentRedirectUri: 'http://localhost:1337/oidc-silent-renew.html',
  automaticSilentRenew: true, // If true oidc-client will try to renew your token when it is about to expire
  automaticSilentSignin: true // If true vuex-oidc will try to silently signin unauthenticated users on public routes. Defaults to true
}
```

You have to make sure that you have an endpoint that matches the silentRedirectUri setting. It should run the following code:

```js
import { vuexOidcProcessSilentSignInCallback } from 'vuex-oidc'

vuexOidcProcessSilentSignInCallback()
```

## 10) Optional: setup oidc-client event listeners

oidc-client has an [events api](https://github.com/IdentityModel/oidc-client-js/wiki#events) that you can choose to implement by passing a third argument into vuexOidcCreateStoreModule

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/store.js

import Vue from 'vue'
import Vuex from 'vuex'
import { vuexOidcCreateStoreModule } from 'vuex-oidc'
import { oidcSettings } from './config/oidc'

Vue.use(Vuex)

export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(
      oidcSettings,
      // Optional OIDC store settings
      {
        dispatchEventsOnWindow: true
      },
      // Optional OIDC event listeners
      {
        userLoaded: (user) => console.log('OIDC user is loaded:', user),
        userUnloaded: () => console.log('OIDC user is unloaded'),
        accessTokenExpiring: () => console.log('Access token will expire'),
        accessTokenExpired: () => console.log('Access token did expire'),
        silentRenewError: () => console.log('OIDC user is unloaded'),
        userSignedOut: () => console.log('OIDC user is signed out'),
        oidcError: (payload) => console.log(`An error occured at ${payload.context}:`, payload.error),
        automaticSilentRenewError: (payload) => console.log('Automatic silent renew failed.', payload.error)
      }
    )
  }
})
```

The events oidcError and automaticSilentRenewError are not from `oidc-client`, but are implemented in `vuex-oidc`.

If you want to listen to the events from inside your application the events can also be dispatched in the browser as custom events by vuex-oidc (prefixed with `vuexoidc:`). If you want this you pass `dispatchEventsOnWindow: true` as a storeSetting (second argument to the vuexOidcCreateStoreModule function).

```js
// https://github.com/perarnborg/vuex-oidc-example/tree/master/src/store.js
// How to make vuex-oidc dispatch events on window

export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(
      oidcSettings,
      // Optional OIDC store settings
      {
        dispatchEventsOnWindow: true
      }
    )
  }
})


// https://github.com/perarnborg/vuex-oidc-example/tree/master/src/App.vue
// How to listen to oidc-client events on window

export default {
  name: 'App',
  methods: {
    userLoaded: function () {
      console.log('I am listening to the user loaded event in vuex-oidc')
    }
  },
  mounted () {
    window.addEventListener('vuexoidc:userLoaded', this.userLoaded)
  },
  destroyed () {
    window.removeEventListener('vuexoidc:userLoaded', this.userLoaded)
  }
}
```

## API

The router middleware handles authentication of routes automatically, but there are also actions you can use directly. If you want you can skip the router middleware to customize the authentication and access control behaviour of your app.

```vue
<script>
import { mapGetters, mapActions } from 'vuex'

export default {
  name: 'App',
  computed: {
    ...mapGetters([
      'oidcIsAuthenticated',
      'oidcUser',
      'oidcAccessToken',
      'oidcAccessTokenExp',
      'oidcIdToken',
      'oidcIdTokenExp',
      'oidcRefreshToken',
      'oidcRefreshTokenExp',
      'oidcAuthenticationIsChecked',
      'oidcError'
    ]),
    hasAccess: function() {
      return this.oidcIsAuthenticated || this.$route.meta.isPublic
    }
  },
  methods: {
    ...mapActions([
      'authenticateOidc', // Authenticates with redirect to sign in if not signed in
      'oidcSignInCallback', // Handles callback from authentication redirect. Has an optional url parameter
      'authenticateOidcSilent', // Authenticates if signed in. No redirect is made if not signed in
      'getOidcUser', // Get user from oidc-client storage and update it in vuex store. Returns a promise
      'signOutOidc', // Signs out user in open id provider
      'signOutOidcSilent', // Signs out user in open id provider using a hidden iframe
      'removeOidcUser' // Signs out user in vuex and browser storage, but not in open id provider
    ])
  }
}
</script>
```
