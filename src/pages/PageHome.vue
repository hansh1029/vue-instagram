<template>
  <q-page class="constrain q-pa-md">
    <transition
      appear
      enter-active-class="animated fadeIn"
      leave-active-class="animated fadeOut"
    >
      <div
        v-if="showNotificationsBanner && pushNotificationsSupported"
        class="banner-container bg-primary"
      >
        <div class="constrain">
          <q-banner class="bg-grey-3 q-mb-md">
            <template v-slot:avatar>
              <q-icon name="eva-bell-outline" color="primary" />
            </template>
            Would you like to enable notifications?
            <template v-slot:action>
              <q-btn
                @click="enableNotifications"
                dense
                class="q-px-sm"
                flat
                label="Yes"
                color="primary"
              />
              <q-btn
                @click="showNotificationsBanner = false"
                dense
                class="q-px-sm"
                flat
                label="Later"
                color="primary"
              />
              <q-btn
                @click="neverShowNotificationsBanner"
                dense
                class="q-px-sm"
                flat
                label="Never"
                color="primary"
              />
            </template>
          </q-banner>
        </div>
      </div>
    </transition>
    <div class="row q-col-gutter-lg">
      <div class="col-12 col-sm-8">
        <template v-if="!loadingPosts && posts.length">
          <q-card
            v-for="post in posts"
            :key="post.id"
            class="card-post q-mb-md"
            :class="{ 'bg-red-1': post.offline }"
            flat
            bordered
          >
            <q-badge
              v-if="post.offline"
              class="badge-offline absolute-top-right"
              color="red"
            >
              Stored offline
            </q-badge>
            <q-item>
              <q-item-section avatar>
                <q-avatar>
                  <img src="https://cdn.quasar.dev/img/boy-avatar.png" />
                </q-avatar>
              </q-item-section>

              <q-item-section>
                <q-item-label class="text-bold">melolon_quasagram</q-item-label>
                <q-item-label caption>{{ post.location }}</q-item-label>
              </q-item-section>
            </q-item>

            <q-separator />

            <q-img :src="post.imageUrl" />

            <q-card-section>
              <div>{{ post.caption }}</div>
              <div class="text-caption text-grey">
                {{ post.date | niceDate }}
              </div>
            </q-card-section>
          </q-card>
        </template>
        <template v-else-if="!loadingPosts && !posts.length">
          <h5 class="text-center text-grey">No posts yet.</h5>
        </template>
        <template v-else>
          <q-card flat bordered>
            <q-item>
              <q-item-section avatar>
                <q-skeleton type="QAvatar" animation="fade" size="40px" />
              </q-item-section>

              <q-item-section>
                <q-item-label>
                  <q-skeleton type="text" animation="fade" />
                </q-item-label>
                <q-item-label caption>
                  <q-skeleton type="text" animation="fade" />
                </q-item-label>
              </q-item-section>
            </q-item>

            <q-skeleton height="200px" square animation="fade" />

            <q-card-section>
              <q-skeleton type="text" class="text-subtitle2" animation="fade" />
              <q-skeleton
                type="text"
                width="50%"
                class="text-subtitle2"
                animation="fade"
              />
            </q-card-section>
          </q-card>
        </template>
      </div>
      <div class="col-4 large-screen-only">
        <q-item class="fixed">
          <q-item-section avatar>
            <q-avatar size="48px">
              <img src="https://cdn.quasar.dev/img/boy-avatar.png" />
            </q-avatar>
          </q-item-section>

          <q-item-section>
            <q-item-label class="text-bold">melolon_quasagram</q-item-label>
            <q-item-label caption>Lemon Melolon</q-item-label>
          </q-item-section>
        </q-item>
      </div>
    </div>
  </q-page>
</template>

<script>
import { date } from "quasar";
import { openDB } from "idb";
let qs = require("qs");

