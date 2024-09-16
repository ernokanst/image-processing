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
    <v-file-input accept="image/*" v-model="image" @change="imageFromFile"
      label="Загрузить изображение с компьютера"></v-file-input>
    <v-text-field v-model="photoLink" @keydown.enter="imageFromWeb" label="Ссылка на изображение"></v-text-field>
  </div>
  <canvas @mousemove="getPixelData" @click="getColor" id="myCanvas" style="border:1px solid #000000;"></canvas>
  <v-dialog v-model="sizedialog" width="auto">
    <v-card max-width="400" prepend-icon="mdi-image-size-select-large" title="Изменить размер">
      <div style="margin: 8px;">
        <p>Оригинальный размер: {{ dimData }}</p>
        <p>Текущий масштаб: {{ scaleData }}%</p>
        <v-select label="Единицы" v-model="sizeChoice" :items="sizeMeasures"
          @update:model-value="changeMeasure"></v-select>
        <p>Текущий размер:</p>
        <v-text-field @input="fixedRatioW" v-model="measureW" type="number" label="Ширина"></v-text-field>
        <v-text-field @input="fixedRatioH" v-model="measureH" type="number" label="Высота"></v-text-field>
        <p>({{ newMP }} МПикс)</p>
        <v-checkbox label="Сохранить пропорции" v-model="preserveRatio"></v-checkbox>
        <v-select label="Метод интерполяции" v-tooltip:top=interExplain v-model="interChoice" :items="interMethods"
          @update:model-value="changeInter"></v-select>
      </div>
      <template v-slot:actions>
        <v-btn class="ms-auto" text="Отмена" @click="sizedialog = false"></v-btn>
        <v-btn class="ms-auto" text="Ок" @click="updateSize"></v-btn>
      </template>
    </v-card>
  </v-dialog>
  <footer>
    <div class="picker" id="picker" style="display: none;">
      <h3>Пипетка</h3>
      <v-row style="margin: 8px;">
        <div class="swatch" :style="{ backgroundColor: swatch1 }"></div>
        <p>{{ color1 }}</p>
      </v-row>
      <v-row style="margin: 8px;">
        <div class="swatch" :style="{ backgroundColor: swatch2 }"></div>
        <p>{{ color2 }}</p>
      </v-row>
      <p>{{ contrast }}</p>
      <v-btn variant="text" prepend-icon="mdi-close" @click="closePicker">Закрыть</v-btn>
    </div>
    <table>
      <tr>
        <h2>
          <td>
            <v-btn-toggle v-model="toggle" variant="outlined" divided>
              <v-btn icon="mdi-hand-back-right-outline" v-tooltip:top="'Рука: передвигайте изображение'"></v-btn>
              <v-btn icon="mdi-eyedropper" v-tooltip:top="'Пипетка: получайте информацию о цветах'"
                @click="openPicker"></v-btn>
            </v-btn-toggle>
          </td>
          <td id="dim">Размер изображения: </td>
          <td>Позиция: {{ posData }}</td>
          <td>Цвет: {{ colorData }}</td>
        </h2>
      </tr>
    </table>
    <v-row style="padding: 20px;">
      <v-slider v-model="scaleSlider" @update:model-value="changeImageSize" :max="300" :min="12" class="align-center"
        thumb-label></v-slider>
      <v-btn variant="text" @click="sizedialog = true">Изменить размер</v-btn>
      <v-btn variant="text" @click="saveImage">Сохранить</v-btn>
    </v-row>
  </footer>
</template>

