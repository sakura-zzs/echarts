<template>
  <input type="number" v-model="duration" placeholder="输入持续时间（毫秒）" />
  <input type="number" v-model="count" placeholder="输入数据数量(每100ms)" />
  <button @click="startChart">开始</button>
  <button @click="stopChart">停止</button>
  <button @click="resetChart">重置</button>
  <div id="container"></div>
</template>
<script setup ts>
import { onMounted, getCurrentInstance, ref } from "vue";
const { proxy } = getCurrentInstance();
const chart=ref(null)
const duration = ref(30000);
const count = ref(100);
let timer = null;

const generateRandomData = (duration, count) => {
  const loopCount = duration / 100;
  let t = 0;
  const res = [];
  timer = setInterval(() => {
    t++;
    for (let i = 0; i < count; i++) {
      res.push([Date.now(), Math.random() * 100]);
    }
    updateChart(res);
    if (t > loopCount) {
      clearInterval(timer);
      return;
    }
  }, 100);
  return res;
};

const updateChart = (data) => {
  chart.value.setOption({
    series: [
      {
        data,
      },
    ],
  });
};

const startChart = () => {
  generateRandomData(duration.value, count.value);
};

const stopChart = () => {
  clearInterval(timer);
};

const resetChart = () => {
  timer && clearInterval(timer);
  chart.value.setOption({
    series: [
      {
        data: [],
      },
    ],
  });
};



const createChart = () => {
  const container = document.getElementById("container");
  chart.value = proxy.$echarts.init(container);
  const option = {
    xAxis: {
      type: "time",
      data: [],
    },
    yAxis: {
      type: "value",
    },
    series: [
      {
        data: [],
        type: "line",
        showSymbol: true,
      },
    ],
    animation: true,
  };
  chart.value.setOption(option);
};

onMounted(() => {
  createChart();
});
</script>
<style scoped>
#container {
  width: 1200px;
  height: 600px;
}
</style>
