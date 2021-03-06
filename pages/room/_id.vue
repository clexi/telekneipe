<template>
  <div class="container">
    <div ref="videoContainer" class="video-container">
      <transition name="fade">
        <stream
          v-show="showVideos"
          ref="stream"
          width="100%"
          :socket-u-r-l="'https://telekneipe-server.classen.rocks/'"
          :room-id="roomId"
          :camera-height="videoHeight"
          :class="'videos ' + videoCountClass"
          :controls="false"
          @openedRoom="openedRoom"
          @joinedRoom="joinedRoom"
          @leftRoom="leftRoom"
          @onmessage="handleAction"
          @sendAction="sendAction"
        />
      </transition>
    </div>
    <message :show-message="message" />
    <audio
      ref="soundBeer"
      preload="true"
      class="sound beer"
      src="/sounds/beer.mp3"
    ></audio>
    <audio
      ref="soundDoor"
      preload="true"
      class="sound door"
      src="/sounds/door.mp3"
    ></audio>
    <audio
      ref="soundClink"
      preload="true"
      class="sound clink"
      src="/sounds/clink.mp3"
    ></audio>
    <controls
      :should-show-controls="showingControls || showingControlsBlocking"
      @blockHiding="showingControlsBlocking = true"
      @hide="showingControlsBlocking = false"
      @toggleSound="toggleSound"
      @leaveGoBack="leaveGoBack"
      @leaveGotoHome="leaveGotoHome"
    />
  </div>
</template>

<script>
import stream from '@/components/stream/stream'

export default {
  components: {
    stream,
    Controls: () => import('@/components/controls/Controls'),
    Message: () => import('@/components/actions/message')
  },
  data() {
    return {
      roomId: this.$route.params.id,
      videoHeight: 'auto',
      playing: false,
      showVideos: false,
      showingControls: false,
      showingControlsBlocking: false,
      videos: [],
      videoObjects: {},
      message: false
    }
  },
  computed: {
    roomRunning() {
      return typeof this.$refs.stream !== 'undefined'
    },
    localVideoId() {
      return this.$refs.stream.localVideo.id
    },
    videoCountClass() {
      if (this.videos.length > 8) {
        return 'grid-50-100-max'
      } else if (this.videos.length > 3) {
        return 'grid-50-50'
      } else if (this.videos.length > 2) {
        return 'grid-50'
      } else if (this.videos.length <= 1) {
        return 'grid-100'
      } else {
        return 'grid-100'
      }
    },
    audioTracks() {
      return [this.$refs.soundBeer, this.$refs.soundDoor, this.$refs.soundClink]
    }
  },
  created() {
    if (this.roomId) {
      this.$store.commit('setRoomId', this.roomId)
    } else {
      this.$router.push('/')
    }
  },
  mounted() {
    if (this.roomId) {
      this.joinRoom()
    }
    document.body.addEventListener('touchstart', this.canPlayAudio, false)
    document.addEventListener('mousemove', this.showControls)
    this.$refs.stream.rtcmConnection.onmessage(this.handleAction)
  },
  updated() {
    if (typeof this.$refs.stream.$refs.videos !== 'undefined') {
      this.$refs.stream.$refs.videos.map(($v) => {
        $v.controls = false
      })
    }
  },
  beforeDestroy() {
    document.body.removeEventListener('touchstart', this.canPlayAudio, false)
    document.removeEventListener('mousemove', this.showControls)
  },
  methods: {
    canPlayAudio() {
      if (this.audioTracks) {
        for (const audio of this.audioTracks) {
          audio.play()
          audio.pause()
          audio.currentTime = 0
        }
      }
    },
    isPlaying() {
      this.playing = !this.playing
    },
    joinRoom() {
      this.$refs.stream.join()
      this.$store.commit(
        'setVideoLink',
        window.location.origin + '?roomId=' + encodeURIComponent(this.roomId)
      )
    },
    leaveRoom() {
      this.$refs.stream.leave()
    },
    openedRoom(video) {},
    joinedRoom(video) {
      this.showVideos = true
      this.$refs.soundBeer.play()
      if (typeof this.$refs.stream.$refs.videos !== 'undefined')
        this.videos = this.$refs.stream.$refs.videos
      if (typeof this.$refs.stream.videoList !== 'undefined')
        this.videoObjects = this.$refs.stream.videoList
      // console.log(this.$refs.stream.videoList)
    },
    leftRoom(video) {
      if (typeof this.$refs.stream.$refs.videos !== 'undefined')
        this.videos = this.$refs.stream.$refs.videos
      if (typeof this.$refs.stream.videoList !== 'undefined')
        this.videoObjects = this.$refs.stream.videoList
      this.$refs.soundDoor.play()
    },
    showControls() {
      setTimeout(() => {
        this.showingControls = false
      }, 5000)
      this.showingControls = true
    },
    toggleSound(sound) {
      if (this.videos.length) {
        this.videos.map((item) => {
          if (item.id !== this.localVideoId) item.muted = !sound
        })
      }
    },
    leaveGoBack() {
      if (this.roomRunning) {
        this.leaveRoom()
        this.$router.go(-1)
      }
    },
    leaveGotoHome() {
      if (this.roomRunning) {
        this.leaveRoom()
        this.$router.push('/')
      }
    },
    sendAction(event) {
      if (event.to === 'all') {
        this.$refs.stream.rtcmConnection.send({
          to: 'all',
          message: 'cheers all'
        })
        this.cheersAll()
      } else {
        this.videoObjects.map(($v) => {
          if ($v.id === event.to) {
            const message = {
              from: this.localVideoId,
              to: $v.id,
              message: 'cheers'
            }
            this.$refs.stream.rtcmConnection.send(message, $v.id)
            this.cheersOne(message)
          }
        })
      }
    },
    handleAction(message) {
      if (typeof message !== 'function') {
        if (message.data.to === 'all') {
          this.cheersAll()
        } else {
          this.cheersOne(message.data)
        }
      }
    },
    cheersAll() {
      this.$refs.stream.$refs.videos.map(($item) => {
        $item.allActive = true
      })
      this.message = true
      this.$refs.soundClink.play()

      setTimeout(() => {
        this.message = false
        this.$refs.stream.$refs.videos.map(($item) => {
          $item.allActive = false
        })
      }, 1000)
    },
    cheersOne(message) {
      this.$refs.stream.$refs.videos.map(($item) => {
        if ($item.id === message.to || $item.id === message.from) {
          $item.activeVideo = true
        }
      })
      this.message = true
      this.$refs.soundClink.play()

      setTimeout(() => {
        this.message = false
        this.$refs.stream.$refs.videos.map(($item) => {
          $item.activeVideo = false
        })
      }, 1000)
    }
  },
  head() {
    return {
      title: this.roomId + ' Telekneipe'
    }
  },
  beforeRouteLeave(to, from, next) {
    if (this.roomRunning) {
      this.leaveRoom()
    }
    next()
  }
}
</script>

