<template>
  <div class="app-wrapper">
    <!-- 公司 Logo -->
    <img :src="companyLogo" alt="Company Logo" class="company-logo" />
    
    <!-- 下雪动画 Canvas -->
    <canvas id="snowCanvas"></canvas>
    
    <div class="app-container">
      <!-- 左侧半透明区域 -->
      <div class="left-column">
        <!-- 自定义轮播图 -->
        <CustomCarousel :images="carouselImages" :autoplay="true" :delay="3000" />
        
        <!-- 版权声明和规则说明 -->
        <FooterInfo />
        
        <!-- 音乐控制按钮 -->
        <button class="music-toggle" @click="toggleMusic">
          <i :class="musicPlaying ? 'fas fa-volume-up' : 'fas fa-volume-mute'"></i>
        </button>
        
        <!-- 其他左侧内容（可选） -->
      </div>
      
      <!-- 右侧不透明区域 -->
      <div class="right-column">
        <h1 class="title">过年送大礼</h1>
        
        <!-- 横向卡片容器 -->
        <div class="horizontal-cards">
          <!-- 输入区域卡片 -->
          <div class="card input-card">
            <div class="input-group">
              <label for="totalParticipants">请输入参与抽奖的总人数:</label>
              <input
                type="number"
                id="totalParticipants"
                v-model.number="totalParticipants"
                min="1"
                placeholder="输入1到1000之间的数字"
              />
            </div>
             
            <div class="input-group">
              <label for="choosePrize">请选择要抽取的奖项:</label>
              <select name="choosePrize" id="choosePrize" v-model="selectedAward">
                <option 
                  v-for="([awardName], index) in awardEntries" 
                  :key="index" 
                  :value="awardName"
                  :disabled="drawnAwards.has(awardName)"
                >
                  {{ awardName }} <span v-if="drawnAwards.has(awardName)">(已抽取)</span>
                </option>
              </select>
            </div>
          </div>
      
          <!-- 按钮组卡片 -->
          <div class="card button-card">
            <div class="button-group">
              <button @click="startDrawing" :disabled="!canStartDrawing || isDrawing || !selectedAward">
                <span v-if="isDrawing">抽奖中...</span>
                <span v-else>开始抽奖</span>
              </button>
              <button @click="openResultsModal" :disabled="currentDrawnNumbers.length === 0 || isDrawing">
                查看中奖结果
              </button>
              <button @click="openHistoryModal" :disabled="history.length === 0 || isDrawing">
                历史记录
              </button>
            </div>
          </div>
        </div>
        
 <!-- 历史记录模态弹窗 -->
        <div :class="['modal-overlay', { active: isHistoryModalOpen }]" @click.self="closeHistoryModal">
          <div class="modal">
            <div class="modal-header">
              <h2>历史抽奖记录</h2>
              <button class="close-button" @click="closeHistoryModal">&times;</button>
            </div>
            <div class="modal-content">
              <div v-for="(record, index) in history" :key="index" class="record">
                <h3>{{ record.awardName }} ({{ record.date }}) - {{ record.type === 'redraw' ? '重新抽取' : '初次抽取' }}</h3>
                <div v-if="record.type === 'draw'">
                  <strong>获奖号码:</strong>
                  <div class="number-list">
                    <span
                      v-for="number in record.numbers"
                      :key="number"
                      @click="MarkAbsentNumber(number, record.awardName)"
                      :class="{ absent: absentNumbers.has(number) }"
                      class="number"
                    >
                      {{ number }}
                    </span>
                  </div>
                </div>
                <div v-else-if="record.type === 'redraw'">
                  <div>
                    <strong>缺席号码:</strong>
                    <div class="number-list">
                      <span
                        v-for="number in record.originalNumbers"
                        :key="number"
                        class="absent number"
                      >
                        {{ number }}
                      </span>
                    </div>
                  </div>
                  <div>
                    <strong>重新抽取的号码:</strong>
                    <div class="number-list">
                      <span
                        v-for="number in record.newNumbers"
                        :key="number"
                        @click="MarkAbsentNumber(number, record.awardName)"
                        :class="{ absent: absentNumbers.has(number) }"
                        class="number"
                      >
                        {{ number }}
                      </span>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div class="modal-footer">
              <button @click="closeHistoryModal">关闭</button>
            </div>
          </div>
        </div>
  
        <!-- 中奖结果模态弹窗 -->
        <div :class="['modal-overlay', { active: isResultsModalOpen }]" @click.self="closeResultsModal">
          <div class="modal">
            <div class="modal-header">
              <h2>当前抽奖结果</h2>
              <button class="close-button" @click="closeResultsModal">&times;</button>
            </div>
            <div class="modal-content">
              <div class="numbers">
                <strong>{{ selectedAward }}:</strong>
                <div class="number-list">
                  <span
                    v-for="number in currentDrawnNumbers"
                    :key="number"
                    @click="MarkAbsentNumber(number, selectedAward)"
                    :class="{ absent: absentNumbers.has(number), 'number-reveal': true }"
                    class="number"
                  >
                    <!-- 条件渲染 AnimatedNumber 或直接显示数字 -->
                    <template v-if="isDrawing">
                      <AnimatedNumber :finalNumber="number" @animationComplete="handleAnimationComplete" />
                    </template>
                    <template v-else>
                      {{ number }}
                    </template>
                  </span>
                </div>
              </div>
            </div>
            <div class="modal-footer">
              <button @click="reDrawAbsentWinners" :disabled="absentNumbers.size === 0 || isDrawing">
                重新抽取
              </button>
              <button @click="closeResultsModal">关闭</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