export default {
  name: "PageHome",
  data() {
    return {
      posts: [],
      loadingPosts: false,
      showNotificationsBanner: false,
    };
  },
  computed: {
    serviceWorkerSupported() {
      if ("serviceWorker" in navigator) return true;
      return false;
    },
    pushNotificationsSupported() {
      if ("PushManager" in window) return true;
      return false;
    },
  },
  methods: {
    getPosts() {
      this.loadingPosts = true;
      this.$axios
        .get(`${process.env.API}/posts`)
        .then((response) => {
          this.posts = response.data;
          this.loadingPosts = false;
          if (!navigator.onLine) {
            this.getOfflinePosts();
          }
        })
        .catch((err) => {
          // console.log(err);
          this.$q.dialog({
            title: "Error",
            message: "Could not load posts.",
          });
          this.loadingPosts = false;
        });
    },
    getOfflinePosts() {
      let db = openDB("workbox-background-sync").then((db) => {
        db.getAll("requests")
          .then((failedRequests) => {
            failedRequests.forEach((failedRequest) => {
              if (failedRequest.queueName == "createPostQueue") {
                let request = new Request(
                  failedRequest.requestData.url,
                  failedRequest.requestData
                );
                request.formData().then((formData) => {
                  let offlinePost = {};
                  offlinePost.id = formData.get("id");
                  offlinePost.caption = formData.get("caption");
                  offlinePost.location = formData.get("location");
                  offlinePost.date = parseInt(formData.get("date"));
                  offlinePost.offline = true;

                  let reader = new FileReader();
                  reader.readAsDataURL(formData.get("file"));
                  reader.onloadend = () => {
                    offlinePost.imageUrl = reader.result;
                    this.posts.unshift(offlinePost);
                  };
                });
              }
            });
          })
          .catch((err) => {
            console.log("Error accessing IndexedDB: ", err);
          });
      });
    },
    listenForOfflinePostUploaded() {
      if (this.serviceWorkerSupported) {
        const channel = new BroadcastChannel("sw-messages");
        channel.addEventListener("message", (event) => {
          console.log("Received", event.data);
          if (event.data.msg == "offline-post-uploaded") {
            let offlinePostCount = this.posts.filter(
              (post) => post.offline == true
            ).length;
            this.posts[offlinePostCount - 1].offline = false;
          }
        });
      }
    },
    initNotificationsBanner() {
      let neverShowNotificationsBanner = this.$q.localStorage.getItem(
        "neverShowNotificationsBanner"
      );

      if (!neverShowNotificationsBanner) {
        this.showNotificationsBanner = true;
      }
    },
    enableNotifications() {
      if (this.pushNotificationsSupported) {
        Notification.requestPermission((result) => {
          console.log("result: ", result);
          this.neverShowNotificationsBanner();
          if (result == "granted") {
            // this.displayGrantedNotification();
            this.checkForExistingPushSubscription();
          }
        });
      }
    },
    checkForExistingPushSubscription() {
      if (this.serviceWorkerSupported && this.pushNotificationsSupported) {
        let reg;
        navigator.serviceWorker.ready
          .then((swreg) => {
            reg = swreg;
            return swreg.pushManager.getSubscription();
          })
          .then((sub) => {
            if (!sub) {
              this.createPushSubscription(reg);
            }
          });
      }
    },
    createPushSubscription(reg) {
      //PUSH_PUBLIC_KEY is saved in dotenv file and created by web-push generator
      let vapidPublicKey = process.env.PUSH_PUBLIC_KEY;
      console.log("vapidPublicKey:", vapidPublicKey);
      let vapidPublicKeyConverted = this.urlBase64ToUint8Array(vapidPublicKey);
      reg.pushManager
        .subscribe({
          applicationServerKey: vapidPublicKeyConverted,
          userVisibleOnly: true,
        })
        .then((newSub) => {
          //convert push subscription object to javascript object
          let newSubData = newSub.toJSON(),
            newSubDataQS = qs.stringify(newSubData);
          return this.$axios.post(
            `${process.env.API}/createSubscription?${newSubDataQS}`
          );
        })
        .then((response) => {
          console.log("response: ", response);
          this.displayGrantedNotification();
        })
        .catch((err) => {
          console.log("err: ", err);
        });
    },
    displayGrantedNotification() {
      //notification without using service worker
      // new Notification("You're subscribed to notifications!");
      //push notification with using service worker
      if (this.serviceWorkerSupported && this.pushNotificationsSupported) {
        // swreg = service worker regitstration object
        navigator.serviceWorker.ready.then((swreg) => {
          swreg.showNotification("You're subscribed to notifications!", {
            body: "Thanks for subscribing!",
            icon: "icons/icon-128x128.png",
            image: "icons/icon-128x128.png",
            badge: "icons/icon-128x128.png",
            dir: "ltr",
            lang: "en-US",
            vibrate: [100, 50, 200],
            tag: "confirm-notification",
            renotify: true,
            actions: [
              {
                action: "hello",
                title: "Hello",
                icon: "icons/icon-128x128.png",
              },
              {
                action: "goodbye",
                title: "Goodbye",
                icon: "icons/icon-128x128.png",
              },
            ],
          });
        });
      }
    },
    neverShowNotificationsBanner() {
      this.showNotificationsBanner = false;
      this.$q.localStorage.set("neverShowNotificationsBanner", true);
    },
    urlBase64ToUint8Array(base64String) {
      const padding = "=".repeat((4 - (base64String.length % 4)) % 4);
      const base64 = (base64String + padding)
        .replace(/-/g, "+")
        .replace(/_/g, "/");

      const rawData = window.atob(base64);
      const outputArray = new Uint8Array(rawData.length);

      for (let i = 0; i < rawData.length; ++i) {
        outputArray[i] = rawData.charCodeAt(i);
      }
      return outputArray;
    },
  },
  filters: {
    niceDate(value) {
      return date.formatDate(value, "MMMM D h:mmA");
    },
  },
  activated() {
    console.log("activated");
    this.getPosts();
  },
  created() {
    this.listenForOfflinePostUploaded();
    this.initNotificationsBanner();
  },
};
</script>

<style lang="sass">
.card-post
  .badge-offline
    border-top-left-radius: 0 !important
  .q-img
    min-height: 200px
</style>