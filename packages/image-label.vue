<script setup lang="ts">
import { defineProps, onMounted, ref } from "vue";
import LoadingIcon from "../images/spinner.gif";

interface IContainer {
  width?: number;
  height?: number;
}

interface IPoint {
  size?: number;
  color?: string;
}

type Point = {
  x: number;
  y: number;
};

interface ILine {
  width?: number;
  color?: string;
}

interface IImageLabel {
  url: string;
  containerProps?: IContainer;
  lineProps?: ILine;
  pointProps?: IPoint;
  getPoints?: (points: Point[]) => void;
}
// 组件属性
const { url, containerProps, lineProps, pointProps, getPoints } =
  defineProps<IImageLabel>();

const points: Point[] = []; // 坐标点
let __img_scale_ = 1; // 容器中的图片相对于原始图的缩放倍数
let ctx = {} as CanvasRenderingContext2D;
let img = {} as HTMLImageElement;
let currentSelectedPointIndex = -1;
let mouseStyleTimer = 0;

// 容器元素
const container = ref<HTMLDivElement>();
// 画布元素
const canvas = ref<HTMLCanvasElement>();

const loading = ref(false);
const containerW = ref(containerProps?.width || 1000);
const containerH = ref(containerProps?.height || 800);

onMounted(() => {
  loadPictureAndInitCanvas();
  canvas?.value?.addEventListener("wheel", mouseWhell);
  canvas?.value?.addEventListener("mousedown", listenMouseDown);
  canvas?.value?.addEventListener("mousemove", listenMouseMove);
});
// 加载图片并初始化画布
const loadPictureAndInitCanvas = () => {
  ctx = canvas?.value?.getContext("2d") as CanvasRenderingContext2D;
  loading.value = true;
  img = document.createElement("img");
  img.alt = "trace img";
  img.id = "img";
  img.src = url;
  // image loaded
  img.onload = function () {
    loading.value = false;
    container.value?.append(img);
    const { width, height } = img;
    if (width > height) {
      img.style.width = "80%";
    } else {
      img.style.height = "80%";
    }
    // 容器中的图片尺寸
    const { clientWidth, clientHeight } = img;
    __img_scale_ = clientWidth / width;
    if (canvas.value) {
      canvas.value.style.width = clientWidth + "px";
      canvas.value.style.height = clientHeight + "px";
      canvas.value.width = clientWidth;
      canvas.value.height = clientHeight;
      canvas.value.style.left = `calc(50% - ${clientWidth / 2}px)`;
      canvas.value.style.top = `calc(50% - ${clientHeight / 2}px)`;
    }
    ctx.drawImage(img, 0, 0, clientWidth, clientHeight);
    img.style.scale = "1";
    initPoints(clientWidth, clientHeight);
    mouseWhellDrawPointAndLine(1);
  };
  // image load error
  img.onerror = function () {
    loading.value = false;
  };
};

// 获取图片缩放倍数
const getScale = (): number => {
  return Number(img.style.scale);
};

// 鼠标滚动事件监听
const mouseWhell = (e: MouseEvent) => {
  e.preventDefault();
  mouseWheelDrawPicture(e);
  mouseWhellDrawPointAndLine(getScale());
};

// 绘制图片
const mouseWheelDrawPicture = (e: MouseEvent) => {
  /**
   * deltaY 正值:向下滚动 缩小; 负值:向上滚动 放大
   */
  let top = 0;
  let left = 0;
  // @ts-ignore
  const { deltaY, offsetX, offsetY } = e;
  if (deltaY < 0 && getScale() < 5 && canvas?.value) {
    img.style.scale = JSON.stringify(getScale() + 0.1);
    // 计算出这次放大相比于上一次调整了canvas多少的offsetTop和offsetLeft
    left =
      canvas.value?.offsetLeft -
      ((offsetX * getScale()) / (getScale() - 0.1) - offsetX);
    top =
      canvas.value.offsetTop -
      ((offsetY * getScale()) / (getScale() - 0.1) - offsetY);
    canvas.value.style.left = left + "px";
    canvas.value.style.top = top + "px";
    canvas.value.style.width = img.clientWidth * getScale() + "px";
    canvas.value.style.height = img.clientHeight * getScale() + "px";
    canvas.value.width = img.clientWidth * getScale();
    canvas.value.height = img.clientHeight * getScale();
  }
  if (deltaY >= 0 && getScale() - 0.1 >= 0.1 && canvas?.value) {
    img.style.scale = JSON.stringify(getScale() - 0.1);
    // 计算出这次缩小相比于上一次调整了canvas多少的offsetTop和offsetLeft
    left =
      canvas.value.offsetLeft -
      (offsetX * getScale()) / (getScale() + 0.1) +
      offsetX;
    top =
      canvas.value.offsetTop -
      (offsetY * getScale()) / (getScale() + 0.1) +
      offsetY;

    canvas.value.style.left = left + "px";
    canvas.value.style.top = top + "px";
    canvas.value.style.width = img.clientWidth * getScale() + "px";
    canvas.value.style.height = img.clientHeight * getScale() + "px";
    canvas.value.width = img.clientWidth * getScale();
    canvas.value.height = img.clientHeight * getScale();
  }

  ctx.drawImage(
    img,
    0,
    0,
    img.clientWidth * getScale(),
    img.clientHeight * getScale()
  );
};

