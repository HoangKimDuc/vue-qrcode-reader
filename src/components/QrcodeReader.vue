<template lang="html">
  <div>
    <div class="qrcode-reader">
      <video ref="video" class="qrcode-reader__camera" @loadeddata="onStreamLoaded"></video>
      <canvas ref="canvas" class="qrcode-reader__snapshot"></canvas>
      <button id="camera-switch" v-if="cameras.length >= 2" @click.stop="facingMode=!facingMode">
        <i class="fa fa-refresh" aria-hidden="true"></i>
      </button>
      <div class="qrcode-reader__overlay">
        <slot name="qr-code-content"></slot>
      </div>
    </div>
  </div>
</template>
<script>
  import {
    scan,
  } from '../scanner.js'

  const SCAN_INTERVAL = 5 // 1000ms / 5ms = 200fps

  export default {
    props: {
      paused: {
        type: Boolean,
        default: false,
      },
      facing: {
        type: Number,
        default: 0,
      },

    },

    data () {
      return {
        cameras: [],
        facingMode: false,
        isfirtChange: true,
        scanLoop: -1, // ID returned by setInterval

        initReject: null,
        initResolve: null,

        stream: null,

        content: null,
        location: null,
      }
    },

    computed: {

      points () {
        if (this.location === null) {
          return []
        } else {
          const video = this.$refs.video

          const widthRatio = video.offsetWidth / video.videoWidth
          const heightRatio = video.offsetHeight / video.videoHeight

          const points = [
            this.location.bottomLeft,
            this.location.topLeft,
            this.location.topRight,
          ]

          return points.map(({
            x,
            y,
          }) => ({
            x: x * widthRatio,
            y: y * heightRatio,
          }))
        }
      },

    },

    watch: {
      paused (newValue) {
        if (newValue === true) {
          this.$refs.video.pause()
          this.stopScanning()
        } else {
          this.$refs.video.play()
          this.startScanning()
        }
      },

      content (newValue) {
        this.$emit('decode', newValue)
      },

      points (newValue) {
        this.$emit('locate', newValue)
      },
    },

    mounted () {
      const initPromise = new Promise(
        (resolve, reject) => {
          this.initResolve = resolve
          this.initReject = reject
        }
      )
      this.$emit('init', initPromise)
      const canvas = this.$refs.canvas
      if (!(canvas.getContext && canvas.getContext('2d'))) {
        this.initReject(new Error('HTML5 Canvas not supported in this browser.'))
      } else if (!(navigator.mediaDevices && navigator.mediaDevices.getUserMedia)) {
        this.initReject(new Error('WebRTC API not supported in this browser'))
      } else {
        this.startCamera()
      }
    },

    beforeDestroy () {
      this.stopCamera()
      this.stopScanning()
    },

    methods: {
      async startCamera () {
        await navigator.mediaDevices.enumerateDevices()
          .then(this.gotDevices).then(this.getStream).catch(this.handleError)
      },
      handleError (error) {
        console.log('Error: ', error)
      },
      gotDevices (deviceInfos) {
        //  label:"camera 1, facing front"
        this.cameras = []
        for (var i = 0; i !== deviceInfos.length; ++i) {
          var deviceInfo = deviceInfos[i]
          if (deviceInfo.kind === 'videoinput') {
            this.cameras.push(deviceInfo)
            console.log(deviceInfo)
          }
        }
      },
      gotStream (stream) {
        this.stream = stream
        const video = this.$refs.video
        if (video.srcObject !== undefined) {
          video.srcObject = this.stream
        } else if (video.mozSrcObject !== undefined) {
          video.mozSrcObject = this.stream
        } else if (window.URL.createObjectURL) {
          video.src = window.URL.createObjectURL(this.stream)
        } else if (window.webkitURL) {
          video.src = window.webkitURL.createObjectURL(this.stream)
        } else {
          video.src = this.stream
        }
        video.playsInline = true
        video.play()
      },
      async getStream () {
        if (this.stream) {
          this.stream.getTracks().forEach(function (track) {
            track.stop()
          })
        }
        let id = null
        id = this.cameras[this.facing].deviceId
        var constraints = {
          audio: false,
          video: {
            facingMode: 'user',
            deviceId: {
              exact: id,
            },
            width: {
              min: 360,
              ideal: 1280,
              max: 1920,
            },
            height: {
              min: 240,
              ideal: 720,
              max: 1080,
            },
          },
        }
        await navigator.mediaDevices.getUserMedia(constraints).then(this.gotStream).catch(this.handleError)
      },
      stopCamera () {
        if (this.stream) {
          this.stream.getTracks().forEach(
            track => track.stop()
          )
        }
      },

      captureFrame () {
        const video = this.$refs.video
        const canvas = this.$refs.canvas

        canvas.width = video.videoWidth
        canvas.height = video.videoHeight
        if (canvas.width === 0) {
          return 0
        }
        const ctx = canvas.getContext('2d')
        const bounds = [0, 0, canvas.width, canvas.height]
        ctx.drawImage(video, ...bounds)
        return ctx.getImageData(...bounds)
      },
      startScanning () {
        this.stopScanning()
        this.scanLoop = window.setInterval(() => {
          const imageData = this.captureFrame()
          window.requestAnimationFrame(() => {
            if (imageData !== 0) {
              const {
                content,
                location,
              } = scan(imageData)
              if (content !== null) {
                this.content = content
              }
              this.location = location
            }
          })
        }, SCAN_INTERVAL)
      },
      stopScanning () {
        window.clearInterval(this.scanLoop)
      },
      onStreamLoaded () { // first frame finished loading
        this.initResolve()
        this.startScanning()
      },
    },
  }

</script>

<style>
.qrcode-reader {
  position: relative;
  display: block;
}

.qrcode-reader__camera {
  display: block;
  object-fit: contain;
  max-width: 100%;
  max-height: 100%;
}

.qrcode-reader__snapshot {
  display: none;
}

.qrcode-reader__overlay {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
}

#camera-switch {
  position: fixed;
  z-index: 9999 !important;
  background: transparent !important;
  color: #fff;
  font-size: 2em;
  padding: 10px 20px;
  top: 50px;
  right: 20px;
}
</style>