// 导入自定义轮播组件
import CustomCarousel from './components/CustomCarousel.vue';
import FooterInfo from './components/FooterInfo.vue'; 
// 其他导入保持不变
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue';
import { AWARDS } from './config';
import confetti from 'canvas-confetti'; // 导入 canvas-confetti
import AnimatedNumber from './components/AnimatedNumber.vue'; // 导入 AnimatedNumber 组件

// 导入音效文件
import drawSound from './assets/sounds/draw.mp3';
import revealSound from './assets/sounds/reveal.mp3';
import winSound from './assets/sounds/win.mp3';
import backgroundMusic from './assets/sounds/background.mp3'; // 导入背景音乐

// 导入 Logo
import logo from './assets/logo.png'; // 使用相对路径

// 导入轮播图图片
import carousel1 from './assets/carousel/carousel1.jpg';
import carousel2 from './assets/carousel/carousel2.jpg';
import carousel3 from './assets/carousel/carousel3.jpg';
import carousel4 from './assets/carousel/carousel4.jpg';

export default {
  name: 'App',
  components: {
    FooterInfo ,
    CustomCarousel, // 注册自定义轮播组件
    AnimatedNumber
  },
  setup() {
    // 将 logo 添加到响应式数据中
    const companyLogo = ref(logo);

    // 轮播图图片数组
    const carouselImages = ref([
      carousel1,
      carousel2,
      carousel3,
      carousel4
    ]);

    // 创建 Audio 实例
    const drawAudio = new Audio(drawSound);
    const revealAudio = new Audio(revealSound);
    const winAudio = new Audio(winSound);
    const music = new Audio(backgroundMusic); // 背景音乐
    music.volume = 0.5; // 设置音量为 50%
    music.loop = true; // 设置循环播放

    const musicPlaying = ref(false); // 追踪音乐是否正在播放

    const toggleMusic = () => {
      if (musicPlaying.value) {
        music.pause();
        musicPlaying.value = false;
      } else {
        music.play().catch((error) => {
          console.error("音乐播放失败:", error);
        });
        musicPlaying.value = true;
      }
    };

    const totalParticipants = ref(0); // 总参与人数
    const currentPool = ref([]); // 当前抽奖池
    const winners = ref(new Map()); // 所有抽奖结果
    const absentNumbers = ref(new Map()); // 缺席号码
    const isDrawing = ref(false); // 抽奖中状态
    const history = ref([]); // 抽奖历史记录
    const drawnAwards = ref(new Set()); // 已抽取的奖项

    const awardEntries = Object.entries(AWARDS); // 奖项列表

    const getTotalPrizes = () => {
      return awardEntries.reduce((acc, [_, val]) => acc + val, 0);
    };

    const canStartDrawing = computed(() => {
      return totalParticipants.value > 0 && totalParticipants.value <= 1000 && totalParticipants.value >= getTotalPrizes();
      // 限制总参与人数不超过 1000，可根据需求调整
    });

    const selectedAward = ref(null); // 用户选择的奖项

    const currentDrawnNumbers = ref([]); // 当前抽奖的号码

    // 控制历史记录模态弹窗的显示
    const isHistoryModalOpen = ref(false);

    // 控制中奖结果模态弹窗的显示
    const isResultsModalOpen = ref(false);

    // 监听 selectedAward 的变化，清空 currentDrawnNumbers
    watch(selectedAward, () => {
      currentDrawnNumbers.value = [];
    });

    // 定义揭示音效播放处理函数
    const handleAnimationComplete = (number) => {
      revealAudio.currentTime = 0;
      revealAudio.play();
    };

    const startDrawing = async () => {
      if (!selectedAward.value) {
        alert("请选择一个奖项进行抽奖。");
        return;
      }

      if (!Number.isInteger(totalParticipants.value) || totalParticipants.value < 1) {
        alert("请输入有效的参与人数（正整数）。");
        return;
      }

      // 检查是否已抽取过该奖项
      if (drawnAwards.value.has(selectedAward.value)) {
        alert(`奖项 ${selectedAward.value} 已经抽取过了。`);
        return;
      }

      isDrawing.value = true;

      // 打开结果弹窗
      openResultsModal();

      // 播放抽奖开始音效
      drawAudio.currentTime = 0;
      drawAudio.play();

      // 如果是第一次抽奖，初始化号码池
      if (currentPool.value.length === 0) {
        currentPool.value = Array.from({ length: totalParticipants.value }, (_, i) => i + 1);
      }

      // 获取奖项数量
      const award = awardEntries.find(([name, _]) => name === selectedAward.value);
      if (!award) {
        alert("无效的奖项选择。");
        isDrawing.value = false;
        return;
      }

      const [awardName, quantity] = award;

      // 检查剩余号码池是否足够抽取
      if (currentPool.value.length < quantity) {
        alert(`奖项 ${awardName} 的获奖人数超过了剩余的号码池大小。`);
        isDrawing.value = false;
        return;
      }

      // 抽取中奖号码
      const drawn = drawWinners([awardName, quantity], currentPool.value);

      const numbers = drawn.get(awardName) || [];

      // 更新 winners
      if (winners.value.has(awardName)) {
        winners.value.get(awardName).push(...numbers);
      } else {
        winners.value.set(awardName, numbers);
      }

      // 更新历史记录
      history.value.push({
        type: 'draw',
        awardName,
        numbers: [...numbers],
        date: new Date().toLocaleString(),
      });

      // 标记奖项已抽取
      drawnAwards.value.add(awardName);

      // 触发庆祝动画
      triggerConfetti();

      // 逐个显示中奖号码，带缓冲动画
      let index = 0;
      currentDrawnNumbers.value = []; // 清空当前号码

      const revealInterval = setInterval(() => {
        if (index < numbers.length) {
          currentDrawnNumbers.value.push(numbers[index]);

          index += 1;
        } else {
          clearInterval(revealInterval);

          // 播放庆祝音效
          winAudio.currentTime = 0;
          winAudio.play();

          isDrawing.value = false;
        }
      }, 500); // 每500毫秒揭示一个号码
    };

    const drawWinners = (award, pool) => {
      let poolCopy = [...pool];
      const [awardName, quantity] = award;
      const drawnWinners = new Map();

      if (poolCopy.length < quantity) {
        alert(`奖项 ${awardName} 的获奖人数超过了剩余的号码池大小。`);
        drawnWinners.set(awardName, poolCopy.splice(0, poolCopy.length));
      } else {
        const luckyNumbers = getRandomSample(poolCopy, quantity);
        drawnWinners.set(awardName, luckyNumbers);
        poolCopy = poolCopy.filter(num => !luckyNumbers.includes(num));
      }

      currentPool.value = poolCopy;
      return drawnWinners;
    };

    const getRandomSample = (array, size) => {
      const shuffled = [...array].sort(() => 0.5 - Math.random());
      return shuffled.slice(0, size);
    };

    // 切换标记未到场的号码
    const MarkAbsentNumber = (number, award) => {
      if (absentNumbers.value.has(number)) {
        absentNumbers.value.delete(number);
      } else {
        absentNumbers.value.set(number, award);
      }
    };

    // 根据标记的未到场号码重新抽取
    const reDrawAbsentWinners = async () => {
      if (isDrawing.value) return; // 防止重复点击

      if (absentNumbers.value.size === 0) {
        alert("没有缺席的号码需要重新抽取。");
        return;
      }

      if (!confirm("您确定要重新抽取缺席的号码吗？")) {
        return;
      }

      isDrawing.value = true;

      // 打开结果弹窗
      openResultsModal();

      // 播放抽奖开始音效
      drawAudio.currentTime = 0;
      drawAudio.play();

      // 创建一个新的 Map 来存储要重新抽取的奖项及缺席的数量
      const redrawMap = new Map();
      // 存储每个奖项对应的缺席号码
      const absentNumbersByAward = new Map();

      // 按奖项分组缺席号码
      absentNumbers.value.forEach((award, number) => {
        if (redrawMap.has(award)) {
          redrawMap.set(award, redrawMap.get(award) + 1);
          absentNumbersByAward.get(award).push(number);
        } else {
          redrawMap.set(award, 1);
          absentNumbersByAward.set(award, [number]);
        }
      });

      // 存储重新抽取的号码，用于更新 currentDrawnNumbers
      let newSelectedAwardNumbers = [];

      // 遍历每个需要重新抽取的奖项
      for (const [awardName, missingCount] of redrawMap.entries()) {
        const newDraw = drawWinners([awardName, missingCount], currentPool.value);
        const newNumbers = newDraw.get(awardName) || [];

        // 更新 winners
        if (winners.value.has(awardName)) {
          winners.value.get(awardName).push(...newNumbers);
        } else {
          winners.value.set(awardName, newNumbers);
        }

        // 更新历史记录
        history.value.push({
          type: 'redraw',
          awardName: awardName,
          originalNumbers: [...(absentNumbersByAward.get(awardName) || [])],
          newNumbers: [...newNumbers],
          date: new Date().toLocaleString(),
        });

        // 标记奖项已抽取
        drawnAwards.value.add(awardName);

        // 如果重新抽取的奖项是当前选中的奖项，准备更新 currentDrawnNumbers
        if (awardName === selectedAward.value) {
          newSelectedAwardNumbers = newNumbers;
        }
      }

      // 从 winners 中移除缺席的号码
      absentNumbers.value.forEach((award, number) => {
        if (winners.value.has(award)) {
          const numbers = winners.value.get(award);
          const index = numbers.indexOf(number);
          if (index !== -1) {
            numbers.splice(index, 1);
          }
        }
      });

      // 清空缺席号码
      absentNumbers.value.clear();

      // 触发庆祝动画
      triggerConfetti();

      // 播放庆祝音效
      winAudio.currentTime = 0;
      winAudio.play();

      // 如果当前选中的奖项有重新抽取的号码，更新 currentDrawnNumbers
      if (newSelectedAwardNumbers.length > 0) {
        currentDrawnNumbers.value = []; // 清空当前号码
        let index = 0;
        const revealInterval = setInterval(() => {
          if (index < newSelectedAwardNumbers.length) {
            currentDrawnNumbers.value.push(newSelectedAwardNumbers[index]);

            index += 1;
          } else {
            clearInterval(revealInterval);
            isDrawing.value = false;
          }
        }, 500); // 每500毫秒揭示一个号码
      } else {
        isDrawing.value = false;
      }
    };

    // 打开历史记录模态弹窗
    const openHistoryModal = () => {
      isHistoryModalOpen.value = true;
    };

    // 关闭历史记录模态弹窗
    const closeHistoryModal = () => {
      isHistoryModalOpen.value = false;
    };

    // 打开中奖结果模态弹窗
    const openResultsModal = () => {
      isResultsModalOpen.value = true;
    };

    // 关闭中奖结果模态弹窗
    const closeResultsModal = () => {
      isResultsModalOpen.value = false;
    };

    // 触发彩带庆祝动画
    const triggerConfetti = () => {
      confetti({
        particleCount: 100,
        spread: 70,
        origin: { y: 0.6 },
      });
    };

    // 下雪动画相关
    let animationId;
    let canvasElement;
    let ctx;
    let setCanvasSize; // 声明 setCanvasSize 以便在 onBeforeUnmount 中使用

    class Snowflake {
      constructor(width, height) {
        this.width = width;
        this.height = height;
        this.reset();
      }

      reset() {
        this.x = Math.random() * this.width;
        this.y = Math.random() * this.height;
        this.radius = Math.random() * 3 + 2;
        this.speed = Math.random() * 1 + 0.5;
        this.wind = Math.random() * 0.5 - 0.25;
      }

      update() {
        this.y += this.speed;
        this.x += this.wind;

        // 如果雪花超出底部，重新从顶部生成
        if (this.y > this.height) {
          this.reset();
          this.y = -this.radius;
        }

        // 如果雪花超出左右边界，重新从另一侧生成
        if (this.x > this.width + this.radius) {
          this.x = -this.radius;
        } else if (this.x < -this.radius) {
          this.x = this.width + this.radius;
        }
      }

      draw(ctx) {
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
        ctx.fillStyle = 'rgba(255, 255, 255, 0.8)'; /* 白色雪花 */
        ctx.fill();
      }
    }

    const setupSnowAnimation = () => {
      canvasElement = document.getElementById('snowCanvas');
      ctx = canvasElement.getContext('2d');

      // 设置 Canvas 尺寸
      setCanvasSize = () => {
        canvasElement.width = window.innerWidth;
        canvasElement.height = window.innerHeight;
      };

      setCanvasSize();
      window.addEventListener('resize', setCanvasSize);

      // 创建雪花数组
      const snowflakes = [];
      const snowflakeCount = 150; // 增加雪花数量以覆盖整个背景

      for (let i = 0; i < snowflakeCount; i++) {
        snowflakes.push(new Snowflake(canvasElement.width, canvasElement.height));
      }

      // 动画循环
      const animate = () => {
        ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);

        snowflakes.forEach(snowflake => {
          snowflake.update();
          snowflake.draw(ctx);
        });

        animationId = requestAnimationFrame(animate);
      };

      animate();
    };

    onMounted(() => {
      setupSnowAnimation();
    });

    onBeforeUnmount(() => {
      window.removeEventListener('resize', setCanvasSize);
      cancelAnimationFrame(animationId);
      music.pause(); // 停止背景音乐
    });

    return {
      FooterInfo ,
      selectedAward,
      awardEntries,
      totalParticipants,
      currentPool,
      winners,
      absentNumbers,
      history,
      drawnAwards,
      canStartDrawing,
      isDrawing,
      startDrawing,
      MarkAbsentNumber,
      reDrawAbsentWinners,
      currentDrawnNumbers,
      isHistoryModalOpen,
      openHistoryModal,
      closeHistoryModal,
      triggerConfetti,

      // 新添加的状态和方法
      isResultsModalOpen,
      openResultsModal,
      closeResultsModal,

      // 音乐相关
      musicPlaying,
      toggleMusic,

      // 新添加的事件处理函数
      handleAnimationComplete,

      // Logo 相关
      companyLogo,

      // 轮播图相关
      carouselImages
    };
  }
};
</script>

