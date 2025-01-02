<!-- AnimatedNumber.vue -->
<template>
  <span class="animated-number">
    {{ displayNumber }}
  </span>
</template>

<script>
import { ref, onMounted, onUnmounted, watch } from 'vue';

export default {
  name: 'AnimatedNumber',
  props: {
    finalNumber: {
      type: Number,
      required: true
    },
    animationDuration: {
      type: Number,
      default: 3000 // 动画持续时间（毫秒）
    },
    interval: {
      type: Number,
      default: 50 // 切换数字的时间间隔（毫秒）
    }
  },
  setup(props, { emit }) { // 获取 emit 函数
    const displayNumber = ref(0);
    let animationInterval = null;

    const startAnimation = () => {
      const startTime = Date.now();
      animationInterval = setInterval(() => {
        displayNumber.value = Math.floor(Math.random() * 300) + 1; // 假设号码范围为1到300
        if (Date.now() - startTime >= props.animationDuration) {
          clearInterval(animationInterval);
          displayNumber.value = props.finalNumber;
          emit('animationComplete', props.finalNumber); // 发出事件
        }
      }, props.interval);
    };

    onMounted(() => {
      startAnimation();
    });

    watch(
      () => props.finalNumber,
      () => {
        startAnimation();
      }
    );

    onUnmounted(() => {
      clearInterval(animationInterval);
    });

    return {
      displayNumber
    };
  }
};
</script>

<style scoped>
.animated-number {
  /* 继承父元素的颜色和背景色 */
  color: inherit;
  background-color: inherit;
  display: inline-block; /* 改为 inline-block 以确保 text-decoration 继承 */
  text-decoration: inherit; /* 确保继承父元素的 text-decoration */
  transition: transform 0.3s ease;
}

.animated-number:hover {
  transform: scale(1.2);
}
</style>
