<template>
  <div class="app-wrapper">
    <!-- 下雪动画 Canvas -->
    <canvas id="snowCanvas"></canvas>
    
    <div class="container">
      <h1 class="title">年会抽奖系统</h1>
      
      <!-- 输入区域卡片 -->
      <div class="card input-card">
        <!-- 输入总参与人数 -->
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
         
        <!-- 选择奖项 -->
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
          <button @click="reDrawAbsentWinners" :disabled="absentNumbers.size === 0 || isDrawing">
            重新抽取
          </button>
          <button @click="openHistoryModal" :disabled="history.length === 0 || isDrawing">
            历史记录
          </button>
        </div>
      </div>
      
      <!-- 抽奖结果卡片 -->
      <div v-if="currentDrawnNumbers.length > 0" class="card results-card">
        <h2>当前抽奖结果:</h2>
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
              <AnimatedNumber :finalNumber="number" />
            </span>
          </div>
        </div>
      </div>
      
      <!-- 历史记录模态弹窗 -->
      <div v-if="isHistoryModalOpen" class="modal-overlay" @click.self="closeHistoryModal">
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
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue';
import { AWARDS } from './config';
import confetti from 'canvas-confetti'; // 导入 canvas-confetti
import AnimatedNumber from './components/AnimatedNumber.vue'; // 导入 AnimatedNumber 组件

// 导入音效文件
import drawSound from '../src/assets/sounds/draw.mp3';
import revealSound from '../src/assets/sounds/reveal.mp3';
import winSound from '../src/assets/sounds/win.mp3';

export default {
  name: 'App',
  components: {
    AnimatedNumber
  },
  setup() {
    // 创建 Audio 实例
    const drawAudio = new Audio(drawSound);
    const revealAudio = new Audio(revealSound);
    const winAudio = new Audio(winSound);

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

    // 监听 selectedAward 的变化，清空 currentDrawnNumbers
    watch(selectedAward, () => {
      currentDrawnNumbers.value = [];
    });

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

          // 播放揭示音效
          revealAudio.currentTime = 0;
          revealAudio.play();

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

      isDrawing.value = true;

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

            // 播放揭示音效
            revealAudio.currentTime = 0;
            revealAudio.play();

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
    let canvas;
    let ctx;

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
        ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
        ctx.fill();
      }
    }

    const setupSnowAnimation = () => {
      canvas = document.getElementById('snowCanvas');
      ctx = canvas.getContext('2d');

      // 设置 Canvas 尺寸
      const setCanvasSize = () => {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
      };

      setCanvasSize();
      window.addEventListener('resize', setCanvasSize);

      // 创建雪花数组
      const snowflakes = [];
      const snowflakeCount = 150; // 增加雪花数量以覆盖整个背景

      for (let i = 0; i < snowflakeCount; i++) {
        snowflakes.push(new Snowflake(canvas.width, canvas.height));
      }

      // 动画循环
      const animate = () => {
        ctx.clearRect(0, 0, canvas.width, canvas.height);

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
      window.removeEventListener('resize', () => {});
      cancelAnimationFrame(animationId);
    });

    return {
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
      triggerConfetti, // 公开方法以备其他组件调用
    };
  }
};
</script>

<style scoped>
/* 包裹整个应用的外层 */
.app-wrapper {
  position: relative;
  width: 100%;
  height: 100vh; /* 确保覆盖整个视口高度 */
  overflow: hidden; /* 防止内容溢出 */
  background: linear-gradient(135deg, #71b7e6, #9b59b6); /* 增加渐变背景 */
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
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 40px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* 标题样式 */
.title {
  font-family: 'Roboto', sans-serif;
  font-size: 2.5em;
  color: #fff;
  margin-bottom: 30px;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
}

/* 卡片通用样式 */
.card {
  background: #ffffff; /* 纯白背景 */
  border-radius: 15px;
  padding: 20px 30px;
  margin-bottom: 30px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); /* 更柔和的阴影 */
  border: 1px solid #e0e0e0; /* 轻微边框 */
  width: 100%;
  box-sizing: border-box;
}

/* 特定卡片样式 */
.input-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #f9f9f9; /* 更浅的背景色 */
}

.button-card {
  display: flex;
  justify-content: center;
  background: #f9f9f9; /* 更浅的背景色 */
}

