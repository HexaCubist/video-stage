<template>
  <div id="wrapper" class="row">
    <header class="col-md-3">
      <h1>Video Stage</h1>
      <a href="/" class="btn btn-large btn-warning">Restart</a>
    </header>
    <main class="offset-md-3 col-md-9">
      <div class="intro">
        <p>
          Welcome to Video Stage. To use this app, you will need one or more DMX lights (RGB) as well as an e1.31 adapter to connect them to your computer or network.
          Begin by configuring your DMX devices below:
        </p>
      </div>
      <div class="dmx-config row">
        <div
        v-for="(device, index) in dmxconf.devices"
        :key="'device-' + index"
        class="col-xl-3 col-lg-4 col-sm-6">
          <div
          :class="{'conflicting': channelConflict (device, index)}"
          class="dmx-device">
            <input type="text" class="form-control" id="name" placeholder="Enter name" v-model="device.name">
            <div class="form-group">
              <label for="unverse">Universe</label>
              <select class="form-control" id="unverse" v-model.number="device.universe">
                <option v-for="n in dmxconf.universes" :key="'uniOpt-' + n">{{n}}</option>
              </select>
            </div>
            <div class="form-group">
              <label for="unverse">Start Channel</label>
              <input type="number" class="form-control" id="numCh" placeholder="# of channels" v-model.number="device.startCh" min="1" max="512">
            </div>
            <div class="form-group">
              <label for="numCh">Number of channels</label>
              <input type="number" class="form-control" id="numCh" placeholder="# of channels" v-model.number="device.numCh">
            </div>
            <div class="form-group">
              <label>R G B Channel offsets</label>
              <div class="input-group">
                <input type="number" class="form-control" placeholder="R" v-model.number="device.rgb[0]" min="0" :max="device.numCh - 1">
                <input type="number" class="form-control" placeholder="G" v-model.number="device.rgb[1]" min="0" :max="device.numCh - 1">
                <input type="number" class="form-control" placeholder="B" v-model.number="device.rgb[2]" min="0" :max="device.numCh - 1">
              </div>
            </div>
            <button class="btn btn-danger dmx-delete" @click="delDMX(index)">Delete</button>
          </div>
        </div>
        <div class="col-xl-3 col-lg-4 col-sm-6">
          <div class="dmx-device dmx-device-new" @click="addDMX()">
            <div>
              <span>+</span>
              <h3>Create Device</h3>
            </div>
          </div>
        </div>
      </div>
      <div class="video-config">
        <p>Once you have configured your DMX devices, drag a video here or select one by clicking on the area to begin playing. DMX colours will be set to the average of the entire video feed.</p>
        <div
        @dragover="dragover"
        @dragleave="dragleave"
        @dragend="dragleave"
        @drop="loadFile"
        @click="onclick"
        id="dropzone-area-1">
          <p class="info-text animate">Drop Files Here</p>
        </div>
        <div>
          <h3>DMX Feed</h3>
          <video id="video-hidden"
          muted
          width="320"
          height="180"></video>
          <h3>Screen Feed</h3>
          <div id="video-container"><video id="video"
          controls
          width="320"
          height="180"></video></div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import SystemInformation from './LandingPage/SystemInformation'
const FastAverageColor = require('fast-average-color')
const {dialog} = require('electron').remote
var e131 = require('e131')

// Vue Data Element
console.log(window.vue)

var client = new e131.Client('127.0.0.1')
var packet = client.createPacket(24)
var slotsData = packet.getSlotsData()
packet.setSourceName('test E1.31 client')
packet.setUniverse(0x01) // make universe number consistent with the client
// packet.setOption(packet.Options.PREVIEW, true) // don't really change any fixture
packet.setPriority(packet.DEFAULT_PRIORITY) // not strictly needed, done automatically

