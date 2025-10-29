<template>
  <input type="number" v-model="duration" placeholder="输入持续时间（毫秒）" />
  <input type="number" v-model="count" placeholder="输入数据数量(每100ms)" />
  <button @click="startChart">开始</button>
  <button @click="stopChart">停止</button>
  <button @click="resetChart">重置</button>
  <div id="container"></div>
</template>
<script setup lang="ts">
import { onMounted, getCurrentInstance, ref } from "vue";
const { proxy } = getCurrentInstance();
const chart=ref(null)
const duration = ref(30000);
const count = ref(100);
let timer = null;
const incomingBuffer:Array<[number,number]> = [];// incomingBuffer 用于缓存实时数据，避免频繁更新图表。
let flushTimer:number|null = null;
const FLUSH_INTERVAL = 300; // 每300ms刷新一次
const dataSource:Array<[number,number]> = [];
const Max_Data_Count = 6000; // 最大数据量，超过此数量时，会删除旧数据
let resyncTimer:number|null = null;
// appendData 长时间运行后，图表内部状态可能累积；每隔一段时间做一次全量同步，重置状态并确保一致性。
const RESYNC_INTERVAL = 10000; // 每10s重新同步一次数据

const resyncFullData = () => {
  chart.value.setOption({
    series: [
      {
        data: dataSource.slice(),
      },
    ],
  },
  true,//全量刷新
  true,//延迟更新，减少立即计算的压力lazyUpdate
  );
};

const flushBuffer = () => {
  if (incomingBuffer.length === 0) {
    return;
  }
  const batch = incomingBuffer.splice(0, incomingBuffer.length);
  dataSource.push(...batch);
  if (dataSource.length > Max_Data_Count) {
    // 删除旧数据，保留最新的 Max_Data_Count 条数据
    dataSource.splice(0, dataSource.length - Max_Data_Count);
  }
  // 增量更新
  chart.value.appendData({
    seriesIndex: 0,//折线序列索引
    data: batch,
  });
};

const generateRandomData = (duration, count) => {
  const loopCount = duration / 100;
  let t = 0;
  const res = [];
  timer = setInterval(() => {
    t++;
    for (let i = 0; i < count; i++) {
      res.push([Date.now(), Math.random() * 100]);
    }
    incomingBuffer.push(...res);
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
  flushTimer && clearInterval(flushTimer);
  generateRandomData(duration.value, count.value);
  flushTimer = setInterval(flushBuffer, FLUSH_INTERVAL);
  resyncTimer = setInterval(resyncFullData, RESYNC_INTERVAL);
};

const stopChart = () => {
  clearInterval(timer);
  flushTimer && clearInterval(flushTimer);
  resyncTimer && clearInterval(resyncTimer);
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
