<template>
  <div class="carousel">
    <!-- 轮播图容器 -->
    <div class="carousel-inner" :style="{ transform: `translateX(-${currentIndex * 100}%)` }">
      <div class="carousel-item" v-for="(image, index) in images" :key="index">
        <img :src="image" :alt="'Carousel Image ' + (index + 1)" />
      </div>
    </div>
    
    <!-- 前进和后退按钮 -->
    <button class="carousel-control prev" @click="prevSlide">
      &#10094;
    </button>
    <button class="carousel-control next" @click="nextSlide">
      &#10095;
    </button>
    
    <!-- 分页指示器 -->
    <div class="carousel-indicators">
      <span 
        v-for="(image, index) in images" 
        :key="index" 
        :class="['indicator', { active: index === currentIndex }]" 
        @click="goToSlide(index)"
      ></span>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';

export default {
  name: 'CustomCarousel',
  props: {
    images: {
      type: Array,
      required: true
    },
    autoplay: {
      type: Boolean,
      default: true
    },
    delay: {
      type: Number,
      default: 3000 // 3秒
    }
  },
  setup(props) {
    const currentIndex = ref(0);
    let autoplayInterval = null;

    const nextSlide = () => {
      currentIndex.value = (currentIndex.value + 1) % props.images.length;
    };

    const prevSlide = () => {
      currentIndex.value = (currentIndex.value - 1 + props.images.length) % props.images.length;
    };

    const goToSlide = (index) => {
      currentIndex.value = index;
    };

    const startAutoplay = () => {
      if (props.autoplay) {
        autoplayInterval = setInterval(() => {
          nextSlide();
        }, props.delay);
      }
    };

    const stopAutoplay = () => {
      if (autoplayInterval) {
        clearInterval(autoplayInterval);
        autoplayInterval = null;
      }
    };

    onMounted(() => {
      startAutoplay();
    });

    onBeforeUnmount(() => {
      stopAutoplay();
    });

    // 当 currentIndex 变化时，重启自动播放
    watch(currentIndex, () => {
      stopAutoplay();
      startAutoplay();
    });

    return {
      currentIndex,
      nextSlide,
      prevSlide,
      goToSlide
    };
  }
};
</script>

<style scoped>
.carousel {
  position: relative;
  width: 300px; /* 调整为竖向矩形的宽度 */
  height: 500px; /* 调整为竖向矩形的高度 */
  overflow: hidden;
  border-radius: 10px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.carousel-inner {
  display: flex;
  transition: transform 0.5s ease-in-out;
  height: 100%;
}

.carousel-item {
  min-width: 100%;
  height: 100%;
  flex-shrink: 0;
}

.carousel-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.carousel-control {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  background-color: rgba(255, 255, 255, 0.7);
  border: none;
  padding: 10px;
  border-radius: 50%;
  cursor: pointer;
  font-size: 1.5em;
  color: #FF8C00; /* 主色调 */
  transition: background-color 0.3s ease;
}

.carousel-control:hover {
  background-color: rgba(255, 255, 255, 1);
}

.carousel-control.prev {
  left: 10px;
}

.carousel-control.next {
  right: 10px;
}

.carousel-indicators {
  position: absolute;
  bottom: 10px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 8px;
}

.indicator {
  width: 12px;
  height: 12px;
  background-color: rgba(255, 255, 255, 0.7);
  border-radius: 50%;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.indicator.active {
  background-color: #FF8C00; /* 主色调 */
}

.carousel-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 10px; /* 添加圆角 */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2); /* 添加阴影 */
}

</style>