export default {
  name: 'landing-page',
  components: { SystemInformation },
  methods: {
    open (link) {
      this.$electron.shell.openExternal(link)
    },
    addDMX () {
      let lastNumCh = 8
      let lastRGB = [0, 1, 2]
      let uniStartCh = new Array(this.dmxconf.universes).fill(1)
      let defaultStartCh = 1
      let defaultStartUni = 1
      for (let index = 0; index < this.dmxconf.devices.length; index++) { // Get max start ch for each universe
        const device = this.dmxconf.devices[index]
        uniStartCh[device.universe - 1] = Math.max(uniStartCh[device.universe - 1], device.startCh + device.numCh)
        lastNumCh = device.numCh
        lastRGB = device.rgb
      }
      console.log(uniStartCh)
      // Get first non-full universe
      for (let i = 0; i < uniStartCh.length; i++) {
        if (uniStartCh[i] + lastNumCh < 512) {
          defaultStartCh = uniStartCh[i]
          defaultStartUni = i + 1 // 0 indexed to 1 indexed
          break
        }
      }
      this.dmxconf['devices'].push({
        name: 'Click to edit', // Device name (flavor)
        universe: defaultStartUni, // Universe number (1-indexed)
        startCh: defaultStartCh, // The first channel used for this device
        numCh: lastNumCh, // The number of channels this device uses
        rgb: lastRGB // The R,G,B channels of this device (relative to ch start)
      })
    },
    delDMX (index) {
      this.dmxconf.devices.splice(index, 1)
    },
    channelConflict (deviceA, deviceindex) {
      // Returns true or false depending on whether channel is in use
      // If device is given ignore channels used by current device
      for (let index = 0; index < this.dmxconf.devices.length; index++) {
        const deviceB = this.dmxconf.devices[index]
        if (deviceA.universe !== deviceB.universe || index === deviceindex) continue

        if (deviceA.startCh <= deviceB.startCh + deviceB.numCh - 1 && deviceA.startCh + deviceA.numCh - 1 >= deviceB.startCh) {
          console.log(deviceA, deviceB)
          return true // if there is overlap return that there is a conflict
        }
      }
      return false
    },
    dragover (e) {
      e.preventDefault()
      var holder = document.getElementById('dropzone-area-1')
      holder.classList.add('dragover')
      console.log(e)
      return false
    },
    dragleave () {
      var holder = document.getElementById('dropzone-area-1')
      holder.classList.remove('dragover')
      return false
    },
    loadFile (e) {
      e.preventDefault()
      var holder = document.getElementById('dropzone-area-1')
      holder.classList.remove('dragover')

      let filepaths = []

      for (let f of e.dataTransfer.files) {
        console.log('File(s) you dragged here: ', f.path)
        if (f.type !== 'video/mp4') {
          holder.classList.add('invalid')
          holder.classList.remove('valid')
          return false
        }
        filepaths.push(f.path)
      }
      holder.classList.remove('invalid')
      holder.classList.add('valid')

      this.playVideo(holder, filepaths[0])

      return false
    },
    onclick (e) {
      e.preventDefault()
      var holder = document.getElementById('dropzone-area-1')
      holder.classList.remove('dragover')

      let file = dialog.showOpenDialog({
        title: 'Select Video to play',
        filters: [{ name: 'Video files', extensions: ['mp4'] }],
        properties: ['createDirectory', 'openFile']
      })
      console.log(file)

      if (!file) { return }

      console.log(file)

      holder.classList.remove('invalid')
      holder.classList.add('valid')

      // Get started!
      console.log('Getting started with this file:')
      console.log(file)
      this.playVideo(holder, file[0])
    },
    playVideo (holder, src) {
      let that = this

      let videoHidden = document.getElementById('video-hidden')
      let video = document.getElementById('video')
      let container = document.getElementById('video-container')
      let source = document.createElement('source')
      let source2 = document.createElement('source')

      source.setAttribute('src', 'file://' + src)
      source2.setAttribute('src', 'file://' + src)

      videoHidden.appendChild(source)
      video.appendChild(source2)
      videoHidden.play()
      video.pause()

      video.addEventListener('timeupdate', () => {
        // Compare times and sync if needed (rounded to 0.5s increments)
        let timeA = Math.ceil(video.currentTime / 0.5) * 0.5
        let timeB = Math.ceil((videoHidden.currentTime - 0.35) / 0.5) * 0.5
        if (timeA !== timeB) {
          console.log(`Forcing time match to: ${video.currentTime + 0.35} (Shown video time: + ${timeA} + , Hidden video time: ${timeB}`)
          videoHidden.currentTime = video.currentTime + 0.35
        }
      })

      video.addEventListener('pause', () => videoHidden.pause())
      video.addEventListener('play', () => videoHidden.play())

      window.setTimeout(
        () => that.Ambilight4Sides_init(videoHidden, container),
        100
      )

      // TODO: Make configurable delay
      window.setTimeout(
        () => document.getElementById('video').play(),
        350
      )
    },
    Ambilight4Sides_init: function (video, container) {
      this.Ambilight4Sides.ac = new FastAverageColor()
      this.Ambilight4Sides.video = video
      this.Ambilight4Sides.container = container

      this.Ambilight4Sides_radius = '200px'
      this.Ambilight4Sides_delta = '200px'
      this.Ambilight4Sides_size = 70

      this.Ambilight4Sides_bindEvents()

      !video.paused && this.Ambilight4Sides.onplay()
    },
    Ambilight4Sides_destroy: function () {
      if (this.Ambilight4Sides.video) {
        this.Ambilight4Sides.video.removeEventListener('play', this.Ambilight4Sides.onplay, false)
        this.Ambilight4Sides.video.removeEventListener('pause', this.Ambilight4Sides.onpause, false)
        this.Ambilight4Sides.container.style.boxShadow = 'none'
      }

      window.cancelAnimationFrame(this.Ambilight4Sides.requestId)

      delete this.Ambilight4Sides.ac
      delete this.Ambilight4Sides.video
      delete this.Ambilight4Sides.container
    },
    Ambilight4Sides_bindEvents: function () {
      var that = this
      console.log(that.Ambilight4Sides.video)
      console.log(that.Ambilight4Sides.video.videoWidth)

      this.Ambilight4Sides.onplay = function () {
        that.Ambilight4Sides.width = that.Ambilight4Sides.video.videoWidth
        that.Ambilight4Sides.height = that.Ambilight4Sides.video.videoHeight

        that.Ambilight4Sides.requestId = window.requestAnimationFrame(that.Ambilight4Sides_updateBoxShadows.bind(that))
      }

      this.Ambilight4Sides.onpause = function () {
        window.cancelAnimationFrame(that.Ambilight4Sides.requestId)
      }

      this.Ambilight4Sides.video.addEventListener('play', this.Ambilight4Sides.onplay, false)
      this.Ambilight4Sides.video.addEventListener('pause', this.Ambilight4Sides.onpause, false)
    },
    Ambilight4Sides_updateBoxShadows: function () {
      var ac = this.Ambilight4Sides.ac
      var width = this.Ambilight4Sides.width
      var height = this.Ambilight4Sides.height
      var size = this.size
      var video = this.Ambilight4Sides.video
      var colorTop = ac.getColor(video, {
        left: 0,
        top: 0,
        height: size,
        width: width
      })
      var colorRight = ac.getColor(video, {
        left: width - size,
        top: 0,
        width: size,
        height: height
      })
      var colorLeft = ac.getColor(video, {
        left: 0,
        top: 0,
        width: size,
        height: height
      })
      var colorBottom = ac.getColor(video, {
        left: 0,
        top: height - size,
        width: width,
        height: size
      })
      var colorFull = ac.getColor(video)

      for (let index = 0; index < this.dmxconf.devices.length; index++) {
        const device = this.dmxconf.devices[index]
        slotsData[0] = 255
        slotsData[device.startCh + device.rgb[0] - 1] = colorFull.value[0]
        slotsData[device.startCh + device.rgb[1] - 1] = colorFull.value[1]
        slotsData[device.startCh + device.rgb[2] - 1] = colorFull.value[2]
      }

      client.send(packet)

      this.Ambilight4Sides.container.style.boxShadow = [
        '0 -{delta} {radius} ' + colorTop.rgb,
        '{delta} 0 {radius} ' + colorRight.rgb,
        '0 {delta} {radius} ' + colorBottom.rgb,
        '-{delta} 0 {radius} ' + colorLeft.rgb
      ]
        .join(', ')
        .replace(/\{delta\}/g, this.delta)
        .replace(/\{radius\}/g, this.radius)

      this.Ambilight4Sides.requestId = window.requestAnimationFrame(this.Ambilight4Sides_updateBoxShadows.bind(this))
    }
  },
  data () {
    return {
      dmxconf: {
        universes: 4,
        devices: []
      },
      Ambilight4Sides: {}
    }
  },
  computed: {
    disabledChs () {
      let universes = this.dmxconf.universes
      var channels = new Array(universes)
      for (var i = 0; i < universes; i++) { channels[i] = new Array(512).fill(1) }
      // Channels is now a 2d array of universes and their channels
      for (let index = 0; index < this.dmxconf.devices.length; index++) {
        const device = this.dmxconf.devices[index]
        channels[device.universe - 1].fill(0, device.startCh - 1, device.startCh + device.numCh - 1)
      }
      return channels
    }
  }
}

