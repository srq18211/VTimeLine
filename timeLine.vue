<template>
  <div
    ref="timeLineWrap"
    id="timeLineWrap"
    style="height:200px;background: #fff"
  >
    <svg
      ref="timeLine"
      width="100%"
      height="100%"
      :viewBox="`0 0 ${viewBoxWidth} 150`"
      style="transition:all 0.3s"
    >
      <!--坐标 轴-->
      <line
        ref="baseLine"
        x1="20"
        y1="75"
        :x2="viewBoxWidth - 20"
        y2="75"
        stroke="#eeedf0"
        stroke-width="5"
      ></line>

      <!--坐标 刻度-->
      <template v-for="(item, index) in rangeLength">
        <line
          v-if="index !== 0"
          :x1="index * rangeItemWidth"
          y1="65"
          :x2="index * rangeItemWidth"
          y2="75"
          stroke="#eeedf0"
          stroke-width="1"
        />
      </template>

      <!--坐标 刻度 信息 -->
      <template v-for="(item, index) in rangeLength">
        <!-- 刻度 文字 第一个-->
        <text
          v-if="index == 0"
          text-anchor="middle"
          :x="index * rangeItemWidth + 20"
          y="90"
          class="coordinates"
          color="#000"
        >
          0:00
        </text>
        <!-- 刻度 文字 -->
        <text
          v-else
          text-anchor="middle"
          :x="index * rangeItemWidth"
          y="90"
          class="coordinates"
        >
          {{ index + start }}:00
        </text>
      </template>
      <!-- 坐标 结尾 -->
      <text
        text-anchor="middle"
        :x="rangeLength * rangeItemWidth - 20"
        y="90"
        class="coordinates"
        color="#000"
      >
        24:00
      </text>

      <template v-for="(item, index) in renderData">
        <!--时间段 开始 竖线-->
        <line
          stroke-dasharray="3,2"
          :x1="item.X1"
          :y1="60 - item.X1offset * 20"
          :x2="item.X1"
          y2="75"
          stroke="blue"
          stroke-width="1"
        ></line>
        <!--时间段 开始 圆圈-->
        <circle
          class="time-item-circle"
          :cx="item.X1"
          :cy="60 - item.X1offset * 20"
          r="2"
          fill="#fff"
          stroke="blue"
        />
        <!-- 时间段 开始 文字 -->
        <text
          class="time-text"
          text-anchor="middle"
          :x="item.X1"
          :y="50 - item.X1offset * 20"
        >
          {{ item.start }}
        </text>

        <line
          :x1="item.X1"
          y1="75"
          :x2="item.X2"
          y2="75"
          stroke="blue"
          stroke-width="5"
        ></line>

        <!--结束时间 竖线-->
        <line
          stroke-dasharray="3,2"
          :x1="item.X2"
          y1="75"
          :x2="item.X2"
          :y2="100 + item.X2offset * 20"
          stroke="blue"
          stroke-width="1"
        ></line>
        <!-- 结束时间 圆圈 -->
        <circle
          class="time-item-circle"
          :cx="item.X2"
          :cy="95 + item.X2offset * 20"
          r="2"
          fill="#fff"
          stroke="blue"
        />
        <!-- 结束时间 文字提示 -->
        <text
          class="time-text"
          text-anchor="middle"
          :x="item.X2"
          :y="110 + item.X2offset * 20"
        >
          {{ item.end }}
        </text>
      </template>
    </svg>
  </div>
</template>

