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
      <table class="picker">
        <tr class="picker">
          <td class="picker"></td>
          <td class="picker">Позиция</td>
          <td class="picker">RGB</td>
          <td class="picker">XYZ</td>
          <td class="picker">Lab</td>
        </tr>
        <tr class="picker">
          <td class="picker"><div class="swatch" :style="{ backgroundColor: swatch1 }"></div></td>
          <td class="picker">{{ pos1 }}</td>
          <td class="picker">{{ rgb1 }}</td>
          <td class="picker">{{ xyz1 }}</td>
          <td class="picker">{{ lab1 }}</td>
        </tr>
        <tr class="picker">
          <td class="picker"><div class="swatch" :style="{ backgroundColor: swatch2 }"></div></td>
          <td class="picker">{{ pos2 }}</td>
          <td class="picker">{{ rgb2 }}</td>
          <td class="picker">{{ xyz2 }}</td>
          <td class="picker">{{ lab2 }}</td>
        </tr>
      </table>
      <v-row style="margin: 8px;">
        <v-btn prepend-icon="mdi-help-circle-outline" variant="text"
          v-tooltip:bottom="'RGB – аддитивная цветовая модель, описывающая способ кодирования цвета для цветовоспроизведения с помощью трёх цветов, которые принято называть основными. Каждый параметр (красный, зеленый и синий) определяет интенсивность цвета со значением от 0 до 255.'">RGB</v-btn>
        <v-btn prepend-icon="mdi-help-circle-outline" variant="text"
          v-tooltip:bottom="'Цветовое пространство XYZ моделирует цвета в соответствии с типичной чувствительностью трех типов колбочек человеческого глаза. Канал Y представляет яркость цвета. Канал Z приблизительно представляет количество синего цвета в изображении, но значение Z в цветовом пространстве XYZ не идентично значению B в цветовом пространстве RGB. Канал X не имеет четкой цветовой аналогии. Однако, если рассматривать цветовое пространство XYZ как трехмерную систему координат, то значения X лежат вдоль оси, которая ортогональна оси Y (яркость) и оси Z.'">XYZ</v-btn>
        <v-btn prepend-icon="mdi-help-circle-outline" variant="text"
          v-tooltip:bottom="'Цветовое пространство L*a*b* обеспечивает более перцептивно однородное цветовое пространство, чем модель XYZ. Цвета в цветовом пространстве L*a*b* могут существовать за пределами гаммы RGB (допустимого набора цветов RGB). L* – яркость изображения, 0 указывает черный цвет, а 100 — белый. a* – количество красных или зеленых тонов в изображении, большое положительное значение a* соответствует красному/пурпурному, большое отрицательное значение a* соответствует зеленому цвету. b* – количество желтых или синих тонов в изображении, большое положительное значение b* соответствует желтому цвету, большое отрицательное значение b* соответствует синему цвету.'">Lab</v-btn>
      </v-row>
      <p>{{ contrast }}</p>
      <v-btn variant="text" prepend-icon="mdi-close" @click="closePicker">Закрыть</v-btn>
    </div>
    <div class="curves" id="curves" style="display: none;">
      <h3>Кривые</h3>
      <v-row>
        <v-col>
          <v-number-input :max="255" :min="0" @input="updateDots" v-model="posx1" type="number"
            label="X1"></v-number-input>
          <v-number-input :max="255" :min="0" @input="updateDots" v-model="posy1" type="number"
            label="Y1"></v-number-input>
        </v-col>
        <v-col>
          <v-number-input :max="255" :min="0" @input="updateDots" v-model="posx2" type="number"
            label="X2"></v-number-input>
          <v-number-input :max="255" :min="0" @input="updateDots" v-model="posy2" type="number"
            label="Y2"></v-number-input>
        </v-col>
      </v-row>
      <canvas id="selector" class="curve" @mousemove="checkDots" height="256" width="256"></canvas>
      <v-checkbox label="Предпросмотр" id="preview" v-model="preview" @click="checkPreview"></v-checkbox>
      <v-row style="margin: 8px;">
        <v-btn variant="text" prepend-icon="mdi-close" @click="closeCurves">Закрыть</v-btn>
        <v-btn variant="text" prepend-icon="mdi-reload" @click="resetCurves">Сброс</v-btn>
        <v-btn variant="text" prepend-icon="mdi-check" @click="applyCurves">Применить</v-btn>
      </v-row>
    </div>
    <table class="info">
      <tr class="info">
        <h2>
          <td class="info">
            <v-btn-toggle v-model="toggle" variant="outlined" divided>
              <v-btn icon="mdi-hand-back-right-outline" v-tooltip:top="'Рука: передвигайте изображение'"></v-btn>
              <v-btn icon="mdi-eyedropper" v-tooltip:top="'Пипетка: получайте информацию о цветах'"
                @click="openPicker"></v-btn>
              <v-btn icon="mdi-chart-bell-curve-cumulative" v-tooltip:top="'Кривые: управляйте цветами'"
                @click="openCurves"></v-btn>
            </v-btn-toggle>
          </td>
          <td id="dim" class="info">Размер изображения: </td>
          <td class="info">Позиция: {{ posData }}</td>
          <td class="info">Цвет: {{ colorData }}</td>
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
var curvedImg = null;
var oldCCpoints = [0, 0, 1, 1];
var colorHist = {
  r: new Array(256).fill(0),
  g: new Array(256).fill(0),
  b: new Array(256).fill(0),
};
document.body.onmousedown = function () {
  mouseDown = true;
}
document.body.onmouseup = function () {
  mouseDown = false;
}
addEventListener("wheel", (evt) => {
  if (evt.altKey) {
    var c = document.getElementById("myCanvas");
    var ctx = c.getContext("2d", { willReadFrequently: true });
    var newimgpos = { x: imgpos.x + evt.deltaX, y: imgpos.y + evt.deltaY };
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
    imgpos = newimgpos;
    checkCurve();
  }
});
var cModified = document.createElement('canvas');
function ColorCurve(canvas, callback) {
  'use strict';

  this.points = [];
  this.currentPoint = -1;
  this.c = document.getElementById(canvas);
  this.ctx = this.c.getContext("2d", { willReadFrequently: true });
  this.height = this.c.height;
  this.width = this.c.width;
  this.redraw = 0;
  this.values = [];
  this.rgb = [];
  this.onChange = callback;

  if (this.height != this.width) {
    console.error("ERROR: Canvas must have same width and height.");
    return undefined;
  }

  this.points.push({ x: 0, y: 0 });
  this.points.push({ x: 1.0, y: 1.0 });

  var me = this;

  this.c.addEventListener('mousedown', function (ev) {
    me.mouseDown(ev);
  }, false);

  this.c.addEventListener('mouseup', function (ev) {
    me.mouseUp(ev);
    me.draw();
  }, false);

  this.c.addEventListener('mouseout', function (ev) {
    me.mouseUp(ev);
    me.draw();
  }, false);

  this.c.addEventListener('mousemove', function (ev) {
    me.mouseMove(ev);
    if (me.redraw == 1) {
      me.draw();
      me.updateValues();
      me.redraw = 0;
    }
  }, false);

  this.draw();
  this.updateValues();
}
ColorCurve.prototype.updateValues = function () {
  'use strict';

  this.rgb.splice(0, this.rgb.length);
  for (var i = 0; i < 256; i++) this.rgb.push(Math.round(this.getY(i / 255.0) * 255));

  if (typeof this.onChange !== "undefined") {
    this.onChange();
  }

}
ColorCurve.prototype.isEqual = function (p1, p2) {
  'use strict';

  if (p1.x == p2.x && p1.y == p2.y) {
    return true;
  } else {
    return false;
  }

}
ColorCurve.prototype.draw = function () {
  'use strict';

  var p1, p4;

  this.values.splice(0, this.values.length);
  this.ctx.clearRect(0, 0, this.width, this.height);
  this.drawColorHist();
  this.drawGrid();

  for (var i = 0; i < this.points.length - 1; i++) {
    if (i < 1) {
      p1 = this.points[0];
    } else {
      p1 = this.points[i - 1];
    }
    if (i + 2 > this.points.length - 1) {
      p4 = this.points[i + 1];
    } else {
      p4 = this.points[i + 2];
    }
    this.quadratic(p1, this.points[i], this.points[i + 1], p4);
  }
  this.drawPoints();

}
ColorCurve.prototype.drawGrid = function () {
  'use strict';

  var space = this.width / 4.0;

  this.ctx.beginPath();
  this.ctx.lineWidth = 1;
  this.ctx.strokeStyle = '#aaaaaa';

  for (var i = 0; i < this.height - space; i += space) {
    this.ctx.moveTo(0, i + space); this.ctx.lineTo(this.height, i + space);
  }
  for (var i = 0; i < this.height - space; i += space) {
    this.ctx.moveTo(i + space, 0); this.ctx.lineTo(i + space, this.height);
  }
  this.ctx.stroke();
}
ColorCurve.prototype.drawColorHist = function () {
  var colors = ['rgb(255, 0, 0)', 'rgb(0, 255, 0)', 'rgb(0, 0, 255)'];
  var histograms = [colorHist.r, colorHist.g, colorHist.b];
  histograms.forEach((hist, index) => {
    this.ctx.fillStyle = colors[index];
    hist.forEach((count, i) => {
      var height = (count / Math.max(...colorHist.r, ...colorHist.g, ...colorHist.b)) * this.c.height;
      this.ctx.fillRect(i, this.c.height - height, 1, height);
    });
  });
}
ColorCurve.prototype.quadratic = function (p1, p2, p3, p4) {
  'use strict';

  var x0, x1, x2, x3, y0, y1, y2, y3, dx, dy;

  this.ctx.strokeStyle = '#000000';
  this.ctx.lineWidth = 1.5;
  var slope = 0;

  x0 = p2.x;
  x3 = p3.x;

  y0 = p2.y;
  y3 = p3.y;

  dx = x3 - x0;
  dy = y3 - y0;

  x1 = ((2.0 * x0) / 3.0) + (x3 / 3.0)
  x2 = (x0 / 3.0) + ((2.0 * x3) / 3.0);


  if (this.isEqual(p1, p2) && this.isEqual(p3, p4)) {
    y1 = y0 + (dy / 3.0);
    y2 = y0 + ((dy * 2.0) / 3.0);
  }
  if (this.isEqual(p1, p2) && !this.isEqual(p3, p4)) {

    slope = ((p4.y) - y0) / (p4.x - x0);
    y2 = y3 - ((slope * dx) / 3.0);
    y1 = y0 + ((y2 - y0) / 2.0);

  }
  if (!this.isEqual(p1, p2) && this.isEqual(p3, p4)) {
    slope = (y3 - (p1.y)) / (x3 - p1.x);

    y1 = y0 + ((slope * dx) / 3.0);
    y2 = y3 + ((y1 - y3) / 2.0);
  }

  if (!this.isEqual(p1, p2) && !this.isEqual(p3, p4)) {
    slope = (y3 - (p1.y)) / (x3 - p1.x);
    y1 = y0 + ((slope * dx) / 3.0);
    slope = ((p4.y) - y0) / (p4.x - x0);
    y2 = y3 - ((slope * dx) / 3.0);
  }

  this.ctx.beginPath();
  this.ctx.moveTo(x0 * this.width, this.height - (y0 * this.height));

  var step = (x3 - x0) / 20.0;
  var tx = x0;

  for (var i = 0.0; i <= 1.05; i += 0.05) {

    var ty = (y0 * Math.pow((1 - i), 3)) +
      (3 * y1 * Math.pow((1 - i), 2) * i) +
      (3 * y2 * (1 - i) * i * i) +
      (y3 * i * i * i);

    this.ctx.lineTo(tx * this.width, this.height - (ty * this.height));
    this.values.push({ x: tx, y: ty });
    tx = tx + step;

  }

  this.ctx.moveTo(0, this.height - (this.points[0].y * this.height));
  this.ctx.lineTo(this.points[0].x * this.width, this.height - (this.points[0].y * this.height));

  this.ctx.moveTo(this.points[1].x * this.width, this.height - (this.points[1].y * this.height));
  this.ctx.lineTo(this.width, this.height - (this.points[1].y * this.height));

  this.ctx.stroke();
  return true;
}
ColorCurve.prototype.drawPoints = function () {
  'use strict';

  this.ctx.fillStyle = '#ff0000';
  this.ctx.beginPath();

  for (var i = 0; i < this.points.length; i++) {
    this.ctx.moveTo(this.points[i].x * this.width, this.height - (this.points[i].y * this.height));
    this.ctx.arc(this.points[i].x * this.width, this.height - (this.points[i].y * this.height), 3, 0, 2 * Math.PI, false);
  }
  this.ctx.fill();
}
ColorCurve.prototype.mouseDown = function (event) {
  'use strict';

  if (!event) var event = window.event;
  var rect = this.c.getBoundingClientRect();
  var x = (event.clientX - rect.left) / this.width, y = (event.clientY - rect.top) / this.height;

  var dis = 10000;
  var punto = -1;

  for (var i = 0; i < this.points.length; i++) {
    var x1 = x - this.points[i].x;
    var y1 = y - (1.0 - this.points[i].y);

    var tdis = x1 * x1 + y1 * y1;
    var tdis = Math.sqrt(tdis);

    if (tdis < dis) {
      dis = tdis;
      punto = i;
    }
  }
  this.currentPoint = (dis < 8.0) ? punto : this.currentPoint;
}
ColorCurve.prototype.mouseUp = function (event) {
  'use strict';
  if (this.currentPoint != -1) {
    this.updateValues();
  }
  this.currentPoint = -1;
}
ColorCurve.prototype.mouseMove = function (event) {

  'use strict';

  var prevx, nextx, prevy, nexty;

  if (this.currentPoint == -1) return;

  if (this.currentPoint > 0) prevx = this.points[this.currentPoint - 1].x; else prevx = 0;
  if (this.currentPoint == this.points.length - 1) nextx = 1.0; else nextx = this.points[this.currentPoint + 1].x;
  if (this.currentPoint > 0) prevy = this.points[this.currentPoint - 1].y; else prevy = 0;
  if (this.currentPoint == this.points.length - 1) nexty = 1.0; else nexty = this.points[this.currentPoint + 1].y;

  var x = (event.pageX - this.c.offsetLeft) / this.width;
  var y = 1.0 - ((event.pageY - this.c.offsetTop) / this.height);

  if (x > prevx && x < nextx && y > prevy && y < nexty) {
    this.points[this.currentPoint].x = x;
    this.points[this.currentPoint].y = y;

    this.redraw = 1;
  }
}
ColorCurve.prototype.getY = function (xpos) {
  'use strict';

  if (xpos < this.values[0].x) xpos = this.values[0].x;
  if (xpos > this.values[this.values.length - 1].x) xpos = this.values[this.values.length - 1].x;

  for (var i = 0; i < this.values.length - 2; i++) {
    if (xpos >= this.values[i].x && xpos < this.values[i + 1].x) break;
  }
  var valuea = (xpos - this.values[i].x) / (this.values[i + 1].x - this.values[i].x);
  var valueb = valuea * (this.values[i + 1].y - this.values[i].y);

  var ret = this.values[i].y + valueb;

  if (ret < 0.0) return 0.0;
  if (ret > 1.0) return 1.0;

  return ret;
}
var Filter = Filter || {};
var CC;
Filter.Init = function () {
  if (imgheight > imgwidth) {
    cModified.height = imgheight;
    cModified.width = imgheight;
  } else {
    cModified.height = imgwidth;
    cModified.width = imgwidth;
  }
  var c = document.getElementById("myCanvas");
  var ctx = c.getContext("2d", { willReadFrequently: true });
  var imageData = ctx.getImageData(imgpos.x, imgpos.y, imgwidth, imgheight);
  this.numPixels = imageData.width * imageData.height;
  this.originalPixels = imageData.data;
  var ctx = cModified.getContext("2d", { willReadFrequently: true });
  ctx.putImageData(imageData, 0, 0);
  this.canvas = cModified;
  this.ctx = this.canvas.getContext("2d", { willReadFrequently: true });
}
Filter.applyFilter = function () {
  if (CC == undefined) return;
  var imageData = this.ctx.getImageData(0, 0, this.canvas.width, this.canvas.height);
  var pix = imageData.data;
  for (var i = 0; i < pix.length; i++) {
    pix[i] = CC.rgb[Filter.originalPixels[i]];
  }
  colorHist = {
    r: new Array(256).fill(0),
    g: new Array(256).fill(0),
    b: new Array(256).fill(0),
  };
  for (var i = 0; i < pix.length; i += 4) {
    colorHist.r[pix[i]]++;
    colorHist.g[pix[i + 1]]++;
    colorHist.b[pix[i + 2]]++;
  }
  Filter.ctx.putImageData(imageData, 0, 0);
  curvedImg = this.ctx.getImageData(0, 0, imgwidth, imgheight);
  if (document.getElementById("preview").checked) {
    drawFilter();
  }
}
function drawFilter() {
  var c = document.getElementById("myCanvas");
  var ctx = c.getContext("2d", { willReadFrequently: true });
  c.style.imageRendering = "pixelated";
  ctx.clearRect(0, 0, c.width, c.height);
  ctx.putImageData(curvedImg, imgpos.x, imgpos.y);
}
function checkCurve() {
  if (CC == undefined) return;
  Filter.Init();
  Filter.applyFilter();
  drawFilter();
}
export default {
  data() {
    return {
      sizedialog: false,
      image: null,
      colorData: "",
      posData: "",
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
      rgb1: "",
      rgb2: "",
      xyz1: "",
      xyz2: "",
      lab1: "",
      lab2: "",
      pos1: "",
      pos2: "",
      swatch1: "",
      swatch2: "",
      contrast: "",
      preview: false,
      posx1: 0,
      posy1: 0,
      posx2: 255,
      posy2: 255,
    }
  },
  methods: {
    createImage() {
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d", { willReadFrequently: true });
      c.style.imageRendering = "pixelated";
      c.width = window.innerWidth;
      c.height = window.innerHeight - $("footer").height();
      img.onload = function () {
        newimgwidth = img.width;
        newimgheight = img.height;
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
          scale = (width * height) / (img.width * height);
          this.scaleSlider = scale;
          imgpos = { x: c.width / 2 - width / 2, y: c.height / 2 - height / 2 };
          imgwidth = width;
          imgheight = height;
          ctx.drawImage(img, imgpos.x, imgpos.y, imgwidth, imgheight);
          document.getElementById("dim").textContent = "Размер изображения: " + [Math.floor(imgwidth.toFixed(2)), Math.floor(imgheight.toFixed(2))].toString();
        }
        this.newMP = (imgwidth * imgheight / 1000000).toFixed(2);
        document.getElementById("dim").textContent = "Размер изображения: " + [Math.floor(imgwidth.toFixed(2)), Math.floor(imgheight.toFixed(2))].toString();
        var sizes = [];
        sizes[0] = "";
        var imgdata = ctx.getImageData(imgpos.x, imgpos.y, imgwidth, imgheight).data;
        for (var i = 0; i < imgdata.length; i += 4) {
          colorHist.r[imgdata[i]]++;
          colorHist.g[imgdata[i + 1]]++;
          colorHist.b[imgdata[i + 2]]++;
        }
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
      var ctx = c.getContext("2d", { willReadFrequently: true });
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
      checkCurve();
    },
    updateSize() {
      if (parseFloat(this.measureW) <= 0 || parseFloat(this.measureH) <= 0) {
        alert("Размеры изображения не должны быть меньше или равны 0");
        return;
      }
      this.sizedialog = false;
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d", { willReadFrequently: true });
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
      checkCurve();
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
      if (this.toggle == 1) this.toggle = -1;
    },
    openCurves() {
      var x = document.getElementById("curves");
      x.style.display = "block";
      this.curveStart();
    },
    closeCurves() {
      CC.points[0].x = oldCCpoints[0];
      CC.points[0].y = oldCCpoints[1];
      CC.points[1].x = oldCCpoints[2];
      CC.points[1].y = oldCCpoints[3];
      CC.draw();
      CC.updateValues();
      this.checkDots();
      checkCurve();
      var x = document.getElementById("curves");
      x.style.display = "none";
      if (this.toggle == 2) this.toggle = -1;
    },
    resetCurves() {
      CC.points[0].x = 0;
      CC.points[0].y = 0;
      CC.points[1].x = 1;
      CC.points[1].y = 1;
      CC.draw();
      CC.updateValues();
      this.checkDots();
      checkCurve();
    },
    applyCurves() {
      Filter.applyFilter();
      drawFilter();
      oldCCpoints[0] = CC.points[0].x;
      oldCCpoints[1] = CC.points[0].y;
      oldCCpoints[2] = CC.points[1].x;
      oldCCpoints[3] = CC.points[1].y;
      var x = document.getElementById("curves");
      x.style.display = "none";
      if (this.toggle == 2) this.toggle = -1;
    },
    curveStart() {
      if (CC == undefined) {
        CC = new ColorCurve('selector', function () { Filter.applyFilter(); });
      }
      Filter.Init();
    },
    getColor(evt) {
      if (this.toggle == 1) {
        var c = document.getElementById("myCanvas");
        var ctx = c.getContext("2d", { willReadFrequently: true });
        var pos = this.getMousePos(c, evt);
        if ((pos.x >= imgpos.x) && (pos.x < imgpos.x + imgwidth) && (pos.y >= imgpos.y) && (pos.y < imgpos.y + imgheight)) {
          var pixelData = ctx.getImageData(pos.x, pos.y, 1, 1);
          var xyz = this.RGBtoXYZ([pixelData.data[0], pixelData.data[1], pixelData.data[2]]);
          if (evt.altKey) {
            this.swatch2 = this.colorData;
            this.pos2 = this.posData;
            this.rgb2 = this.colorData;
            this.xyz2 = "XYZ(" + (xyz[0] * 100).toFixed(2) + "%, " + (xyz[1] * 100).toFixed(2) + "%, " + (xyz[2] * 100).toFixed(2) + "%)";
            this.lab2 = this.RGBtoLAB([pixelData.data[0], pixelData.data[1], pixelData.data[2]]);
            luma2 = xyz[1];
          } else {
            this.swatch1 = this.colorData;
            this.pos1 = this.posData;
            this.rgb1 = this.colorData;
            this.xyz1 = "XYZ(" + (xyz[0] * 100).toFixed(2) + "%, " + (xyz[1] * 100).toFixed(2) + "%, " + (xyz[2] * 100).toFixed(2) + "%)";
            this.lab1 = this.RGBtoLAB([pixelData.data[0], pixelData.data[1], pixelData.data[2]]);
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
      var ctx = c.getContext("2d", { willReadFrequently: true });
      var pos = this.getMousePos(c, evt);
      var pixelData = ctx.getImageData(pos.x, pos.y, 1, 1);
      if (this.toggle == 0) {
        if (!mouseDown) {
          imgpos = { x: imgpos.x + (pos.x - initPos.x), y: imgpos.y + (pos.y - initPos.y) };
          initPos = null;
          checkCurve();
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
      var c = document.getElementById("myCanvas");
      var ctx = c.getContext("2d", { willReadFrequently: true });
      var imageData = ctx.getImageData(imgpos.x, imgpos.y, imgwidth, imgheight);
      var c = document.createElement("canvas");
      var ctx = c.getContext("2d", { willReadFrequently: true });
      c.style.imageRendering = "pixelated";
      c.width = imgwidth
      c.height = imgheight
      ctx.putImageData(imageData, 0, 0);
      var link = document.createElement("a");
      link.download = "image.png";
      link.href = c.toDataURL()
      link.click();
    },
    checkPreview() {
      if (document.getElementById("preview").checked) {
        Filter.applyFilter();
      } else {
        var tmp = [CC.points[0].x, CC.points[0].y, CC.points[1].x, CC.points[1].y];
        CC.points[0].x = oldCCpoints[0];
        CC.points[0].y = oldCCpoints[1];
        CC.points[1].x = oldCCpoints[2];
        CC.points[1].y = oldCCpoints[3];
        CC.draw();
        CC.updateValues();
        Filter.applyFilter();
        drawFilter();
        CC.points[0].x = tmp[0];
        CC.points[0].y = tmp[1];
        CC.points[1].x = tmp[2];
        CC.points[1].y = tmp[3];
        CC.draw();
        CC.updateValues();
      }
    },
    checkDots() {
      if (CC == undefined) return;
      this.posx1 = Math.ceil(CC.points[0].x * 255);
      this.posy1 = Math.ceil(CC.points[0].y * 255);
      this.posx2 = Math.ceil(CC.points[1].x * 255);
      this.posy2 = Math.ceil(CC.points[1].y * 255);
    },
    updateDots() {
      if (CC == undefined) return;
      if (this.posx1 < 0) this.posx1 = 0;
      if (this.posy1 < 0) this.posy1 = 0;
      if (this.posx2 > 255) this.posx2 = 255;
      if (this.posy2 > 255) this.posy2 = 255;
      if (this.posx1 > this.posx2) this.posx1 = this.posx2 - 1;
      if (this.posy1 > this.posy2) this.posy1 = this.posy2 - 1;
      if (this.posx2 < this.posx1) this.posx2 = this.posx1 + 1;
      if (this.posy2 < this.posy1) this.posy2 = this.posy1 + 1;
      CC.points[0].x = this.posx1 / 255;
      CC.points[0].y = this.posy1 / 255;
      CC.points[1].x = this.posx2 / 255;
      CC.points[1].y = this.posy2 / 255;
      CC.draw();
      CC.updateValues();
    }
  }
}
</script>
