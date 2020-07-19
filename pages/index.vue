<template lang="pug">
  div.container
    video(ref="video" @play="onVideoPlay" width="720" height="560" autoplay muted)
    canvas(ref="canvas" width="720" height="560")
</template>

<script>
import * as faceapi from 'face-api.js'

const MODEL_URL = '/models'
  , interval = 200

let prevDetections = null
  , notDetectedCount = 0
  , timer = null

export default {
  data: () => {
    return {
      videoSize : null,
    }
  },
  mounted() {
    this.videoSize = {
      width: this.$refs.video.width,
      height: this.$refs.video.height
    };
  },
  fetch() {
    return this.loadModels()
      .then(this.askUserToStart)
      .then(this.startVideo);
  },
  methods: {
    loadModels() {
      return Promise.all([
        faceapi.nets.tinyFaceDetector.loadFromUri(MODEL_URL),
        faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL),
        faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL),
        faceapi.nets.faceExpressionNet.loadFromUri(MODEL_URL),
      ]);
    },
    askUserToStart() {
      return new Promise((resolve, reject) => {
        if (confirm('videoを利用します。')) {
          resolve();
        } else {
          reject();
        }
      });
    },
    startVideo() {
      navigator.mediaDevices.getUserMedia(
        { video: this.videoSize, audio: false }
      )
      .then(stream => this.$refs.video.srcObject = stream)
      .catch(err => console.error(err))
    },
    onVideoPlay() {
      const canvas = this.$refs.canvas;
      faceapi.matchDimensions(canvas, this.videoSize);
      if (timer) {
        clearInterval(timer);
      }
      timer = setInterval(this.faceDetect, interval);
    },
    async faceDetect() {
      let results = null;
      try {
        results = await faceapi.detectAllFaces(
            this.$refs.video,
            new faceapi.TinyFaceDetectorOptions())
          .withFaceExpressions();
      } catch (e) {
        console.log(e);
      }

      if (!results || !results.length) {
        notDetectedCount ++;
        if (5000 < interval * notDetectedCount) {
          this.clearCanvas();
        }
        return;
      }
      this.renderDetection(results);
    },
    renderDetection(results) {
      const canvas = this.$refs.canvas;
      const resizedResults = faceapi.resizeResults(results, this.videoSize);
      this.clearCanvas();
      resizedResults.map((result) => {
        this.renderExpression(result);
      });
    },
    renderExpression(result) {
      const ctx = this.$refs.canvas.getContext('2d');
      const box = result.detection.box;
      const fontsize = Math.max(box.width, box.height);
      ctx.font = fontsize + "px serif";
      ctx.fillStyle = "white";
      const x = box.x + ((box.width - fontsize) / 2);
      const y = box.y + fontsize - 50;
      const emoji = this.expression2emoji(result.expressions);
      ctx.fillText(emoji, x, y);
    },
    expression2emoji(expressions) {
      const expression = Object.entries(expressions).reduce((cur, prev) => {
        return cur[1] < prev[1]? prev : cur;
      });
      switch(expression[0]) {
        case "happy":
          return "😄";
        case "sad":
          return "😢";
        case "angry":
          return "😠";
        case "fearful":
          return "😨";
        case "disgusted":
          return "😩";
        case "surprised":
          return "😲";
        case "neutral":
        default:
          return "😶";
      }
    },
    clearCanvas() {
      this.$refs.canvas.getContext('2d').clearRect(0, 0, this.videoSize.width, this.videoSize.height);
    }
  },
  
}
</script>

<style>
.container {
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
video, canvas {
  position: absolute;
}
</style>
