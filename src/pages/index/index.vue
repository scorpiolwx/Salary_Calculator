<template>
  <view class="content">
    <view class="header">
      <text class="title">窝囊费计算器</text>
    </view>
    <view class="form-container" :class="{ 'form-container-active': isActive }">
      <view class="form-item" v-for="(item, index) in formItems" :key="index" :class="{ 'form-item-animate': isActive }"
            :style="{ animationDelay: `${index * 0.1}s` }">
        <text class="label">{{ item.label }}：</text>
        <template v-if="item.key === 'selectedMonth'">
          <picker mode="date" fields="month" :value="formData[item.key]" @change="handleDateChange" class="input picker-input">
            <view class="picker-text">
              {{ formData[item.key] || item.placeholder }}
            </view>
          </picker>
        </template>
        <input
            v-else
            type="digit"
            v-model.number="formData[item.key]"
            class="input"
            :placeholder="item.placeholder"
            :data-key="item.key"
            @focus="handleInputFocus"
            @blur="handleInputBlur"
        />
        <view class="input-icon" :class="{ 'input-icon-active': activeInput === item.key }"></view>
      </view>
    </view>

    <!-- 实发工资展示区域 -->
    <view class="salary-display" :class="{ 'salary-animation': salaryAnimation }">
      <text class="salary-display-title">窝囊费：</text>
      <text class="salary-display-amount" :class="{ 'salary-amount-animation': salaryAnimation }">¥{{ formatNumber(estimatedSalary) }}</text>
    </view>

    <!-- 艺术字效果 -->
    <view class="art-text-container" v-if="showArtText">
      <view class="art-text" :class="{ 'art-text-animation': showArtText }">
        <text class="art-text-char">真</text>
        <text class="art-text-char">姬</text>
        <text class="art-text-char">八</text>
        <text class="art-text-char">穷</text>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      isActive: false,
      activeInput: '',
      estimatedSalary: 0,
      hasEnoughData: false,
      salaryAnimation: false,
      showArtText: false,
      // 时间API，不需要注册即可使用
      holidayApi: 'http://timor.tech/api/holiday/year',
      formData: {
        baseSalary: 0,
        attendanceDays: 0,
        actualAttendance: 0,
        statutoryHolidaySalary: 0,
        statutoryHolidayDays: 0,
        insuranceDeduction: 662.54,
        otherDeduction: 0,
        selectedMonth: ''
      },
      formItems: [
        {label: '选择月份', key: 'selectedMonth', placeholder: '请选择月份'},
        {label: '工资基数', key: 'baseSalary', placeholder: '请输入工资基数'},
        {label: '应出勤天数', key: 'attendanceDays', placeholder: '请输入应出勤天数'},
        {label: '出勤天数', key: 'actualAttendance', placeholder: '请输入出勤天数'},
        {label: '法定假工资', key: 'statutoryHolidaySalary', placeholder: '请输入法定假工资'},
        {label: '法定假天数', key: 'statutoryHolidayDays', placeholder: '请输入法定假天数'},
        {label: '扣除保险', key: 'insuranceDeduction', placeholder: '请输入扣除保险'},
        {label: '其他扣除', key: 'otherDeduction', placeholder: '请输入其他扣除'}
      ]
    }
  },
  onLoad() {
    // 设置默认月份为当前日期的上一个月
    const now = new Date();
    let year = now.getFullYear();
    let month = now.getMonth(); // 0-11

    if (month === 0) {
      year--;
      month = 11;
    } else {
      month--;
    }

    const monthStr = (month + 1).toString().padStart(2, '0');
    this.formData.selectedMonth = `${year}-${monthStr}`;

    // 调用接口获取默认月份的法定假天数
    this.getStatutoryHolidayDays(this.formData.selectedMonth);

    // 页面加载时添加动画效果
    setTimeout(() => {
      this.isActive = true;
    }, 100);
  },
  methods: {
    handleInputFocus(e) {
      this.activeInput = e.target.dataset.key;
    },
    handleInputBlur() {
      this.activeInput = '';
      // 计算实发工资
      this.calculateEstimatedSalary();
    },
    calculateEstimatedSalary() {
      const {
        baseSalary,
        attendanceDays,
        actualAttendance,
        statutoryHolidaySalary,
        statutoryHolidayDays,
        insuranceDeduction,
        otherDeduction
      } = this.formData;

      if (!statutoryHolidayDays) statutoryHolidayDays = 0;

      // 检查是否有足够的数据来计算
      if (baseSalary > 0 && actualAttendance > 0) {
        // 计算月考勤天数 = 应出勤天数 + 法定假天数
        const attendanceDaysTotal = attendanceDays + statutoryHolidayDays;

        // 计算实发工资
        // 工资/月考勤天数*实际出勤天数+4000/月考勤天数*月法定假天数-扣除保险-其他扣除
        const estimatedSalary = (baseSalary / attendanceDaysTotal) * actualAttendance + (statutoryHolidaySalary / attendanceDaysTotal) * statutoryHolidayDays - insuranceDeduction - otherDeduction;

        this.estimatedSalary = Math.max(0, estimatedSalary);
        this.hasEnoughData = true;
        
        // 触发工资动画
        this.triggerSalaryAnimation();
      } else {
        this.hasEnoughData = false;
      }
    },
    
    // 触发工资动画
    triggerSalaryAnimation() {
      this.salaryAnimation = true;
      setTimeout(() => {
        this.salaryAnimation = false;
      }, 1000);
      
      // 显示艺术字
      this.showArtText = true;
      setTimeout(() => {
        this.showArtText = false;
      }, 1000);
    },

    handleDateChange(e) {
      const selectedDate = e.detail.value;
      this.formData.selectedMonth = selectedDate;
      // 调用接口获取法定假天数
      this.getStatutoryHolidayDays(selectedDate);
    },
    // 计算指定月份的周六日天数
    getWeekendDays(year, month) {
      let days = 0;
      const lastDay = new Date(year, month, 0).getDate();

      for (let day = 1; day <= lastDay; day++) {
        const date = new Date(year, month - 1, day);
        const dayOfWeek = date.getDay();
        if (dayOfWeek === 0 || dayOfWeek === 6) {
          days++;
        }
      }

      return days;
    },

    getStatutoryHolidayDays(month) {
      // 从月份中提取年份和月份
      const [year, monthNum] = month.split('-').map(Number);
      const monthStr = monthNum.toString().padStart(2, '0');

      // 调用时间API获取法定节假日
      uni.request({
        url: `${this.holidayApi}?year=${year}`,
        method: 'GET',
        success: (res) => {
          if (res.data.code === 0) {
            // 计算该月份的法定假天数（扣除周六日和补班）
            let statutoryDays = 0;
            let makeUpWorkDays = 0;
            const holidays = res.data.holiday;

            // 遍历所有节假日，统计该月份的法定假天数和补班天数
            for (const dateKey in holidays) {
              if (dateKey.startsWith(monthStr)) {
                const holiday = holidays[dateKey];
                if (holiday.holiday) {
                  // 检查是否是周六日
                  const [, day] = dateKey.split('-').map(Number);
                  const date = new Date(year, monthNum - 1, day);
                  const dayOfWeek = date.getDay();

                  // 不是周六日，计入法定假天数
                  if (dayOfWeek !== 0 && dayOfWeek !== 6) {
                    statutoryDays++;
                  }
                } else if (holiday.name && holiday.name.includes('补班')) {
                  // 统计补班天数
                  makeUpWorkDays++;
                }
              }
            }

            // 扣除补班天数
            const finalStatutoryDays = Math.max(0, statutoryDays - makeUpWorkDays);
            this.formData.statutoryHolidayDays = finalStatutoryDays;
            // 计算实发工资
            this.calculateEstimatedSalary();
          } else {
            console.error('获取法定节假日失败:', res.data);
            // 失败时使用默认值
            this.formData.statutoryHolidayDays = 0;
            // 计算实发工资
            this.calculateEstimatedSalary();
          }
        },
        fail: (err) => {
          console.error('请求法定节假日API失败:', err);
          // 失败时使用默认值
          this.formData.statutoryHolidayDays = 0;
          // 计算实发工资
          this.calculateEstimatedSalary();
        }
      });
    },
    // 格式化数字，添加千分符
    formatNumber(num) {
      return num.toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
    }
  }
}
</script>