<style scoped>
/* 保持之前的样式 */

.app-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  width: 100%;
  min-height: 100vh; /* 保持最小高度为视口高度 */
  overflow: hidden; /* 防止内容溢出 */
  border-radius: 20px; /* 为整个页面添加圆角 */
  background-color: transparent; /* 确保背景透明 */
}

/* 公司 Logo 样式 */
.company-logo {
  position: absolute;
  top: 60px; /* 根据需要调整距离顶部的距离 */
  left: 50%;
  transform: translateX(-50%);
  width: 150px; /* 根据需要调整 logo 的大小 */
  height: auto; /* 保持纵横比 */
  z-index: 1002; /* 确保 logo 在最前面 */
}

/* 下雪动画 Canvas 样式 */
#snowCanvas {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none; /* 允许点击穿透 */
  z-index: -1; /* 位于所有内容后面 */
  background-color: transparent; /* 确保背景透明 */
}

/* 容器样式 */
.app-container {
  display: flex;
  flex-direction: row;
  width: 84%;
  height: 100%;
  max-width: 1200px;
  border-radius: 20px; /* 为整体容器添加圆角 */
  overflow: hidden; /* 确保子元素不超出圆角边界 */
}

/* 左侧半透明区域 */
.left-column {
  padding: 40px 20px;
  position: relative; /* 设置为相对定位，为内部绝对定位元素提供参考 */
  flex: 1;
  background: rgba(255, 243, 224, 0.8); /* 柔和的杏色背景，增加透明度以保持轻松感 */
  box-shadow: 2px 0 5px rgba(0, 0, 0, 0.1); /* 轻微阴影，增强厚度感 */
  padding: 20px; /* 内边距，增强厚度感 */
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* 右侧不透明区域 */
.right-column {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40px;
  box-sizing: border-box;
  background: var(--background-color) !important; /* 使用主背景色，并提高优先级 */
  border-left: none; /* 移除左侧边框 */
}

/* 标题样式 */
.title {
  font-family: 'Great Vibes', cursive; /* 装饰性字体 */
  font-size: 3em; /* 增大标题字体 */
  color: var(--primary-color); /* 使用主色调 */
  margin-bottom: 30px;
  text-align: center;
  border-bottom: 2px solid #FFE0B2; /* 浅黄色边框 */
  padding-bottom: 10px;
}

/* 新增横向卡片容器样式 */
.horizontal-cards {
  display: flex;
  flex-direction: row;
  gap: 25px; /* 卡片之间的间距 */
  width: 100%;
  justify-content: center; /* 将卡片居中对齐 */
  flex-wrap: wrap; /* 当屏幕较小时自动换行 */
  margin-bottom: 30px; /* 与下方内容的间距 */
}

/* 卡片通用样式 */
.card {
  background: var(--card-background); /* 使用卡片背景色 */
  border-radius: 15px;
  padding: 20px 30px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); /* 柔和的阴影 */
  border: 1px solid #FFE0B2; /* 浅黄色边框 */
  box-sizing: border-box;
}

