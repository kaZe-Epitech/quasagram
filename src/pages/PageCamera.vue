<template>
  <q-page class="constrain-more q-pa-md">
    <div class="camera-frame q-pa-md">
      <video 
        v-show='!imageCaptured'
        ref='video'
        class="full-width"
        autoplay
        playsinline
      />
      <canvas 
        v-show='imageCaptured'
        ref="canvas"
        class="full-width"
        height="240px"
      />
    </div>
    <div class="text-center q-pa-md">
      <q-btn
        v-if="hasCameraSupport"
        round
        color="red"
        icon="camera"
        size="lg" 
        @click="captureImage" 
      />
      <q-file
        v-else
        accept="image/*"
        @input="captureImageFallback"
        outlined
        v-model="imageUpload"
        label="Choose an image to upload">
        <template v-slot:prepend>
          <q-icon name="attach_file" />
        </template>
      </q-file>
    </div>
    <div class="row justify-center q-ma-md">
      <q-input
        v-model="post.caption"
        class="col col-sm-6"
        label="Caption" 
        dense />
    </div>
    <div class="row justify-center q-ma-md">
      <q-input
        :loading='locationLoading && locationSupported'
        v-model="post.location"
        class="col col-sm-6"
        label="Location" 
        dense>
        <template v-slot:append>
          <q-btn v-if='!locationLoading' @click="getLocation" round dense flat icon="my_location" />
        </template>
      </q-input>
    </div>
    <div class="row justify-center q-mt-lg">
      <q-btn
        unelevated
        rounded
        color="primary"
        label="Post image" />
    </div>

    <!-- dialog confirm -->
    <q-dialog v-model="confirm" persistent>
      <q-card>
        <q-card-section class="row items-center">
          <q-avatar icon="wrong_location" color="primary" text-color="white" />
          <span class="q-ml-sm">Sorry but we could not find your current location.</span>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn flat label="OK" color="primary" v-close-popup />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script>
import { uid } from 'quasar'
require('md-gum-polyfill')

export default {
  name: 'PageCamera',
  data () {
    return {
      post: {
        id: uid(),
        caption: '',
        location: '',
        photo: null,
        date: Date.now()
      },
      imageCaptured: false,
      hasCameraSupport: true,
      imageUpload: [],
      confirm: false,
      locationLoading: false
    }
  },
  computed: {
    locationSupported () {
      if (geolocation in navigator) 
        return true
      return false
    }
  },
  methods: {
    initCamera () {
      console.log('initialize the camera')
      navigator.mediaDevices.getUserMedia({
        video: true
      }).then(stream => {
        this.$ref.video.srcObject = stream
      }).catch(error => {
        this.hasCameraSupport = false
      })
    },
    captureImage () {
      console.log("capturing an image")
      let video = this.$refs.video
      let canvas = this.$refs.canvas
      canvas.width = video.getBoundingCLientRect().width
      canvas.height = video.getBoundingCLientRect().height
      let context = canvas.getContext('2d')
      context.drawImage(video, 0, 0, canvas.width, canvas.height)
      this.imageCaptured = true
      this.post.photo = this.dataURItoBlob(canvas.toDataUrl())
      this.disableCamera()
    },
    captureImageFallback (file) {
      this.post.photo = file

      let canvas = this.$refs.canvas
      let context = canvas.getContext('2d')
      var reader = new FileReader()
      reader.onload = event => {
        var img = new Image()
        img.onload = () => {
          canvas.width = img.width
          canvas.height = img.height
          context.drawImage(img,0,0)
          this.imageCaptured = true
        }
        img.src = event.target.result
      }
      reader.readAsDataURL(file)  
    },
    disableCamera () {
      this.$refs.video.srcObject.getVideoTracks().foreach(track => {
        track.stop()
      })
    },
    dataURItoBlob (dataURI) {
      // convert base64 to raw binary data held in a string
      // doesn't handle URLEncoded DataURIs - see SO answer #6850276 for code that does this
      var byteString = atob(dataURI.split(',')[1])

      // separate out the mime component
      var mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0]

      // write the bytes of the string to an ArrayBuffer
      var ab = new ArrayBuffer(byteString.length)

      // create a view into the buffer
      var ia = new Uint8Array(ab)

      // set the bytes of the buffer to the correct values
      for (var i = 0; i < byteString.length; i++) {
          ia[i] = byteString.charCodeAt(i)
      }

      // write the ArrayBuffer to a blob, and you're done
      var blob = new Blob([ab], {type: mimeString})
      return blob

    },
    getLocation () {
      this.locationLoading = true
      navigator.geolocation.getCurrentPosition(position => {
        this.getCityAndCountry(position)
      }, err => {
        this.locationError()
      }, { timeout: 7000 })
    },
    getCityAndCountry (position) {
      let apiUrl = `https://geocode.xyz/${ position.coords.latitude },${ position.coords.longitude }?json=1`
      this.$axios.get(apiUrl).then(result => {
        this.locationSuccess(result)
      }).catch(err => {
        this.locationError()
      })
    },
    locationSuccess (result) {
      this.post.location = result.data.city
      if (result.data.country) {
        this.post.location += `, ${ result.data.country }`
      }
      this.locationLoading = false
    },
    locationError () {
      this.confirm = true
      this.locationLoading = false
    }
  },
  mounted () {
    this.initCamera()
  },
  beforeDestroy () {
    if (this.hasCameraSupport)
      this.disableCamera()
  }
}
</script>

<style lang="sass">
.camera-frame
  border: 2px solid $grey-10
  border-radius: 10px
</style>