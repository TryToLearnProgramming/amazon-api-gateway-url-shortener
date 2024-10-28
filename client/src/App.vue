<!-- Copyright 2019 Amazon.com, Inc. or its affiliates. All Rights Reserved.
SPDX-License-Identifier: MIT-0

Permission is hereby granted, free of charge, to any person obtaining a copy of this
software and associated documentation files (the "Software"), to deal in the Software
without restriction, including without limitation the rights to use, copy, modify,
merge, publish, distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED,
INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A
PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE. -->

<template>
  <div id="app">
    <section class="section">
      <nav class="navbar is-fixed-top" role="navigation" aria-label="main navigation">
        <div class="navbar-background"></div>
        <div class="navbar-content">
          <div class="navbar-brand">
            <div class="navbar-item">
              <h1 class="title has-text-white">{{ appName }}</h1>
            </div>
            <a
              role="button"
              class="navbar-burger burger"
              aria-label="menu"
              aria-expanded="false"
              data-target="navbarCollapse"
              v-bind:class="{ 'is-active': isOpen }"
              @click="isOpen = !isOpen"
            >
              <span aria-hidden="true"></span>
              <span aria-hidden="true"></span>
              <span aria-hidden="true"></span>
            </a>
          </div>

          <div
            id="navbarCollapse"
            class="navbar-menu"
            v-bind:class="{ 'is-active': isOpen }"
          >
            <div class="navbar-end">
              <a
                class="navbar-item auth-button"
                v-if="authorized"
                v-on:click="logout()"
                v-bind:href="logOutUrl"
              >Log Out</a>
              <a 
                class="navbar-item auth-button" 
                v-if="!authorized" 
                v-bind:href="signUpUrl"
              >Sign up</a>
              <a 
                class="navbar-item auth-button" 
                v-if="!authorized" 
                v-bind:href="logInUrl"
              >Log in</a>
            </div>
          </div>
        </div>
      </nav>
    </section>
    <section class="section">
      <div class="container">
        <div v-if="authorized">
          <router-view />
        </div>
        <div v-else>
          <h1 class="title">Welcome to {{ appName }}</h1>
          <h2 class="subtitle">Shorten your urls with ease.</h2>
          <p v-if="linkNotFound">
            We're sorry, that link could not be found.
            <a v-bind:href="signUpUrl">Sign up</a> or
            <a v-bind:href="logInUrl">Log in</a> to register it?
          </p>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { mapState } from "vuex";
import axios from "axios";

// Set Cognito Vars
const clientId = process.env.VUE_APP_CLIENT_ID;
const authDomain = process.env.VUE_APP_AUTH_DOMAIN;
const queryStringParams = new URLSearchParams(window.location.search);
const cognitoCode = queryStringParams.get("code") || null;
const lnf = queryStringParams.get("lnf") || null;
const redUrl = window.location.origin;

export default {
  name: "app",
  data() {
    return {
      appName: `Url Shortener ${process.env.VUE_APP_NAME}`,
      signUpUrl: `${authDomain}/signup?response_type=code&client_id=${clientId}&redirect_uri=${redUrl}`,
      logInUrl: `${authDomain}/login?response_type=code&client_id=${clientId}&redirect_uri=${redUrl}`,
      logOutUrl: `${authDomain}/logout?client_id=${clientId}&logout_uri=${redUrl}`,
      linkNotFound: lnf,
      isOpen: false,
    };
  },
  created() {
    if (cognitoCode) this.exchangeToken();
    else this.exchangeRefreshToken();
  },
  methods: {
    convertJSON: function (json) {
      const oAuthTokenBodyArray = Object.entries(json).map(([key, value]) => {
        const encodedKey = encodeURIComponent(key);
        const encodedValue = encodeURIComponent(value);
        return `${encodedKey}=${encodedValue}`;
      });
      return oAuthTokenBodyArray.join("&");
    },
    exchangeRefreshToken: function () {
      const oauthTokenBodyJson = {
        grant_type: "refresh_token",
        client_id: clientId,
        refresh_token: localStorage.getItem("cognitoRefreshToken"),
      };
      const oauthTokenBody = this.convertJSON(oauthTokenBodyJson);

      if (oauthTokenBodyJson.refresh_token) {
        return axios
          .post(`${authDomain}/oauth2/token`, oauthTokenBody, {
            ["Content-Type"]: "application/x-www-form-urlencoded",
          })
          .then((response) => {
            let json = response.data;
            if (json.id_token) {
              localStorage.setItem("cognitoIdentityToken", json.id_token);
              this.$store.commit("authorize");
            }
          })
          .catch(() => {
            this.$store.commit("deAuthorize");
          });
      } else {
        return new Promise((res) => {
          return res({});
        });
      }
    },
    exchangeToken: function () {
      const oauthTokenBodyJson = {
        grant_type: "authorization_code",
        client_id: clientId,
        code: cognitoCode,
        redirect_uri: redUrl,
      };
      const oauthTokenBody = this.convertJSON(oauthTokenBodyJson);

      return axios
        .post(`${authDomain}/oauth2/token`, oauthTokenBody, {
          ["Content-Type"]: "application/x-www-form-urlencoded",
        })
        .then((response) => {
          let json = response.data;
          if (json.id_token) {
            localStorage.setItem("cognitoIdentityToken", json.id_token);
            localStorage.setItem("cognitoRefreshToken", json.refresh_token);
            this.$store.commit("authorize");
          }
          let query = Object.assign({}, this.$route.query);
          delete query.code;
          this.$router.replace({ query });
        })
        .catch(() => {
          this.$store.commit("deAuthorize");
        });
    },
    logout: function () {
      localStorage.setItem("cognitoIdentityToken", null);
      localStorage.setItem("cognitoRefreshToken", null);
    },
  },
  computed: {
    ...mapState(["authorized"]),
  },
};
</script>

<style scoped>
.navbar {
  position: relative;
  padding: 0;
  min-height: 4rem;
}

.navbar-background {
  position: absolute;
  inset: 0;
  background: linear-gradient(to right, #57b3de, #3498db);  
  box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1);
}

.navbar-content {
  position: relative;
  width: 100%;
  padding: 0 1rem;
  height: 4rem;
  display: flex;
  align-items: center;
}

.navbar-brand {
  height: 4rem;
  display: flex;
  align-items: center;
  flex: 1;
}

.navbar-menu {
  background: transparent;
  flex: none;
  padding: 0;
  margin: 0;
}

.navbar-end {
  display: flex;
  align-items: center;
  height: 4rem;
}

.auth-button {
  background-color: rgba(255, 255, 255, 0.1);
  color: white !important;
  border-radius: 0.375rem;
  margin: 0.5rem;
  transition: background-color 0.2s;
  height: auto;
}

.auth-button:hover {
  background-color: rgba(255, 255, 255, 0.2) !important;
}

.navbar-burger {
  color: white;
}

.navbar-menu {
  background: transparent;
}

@media screen and (max-width: 1023px) {
  .navbar-menu.is-active {
    background: linear-gradient(to right, #57b3de, #3498db); 
  }
}
</style>