<script>
var img = new Image();
var imgwidth = 0;
var imgheight = 0;
var imgpos = null;
var newimgwidth = 0;
var newimgheight = 0;
var scale = 1;
var luma1 = null;
var luma2 = null;
var mouseDown = false;
var initPos = null;
document.body.onmousedown = function () {
  mouseDown = true;
}
document.body.onmouseup = function () {
  mouseDown = false;
}
export default {
  data() {
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
      toggle: null,
      color1: "",
      color2: "",
      swatch1: "",
      swatch2: "",
      contrast: "",
    }
  },
  methods: {
    createImage() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      c.style.imageRendering = "pixelated";
      c.width = window.innerWidth;
      c.height = window.innerHeight - $("footer").height();
      img.onload = function () {
        if ((img.width <= (c.width - 100)) && (img.height <= (c.height - 100))) {
          imgpos = { x: c.width / 2 - img.width / 2, y: c.height / 2 - img.height / 2 };
          ctx.drawImage(img, imgpos.x, imgpos.y);
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
          imgpos = { x: c.width / 2 - width / 2, y: c.height / 2 - height / 2 };
          ctx.drawImage(img, imgpos.x, imgpos.y, width, height);
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
      imgpos = { x: imgpos.x - (newimgwidth * parseFloat(this.scaleSlider) / 100 - imgwidth) / 2, y: imgpos.y - (newimgheight * parseFloat(this.scaleSlider) / 100 - imgheight) / 2 };
      imgwidth = newimgwidth * parseFloat(this.scaleSlider) / 100;
      imgheight = newimgheight * parseFloat(this.scaleSlider) / 100;
      this.newMP = (imgwidth * imgheight / 1000000).toFixed(2);
      ctx.drawImage(img, imgpos.x, imgpos.y, imgwidth, imgheight);
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
        imgpos = { x: imgpos.x - (parseFloat(this.measureW) - imgwidth) / 2, y: imgpos.y - (parseFloat(this.measureH) - imgheight) / 2 };
        imgwidth = parseFloat(this.measureW);
        imgheight = parseFloat(this.measureH);
      }
      if (this.sizeChoice == "Проценты") {
        imgpos = { x: imgpos.x - (img.width * parseFloat(this.measureW) / 100 - imgwidth) / 2, y: imgpos.y - (img.height * parseFloat(this.measureH) / 100 - imgheight) / 2 };
        imgwidth = img.width * parseFloat(this.measureW) / 100;
        imgheight = img.height * parseFloat(this.measureH) / 100;
      }
      scale = (imgwidth * imgheight) / (img.width * imgheight);
      newimgwidth = imgwidth / scale;
      newimgheight = imgheight / scale;
      ctx.drawImage(img, imgpos.x, imgpos.y, imgwidth, imgheight);
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
    openPicker() {
      if (this.toggle == 1) {
        var x = document.getElementById("picker");
        x.style.display = "block";
      }
    },
    closePicker() {
      var x = document.getElementById("picker");
      x.style.display = "none";
    },
    getColor(evt) {
      if (this.toggle == 1) {
        var c = document.getElementById("myCanvas");
        var ctx = c.getContext("2d");
        var pos = this.getMousePos(c, evt);
        if ((pos.x >= imgpos.x) && (pos.x < imgpos.x + imgwidth) && (pos.y >= imgpos.y) && (pos.y < imgpos.y + imgheight)) {
          var pixelData = ctx.getImageData(pos.x, pos.y, 1, 1);
          var xyz = this.RGBtoXYZ([pixelData.data[0], pixelData.data[1], pixelData.data[2]]);
          if (evt.altKey) {
            this.color2 = this.posData + " " + this.colorData + " XYZ(" + (xyz[0] * 100).toFixed(2) + "%, " + (xyz[1] * 100).toFixed(2) + "%, " + (xyz[2] * 100).toFixed(2) + "%) " + this.RGBtoLAB([pixelData.data[0], pixelData.data[1], pixelData.data[2]]);
            this.swatch2 = this.colorData;
            luma2 = xyz[1];
          } else {
            this.color1 = this.posData + " " + this.colorData + " XYZ(" + (xyz[0] * 100).toFixed(2) + "%, " + (xyz[1] * 100).toFixed(2) + "%, " + (xyz[2] * 100).toFixed(2) + "%) " + this.RGBtoLAB([pixelData.data[0], pixelData.data[1], pixelData.data[2]]);
            this.swatch1 = this.colorData;
            luma1 = xyz[1];
          }
          if (luma1 != null && luma2 != null) {
            if (luma1 >= luma2) {
              this.contrast = ((luma1 + 0.05) / (luma2 + 0.05)).toFixed(2);
            } else {
              this.contrast = ((luma2 + 0.05) / (luma1 + 0.05)).toFixed(2);
            }
            if (this.contrast < 4.5) {
              this.contrast += " (недостаточный)";
            }
            this.contrast = "Контраст: " + this.contrast;
          }
        }
      }
    },
    RGBtoXYZ(rgb) {
      var r = rgb[0] / 255;
      var g = rgb[1] / 255;
      var b = rgb[2] / 255;
      r = (r > 0.04045) ? Math.pow((r + 0.055) / 1.055, 2.4) : r / 12.92;
      g = (g > 0.04045) ? Math.pow((g + 0.055) / 1.055, 2.4) : g / 12.92;
      b = (b > 0.04045) ? Math.pow((b + 0.055) / 1.055, 2.4) : b / 12.92;
      var x = (r * 0.4124 + g * 0.3576 + b * 0.1805);
      var y = (r * 0.2126 + g * 0.7152 + b * 0.0722);
      var z = (r * 0.0193 + g * 0.1192 + b * 0.9505);
      return [x, y, z];
    },
    RGBtoLAB(rgb) {
      var xyz = this.RGBtoXYZ(rgb);
      var x = xyz[0] / 0.95047;
      var y = xyz[1] / 1.00000;
      var z = xyz[2] / 1.08883;
      x = (x > 0.008856) ? Math.pow(x, 1 / 3) : (7.787 * x) + 16 / 116;
      y = (y > 0.008856) ? Math.pow(y, 1 / 3) : (7.787 * y) + 16 / 116;
      z = (z > 0.008856) ? Math.pow(z, 1 / 3) : (7.787 * z) + 16 / 116;
      var L = (116 * y) - 16;
      var a = 500 * (x - y);
      var b = 200 * (y - z);
      return "Lab(" + L.toFixed(2) + "%, " + a.toFixed(2) + "%, " + b.toFixed(2) + "%)";
    },
    getPixelData(evt) {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d");
      var pos = this.getMousePos(c, evt);
      var pixelData = ctx.getImageData(pos.x, pos.y, 1, 1);
      if (this.toggle == 0) {
        if (!mouseDown) {
          imgpos = { x: imgpos.x + (pos.x - initPos.x), y: imgpos.y + (pos.y - initPos.y) };
          initPos = null;
        }
        if (mouseDown) {
          if (initPos == null) {
            initPos = pos;
          }
          var newimgpos = { x: imgpos.x + (pos.x - initPos.x), y: imgpos.y + (pos.y - initPos.y) };
          if (newimgpos.x < 50 - imgwidth) {
            newimgpos.x = 50 - imgwidth;
          }
          if (newimgpos.x > c.width - 50) {
            newimgpos.x = c.width - 50;
          }
          if (newimgpos.y < 50 - imgheight) {
            newimgpos.y = 50 - imgheight;
          }
          if (newimgpos.y > c.height - 50) {
            newimgpos.y = c.height - 50;
          }
          c.style.imageRendering = "pixelated";
          ctx.clearRect(0, 0, c.width, c.height);
          ctx.drawImage(img, newimgpos.x, newimgpos.y, imgwidth, imgheight);
        }
      }
      this.colorData = "RGB(" + [pixelData.data[0], pixelData.data[1], pixelData.data[2]].toString() + ")";
      if ((pos.x >= imgpos.x) && (pos.x < imgpos.x + imgwidth) && (pos.y >= imgpos.y) && (pos.y < imgpos.y + imgheight)) {
        this.posData = [Math.floor(pos.x) - Math.floor(imgpos.x), Math.floor(pos.y) - Math.floor(imgpos.y)].toString();
      } else {
        this.posData = "";
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
