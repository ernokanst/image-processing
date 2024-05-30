<template>
      <div id="upload">
        <header>
    <v-app-bar :elevation="6" color="primary">
        <v-app-bar-title>
          Обработка изображений
        </v-app-bar-title>
    </v-app-bar>
  </header>
        <h1>Загрузите изображение</h1>
        <v-file-input accept="image/*" v-model="image" @change="imageFromFile" label="Загрузить изображение с компьютера"></v-file-input>
        <v-text-field v-model="photoLink" @keydown.enter="imageFromWeb" label="Ссылка на изображение"></v-text-field>
      </div>
      <canvas @mousemove="getPixelData" id="myCanvas" style="border:1px solid #000000;"></canvas>
    <v-dialog
      v-model="sizedialog"
      width="auto"
    >
      <v-card
        max-width="400"
        prepend-icon="mdi-image-size-select-large"
        title="Изменить размер"
      >
      <a>Оригинальный размер: {{ dimData }}</a>
      <a>Текущий масштаб: {{ scaleData }}%</a>
      <a>Текущий размер:</a>
      <v-text-field @mousemove="fixedRatioW" v-model="currentW" type="number" label="Ширина"></v-text-field>
      <v-text-field @mousemove="fixedRatioH" v-model="currentH" type="number" label="Высота"></v-text-field>
      <a>({{ newMP }} МПикс)</a>
      <v-checkbox label="Сохранить пропорции" v-model="preserveRatio"></v-checkbox>
        <template v-slot:actions>
          <v-btn
            class="ms-auto"
            text="Отмена"
            @click="sizedialog = false"
          ></v-btn>
          <v-btn
            class="ms-auto"
            text="Ок"
            @click="updateSize"
          ></v-btn>
        </template>
      </v-card>
    </v-dialog>
      <footer>
        <table><tr><h2>
          <td id="dim">Размер изображения: </td>
          <td>{{ posData }}</td>
          <td>{{ colorData }}</td>
        </h2></tr></table>
        <v-row style="padding: 20px;">
        <v-slider v-model="scaleSlider" @update:model-value="changeImageSize" :max="300" :min="12" class="align-center" thumb-label></v-slider>
        <v-btn variant="text" @click="sizedialog = true">Изменить размер</v-btn>
      </v-row>
  </footer>
</template>

<script>
var img = new Image();
var imgwidth = 0;
var imgheight = 0;
var newimgwidth = 0;
var newimgheight = 0;
var scale = 1;
  export default {
  data(){
    return {
      sizedialog: false,
      image: null,
      colorData: "Цвет: ",
      posData: "Позиция: ",
      dimData: "",
      photoLink: "",
      scaleData: "",
      scaleSlider: "100",
      currentW: "",
      currentH: "",
      newMP: "",
      preserveRatio: true,
    }
  },
  methods: {
    createImage() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      c.width = window.innerWidth;
      c.height = window.innerHeight - $( "footer" ).height();
      img.onload = function(){
        if ((img.width <= (c.width - 100)) && (img.height <= (c.height - 100))) {
          ctx.drawImage(img, c.width / 2 - img.width / 2, c.height / 2 - img.height / 2);
          imgwidth = img.width;
          imgheight = img.height;
        } else {
          var MAX_WIDTH = c.width - 100;
          var MAX_HEIGHT = c.height - 100;
          var width = img.width;
          var height = img.height;
          if (width > MAX_WIDTH) {
            scale *= (MAX_WIDTH / width)
            height *= (MAX_WIDTH / width);
            width = MAX_WIDTH;
          }
          if (height > MAX_HEIGHT) {
            scale *= (MAX_HEIGHT / height);
            width *= (MAX_HEIGHT / height);
            height = MAX_HEIGHT;
          }
          imgwidth = width;
          imgheight = height;
          ctx.drawImage(img, c.width / 2 - width / 2, c.height / 2 - height / 2, width, height);
        }
        newimgwidth = imgwidth;
        newimgheight = imgheight;
        this.newMP = (imgwidth * imgheight / 1000000).toFixed(2);
        document.getElementById("dim").textContent = "Размер изображения: " + [img.width, img.height].toString();
      }
      document.getElementById("upload").style.display = 'none';
    },
    changeImageSize() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      ctx.clearRect(0, 0, c.width, c.height);
      scale = parseFloat(this.scaleSlider) / 100;
      imgwidth = newimgwidth * parseFloat(this.scaleSlider) / 100;
      imgheight = newimgheight * parseFloat(this.scaleSlider) / 100;
      this.newMP = (imgwidth * imgheight / 1000000).toFixed(2);
      ctx.drawImage(img, c.width / 2 - imgwidth / 2, c.height / 2 - imgheight / 2, imgwidth, imgheight);
      document.getElementById("dim").textContent = "Размер изображения: " + [imgwidth.toFixed(2), imgheight.toFixed(2)].toString();
      this.currentW = imgwidth.toFixed(2);
      this.currentH = imgheight.toFixed(2);
    },
    updateSize() {
      this.sizedialog = false;
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      ctx.clearRect(0, 0, c.width, c.height);
      imgwidth = parseFloat(this.currentW);
      imgheight = parseFloat(this.currentH);
      scale = (imgwidth * imgheight) / (img.width * imgheight);
      newimgwidth = imgwidth / scale;
      newimgheight = imgheight / scale;
      ctx.drawImage(img, c.width / 2 - imgwidth / 2, c.height / 2 - imgheight / 2, imgwidth, imgheight);
      document.getElementById("dim").textContent = "Размер изображения: " + [imgwidth.toFixed(2), imgheight.toFixed(2)].toString();
    },
    fixedRatioW() {
      if (this.preserveRatio) {
        this.currentH = String(imgheight * (parseFloat(this.currentW) / imgwidth));
      }
      this.newMP = (parseFloat(this.currentW) * parseFloat(this.currentH) / 1000000).toFixed(2);
    },
    fixedRatioH() {
      if (this.preserveRatio) {
        this.currentW = String(imgwidth * (parseFloat(this.currentH) / imgheight));
      }
      this.newMP = (parseFloat(this.currentW) * parseFloat(this.currentH) / 1000000).toFixed(2);
    },
    imageFromWeb() { //Перед использованием включить https://cors-anywhere.herokuapp.com/
      this.createImage();
      img.crossOrigin = "Anonymous";
      img.src = "https://cors-anywhere.herokuapp.com/" + this.photoLink;
    },
    imageFromFile() {
      this.createImage();
      var imgurl = URL.createObjectURL(this.image[0]);
      img.src = imgurl;
      this.photoLink = imgurl;
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
      if ((pos.x >= canvas.width / 2 - imgwidth / 2) && (pos.x < canvas.width / 2 + imgwidth / 2) && (pos.y >= canvas.height / 2 - imgheight / 2) && (pos.y < canvas.height / 2 + imgheight / 2)) {
        this.posData = "Позиция: " + [Math.floor(pos.x) - Math.floor(canvas.width / 2 - imgwidth / 2), Math.floor(pos.y) - Math.floor(canvas.height / 2 - imgheight / 2)].toString();
      } else {
        this.posData = "Позиция: ";
      }
      this.scaleSlider = String(scale * 100);
      this.scaleData = (scale * 100).toFixed(2);
      this.dimData = String(img.width) + ", " + String(img.height) + " (" + (img.width * img.height / 1000000).toFixed(2) + " МПикс)";
      this.newMP = (imgwidth * imgheight / 1000000).toFixed(2);
      this.currentW = imgwidth.toFixed(2);
      this.currentH = imgheight.toFixed(2);
    }
  }
}
</script>