// Animation
/* eslint no-unused-vars: ["error", { "varsIgnorePattern": "Ambilight" }] */
// var Ambilight4Sides = {
//   init: function (video, container) {
//     this._ac = new FastAverageColor()
//     this._video = video
//     this._container = container

//     this.radius = '200px'
//     this.delta = '200px'
//     this.size = 70

//     this.bindEvents()

//     !video.paused && this._onplay()
//   },
//   destroy: function () {
//     if (this._video) {
//       this._video.removeEventListener('play', this._onplay, false)
//       this._video.removeEventListener('pause', this._onpause, false)
//       this._container.style.boxShadow = 'none'
//     }

//     window.cancelAnimationFrame(this._requestId)

//     delete this._ac
//     delete this._video
//     delete this._container
//   },
//   bindEvents: function () {
//     var that = this
//     console.log(that.Ambilight4Sides.video)
//     console.log(that.Ambilight4Sides.video.videoWidth)

//     this._onplay = function () {
//       that.Ambilight4Sides.width = that.Ambilight4Sides.video.videoWidth
//       that.Ambilight4Sides.height = that.Ambilight4Sides.video.videoHeight

//       that.Ambilight4Sides.requestId = window.requestAnimationFrame(that.updateBoxShadows.bind(that))
//     }

