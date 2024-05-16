<template>
    <v-col>
      <div id="upload">
        <h1>Загрузите изображение</h1>
        <v-file-input accept="image/*" v-model="image" @change="uploadImage" label="Загрузить изображение с компьютера"></v-file-input>
        <v-text-field v-model="photoLink" @keydown.enter="getImage" label="Ссылка на изображение"></v-text-field>
      </div>
      <canvas @mousemove="getPixelData" id="myCanvas" style="border:1px solid #000000;"></canvas>
      <v-card style="opacity: 0">
      <v-card-item>
        <v-card-title>{{ dimData }}</v-card-title>
        <v-card-title>{{ posData }}</v-card-title>
        <v-card-title>{{ colorData }}</v-card-title>
      </v-card-item>
      <v-card-actions>
        <v-btn>Button</v-btn>
      </v-card-actions>
    </v-card>
    <v-dialog
      v-model="sizedialog"
      width="auto"
    >
      <v-card
        max-width="400"
        prepend-icon="mdi-image-size-select-large"
        title="Изменить размер"
      >
      <a>Текущий {{ dimData }}</a>
      <v-text-field label="Ширина"></v-text-field>
      <v-text-field label="Высота"></v-text-field>
        <template v-slot:actions>
          <v-btn
            class="ms-auto"
            text="Отмена"
            @click="sizedialog = false"
          ></v-btn>
          <v-btn
            class="ms-auto"
            text="Ок"
            @click="sizedialog = false"
          ></v-btn>
        </template>
      </v-card>
    </v-dialog>
  </v-col>
      <footer>
        <h2>{{ dimData }}</h2>
        <h2>{{ posData }}</h2>
        <h2>{{ colorData }}</h2>
        <v-btn variant="text" @click="sizedialog = true">Изменить размер</v-btn>
  </footer>
</template>

<script>
  export default {
  data(){
    return {
      sizedialog: false,
      image: null,
      colorData: "",
      posData: "",
      dimData: "",
      photoLink: "",
    }
  },
  methods: {
    getImage() { //Перед использованием включить https://cors-anywhere.herokuapp.com/
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      var img = new Image();
      img.onload = function(){
        c.height = img.height;
        c.width = img.width;
        ctx.drawImage(img, 0, 0);
        this.image = null;
      }
      img.crossOrigin = "Anonymous";
      img.src = "https://cors-anywhere.herokuapp.com/" + this.photoLink;
      document.getElementById("upload").style.display = 'none';
    },
    uploadImage() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      var img = new Image();
      img.onload = function(){
        c.height = img.height;
        c.width = img.width;
        ctx.drawImage(img, 0, 0);
      }
      var imgurl = URL.createObjectURL(this.image[0]);
      img.src = imgurl;
      this.photoLink = imgurl;
      document.getElementById("upload").style.display = 'none';
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
      this.posData = "Позиция: " + [Math.floor(pos.x), Math.floor(pos.y)].toString();
      this.dimData = "Размер изображения: " + [canvas.width, canvas.height].toString();
    }
  }
}
</script>