<script>
// import throttling from '@/tools/throttling.js';
function throttle(func, wait) {
  let previous = 0;
  return function() {
    let now = Date.now();
    let context = this;
    let args = arguments;
    if (now - previous > wait) {
      func.apply(context, args);
      previous = now;
    }
  };
}
export default {
  props: {
    data: {
      type: Array,
      default: function() {
        return [];
      }
    }
  },
  name: 'timeLine',
  data() {
    return {
      start: 0, // 展示时段 限制
      end: 24, // 展示时段 限制
      viewBoxWidth: 0,
      textWidth: 0,
      labelOffset: {
        size: 40,
        deep: 3
      }
    };
  },
  computed: {
    // 根据数据 划分时间段
    rangeLength() {
      const { end, start } = this;
      return +end - +start;
    },
    // 1小时 宽度
    rangeItemWidth() {
      return Math.floor(this.viewBoxWidth / this.rangeLength);
    },
    // 模板渲染 数据
    renderData() {
      // 处理 远程数据：筛选超出时段 的数据
      let showData = this.data.filter(i => {
        const [m, M] = i.start.split(':');
        return m >= this.start;
      });

      const { viewBoxWidth, labelOffset } = this;
      // 用户数据 添加 时间段坐标信息
      let arr = showData.map((i, index) => {
        // 获取  时：分
        const [m, M] = i.start.split(':');
        const [m1, M1] = i.end.split(':');

        const s1 = (+m - this.start) * 60 + +M;
        const s2 = (+m1 - this.start) * 60 + +M1;

        // 比如 展示8点到20 点 就是12个小时的秒数 (20-8)*60
        // const HourToSecend = (this.end - this.start) * 60;
        const sday = this.rangeLength * 60;

        // 数据添加 坐标位置 X1：开始时间  X2：结束时间
        return {
          ...i,
          // (s1 / sday) * this.viewBoxWidth - this.rangeItemWidth * this.start,
          X1: (s1 / sday) * viewBoxWidth,
          X2: (s2 / sday) * viewBoxWidth
        };
      });

      // 排序
      arr = arr.sort(function(a, b) {
        return a.X1 - b.X1;
      });

      // 用户数据 添加 碰撞检测 初始化
      arr = arr.map((i, index) => {
        let pre = arr[index - 1];
        return {
          ...i,
          X2offset: pre && i.X2 - pre.X2 <= labelOffset.size ? 1 : 0,
          X1offset: pre && i.X1 - pre.X1 <= labelOffset.size ? 1 : 0
        };
      });

      let temp = {};
      // 用户数据 添加 碰撞检测
      arr.forEach((i, index) => {
        let pre = arr[index - 1];
        arr[index] = {
          ...i,
          X2offset:
            pre && i.X2 - pre.X2 <= labelOffset.size
              ? pre.X2offset >= 0
                ? pre.X2offset + 1 > labelOffset.deep
                  ? 0
                  : pre.X2offset + 1
                : 0
              : 0,
          X1offset:
            pre && i.X1 - pre.X1 <= labelOffset.size
              ? pre.X1offset >= 0
                ? pre.X1offset + 1 > labelOffset.deep
                  ? 0
                  : pre.X1offset + 1
                : 0
              : 0,
          offset: pre && i.X1 - pre.X1
        };
      });

      // 用户数据 重叠数据 合并
      // arr = arr.forEach((i, index) => {
      //   let preV2 = arr[index-1].X2
      //   let currV1 = arr[index].X1
      //   if(preV2 ===currV1){
      //
      //   }
      // });
      return arr;
    }
  },
  created() {
    this.start =
      Math.min.apply(null, this.data.map(i => i.start.split(':')[0])) - 1;
    this.end =
      Math.max.apply(null, this.data.map(i => i.end.split(':')[0])) + 2;
  },
  mounted() {
    // 动态设置svg 视图盒子
    this.viewBoxWidth = this.$refs['timeLineWrap'].scrollWidth;
    window.addEventListener(
      'resize',
      throttling(() => {
        if (this.$refs['timeLineWrap']) {
          this.viewBoxWidth = this.$refs['timeLineWrap'].scrollWidth;
          this.textWidth = this.$refs['textWidth'].scrollWidth;
        }
      }, 200),
      false
    );
  }
};
</script>

<style scoped>
.color-primary {
  color: #313233;
}

.time-text {
  font-size: 10px;
}

.coordinates {
  stroke-width: 0.5;
  stroke: #eeedf0;
  font-size: 10px;
}

.time-item-circle {
  background: #000;
}
</style>