<style scoped>
.video-container {
  width: 100%;
  min-height: 100vh;
  display: flex;
}
.sound {
  visibility: hidden;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}

.fade-enter-active,
.fade-leave-active {
  will-change: opacity;
  transition: opacity var(--transition-default);
}
</style>
<style>
/* unscoped styles for vue-webrtc */
.videos.video-list {
  background: transparent !important;
  display: grid;
  width: 100%;
  grid-template-columns: repeat(auto-fit, minmax(25%, 1fr));
  max-height: 100vh;
  object-fit: cover;
  object-position: center center;
}

.videos.videos.video-list.grid-100 {
}

.videos.video-list.grid-50-100-max {
  grid-template-columns: repeat(auto-fit, minmax(50%, 1fr));
  max-height: 100%;
}
@media screen and (max-width: 992px) and (orientation: portrait) {
  .videos.video-list {
    grid-template-columns: repeat(auto-fill, minmax(100%, 1fr));
    max-height: 100vh;
  }
  .videos.video-list.grid-50 {
    grid-template-columns: repeat(auto-fit, minmax(50%, 1fr));
  }
  .videos.video-list.grid-50-100-max {
    grid-template-columns: repeat(auto-fit, minmax(50%, 1fr));
    max-height: 100%;
  }
}
@media screen and (max-width: 992px) and (orientation: landscape) {
  .videos.video-list {
    grid-template-columns: repeat(auto-fit, minmax(50%, 1fr));
  }
  .videos.video-list.grid-50 {
    grid-template-columns: repeat(auto-fit, minmax(25%, 1fr));
  }
}
</style>
