<script setup lang="ts">
import { ref } from 'vue'
import VueLabelImage from "vue-image-label";

interface Point {x: number; y:number}

const points = ref<Point[]>([])

const getPoints = (params: Point[]) => {
  points.value = params
}
</script>

<template>
  <div class="wrapper">
    <VueLabelImage
      url="https://stbrain.kjt.zj.gov.cn/Documents/ac681331-de00-4fcb-b082-27159b32f6c5/Instruments/43%20%E7%A2%B3%E7%A1%AB%E5%88%86%E6%9E%90%E4%BB%AA.jpg"
      :get-points="getPoints"
    />
    <div v-if="points.length" className="footer">
      <h3>坐标点信息:</h3>
      <ul className="points">
        <li v-for="point in points" :key="point.x + point.y">
          {{point.x}}, {{point.y}}
        </li>
      </ul>
      </div>
  </div>
</template>

<style scoped>

.wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100vw;
  height: 100vh;
  padding:  24px 0;
  background-color: #000;
  box-sizing: border-box;
}
.footer {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 1000px;
  background: #333;
  height: 60px;
  margin-top: 24px;
}
h3 {
  color: aliceblue;
}
.points {
  display: flex;
  list-style: none;
  min-width: 600px;
}

.points li {
  margin-left: 24px;
  color: aliceblue;
}


</style>
