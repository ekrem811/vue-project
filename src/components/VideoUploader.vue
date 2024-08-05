<template>
  <div>
    <input type="file" @change="onFileChange" accept="video/*" />
    <video ref="video" controls></video>
    <canvas ref="canvas" style="display: none;"></canvas>
    <div v-if="processedImage">
      <h3>Processed Frame</h3>
      <img :src="processedImage" alt="Processed frame" />
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      websocket: null,
      processedImage: null
    };
  },
  methods: {
    onFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        const url = URL.createObjectURL(file);
        this.$refs.video.src = url;
        this.$refs.video.onloadedmetadata = () => {
          this.$refs.video.play();
          this.startStreaming();
        };
      }
    },
    startStreaming() {
      const video = this.$refs.video;
      const canvas = this.$refs.canvas;
      const context = canvas.getContext('2d');
      const fps = 10; // Change FPS as needed


      this.websocket = new WebSocket('ws://localhost:3000');
      this.websocket.binaryType = 'arraybuffer';

      this.websocket.onopen = () => {
        console.log('WebSocket connection opened');
      };

      this.websocket.onmessage = (event) => {
        console.log("new frame received")
        const blob = new Blob([event.data], { type: 'image/jpeg' });
        const url = URL.createObjectURL(blob);
        this.processedImage = url;
      };

      this.websocket.onerror = (error) => {
        console.error('WebSocket error:', error);
      };

      this.websocket.onclose = () => {
        console.log('WebSocket connection closed');
      };

      const streamFrames = () => {
        if (video.paused || video.ended) return;
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        context.drawImage(video, 0, 0, canvas.width, canvas.height);
        canvas.toBlob((blob) => {
          if (blob && this.websocket.readyState === WebSocket.OPEN) {
            const reader = new FileReader();
            reader.onload = () => {
              const arrayBuffer = reader.result;
              this.websocket.send(arrayBuffer);
            };
            reader.readAsArrayBuffer(blob);
          }
        }, 'image/jpeg');
        setTimeout(streamFrames, 1000 / fps);
      };

      video.addEventListener('play', () => {
        streamFrames();
      });
    }
  },
  beforeDestroy() {
    if (this.websocket) {
      this.websocket.close();
    }
  }
};
</script>

<style scoped>
/* Add your styles here */
</style>