/* 调整输入卡片和按钮卡片的宽度 */
.input-card, .button-card {
  flex: 1 1 300px; /* 允许卡片在300px以上扩展 */
  max-width: 400px; /* 最大宽度限制 */
}

/* 特定卡片样式 */
.input-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: var(--card-background); /* 使用卡片背景色 */
}

.button-card {
  display: flex;
  justify-content: center;
  background: var(--card-background); /* 使用卡片背景色 */
}

/* 输入组样式 */
.input-group {
  display: flex;
  flex-direction: column;
  margin-bottom: 20px;
  align-items: center; /* 水平居中 */
}

.input-group label {
  margin-bottom: 8px;
  font-weight: bold;
  color: var(--text-color); /* 使用文本颜色 */
  text-align: center; /* 可选：使标签文本居中 */
}


.input-group input,
.input-group select {
  padding: 12px 20px;
  border: 2px solid var(--input-border); /* 使用输入边框色 */
  border-radius: 8px;
  font-size: 1em;
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
  background: var(--input-background); /* 使用输入背景色 */
  color: var(--text-color); /* 使用文本颜色 */
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.05);
  max-width: 400px;
  width: 100%;
}

.input-group input:focus,
.input-group select:focus {
  outline: none;
  background: #FFE0B2; /* 柔和的浅黄色背景 */
  border-color: var(--primary-color); /* 使用主色调边框 */
}

