<template>
  <q-page class="constrain-more q-pa-md">
    <div class="camera-frame q-pa-md">
      <video 
        v-show="!imageCaptured"
        ref="video"
        class="full-width"
        autoplay
        playsinline
      />
      <canvas
        v-show="imageCaptured"
        ref="canvas"
        class="full-width"
        height="240"
      />
    </div>
    <div class="text-center q-pa-md">
      <q-btn
        v-if="hasCameraSupport"
        @click="captureImage"
        :disable="imageCaptured"
        round
        color="grey-10"
        size="lg"
        icon="eva-camera"
      />
      <q-file
        v-else
        v-model="imageUpload"
        @input="captureImageFallback"
        label="Choose an image"
        accept="image/*"
        outlined
      >
        <template v-slot:prepend>
          <q-icon name="eva-attach-outline" />
        </template>
      </q-file>
      <div class="row justify-center q-ma-md">
        <q-input
          v-model="post.caption"
          class="col col-sm-6"
          label="Caption *"
          dense
        />
      </div>
      <div class="row justify-center q-ma-md">
        <q-input
          v-model="post.location"
          :loading="locationLoading"
          class="col col-sm-6"
          label="Location"
          dense
        >
          <template v-slot:append>
            <q-btn
              v-if="!locationLoading && locationSupported"
              @click="getLocation"
              round
              dense
              flat
              icon="eva-navigation-2-outline"
            />
          </template>
        </q-input>
      </div>
      <div class="row justify-center q-mt-lg">
        <q-btn
          @click="addPost()"
          :disable="!post.caption || !post.photo"
          class="q-mb-lg"
          unelevated
          rounded
          color="primary"
          label="Post Image"
        />
      </div>
    </div>
  </q-page>
</template>

<script>
import { uid } from "quasar";
//old version browser can use getUserMedia with md-gum-polyfill
require("md-gum-polyfill");

export default {
  name: "PageCamera",
  data() {
    return {
      post: {
        id: uid(),
        caption: "",
        location: "",
        photo: null,
        date: Date.now(),
      },
      imageCaptured: false,
      imageUpload: [],
      hasCameraSupport: true,
      locationLoading: false,
    };
  },
  computed: {
    locationSupported() {
      if ("geolocation" in navigator) return true;
      return false;
    },
    backgroundSyncSupported() {
      if ("serviceWorker" in navigator && "SyncManager" in window) return true;
      return false;
    },
  },
  methods: {
    initCamera() {
      navigator.mediaDevices
        .getUserMedia({
          video: true,
        })
        .then((stream) => {
          this.$refs.video.srcObject = stream;
        })
        .catch((err) => {
          this.hasCameraSupport = false;
          this.$q.dialog({
            title: "Alert",
            message: "Cannot use a camera.",
          });
        });
    },
    captureImage() {
      let video = this.$refs.video;
      let canvas = this.$refs.canvas;
      canvas.width = video.getBoundingClientRect().width;
      canvas.height = video.getBoundingClientRect().height;
      let context = canvas.getContext("2d");
      context.drawImage(video, 0, 0, canvas.width, canvas.height);
      this.imageCaptured = true;
      this.post.photo = this.dataURItoBlob(canvas.toDataURL());
      this.disableCamera();
    },
    captureImageFallback(file) {
      this.post.photo = file;

      let canvas = this.$refs.canvas;
      let context = canvas.getContext("2d");

      var reader = new FileReader();
      reader.onload = (event) => {
        var img = new Image();
        img.onload = () => {
          canvas.width = img.width;
          canvas.height = img.height;
          context.drawImage(img, 0, 0);
          this.imageCaptured = true;
        };
        img.src = event.target.result;
      };
      reader.readAsDataURL(file);
    },
    disableCamera() {
      this.$refs.video.srcObject.getVideoTracks().forEach((track) => {
        track.stop();
      });
    },
    dataURItoBlob(dataURI) {
      // convert base64 to raw binary data held in a string
      // doesn't handle URLEncoded DataURIs - see SO answer #6850276 for code that does this
      var byteString = atob(dataURI.split(",")[1]);

      // separate out the mime component
      var mimeString = dataURI.split(",")[0].split(":")[1].split(";")[0];

      // write the bytes of the string to an ArrayBuffer
      var ab = new ArrayBuffer(byteString.length);

      // create a view into the buffer
      var ia = new Uint8Array(ab);

      // set the bytes of the buffer to the correct values
      for (var i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i);
      }

      // write the ArrayBuffer to a blob, and you're done
      var blob = new Blob([ab], { type: mimeString });
      return blob;
    },
    getLocation() {
      this.locationLoading = true;
      navigator.geolocation.getCurrentPosition(
        (position) => {
          this.getCityAndCountry(position);
        },
        (err) => {
          this.locationError();
        },
        { timeout: 7000 }
      );
    },
    getCityAndCountry(position) {
      let apiUrl = `https://geocode.xyz/${position.coords.latitude},${position.coords.longitude}?json=1`;
      this.$axios
        .get(apiUrl)
        .then((result) => {
          this.locationSuccess(result);
        })
        .catch((err) => {
          this.locationError();
        });
    },
    locationSuccess(result) {
      this.post.location = result.data.city;
      if (result.data.country) {
        this.post.location += `, ${result.data.country}`;
      }
      this.locationLoading = false;
    },
    locationError() {
      this.$q.dialog({
        title: "Error",
        message: "Could not find your location.",
      });
      this.locationLoading = false;
    },
    addPost() {
      this.$q.loading.show();

      let postCreated = this.$q.localStorage.getItem("postCreated");

      if (this.$q.platform.is.android && !postCreated && !navigator.onLine) {
        this.addPostError();
        this.$q.loading.hide();
      } 
      else {
        let formData = new FormData();
        formData.append("id", this.post.id);
        formData.append("caption", this.post.caption);
        formData.append("location", this.post.location);
        formData.append("date", this.post.date);
        formData.append("file", this.post.photo, this.post.id + ".png");

        this.$axios
          .post(`${process.env.API}/createPost`, formData)
          .then((response) => {
            // console.log("response", response);
            this.$router.push("/");
            this.$q.notify({
              message: "Post created!",
              actions: [
                {
                  label: "Dismiss",
                  color: "white",
                },
              ],
            });
            this.$q.loading.hide();
            if (this.$q.platform.is.safari) {
              setTimeout(() => {
                window.location.href = "/";
              }, 1000);
            }
          })
          .catch((err) => {
            //console.log("err", err);
            if (!navigator.onLine && this.backgroundSyncSupported) {
              // redirect to the home page
              this.$q.notify("Post created offline");
              this.$router.push("/");
            } else {
              this.addPostError();
            }
            this.$q.loading.hide();
          });
      }
    },
    addPostError() {
      this.$q.dialog({
        title: "Error",
        message: "Sorry, could not create post!",
      });
    },
  },
  mounted() {
    this.initCamera();
  },
  beforeDestroy() {
    if (this.hasCameraSupport) {
      this.disableCamera();
    }
  },
};
</script>

<style lang="sass">
.camera-frame
  border: 2px solid $grey-10
  border-radius: 10px
</style>