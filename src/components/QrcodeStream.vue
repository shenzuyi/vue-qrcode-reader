<template lang="html">
  <div class="wrapper">
    <!--
    Note that the order of DOM elements matters.
    It defines the stacking order.
    The first element is at the very bottom, the last element is on top.
    This eliminates the need for `z-index`.
    -->
    <video
      ref="video"
      v-show="shouldScan"
      class="camera"
      autoplay
      muted
      playsinline
    ></video>

    <canvas ref="pauseFrame" v-show="!shouldScan" class="pause-frame"></canvas>

    <canvas ref="trackingLayer" class="tracking-layer"></canvas>

    <div class="overlay">
      <slot></slot>
    </div>
  </div>
</template>

<script>
import { keepScanning } from "../misc/scanner.js";
import { thinSquare } from "../misc/track-func.js";
import Camera from "../misc/camera.js";
import CommonAPI from "../mixins/CommonAPI.vue";
import Worker from "../worker/jsqr.js";

export default {
  name: "qrcode-stream",

  mixins: [CommonAPI],

  props: {
    camera: {
      type: String,
      default: "auto",

      validator(camera) {
        return ["auto", "rear", "front", "off"].includes(camera);
      }
    },

    torch: {
      type: Boolean,
      default: false
    },

    track: {
      type: [Function, Boolean],
      default: true
    },

    worker: {
      type: Function,
      default: Worker
    }
  },

  data() {
    return {
      cameraInstance: null,
      destroyed: false,
      stopScanning: () => {}
    };
  },

  computed: {
    shouldStream() {
      return this.destroyed === false && this.camera !== "off";
    },

    shouldScan() {
      return this.shouldStream === true && this.cameraInstance !== null;
    },

    /**
     * Minimum delay in milliseconds between frames to be scanned. Don't scan
     * so often when visual tracking is disabled to improve performance.
     */
    scanInterval() {
      if (this.track === false) {
        return 500;
      } else {
        return 40; // ~ 25fps
      }
    },

    trackRepaintFunction() {
      if (this.track === true) {
        return thinSquare({ color: "#ff0000" });
      } else if (this.track === false) {
        return undefined;
      } else {
        return this.track;
      }
    }
  },

  watch: {
    shouldStream(shouldStream) {
      if (!shouldStream) {
        const frame = this.cameraInstance.captureFrame();
        this.paintPauseFrame(frame);
      }
    },

    shouldScan(shouldScan) {
      if (shouldScan) {
        this.clearPauseFrame();
        this.clearTrackingLayer();
        this.startScanning();
      } else {
        this.stopScanning();
      }
    },

    torch() {
      this.init();
    },

    camera() {
      this.init();
    }
  },

  mounted() {
    this.init();
  },

  beforeDestroy() {
    this.beforeResetCamera();
    this.stopScanning();
    this.destroyed = true;
  },

  methods: {
    init() {
      const promise = (async () => {
        this.beforeResetCamera();

        if (this.camera === "off") {
          this.cameraInstance = null;

          return {
            capabilities: {}
          };
        } else {
          this.cameraInstance = await Camera(this.$refs.video, {
            camera: this.camera,
            torch: this.torch
          });

          const capabilities = this.cameraInstance.getCapabilities();

          // if the component is destroyed before `cameraInstance` resolves a
          // `beforeDestroy` hook has no chance to clear the remaining camera
          // stream.
          if (this.destroyed) {
            this.cameraInstance.stop();
          }

          return {
            capabilities
          };
        }
      })();

      this.$emit("init", promise);
    },

    startScanning() {
      const detectHandler = result => {
        this.onDetect(Promise.resolve(result));
      };

      // this.stopScanning()
      this.stopScanning = keepScanning(this.worker, this.cameraInstance, {
        detectHandler,
        locateHandler: this.onLocate,
        minDelay: this.scanInterval
      });
    },

    beforeResetCamera() {
      if (this.cameraInstance !== null) {
        this.cameraInstance.stop();
        this.cameraInstance = null;
      }
    },

    onLocate(location) {
      if (this.trackRepaintFunction === undefined || location === null) {
        this.clearTrackingLayer();
      } else {
        this.repaintTrackingLayer(location);
      }
    },

    repaintTrackingLayer(location) {
      const video = this.$refs.video;
      const canvas = this.$refs.trackingLayer;
      const ctx = canvas.getContext("2d");

      // The visually occupied area of the video element.
      // Because the component is responsive and fills the available space,
      // this can be more or less than the actual resolution of the camera.
      const displayWidth = video.offsetWidth;
      const displayHeight = video.offsetHeight;

      // The actual resolution of the camera.
      // These values are fixed no matter the screen size.
      const resolutionWidth = video.videoWidth;
      const resolutionHeight = video.videoHeight;

      // Dimensions of the video element as if there would be no
      //   object-fit: cover;
      // Thus, the ratio is the same as the cameras resolution but it's
      // scaled down to the size of the visually occupied area.
      const largerRatio = Math.max(
        displayWidth / resolutionWidth,
        displayHeight / resolutionHeight
      );
      const uncutWidth = resolutionWidth * largerRatio;
      const uncutHeight = resolutionHeight * largerRatio;

      const xScalar = uncutWidth / resolutionWidth;
      const yScalar = uncutHeight / resolutionHeight;
      const xOffset = (displayWidth - uncutWidth) / 2;
      const yOffset = (displayHeight - uncutHeight) / 2;

      const coordinatesAdjusted = {};
      for (const key in location) {
        coordinatesAdjusted[key] = {
          x: Math.floor(location[key].x * xScalar + xOffset),
          y: Math.floor(location[key].y * yScalar + yOffset)
        };
      }

      window.requestAnimationFrame(() => {
        canvas.width = displayWidth;
        canvas.height = displayHeight;

        this.trackRepaintFunction(coordinatesAdjusted, ctx);
      });
    },

    clearTrackingLayer() {
      const canvas = this.$refs.trackingLayer;
      const ctx = canvas.getContext("2d");

      window.requestAnimationFrame(() => {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
      });
    },

    paintPauseFrame(imageData) {
      const canvas = this.$refs.pauseFrame;
      const ctx = canvas.getContext("2d");

      window.requestAnimationFrame(() => {
        canvas.width = imageData.width;
        canvas.height = imageData.height;

        ctx.putImageData(imageData, 0, 0);
      });
    },

    clearPauseFrame() {
      const canvas = this.$refs.pauseFrame;
      const ctx = canvas.getContext("2d");

      window.requestAnimationFrame(() => {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
      });
    }
  }
};
</script>

<style lang="css" scoped>
.wrapper {
  position: relative;
  z-index: 0;
  width: 100%;
  height: 100%;
}

.overlay,
.tracking-layer {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
}

.camera,
.pause-frame {
  display: block;
  object-fit: cover;
  width: 100%;
  height: 100%;
}
</style>