/* 按钮组样式 */
.button-group {
  display: flex;
  flex-direction: row; /* 水平排列 */
  gap: 20px; /* 按钮间距 */
}

.button-group button {
  padding: 12px 25px;
  background: var(--button-background); /* 使用按钮背景色 */
  color: var(--text-color); /* 使用文本颜色 */
  border: none;
  border-radius: 30px; /* 圆角按钮 */
  font-size: 1em;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.button-group button:hover {
  background-color: #FFB74D; /* 悬停颜色 */
  transform: translateY(-3px);
  box-shadow: 0 6px 10px rgba(0, 0, 0, 0.15);
}

.button-group button:disabled {
  background: #ccc; /* 灰色表示禁用 */
  cursor: not-allowed;
}

/* 音乐控制按钮样式 */
.music-toggle {
  position: absolute;
  top: 10px; /* 调整顶部距离 */
  left: 10px; /* 调整左侧距离 */
  background: transparent;
  border: none;
  cursor: pointer;
  font-size: 1.5em;
  color: var(--primary-color); /* 使用主色调 */
  z-index: 1001; /* 确保按钮在最前面 */
}
.music-toggle:focus {
  outline: none;
}

.music-toggle i {
  transition: color 0.3s ease;
}

.music-toggle:hover i {
  color: var(--secondary-color); /* 使用珊瑚色，表示可交互 */
}

/* 历史记录模态弹窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5); /* 增加遮罩透明度 */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.3s ease, visibility 0.3s ease;
}