const listenMouseDown = (e: MouseEvent) => {
  if (!canvas.value) {
    return;
  }
  const { clientX, clientY, offsetX, offsetY } = e;
  // 画布距离原点的位置
  const spaceX = clientX - canvas.value.offsetLeft || 0;
  const spaceY = clientY - canvas.value.offsetTop || 0;
  const pointSize = pointProps?.size ? pointProps?.size * 2 : 20;
  points.forEach((point, index) => {
    const scalePoint = { x: point.x * getScale(), y: point.y * getScale() };
    const isSelected =
      Math.abs(offsetX - scalePoint.x) <= pointSize &&
      Math.abs(offsetY - scalePoint.y) <= pointSize;
    if (isSelected) {
      currentSelectedPointIndex = index;
    }
  });
  // 选中点 拖拽点
  if (currentSelectedPointIndex >= 0) {
    canvas.value.onmousemove = function (event) {
      points[currentSelectedPointIndex].x = event.offsetX / getScale();
      points[currentSelectedPointIndex].y = event.offsetY / getScale();
      ctx.clearRect(
        0,
        0,
        img.clientWidth * getScale(),
        img.clientHeight * getScale()
      );
      ctx.drawImage(
        img,
        0,
        0,
        img.clientWidth * getScale(),
        img.clientHeight * getScale()
      );
      mouseWhellDrawPointAndLine(getScale());
    };
  } else {
    // 未选中点 拖拽画布
    document.onmousemove = function (event) {
      if (!canvas.value) {
        return;
      }
      const left = event.clientX - spaceX;
      const top = event.clientY - spaceY;
      canvas.value.style.left = left + "px";
      canvas.value.style.top = top + "px";
    };
  }
  document.onmouseup = function () {
    if (!canvas.value) {
      return;
    }
    canvas.value?.setAttribute("class", "");
    document.onmousemove = null;
    canvas.value.onmousemove = null;
    document.onmouseup = null;
    currentSelectedPointIndex = -1;
  };
};

// 鼠标滚动绘制点线
const mouseWhellDrawPointAndLine = (scale: number) => {
  if (!points.length) {
    return;
  }
  // 点
  points.forEach((point) => {
    ctx.beginPath();
    ctx.arc(
      point.x * scale,
      point.y * scale,
      pointProps?.size || 10,
      0,
      2 * Math.PI
    );
    ctx.fillStyle = pointProps?.color || "red";
    ctx.fill();
    ctx.closePath();
  });
  ctx.beginPath();
  ctx.setLineDash([10, 5]);
  ctx.lineWidth = lineProps?.width || 2;
  ctx.strokeStyle = lineProps?.color || "red";
  // 线
  points.forEach(({ x, y }) => {
    ctx.lineTo(x * scale, y * scale);
  });
  ctx.lineTo(points[0].x * scale, points[0].y * scale);
  ctx.stroke();
  // 执行获取坐标点的回调
  getPoints && getPoints(getOriginPoints());
};

// 获取图片上的原始坐标点
const getOriginPoints = () => {
  if (!points.length) {
    console.error("画布暂时没有初始化");
    return [];
  }
  return points.map(({ x, y }) => ({
    x: Number((x / __img_scale_).toFixed(2)),
    y: Number((y / __img_scale_).toFixed(2)),
  }));
};

// 鼠标移动事件,处理鼠标指针
const listenMouseMove = (e: MouseEvent) => {
  if (!canvas.value) {
    return;
  }
  const { offsetX, offsetY } = e;
  let moveToPoint = false;
  const pointSize = pointProps?.size ? pointProps?.size * 2 : 20;
  points.forEach((point) => {
    const scalePoint = { x: point.x * getScale(), y: point.y * getScale() };
    const confirm =
      Math.abs(offsetX - scalePoint.x) <= pointSize &&
      Math.abs(offsetY - scalePoint.y) <= pointSize;
    if (confirm) {
      moveToPoint = true;
    }
  });
  if (mouseStyleTimer) {
    clearTimeout(mouseStyleTimer);
  } else {
    setTimeout(() => {
      canvas.value?.setAttribute("class", moveToPoint ? "selected" : "");
    }, 100);
  }
};

// 默认点加载， 默认取画布中心点80%的正方形区域
const initPoints = (width: number, height: number) => {
  if (points.length > 0) {
    return;
  }
  const leftTop = { x: width * 0.2, y: height * 0.2 };
  const rightTop = { x: width * 0.8, y: height * 0.2 };
  const rightBottom = { x: width * 0.8, y: height * 0.8 };
  const leftBottom = { x: width * 0.2, y: height * 0.8 };
  points.push(leftTop);
  points.push(rightTop);
  points.push(rightBottom);
  points.push(leftBottom);
};
</script>

<template>
  <div
    ref="container"
    class="image-label__container"
    :style="{
      width: containerW + 'px',
      height: containerH + 'px',
    }"
  >
    <div v-if="loading" class="status">
      <img :src="LoadingIcon" alt="loading icon" width="100" height="100" />
    </div>
    <canvas ref="canvas" />
  </div>
</template>
  
<style>
.image-label__container {
  position: relative;
  overflow: hidden;
  background-color: #f5f5f5;
  background-image: url("data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQAQMAAAAlPW0iAAAAA3NCSVQICAjb4U/gAAAABlBMVEXMzMz////TjRV2AAAACXBIWXMAAArrAAAK6wGCiw1aAAAAHHRFWHRTb2Z0d2FyZQBBZG9iZSBGaXJld29ya3MgQ1M26LyyjAAAABFJREFUCJlj+M/AgBVhF/0PAH6/D/HkDxOGAAAAAElFTkSuQmCC");
}
.image-label__container canvas {
  position: absolute;
  cursor: move;
}

.image-label__container #img {
  position: absolute;
  right: 1000000px;
  bottom: 1000000px;
}

.image-label__container .selected {
  cursor: crosshair;
}
.image-label__container .status {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  z-index: 2;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>
  