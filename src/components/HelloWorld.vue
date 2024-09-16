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
      <v-select label="Единицы" v-model="sizeChoice" :items="sizeMeasures" @update:model-value="changeMeasure"></v-select>
      <a>Текущий размер:</a>
      <v-text-field @input="fixedRatioW" v-model="measureW" type="number" label="Ширина"></v-text-field>
      <v-text-field @input="fixedRatioH" v-model="measureH" type="number" label="Высота"></v-text-field>
      <a>({{ newMP }} МПикс)</a>
      <v-checkbox label="Сохранить пропорции" v-model="preserveRatio"></v-checkbox>
      <v-select label="Метод интерполяции" v-tooltip:top=interExplain v-model="interChoice" :items="interMethods" @update:model-value="changeInter"></v-select>
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
        <v-btn variant="text" @click="saveImage">Сохранить</v-btn>
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
      newMP: "",
      measureW: "",
      measureH: "",
      sizeChoice: "Пиксели",
      sizeMeasures: ["Пиксели", "Проценты"],
      preserveRatio: true,
      interChoice: "Ближайшего соседа",
      interMethods: ["Ближайшего соседа"],
      interExplain: "Для каждого пикселя конечного изображения выбирается один пиксель исходного, наиболее близкий к его положению с учетом масштабирования.",
    }
  },
  methods: {
    createImage() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      c.style.imageRendering = "pixelated";
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
          imgwidth = img.width;
          imgheight = img.height;
          ctx.drawImage(img, c.width / 2 - width / 2, c.height / 2 - height / 2, width, height);
        }
        newimgwidth = imgwidth;
        newimgheight = imgheight;
        this.newMP = (imgwidth * imgheight / 1000000).toFixed(2);
        document.getElementById("dim").textContent = "Размер изображения: " + [Math.floor(img.width), Math.floor(img.height)].toString();
        var sizes = [];
        sizes[0] = ""
      }
      document.getElementById("upload").style.display = "none";
    },
    changeMeasure() {
      if (this.sizeChoice == "Пиксели") {
        this.measureW = Math.floor(imgwidth);
        this.measureH = Math.floor(imgheight);
      }
      if (this.sizeChoice == "Проценты") {
        this.measureW = (imgwidth / img.width * 100).toFixed(2);
        this.measureH = (imgheight / img.height * 100).toFixed(2);
      }
    },
    changeInter() {
      if (this.sizeChoice == "Ближайшего соседа") {
        this.interExplain = "Для каждого пикселя конечного изображения выбирается один пиксель исходного, наиболее близкий к его положению с учетом масштабирования.";
      }
    },
    changeImageSize() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      c.style.imageRendering = "pixelated";
      ctx.clearRect(0, 0, c.width, c.height);
      scale = parseFloat(this.scaleSlider) / 100;
      imgwidth = newimgwidth * parseFloat(this.scaleSlider) / 100;
      imgheight = newimgheight * parseFloat(this.scaleSlider) / 100;
      this.newMP = (imgwidth * imgheight / 1000000).toFixed(2);
      ctx.drawImage(img, c.width / 2 - imgwidth / 2, c.height / 2 - imgheight / 2, imgwidth, imgheight);
      document.getElementById("dim").textContent = "Размер изображения: " + [Math.floor(imgwidth.toFixed(2)), Math.floor(imgheight.toFixed(2))].toString();
      this.changeMeasure();
    },
    updateSize() {
      if (parseFloat(this.measureW) <= 0 || parseFloat(this.measureH) <= 0) {
        alert("Размеры изображения не должны быть меньше или равны 0");
        return;
      }
      this.sizedialog = false;
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      c.style.imageRendering = "pixelated";
      ctx.clearRect(0, 0, c.width, c.height);
      if (this.sizeChoice == "Пиксели") {
        imgwidth = parseFloat(this.measureW);
        imgheight = parseFloat(this.measureH);
      }
      if (this.sizeChoice == "Проценты") {
        imgwidth = img.width * parseFloat(this.measureW) / 100;
        imgheight = img.height * parseFloat(this.measureH) / 100;
      }
      scale = (imgwidth * imgheight) / (img.width * imgheight);
      newimgwidth = imgwidth / scale;
      newimgheight = imgheight / scale;
      ctx.drawImage(img, c.width / 2 - imgwidth / 2, c.height / 2 - imgheight / 2, imgwidth, imgheight);
      document.getElementById("dim").textContent = "Размер изображения: " + [Math.floor(imgwidth.toFixed(2)), Math.floor(imgheight.toFixed(2))].toString();
    },
    fixedRatioW() {
      if (this.sizeChoice == "Пиксели") {
        if (this.preserveRatio) {
          this.measureH = String(Math.floor(imgheight * (parseFloat(this.measureW) / imgwidth)));
        }
        this.newMP = (parseFloat(this.measureW) * parseFloat(this.measureH) / 1000000).toFixed(2);
      }
      if (this.sizeChoice == "Проценты") {
        if (this.preserveRatio) {
          this.measureH = String(((imgheight / img.height * 100) * (parseFloat(this.measureW) / (imgwidth / img.width * 100))).toFixed(2));
        }
        this.newMP = (parseFloat(imgwidth * (parseFloat(this.measureW) / 100)) * parseFloat(imgheight * (parseFloat(this.measureH) / 100)) / 1000000).toFixed(2);
      }
    },
    fixedRatioH() {
      if (this.sizeChoice == "Пиксели") {
        if (this.preserveRatio) {
          this.measureW = String(Math.floor(imgwidth * (parseFloat(this.measureH) / imgheight)));
        }
        this.newMP = (parseFloat(this.measureW) * parseFloat(this.measureH) / 1000000).toFixed(2);
      }
      if (this.sizeChoice == "Проценты") {
        if (this.preserveRatio) {
          this.measureW = String(((imgwidth / img.width * 100) * (parseFloat(this.measureH) / (imgheight / img.height * 100))).toFixed(2));
        }
        this.newMP = (parseFloat(imgwidth * (parseFloat(this.measureW) / 100)) * parseFloat(imgheight * (parseFloat(this.measureH) / 100)) / 1000000).toFixed(2);
      }
    },
    imageFromWeb() { //Перед использованием включить https://cors-anywhere.herokuapp.com/
      this.createImage();
      img.crossOrigin = "Anonymous";
      img.src = "https://cors-anywhere.herokuapp.com/" + this.photoLink;
    },
    imageFromFile() {
      this.createImage();
      var imgurl = URL.createObjectURL(this.image);
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
      this.changeMeasure();
    },
    saveImage() {
      var c = document.createElement("canvas");
      var ctx = c.getContext("2d");
      c.style.imageRendering = "pixelated";
      c.width = imgwidth
      c.height = imgheight
      ctx.drawImage(img, 0, 0, imgwidth, imgheight);
      var link = document.createElement("a");
      link.download = "image.png";
      link.href = c.toDataURL()
      link.click();
    }
  }
}
</script>