.modal-overlay.active {
  opacity: 1;
  visibility: visible;
}

.modal {
  background-color: var(--modal-background); /* 使用模态背景色 */
  width: 90%;
  max-width: 800px;
  border-radius: 10px;
  overflow: hidden;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2); /* 更柔和的阴影 */
  border: 1px solid #FFE0B2; /* 浅黄色边框 */
  animation: fadeIn 0.3s ease-out;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background-color: var(--modal-header-background); /* 浅黄色背景 */
  color: var(--primary-color); /* 使用主色调 */
}

.modal-header h2 {
  margin: 0;
}

.close-button {
  background: none;
  border: none;
  font-size: 1.5em;
  color: var(--primary-color); /* 使用主色调 */
  cursor: pointer;
}

.modal-content {
  padding: 20px;
  max-height: 70vh;
  overflow-y: auto;
}

.modal-footer {
  padding: 20px 25px;
  display: flex; /* 添加Flex布局 */
  justify-content: flex-end; /* 按钮靠右对齐 */
  gap: 15px; /* 设置按钮之间的间距 */
  background-color: var(--modal-footer-background); /* 柔和的浅黄色背景 */
  border-top: 1px solid #ddd; /* 添加顶部边框 */
}
.modal-footer button {
  padding: 10px 20px;
  background-color: var(--button-background); /* 使用按钮背景色 */
  color: var(--text-color); /* 使用文本颜色 */
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.modal-footer button:hover {
  background-color: var(--button-hover); /* 使用按钮悬停背景色 */
}

.record {
  border: 1px solid #FFD180; /* 浅橙色边框 */
  padding: 20px;
  border-radius: 10px; /* 增加圆角 */
  margin-bottom: 20px;
  background-color: var(--input-background); 
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05); /* 轻微阴影 */
  transition: box-shadow 0.3s ease;
}

