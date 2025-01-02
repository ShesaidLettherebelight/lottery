<template>
  <div class="footer-info">
    <p>&copy; 2025璧山电信年会抽奖专用，活动解释权归云中台所有</p>
    <p>活动规则说明：请参与者遵守以下规则...</p>
  </div>
</template>

<script>
export default {
  name: 'FooterInfo',
};
</script>

<style scoped>
.footer-info {
  margin-top: 20px; /* 与轮播图之间的间距 */
  text-align: center; /* 文本居中 */
  font-size: 0.8em; /* 较小的字体大小 */
  color: #555; /* 较浅的文字颜色 */
  line-height: 1.5; /* 行高 */
  font-family: Arial, sans-serif; /* 更改字体 */
}

.footer-info p {
  margin: 5px 0; /* 段落之间的间距 */
}

/* 可选：为版权声明添加图标 */
.footer-info p:first-child::before {
  font-weight: bold;
  margin-right: 5px;
}
</style>
