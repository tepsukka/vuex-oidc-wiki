# How to implement vuex-oidc

The library is available as an [npm package](https://www.npmjs.com/package/vuex-oidc).
To install the package run:

```bash
npm install vuex-oidc --save

```
For reference, there is an example of a working implementation in https://github.com/perarnborg/vuex-oidc-example


## 1) Create your oidc settings

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/config/oidc.js

export const oidcSettings = {
  authority: 'https://your_oidc_authority',
  client_id: 'your_client_id',
  redirect_uri: 'http://localhost:1337/oidc-callback',
  response_type: 'id_token token',
  scope: 'openid profile'
}

```

Check out documentation for oidc-client to see all options: https://github.com/IdentityModel/oidc-client-js/wiki


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
If you want a namespaced module you can pass moduleOptions as a second argument:

```js
export default new Vuex.Store({
  modules: {
    oidcStore: vuexOidcCreateStoreModule(oidcSettings, { namespaced: true })
  }
})

```

## 3) Setup route for Open id callback

Create a callback component. The component will be rendered during the time that the callback is made, so feel free to add any loader/spinner if you want.

If you have created your vuex store module as a namespaced module you will have to map the action from the correct namespace.

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/views/OidcCallback.vue

<script>

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

Setup the route with your callback component. Note the meta properties isOidcCallback and isPublic which are required for this route.

```js
// https://github.com/perarnborg/vuex-oidc-example/blob/master/src/router.js

import OidcCallback from '@/components/OidcCallback'

const routes = [
  ...yourApplicationsRoutes,
  {
    path: '/oidc-callback', // Needs to match redirect_uri in you oidcSettings
    name: 'oidcCallback',
    component: OidcCallback,
    meta: {
      isOidcCallback: true,
      isPublic: true
    }
  }
]

``` 

## 4) Setup vue-router

Create the oidc router middleware with factory funtion vuexOidcCreateRouterMiddleware that takes your vuex store as argument.

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

```
router.beforeEach(vuexOidcCreateRouterMiddleware(store, 'oidcStore'))
```

## 5) Optional: Control rendering in app layout or common components

The router middleware will ensure that routes that require authentication are not rendered. If you want to control rendering outside of the router-view you can use the vuex getter oidcIsAuthenticated to check authentication. 

This can be done in any way you want. Here is an example of how to condition rendering against authentication in a component.

```js
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

```js
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

```js
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

Routes with meta.isPublic will not require authentication. If you have setup a silent_redirect_uri a silent signIn will be made on public routes.


## 9) Optional: setup silent renew callback

```js
export const oidcSettings = {
  ...youOidcOtherSettings,
  silent_redirect_uri: 'http://localhost:1337/oidc-silent-renew.html',
  automaticSilentRenew: true // If true oidc-client will try to renew your token when it is about to expire
}

```

You have to make sure that you have an endpoint that matches the silent_redirect_uri setting. It should run the following code:

```js
import { vuexOidcProcessSilentSignInCallback } from 'vuex-oidc'

import { oidcSettings } from '@/config'

vuexOidcProcessSilentSignInCallback(oidcSettings)

```

## 10) Optional: setup oidc-client event listeners

oidc-client has an events api (https://github.com/IdentityModel/oidc-client-js/wiki#events) that you can choose to implement by passing a third argument into vuexOidcCreateStoreModule

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
      // Optionlaly define the store module as namespaced
      { namespaced: true },
      // Optional OIDC event listeners
      {
        userLoaded: (user) => console.log('OIDC user is loaded:', user),
        userUnloaded: () => console.log('OIDC user is unloaded'),
        accessTokenExpiring: () => console.log('Access token will expire'),
        accessTokenExpired: () => console.log('Access token did expire'),
        silentRenewError: () => console.log('OIDC user is unloaded'),
        userSignedOut: () => console.log('OIDC user is signed out')
      }
    )
  }
})

```

If you want to listen to the events from inside your application the events are also dispatched in the browser as custom events by vuex-oidc (prefixed with vuexoidc:):

```
// https://github.com/perarnborg/vuex-oidc-example/tree/master/src

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

# API

The router middleware handles authentication of routes automatically, but there are also actions you can use directly. If you want you can skip the router middleware to customize the authentication and access control behaviour of your app.

```js
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
      'oidcAuthenticationIsChecked'
    ]),
    hasAccess: function() {
      return this.oidcIsAuthenticated || this.$route.meta.isPublic
    }
  },
  methods: {
    ...mapActions([
      'authenticateOidc', // Authenticates with redirect to sign in if not signed in
      'oidcSignInCallback', // Handles callback from authentication redirect
      'authenticateOidcSilent', // Authenticates if signed in. No redirect is made if not signed in
      'getOidcUser', // Update user in store
      'signOutOidc', // Signs out user
    ])
  }
}
</script>
```
