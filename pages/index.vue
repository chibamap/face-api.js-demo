<template lang="pug">
  div.container
    video(ref="video" width="720" height="560" autoplay muted)
</template>

<script>
import * as faceapi from 'face-api.js'

const MODEL_URL = '/models'

export default {
  data: () => {
    return {
      videoSize : null,
      canvas: null
    }
  },
  mounted() {
    this.$refs.video.addEventListener('play', this.onVideoPlay);
    this.loadModels().then(this.askUserToStart).then(this.startVideo);
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
      navigator.getUserMedia(
        { video: {} },
        stream => this.$refs.video.srcObject = stream,
        err => console.error(err)
      )
    },
    onVideoPlay() {
      console.log('on video play');
      const videoSize = {
        width: this.$refs.video.width,
        height: this.$refs.video.height
      };
      const canvas = faceapi.createCanvasFromMedia(this.$refs.video);
      this.$el.append(canvas);
      faceapi.matchDimensions(canvas, videoSize);

      setInterval(async () => {
        const detections = await faceapi.detectAllFaces(
            this.$refs.video,
            new faceapi.TinyFaceDetectorOptions()
        )
        .withFaceLandmarks()
        .withFaceExpressions();

        const resizedDetections = faceapi.resizeResults(detections, videoSize);
        canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
        faceapi.draw.drawDetections(canvas, resizedDetections);
        const minProbability = 0.05;
        faceapi.draw.drawFaceExpressions(canvas, resizedDetections, minProbability);
      }, 100);
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