.results-card {
  color: #333; /* 深色文本，增强可读性 */
  background: #ffffff; /* 纯白背景 */
  border: 1px solid #e0e0e0; /* 轻微边框 */
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
  color: #333; /* 根据背景调整颜色 */
  text-align: center; /* 可选：使标签文本居中 */
}

.input-group input,
.input-group select {
  padding: 12px 20px;
  border: 2px solid #bdc3c7;
  border-radius: 8px;
  font-size: 1em;
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
  background: #ffffff; /* 纯白背景 */
  color: #333;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.05);
  max-width: 400px;
  width: 100%;
}

.input-group input:focus,
.input-group select:focus {
  outline: none;
  background: #f0f8ff; /* 聚焦时轻微背景变化 */
  border-color: #3498db;
}

/* 按钮组样式 */
.button-group {
  display: flex;
  flex-direction: row; /* 水平排列 */
  gap: 20px; /* 按钮间距 */
}

.button-group button {
  padding: 12px 25px;
  background: #3498db; /* 纯色背景 */
  color: #fff;
  border: none;
  border-radius: 30px; /* 圆角按钮 */
  font-size: 1em;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.button-group button:hover {
  background-color: #2980b9; /* 深色调 */
  transform: translateY(-3px);
  box-shadow: 0 6px 10px rgba(0, 0, 0, 0.15);
}

.button-group button:disabled {
  background: #bdc3c7; /* 灰色表示禁用 */
  cursor: not-allowed;
}

/* 抽奖结果样式 */
.results {
  color: #fff;
}

.results h2 {
  margin-bottom: 15px;
  color: #2ecc71;
}

.numbers {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.number-list {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  margin-top: 10px;
}

.number {
  display: inline-block;
  padding: 12px 18px;
  margin: 8px;
  background-color: #2ecc71; /* 更鲜明的绿色 */
  border-radius: 50%;
  cursor: pointer;
  transition: background-color 0.3s ease, transform 0.2s ease;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  min-width: 50px;
  text-align: center;
  color: #fff;
}

.number:hover {
  background-color: #27ae60;
  transform: scale(1.1);
}

.absent {
  text-decoration: line-through;
  background-color: #e74c3c !important; /* 更鲜明的红色 */
  color: #fff;
  cursor: not-allowed;
}

/* 历史记录模态弹窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.3); /* 调整遮罩透明度 */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal {
  background-color: #ffffff; /* 纯白背景 */
  width: 90%;
  max-width: 800px;
  border-radius: 10px;
  overflow: hidden;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2); /* 更柔和的阴影 */
  border: 1px solid #e0e0e0; /* 轻微边框 */
  animation: fadeIn 0.3s ease-out;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  background-color: #3498db; /* 使用更明亮的颜色 */
  color: #fff;
}

.modal-header h2 {
  margin: 0;
}

.close-button {
  background: none;
  border: none;
  font-size: 1.5em;
  color: #fff;
  cursor: pointer;
}

.modal-content {
  padding: 20px;
  max-height: 70vh;
  overflow-y: auto;
}

.modal-footer {
  padding: 15px 20px;
  text-align: right;
  background-color: #f1f1f1; /* 更浅的背景色 */
}

.modal-footer button {
  padding: 10px 20px;
  background-color: #e74c3c; /* 更鲜明的按钮背景 */
  color: #fff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.modal-footer button:hover {
  background-color: #c0392b;
}

.record {
  border: 1px solid #ddd;
  padding: 15px;
  border-radius: 8px;
  margin-bottom: 15px;
  background-color: #fafafa;
  transition: box-shadow 0.3s ease;
}

.record:hover {
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.record h3 {
  margin-bottom: 10px;
  color: #2c3e50;
}

.number-list {
  display: flex;
  flex-wrap: wrap;
  margin-top: 5px;
}

.number-reveal {
  animation: revealAnimation 0.5s ease-out;
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
  .container {
    width: 90%;
    margin: 20px auto;
    padding: 20px;
  }

  .button-group {
    flex-direction: column; /* 垂直排列按钮 */
    gap: 10px;
  }

  .button-group button {
    width: 100%; /* 在小屏幕上使用全宽按钮 */
  }
}
</style>