//     this._onpause = function () {
//       window.cancelAnimationFrame(that.Ambilight4Sides.requestId)
//     }

//     this._video.addEventListener('play', this._onplay, false)
//     this._video.addEventListener('pause', this._onpause, false)
//   },
//   updateBoxShadows: function () {
//     var ac = this._ac
//     var width = this._width
//     var height = this._height
//     var size = this.size
//     var video = this._video
//     var colorTop = ac.getColor(video, {
//       left: 0,
//       top: 0,
//       height: size,
//       width: width
//     })
//     var colorRight = ac.getColor(video, {
//       left: width - size,
//       top: 0,
//       width: size,
//       height: height
//     })
//     var colorLeft = ac.getColor(video, {
//       left: 0,
//       top: 0,
//       width: size,
//       height: height
//     })
//     var colorBottom = ac.getColor(video, {
//       left: 0,
//       top: height - size,
//       width: width,
//       height: size
//     })
//     var colorFull = ac.getColor(video)

//     // Get window.vue data
//     let dmxconf = window.vue.dmxconf.devices

//     for (let index = 0; index < dmxconf.devices.length; index++) {
//       const device = dmxconf.devices[index]
//       console.log(device)
//       slotsData[0] = 255
//       slotsData[device.startCh + device.rgb[0]] = colorFull.value[0]
//       slotsData[device.startCh + device.rgb[1]] = colorFull.value[1]
//       slotsData[device.startCh + device.rgb[2]] = colorFull.value[2]
//     }