.record:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.record h3 {
  margin-bottom: 15px;
  color: var(--text-color); /* 使用文本颜色变量，确保对比度 */
  font-size: 1.4em; /* 增大字体大小 */
  font-weight: bold; /* 加粗字体 */
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.1); /* 添加轻微的文字阴影，增加可读性 */
}

.number-list {
  display: flex;
  flex-wrap: wrap;
  gap: 10px; /* 增加数字之间的间距 */
  justify-content: center; /* 居中对齐数字 */
  margin-top: 10px;
}

.number-reveal {
  animation: revealAnimation 0.5s ease-out;
}

/* 针对弹窗内的数字进行样式覆盖 */
.modal-content .number {
  background-color: #FF8C00; /* 主色调背景 */
  color: #fff; /* 白色文字 */
  transition: background-color 0.3s ease, color 0.3s ease, text-decoration 0.3s ease;
}

.modal-content .number.absent {
  background-color: #E53935; /* 红色背景 */
  color: #fff; /* 白色文字 */
  text-decoration: line-through; /* 划线效果 */
  cursor: not-allowed; /* 禁用指针 */
  border: 2px solid #fff; /* 白色边框，增加对比度 */
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.2); /* 更明显的阴影 */
}

/* 修改 .number 样式，增加美观性 */
.number {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 15px 20px; /* 增加内边距 */
  margin: 0; /* 移除 margin，使用 gap 代替 */
  background-color: #FF8C00; /* 主色调背景 */
  border-radius: 8px; /* 圆角 */
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  width: 50px; /* 固定宽度 */
  height: 50px; /* 固定高度 */
  text-align: center;
  color: #fff; /* 白色文字 */
  font-size: 1.2em; /* 增大字体 */
}