<style>
/* 全局样式 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* 渐变背景 */
.content {
  display: flex;
  flex-direction: column;
  padding: 30rpx;
  min-height: 100vh;
  background: linear-gradient(-45deg, #ee7752, #e73c7e, #23a6d5, #23d5ab);
  background-size: 400% 400%;
  position: relative;
  overflow: hidden;
  animation: gradientBG 15s ease infinite;
}

/* 背景装饰 */
.content::before {
  content: '';
  position: absolute;
  top: -50%;
  left: -50%;
  width: 200%;
  height: 200%;
  background: radial-gradient(circle, rgba(255, 255, 255, 0.1) 0%, rgba(255, 255, 255, 0) 70%);
  animation: float 15s ease-in-out infinite;
  z-index: 0;
}

/* 背景装饰2 */
.content::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 300%;
  height: 300%;
  background: radial-gradient(circle, rgba(255, 255, 255, 0.05) 0%, rgba(255, 255, 255, 0) 50%);
  transform: translate(-50%, -50%);
  animation: floatReverse 20s ease-in-out infinite;
  z-index: 0;
}

@keyframes gradientBG {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

@keyframes float {
  0%, 100% {
    transform: translate(0, 0) rotate(0deg);
  }
  50% {
    transform: translate(15%, 15%) rotate(180deg);
  }
}

@keyframes floatReverse {
  0%, 100% {
    transform: translate(-50%, -50%) rotate(0deg);
  }
  50% {
    transform: translate(-40%, -40%) rotate(-180deg);
  }
}

/* 头部样式 */
.header {
  text-align: center;
  margin-bottom: 40rpx;
  padding: 30rpx;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 16rpx;
  box-shadow: 0 8rpx 32rpx rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(10rpx);
  z-index: 1;
  transform: translateY(-20rpx);
  opacity: 0;
  animation: slideUp 0.8s ease-out forwards;
}

.title {
  font-size: 48rpx;
  font-weight: bold;
  color: #333;
  margin-bottom: 10rpx;
  background: linear-gradient(90deg, #667eea, #764ba2, #ee7752, #23d5ab);
  background-size: 300% 300%;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  animation: gradientText 8s ease infinite;
  position: relative;
  display: inline-block;
}

@keyframes gradientText {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

.title::after {
  content: '';
  position: absolute;
  bottom: -5rpx;
  left: 0;
  width: 0;
  height: 3rpx;
  background: linear-gradient(90deg, #667eea, #764ba2);
  transition: width 0.8s ease;
}

.header:hover .title::after {
  width: 100%;
}

.subtitle {
  font-size: 24rpx;
  color: #666;
  opacity: 0.8;
}

/* 表单容器 */
.form-container {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20rpx;
  padding: 30rpx;
  box-shadow: 0 12rpx 40rpx rgba(0, 0, 0, 0.15);
  backdrop-filter: blur(15rpx);
  z-index: 1;
  transform: scale(0.95);
  opacity: 0;
  transition: all 0.6s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.form-container-active {
  transform: scale(1);
  opacity: 1;
}

/* 表单项 */
.form-item {
  display: flex;
  align-items: center;
  margin-bottom: 25rpx;
  padding: 20rpx;
  border-radius: 12rpx;
  background: rgba(245, 245, 245, 0.7);
  border: 2rpx solid transparent;
  transition: all 0.3s ease;
  transform: translateX(-20rpx);
  opacity: 0;
}

.form-item-animate {
  transform: translateX(0);
  opacity: 1;
  animation: slideIn 0.5s ease-out forwards;
}

.form-item:hover {
  border-color: #667eea;
  box-shadow: 0 4rpx 16rpx rgba(102, 126, 234, 0.2);
  background: rgba(255, 255, 255, 0.8);
}

@keyframes slideIn {
  from {
    transform: translateX(-20rpx);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

@keyframes slideUp {
  from {
    transform: translateY(-20rpx);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

/* 标签样式 */
.label {
  width: 220rpx;
  font-size: 28rpx;
  font-weight: 600;
  color: #333;
  margin-right: 20rpx;
}

/* 输入框样式 */
.input {
  flex: 1;
  height: 70rpx;
  padding: 0 20rpx;
  border: 2rpx solid #e0e0e0;
  border-radius: 10rpx;
  font-size: 28rpx;
  color: #333;
  background: rgba(255, 255, 255, 0.9);
  transition: all 0.3s ease;
  position: relative;
  z-index: 1;
  display: flex;
  align-items: center;
}

.picker-text {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  color: #333;
}

.picker-text:empty::before {
  content: attr(data-placeholder);
  color: #999;
}

.input:focus {
  outline: none;
  border-color: #667eea;
  box-shadow: 0 0 0 4rpx rgba(102, 126, 234, 0.1);
  transform: translateY(-2rpx);
}

/* 输入框图标 */
.input-icon {
  width: 20rpx;
  height: 20rpx;
  margin-left: 15rpx;
  border-radius: 50%;
  background: #e0e0e0;
  transition: all 0.3s ease;
}

.input-icon-active {
  background: #667eea;
  box-shadow: 0 0 10rpx rgba(102, 126, 234, 0.6);
  animation: pulse 1.5s ease-in-out infinite;
}

/* 实发工资展示区域 */
.salary-display {
  margin-top: 30rpx;
  padding: 25rpx;
  background: linear-gradient(135deg, #667eea, #764ba2);
  border-radius: 16rpx;
  box-shadow: 0 8rpx 32rpx rgba(102, 126, 234, 0.3);
  text-align: center;
}

.salary-display-title {
  font-size: 28rpx;
  color: rgba(255, 255, 255, 0.9);
  margin-bottom: 10rpx;
}

.salary-display-amount {
  font-size: 48rpx;
  font-weight: bold;
  color: white;
  text-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.2);
  margin-left: 20px;
  transition: all 0.3s ease;
}

/* 工资动画效果 */
.salary-animation {
  animation: salaryPulse 1s ease-in-out forwards;
}

.salary-amount-animation {
  animation: salaryScale 1s ease-in-out forwards, salaryGlow 1s ease-in-out forwards;
}

@keyframes salaryPulse {
  0% {
    transform: scale(1);
    box-shadow: 0 8rpx 32rpx rgba(102, 126, 234, 0.3);
  }
  50% {
    transform: scale(1.05);
    box-shadow: 0 12rpx 48rpx rgba(102, 126, 234, 0.6), 0 0 30rpx rgba(255, 255, 255, 0.8);
  }
  100% {
    transform: scale(1);
    box-shadow: 0 8rpx 32rpx rgba(102, 126, 234, 0.3);
  }
}

@keyframes salaryScale {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  25% {
    transform: scale(1.2) rotate(5deg);
    opacity: 1;
  }
  50% {
    transform: scale(1.1) rotate(-3deg);
    opacity: 1;
  }
  75% {
    transform: scale(1.15) rotate(2deg);
    opacity: 1;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

@keyframes salaryGlow {
  0% {
    text-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.2);
    color: white;
  }
  50% {
    text-shadow: 0 0 20rpx rgba(255, 255, 255, 1), 0 0 40rpx rgba(102, 126, 234, 0.8), 0 0 60rpx rgba(118, 75, 162, 0.6);
    color: #fff;
  }
  100% {
    text-shadow: 0 2rpx 8rpx rgba(0, 0, 0, 0.2);
    color: white;
  }
}

/* 艺术字效果 */
.art-text-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  pointer-events: none;
}

.art-text {
  display: flex;
  flex-wrap: wrap;
  width: 80%;
  height: 80%;
  align-items: center;
  justify-content: center;
}

.art-text-char {
  flex: 1 1 45%;
  font-size: 120rpx;
  font-weight: bold;
  text-align: center;
  color: #ff4757;
  text-shadow: 0 0 20rpx rgba(255, 71, 87, 0.8), 0 0 40rpx rgba(255, 71, 87, 0.6);
  transform-origin: center;
  opacity: 0;
}

.art-text-animation {
  animation: artTextExplosion 1s ease-in-out forwards;
}

.art-text-animation .art-text-char {
  animation: artCharExplosion 1s ease-in-out forwards;
}

.art-text-animation .art-text-char:nth-child(1) {
  animation-delay: 0s;
}

.art-text-animation .art-text-char:nth-child(2) {
  animation-delay: 0.1s;
}

.art-text-animation .art-text-char:nth-child(3) {
  animation-delay: 0.2s;
}

.art-text-animation .art-text-char:nth-child(4) {
  animation-delay: 0.3s;
}

@keyframes artTextExplosion {
  0% {
    transform: scale(0);
    opacity: 0;
  }
  30% {
    transform: scale(1.2);
    opacity: 1;
  }
  70% {
    transform: scale(1);
    opacity: 1;
  }
  100% {
    transform: scale(1.5);
    opacity: 0;
  }
}

@keyframes artCharExplosion {
  0% {
    transform: scale(0) rotate(0deg);
    opacity: 0;
  }
  30% {
    transform: scale(1.5) rotate(15deg);
    opacity: 1;
  }
  50% {
    transform: scale(1.2) rotate(-10deg);
    opacity: 1;
  }
  70% {
    transform: scale(1) rotate(5deg);
    opacity: 1;
  }
  100% {
    transform: scale(0.8) rotate(0deg);
    opacity: 0;
  }
}

@keyframes pulse {
  0%, 100% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.2);
    opacity: 0.7;
  }
}

/* 浮动按钮 */
.floating-buttons {
  display: flex;
  justify-content: space-between;
  margin-top: 40rpx;
  z-index: 1;
}

.floating-btn {
  flex: 1;
  height: 80rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 40rpx;
  margin: 0 10rpx;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 6rpx 20rpx rgba(0, 0, 0, 0.15);
  position: relative;
  overflow: hidden;
  transform: translateY(0);
}

.floating-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.2), transparent);
  transition: left 0.6s ease;
}

.floating-btn:hover::before {
  left: 100%;
}

.floating-btn:first-child {
  background: linear-gradient(135deg, #667eea, #764ba2);
  animation: pulseBtn 2s ease-in-out infinite;
  position: relative;
}

.floating-btn:first-child::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(135deg, #764ba2, #667eea);
  border-radius: 40rpx;
  z-index: -1;
  animation: pulseShadow 2s ease-in-out infinite;
}

.floating-btn.secondary {
  background: rgba(255, 255, 255, 0.9);
  border: 2rpx solid #e0e0e0;
  transition: all 0.4s ease;
}

.floating-btn:hover {
  transform: translateY(-6rpx) scale(1.02);
  box-shadow: 0 12rpx 35rpx rgba(0, 0, 0, 0.25);
}

.floating-btn:first-child:hover {
  animation: none;
  transform: translateY(-8rpx) scale(1.05);
}

@keyframes pulseBtn {
  0%, 100% {
    transform: translateY(0) scale(1);
  }
  50% {
    transform: translateY(-3rpx) scale(1.02);
  }
}

@keyframes pulseShadow {
  0%, 100% {
    transform: scale(1);
    opacity: 0.7;
  }
  50% {
    transform: scale(1.05);
    opacity: 0.4;
  }
}

.btn-text {
  font-size: 32rpx;
  color: white;
  z-index: 1;
}

.btn-text.secondary {
  color: #333;
}

/* 结果容器 */
.result-container {
  margin-top: 40rpx;
  padding: 30rpx;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20rpx;
  box-shadow: 0 12rpx 40rpx rgba(0, 0, 0, 0.15);
  backdrop-filter: blur(15rpx);
  text-align: center;
  z-index: 1;
  transform: translateY(20rpx);
  opacity: 0;
  transition: all 0.6s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.result-container-animate {
  transform: translateY(0);
  opacity: 1;
  animation: bounceIn 0.6s ease-out forwards;
}

@keyframes bounceIn {
  0% {
    transform: scale(0.8);
    opacity: 0;
  }
  50% {
    transform: scale(1.05);
    opacity: 1;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}

.result-title {
  font-size: 32rpx;
  color: #666;
  margin-bottom: 10rpx;
}

.result-amount {
  font-size: 56rpx;
  font-weight: bold;
  color: #667eea;
  margin-bottom: 10rpx;
  background: linear-gradient(90deg, #667eea, #764ba2);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.result-subtitle {
  font-size: 24rpx;
  color: #999;
}
</style>
