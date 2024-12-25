<template>
  <div id="calculator">
    <div class="layout">
      <div class="history">
        <div v-for="(entry, index) in history" :key="index" class="history-entry">
          {{ entry }} <!-- 顯示計算歷史記錄 -->
        </div>
        <div class="stack">
            {{ stack }} <!-- 顯示計算堆疊 -->
        </div>
      </div>
      <div class="main">
        <div class="display">
          <div class="operation">{{ formattedOperation }}</div> <!-- 顯示格式化後的運算符 -->
          <div class="result">{{ formattedInput }}</div> <!-- 顯示格式化後的輸入數字 -->
          <div class="current-input">currentInput: {{ currentInput }}</div> <!-- 顯示當前輸入的數字 -->
          <div class="current-operation">currentOperation: {{ currentOperation }}</div> <!-- 顯示當前輸入的數字 -->
        </div>
        <div class="keyboard">
          <button v-for="key in keys" :key="key" @click="handleKeyPress(key)" :class="{ active: key === activeKey }">
            {{ key === '/' ? '÷' : key === '*' ? '×' : key }} <!-- 顯示鍵盤按鍵，替換顯示除號和乘號 -->
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, computed, onMounted, onUnmounted } from 'vue';

export default defineComponent({
  setup() {
    const currentInput = ref('0'); // 當前輸入的數字
    const currentOperation = ref(''); // 當前的運算符
    const activeKey = ref(''); // 當前按下的按鍵
    const resultDisplayed = ref(false); // 是否顯示結果
    const history = ref<string[]>([]); // 計算歷史記錄
    const keys = [
      '7', '8', '9', '/',
      '4', '5', '6', '*',
      '1', '2', '3', '-',
      '0', '.', 'C', '+',
      '='
    ]; // 鍵盤按鍵
    const stack = ref<string[]>([]); // 用於計算的堆疊
    const keyMap: { [key: string]: string } = {
      '7': '7', '8': '8', '9': '9', '/': '/',
      '4': '4', '5': '5', '6': '6', '*': '*',
      '1': '1', '2': '2', '3': '3', '-': '-',
      '0': '0', '.': '.', '=': '=', '+': '+',
      'C': 'C',
      'Enter': '=',
      'Delete': 'C'
    }; // 鍵盤按鍵映射

    const formattedInput = computed(() => {
      if (!currentInput.value || currentInput.value === 'Error') {
        return currentInput.value; // 如果當前輸入為空或錯誤，直接返回
      }
      const parsedValue = parseFloat(currentInput.value.replace(/,/g, '')); // 解析輸入的數字，去除逗號
      return parsedValue.toLocaleString('en', { maximumFractionDigits: 20 }); // 格式化數字，最多顯示20位小數
    }); // 格式化輸入的數字

    const formattedOperation = computed(() => {
      return currentOperation.value.replace(/\*/g, '×').replace(/\//g, '÷'); // 替換運算符顯示
    }); // 格式化運算符

    const handleKeyPress = (key: string): void => {
      activeKey.value = key; // 設置當前按下的按鍵
      setTimeout(() => (activeKey.value = ''), 200); // 200毫秒後清除按鍵狀態

      switch (key) {
        case '+':
        case '-':
        case '*':
        case '/':
          handleOperator(key); // 處理運算符按鍵
          break;
        case '=':
          handleEquals(); // 處理等號按鍵
          break;
        case 'C':
          handleClear(); // 處理清除按鍵
          break;
        default:
          handleNumber(key); // 處理數字按鍵
      }
    }; // 處理按鍵按下事件

    const handleNumber = (number: string): void => {
      if (resultDisplayed.value) {
        currentInput.value = number; // 如果結果已顯示，重置當前輸入
        resultDisplayed.value = false; // 設置結果未顯示
      } else {
        if (currentInput.value === '0' && number !== '.') {
          currentInput.value = number; // 如果當前輸入為0且輸入的不是小數點，重置當前輸入
        } else {
          currentInput.value += number; // 否則，將數字添加到當前輸入
        }
      }
    }; // 處理數字按鍵

    const handleOperator = (operator: string): void => {
      if (currentInput.value !== '') {
        stack.value.push(currentInput.value); // 將當前輸入的數字推入堆疊
        stack.value.push(operator); // 將運算符推入堆疊
        currentOperation.value += currentInput.value + operator; // 更新當前運算符
        currentInput.value = '0'; // 重置當前輸入
      }
      resultDisplayed.value = false; // 設置結果未顯示
    }; // 處理運算符按鍵

    const handleEquals = (): void => {
      if (currentInput.value !== '') {
        stack.value.push(currentInput.value); // 將當前輸入的數字推入堆疊
      }
      try {
        const result = evaluateStack(stack.value); // 計算堆疊中的表達式
        history.value.push(`${currentOperation.value}${currentInput.value} = ${result}`); // 將計算結果添加到歷史記錄
        currentInput.value = result.toString(); // 顯示計算結果
        currentOperation.value = ''; // 重置當前運算符
        stack.value = []; // 清空堆疊
        resultDisplayed.value = true; // 設置結果已顯示
      } catch (error) {
        currentInput.value = 'Error'; // 顯示錯誤
        currentOperation.value = ''; // 重置當前運算符
        stack.value = []; // 清空堆疊
        resultDisplayed.value = false; // 設置結果未顯示
      }
    }; // 處理等號按鍵

    const handleClear = (): void => {
      currentInput.value = '0'; // 重置當前輸入
      currentOperation.value = ''; // 重置當前運算符
      stack.value = []; // 清空堆疊
      resultDisplayed.value = false; // 設置結果未顯示
    }; // 處理清除按鍵

    const evaluateStack = (stack: string[]): number => {
      const resultStack: number[] = [];
      stack.forEach(token => {
        if (!isNaN(parseFloat(token))) {
          resultStack.push(parseFloat(token)); // 如果是數字，推入結果堆疊
        } else {
          const b = resultStack.pop()!; // 彈出結果堆疊中的兩個數字
          const a = resultStack.pop()!;
          switch (token) {
            case '+':
              resultStack.push(a + b); // 加法
              break;
            case '-':
              resultStack.push(a - b); // 減法
              break;
            case '*':
              resultStack.push(a * b); // 乘法
              break;
            case '/':
              resultStack.push(a / b); // 除法
              break;
          }
        }
      });
      return resultStack[0]; // 返回計算結果
    }; // 計算堆疊中的表達式

    onMounted(() => {
      window.addEventListener('keydown', handleKeyboardInput); // 組件掛載時添加鍵盤事件監聽
    });

    onUnmounted(() => {
      window.removeEventListener('keydown', handleKeyboardInput); // 組件卸載時移除鍵盤事件監聽
    });

    const handleKeyboardInput = (event: KeyboardEvent): void => {
      const key = keyMap[event.key];
      if (key) {
        handleKeyPress(key); // 處理鍵盤輸入事件
      }
    };

    return {
      currentInput,
      currentOperation,
      formattedInput,
      formattedOperation,
      keys,
      activeKey,
      handleKeyPress,
      handleOperator,
      handleEquals,
      handleClear,
      handleNumber,
      history,
      stack
    }; // 返回組件中使用的變量和方法
  }
});
</script>

<style scoped>
#calculator {
  width: 400px;
  margin: auto;
  text-align: center;
  border: 1px solid #ccc;
  border-radius: 10px;
  padding: 20px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  display: flex;
  flex-direction: row;
}

