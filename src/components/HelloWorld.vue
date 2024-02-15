<template>
    <v-col>
      <v-file-input accept="image/*" v-model="image" @change="uploadImage" label="Загрузить изображение"></v-file-input>
      <canvas @mousemove="getPixelData" id="myCanvas" style="border:1px solid #000000;"></canvas>
      <v-card class="d-none">
      <v-card-item>
        <v-card-title>{{ dimData }}</v-card-title>
        <v-card-title>{{ posData }}</v-card-title>
        <v-card-title>{{ colorData }}</v-card-title>
      </v-card-item>
    </v-card>
      <footer>
      <v-card>
      <v-card-item>
        <v-card-title>{{ dimData }}</v-card-title>
        <v-card-title>{{ posData }}</v-card-title>
        <v-card-title>{{ colorData }}</v-card-title>
      </v-card-item>
    </v-card>
  </footer>
    </v-col>
</template>

<script>
  export default {
  data(){
    return {
      image: null,
      colorData: "",
      posData: "",
      dimData: "",
    }
  },
  methods: {
    uploadImage() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      var img = new Image();
      img.onload = function(){
        c.height = img.height;
        c.width = img.width;
        ctx.drawImage(img, 0, 0);
      }
      img.src = URL.createObjectURL(this.image[0]);
    },
    getMousePos(canvas, evt) {
      var rect = canvas.getBoundingClientRect();
      return {
        x: evt.clientX - rect.left,
        y: evt.clientY - rect.top
      };
    },
    getPixelData(evt) {
      var canvas = document.getElementById("myCanvas");
      var context = canvas.getContext("2d");
      var pos = this.getMousePos(canvas, evt);
      var pixelData = context.getImageData(pos.x, pos.y, 1, 1);
      this.colorData = "Цвет: " + [pixelData.data[0], pixelData.data[1], pixelData.data[2]].toString();
      this.posData = "Позиция: " + [pos.x, pos.y].toString();
      this.dimData = "Размер изображения: " + [canvas.width, canvas.height].toString();
    }
  }
}
</script>