.number:hover {
  transform: scale(1.1);
}

/* 动画效果 */
@keyframes revealAnimation {
  0% {
    opacity: 0;
    transform: scale(0.5);
  }
  100% {
    opacity: 1;
    transform: scale(1);
  }
}

/* 弹窗出现动画 */
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 确保彩带画布在最上层 */
canvas {
  position: fixed !important;
  pointer-events: none;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: -1; /* 位于所有内容后面 */
}

/* 响应式设计 */
@media (max-width: 950px) {
  .app-container {
    flex-direction: column;
    border-radius: 20px; /* 保持整体圆角 */
  }

  /* 保持左侧显示 */
  .left-column {
    display: block; /* 保持左侧显示 */
  }

  .horizontal-cards {
    justify-content: center; /* 小屏幕下居中对齐卡片 */
  }

  .input-card, .button-card {
    max-width: 100%; /* 小屏幕下卡片宽度占满容器 */
  }


  /* 调整音乐按钮的位置 */
  .music-toggle {
    top: 5px; /* 更靠近顶部 */
    left: 5px; /* 更靠近左侧 */
    font-size: 1.2em;
  }

  /* 调整公司 Logo 在小屏幕上的大小 */
  .company-logo {
    width: 100px; /* 根据需要调整 */
    top: 10px; /* 根据需要调整 */
  }

  /* 轮播图在小屏幕上的调整 */
  .carousel-container {
    max-width: 200px;
    height: 150px;
  }
}

/* 轮播图容器样式 */
.carousel-container {
  width: 100%;
  max-width: 300px; /* 设置轮播图的最大宽度 */
  height: 200px; /* 设置轮播图的高度 */
  margin: 0 auto 20px auto; /* 居中并添加下边距 */
  border-radius: 10px;
  overflow: hidden;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

/* 轮播图图片样式 */
.carousel-image {
  width: 100%;
  height: 100%;
  object-fit: cover; /* 保持图片比例并覆盖容器 */
}

/* 调整 Swiper 分页器和导航按钮的样式（可选） */
.swiper-pagination {
  bottom: 10px !important;
}

.swiper-button-prev,
.swiper-button-next {
  color: #FF8C00; /* 主色调 */
}
</style>