//     client.send(packet)

//     this._container.style.boxShadow = [
//       '0 -{delta} {radius} ' + colorTop.rgb,
//       '{delta} 0 {radius} ' + colorRight.rgb,
//       '0 {delta} {radius} ' + colorBottom.rgb,
//       '-{delta} 0 {radius} ' + colorLeft.rgb
//     ]
//       .join(', ')
//       .replace(/\{delta\}/g, this.delta)
//       .replace(/\{radius\}/g, this.radius)

//     this._requestId = window.requestAnimationFrame(this.Ambilight4Sides_updateBoxShadows.bind(this))
//   }
// }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Space+Mono');

  body {
    font-family: 'Space Mono', monospace;
    background-color: #fffffa !important;
  }

  #wrapper {
    width: 100%;
  }

  header {
      background: rgb(0, 12, 24);
      color: white;
      height: 100vh;
      text-align: center;
      padding: 0.5em;
      padding-top: 2em;
      position: fixed !important;
      z-index: 1;
  }

  main {
      color: rgb(0, 12, 24);
      padding: 5em 0 !important;
  }

  .dmx-config > div {
    padding-left: 5px;
    padding-right: 5px;
    margin-bottom: 10px;
  }
  
  .dmx-device {
      background: white;
      border: 4px solid rgb(255, 227, 142);
      padding: 1em;
      transition: 0.3s ease all;
      box-shadow: 0 0 0px rgba(220, 53, 69, 0.45);
      height: 450px;
  }

  .dmx-device.conflicting {
    border-color: #dc3545;
    box-shadow: 0 0 20px rgba(220, 53, 69, 0.45);
  }

  .dmx-device.dmx-device-new {
      text-align: center;
      user-select: none;
      cursor: pointer;
      display:flex;
      justify-content:center;
      align-items:center;
  }
  
  .dmx-device.dmx-device-new span {
    font-size: 100px;
    margin-top: -65px;
    display: block;
  }

  .dmx-device input#name {
    border: 0;
    padding: 0 1em;
    position: relative;
    top: -1em;
    background: rgb(255, 227, 142);
    border-radius: 0;
    text-align: center;
    transition: 0.3s ease all;
  }

  .dmx-device .dmx-delete {
    float: right;
  }
  
  .dmx-device.conflicting input#name {
    background-color: #dc3545;
    color: white;
  }

  .dmx-device input#name:focus {
    box-shadow: 0 0 0 0.2rem rgba(255, 193, 7, 0.12);
  }
  
  .intro, .dmx-config, .video-config {
    padding: 0 5em;
    position: relative;
  }

  .row.dmx-config {
    z-index: 0;
  }

  .video-config {
    margin-top: 1em;
  }

  #dropzone-area-1 {
    padding: 3em;
    background: white;
    outline: 4px solid hsla(133, 89%, 70%, 1);
    text-align: center;
    cursor: pointer;
    box-shadow: inset 0 0 0px 0px hsla(133, 89%, 70%, 1);
    transition: 0.3s ease all;
  }

  #dropzone-area-1 p {
    margin-bottom: 0;
    font-size: 2em;
  }

  #dropzone-area-1.dragover {
    background: rgb(214,253,223);
  }

  div#dropzone-area-1.valid {
    box-shadow: inset 0 0 1px 100px hsla(133, 89%, 70%, 1);
    background: hsla(133, 89%, 70%, 1);
    font-weight: bold;
  }

  #video-container {
    z-index: 0;
    width: 320px;
    height: 180px;
  }

  /* #video-hidden {
    visibility: hidden
  }

  #video {
    position: absolute;
    width: 100%;
    left: 0;
    margin: auto
  } */

  @media (max-width: 768px)  {
    header {
      position: relative !important;
      height: auto;
    }
  }
</style>