/* 計算器的樣式 */

.layout {
  display: flex;
  width: 100%;
}

/* 布局樣式 */

.history {
  width: 150px;
  max-height: 400px;
  overflow-y: auto;
  border: 1px solid #ccc;
  padding: 10px;
  background: #f9f9f9;
  border-radius: 5px;
}

/* 歷史記錄區域的樣式 */

.history-entry {
  font-size: 14px;
  color: #555;
  margin-bottom: 5px;
  text-align: left;
}

/* 歷史記錄條目的樣式 */

.stack {
  margin-top: 10px;
  font-size: 14px;
  color: #333;
}

/* 堆疊區域的樣式 */

.stack-item {
  font-size: 12px;
  color: #666;
  margin-bottom: 2px;
}

/* 堆疊條目的樣式 */

.main {
  flex: 1;
  display: flex;
  flex-direction: column;
}

/* 主區域的樣式 */

.display {
  margin-bottom: 10px;
  padding: 10px;
  font-size: 18px;
  border: 1px solid #ccc;
  border-radius: 5px;
  background: #fff;
  text-align: right;
  color: #000;
  height: 500px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

/* 顯示區域的樣式 */

.display .operation {
  font-size: 14px;
  color: #888;
}

/* 顯示區域中運算符的樣式 */

.display .result {
  font-size: 18px;
  font-weight: bold;
  color: #000;
}

/* 顯示區域中結果的樣式 */

.keyboard {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
}

/* 鍵盤區域的樣式 */

button {
  padding: 10px;
  font-size: 16px;
  border: none;
  border-radius: 5px;
  background: #007BFF;
  color: #fff;
  cursor: pointer;
  transition: background 0.2s;
}

/* 按鈕的樣式 */

button.active {
  background: #0056b3;
}

/* 按鈕被按下時的樣式 */

button:hover {
  background: #0056b3;
}

/* 按鈕懸停時的樣式 */
</style>
