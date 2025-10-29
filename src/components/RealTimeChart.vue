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
  },
  false,//允许内部合并，避免完全重建notMerge
  true,//延迟更新，减少立即计算的压力lazyUpdate
  );
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
  chart.value = proxy.$echarts.init(container,{
    renderer:"canvas",
    useDirtyRect:true,//切换为 Canvas 渲染并启用 useDirtyRect，减少重绘范围。
  });
  const option = {
    xAxis: {
      type: "time",
      data: [],
      boundaryGap: false
    },
    yAxis: {
      type: "value",
    },
    series: [
      {
        data: [],
        type: "line",
        showSymbol: false,//禁用动画并隐藏数据点符号，降低每次绘制的像素开销。
        sampling: 'lttb',//开启采样（lttb），在大量点位时进行下采样，保留趋势
        smooth: true,//开启平滑曲线，使数据点之间的变化更自然。
      },
    ],
    dataZoom: [
        {
            id: 'dataZoomX',
            type: 'slider',
            xAxisIndex: [0],
            filterMode: 'filter', // 设定为 'filter' 从而 X 的窗口变化会影响 Y 的范围。
            start: 30,
            end: 70
        },
        {
            id: 'dataZoomY',
            type: 'slider',
            yAxisIndex: [0],
            filterMode: 'empty',
            start: 20,
            end: 80
        }
    ],  
    animation: false,
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
