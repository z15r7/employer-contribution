<template>
  <div id="calculator">
    <div class="layout">
      <div class="history">
        <div v-for="(entry, index) in history" :key="index" class="history-entry">
          {{ entry }}
        </div>
      </div>
      <div class="main">
        <div class="display">
          <div class="operation">{{ formattedOperation }}</div>
          <div class="result">{{ formattedInput }}</div>
        </div>
        <div class="keyboard">
          <button v-for="key in keys" :key="key" @click="handleKeyPress(key)" :class="{ active: key === activeKey }">
            {{ key === '/' ? '÷' : key === '*' ? '×' : key }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
// 引入 Vue 的相關功能模組
import { defineComponent, ref, computed, onMounted, onUnmounted } from 'vue';

// 定義並導出 Vue 組件
export default defineComponent({
  setup() {
    // 用於存儲當前輸入的數字，初始化為 '0'
    const currentInput = ref('0');
    // 用於存儲當前的運算式
    const currentOperation = ref('');
    // 用於記錄當前激活的按鍵
    const activeKey = ref('');
    // 用於判斷結果是否已顯示
    const resultDisplayed = ref(false);
    // 用於存儲歷史計算記錄
    const history = ref<string[]>([]);
    // 定義按鍵的數組
    const keys = [
      '7', '8', '9', '/',
      '4', '5', '6', '*',
      '1', '2', '3', '-',
      '0', '.', 'C', '+',
      '='
    ];
    // 鍵盤輸入映射表
    const keyMap: { [key: string]: string } = {
      '7': '7', '8': '8', '9': '9', '/': '/',
      '4': '4', '5': '5', '6': '6', '*': '*',
      '1': '1', '2': '2', '3': '3', '-': '-',
      '0': '0', '.': '.', '=': '=', '+': '+',
      'C': 'C',
      'Enter': '=',
      'Delete': 'C'
    };

    // 格式化輸入數值為帶千分位的形式
    const formattedInput = computed(() => {
      if (!currentInput.value || currentInput.value === 'Error') {
        return currentInput.value;
      }
      const parsedValue = parseFloat(currentInput.value.replace(/,/g, ''));
      return parsedValue.toLocaleString('en', { maximumFractionDigits: 20 });
    });

    // 格式化運算式，將運算符替換為顯示用符號
    const formattedOperation = computed(() => {
      return currentOperation.value.replace(/\*/g, '×').replace(/\//g, '÷');
    });

    // 用於存儲操作數和操作符的棧
    const stack = ref<string[]>([]);

    // 處理按鍵事件
    const handleKeyPress = (key: string): void => {
      activeKey.value = key; // 設置當前按鍵為激活狀態
      setTimeout(() => (activeKey.value = ''), 200); // 200ms後取消激活狀態

      // 根據按鍵類型執行不同操作
      switch (key) {
        case '+':
        case '-':
        case '*':
        case '/':
          handleOperator(key);
          break;
        case '=':
          handleEquals();
          break;
        case 'C':
          handleClear();
          break;
        default:
          handleNumber(key);
      }
    };

    // 處理運算符按鍵
    const handleOperator = (operator: string): void => {
      if (currentInput.value !== '') { // 如果當前有輸入數值
        if (stack.value.length === 0) { // 如果棧為空，直接存入數值
          stack.value.push(currentInput.value);
          currentOperation.value = currentInput.value + operator;
        } else { // 如果棧不為空，計算當前棧的結果
          stack.value.push(currentInput.value);
          const result = evaluateStack(stack.value);
          currentInput.value = result.toString();
          currentOperation.value += currentInput.value + operator;
          stack.value = [result.toString(), operator]; // 將結果和運算符存入棧
        }
        currentInput.value = '0'; // 重置輸入區
      } else {
        if (stack.value.length > 0) { // 如果當前無輸入，更新運算符
          stack.value[stack.value.length - 1] = operator;
          currentOperation.value = currentOperation.value.slice(0, -1) + operator;
        }
      }
      resultDisplayed.value = false; // 設置結果未顯示
    };

    // 處理等於按鍵
    const handleEquals = (): void => {
      if (currentInput.value !== '') { // 如果當前有輸入，存入棧
        stack.value.push(currentInput.value);
      }
      if (['+', '-', '*', '/'].includes(stack.value[stack.value.length - 1])) {
        stack.value.pop(); // 如果最後一個是運算符，刪除
      }
      try {
        const result = evaluateStack(stack.value); // 計算結果
        history.value.push(`${currentOperation.value}${currentInput.value} = ${result}`); // 存入歷史
        currentInput.value = result.toString(); // 更新輸入區為結果
        currentOperation.value = ''; // 清空運算區
        stack.value = []; // 清空棧
        resultDisplayed.value = true; // 設置結果已顯示
      } catch (error) {
        currentInput.value = 'Error';
        currentOperation.value = '';
        stack.value = [];
        resultDisplayed.value = false;
      }
    };

    // 處理清除按鍵
    const handleClear = (): void => {
      currentInput.value = '0';
      currentOperation.value = '';
      stack.value = [];
      resultDisplayed.value = false;
    };

    // 處理數字按鍵
    const handleNumber = (number: string): void => {
      if (resultDisplayed.value) { // 如果結果已顯示，重置輸入區
        currentInput.value = number;
        resultDisplayed.value = false;
      } else {
        if (currentInput.value === 'Error') {
          currentInput.value = ''; // 清空輸入區
          currentOperation.value = ''; 
        }
        if (currentInput.value === '0' && number !== '.') {
          currentInput.value = number; // 替換初始 0
        } else {
          currentInput.value += number; // 拼接數字
        }
      }
    };

    // 處理鍵盤輸入
    const handleKeyboardInput = (event: KeyboardEvent): void => {
      const key = keyMap[event.key];
      if (key) {
        handleKeyPress(key);
      }
    };

    // 計算棧內表達式的結果
    const evaluateStack = (stack: string[]): number => {
      const outputQueue: string[] = []; // 後綴表表達式
      const operatorStack: string[] = []; // 運算符棧
      const precedence: { [key: string]: number } = {
        '+': 1,
        '-': 1,
        '*': 2,
        '/': 2
      };

      // 將中綴表表達式轉為後綴表
      stack.forEach(token => {
        if (!isNaN(parseFloat(token))) {
          outputQueue.push(token);
        } else if (['+', '-', '*', '/'].includes(token)) {
          while (
            operatorStack.length &&
            precedence[operatorStack[operatorStack.length - 1]] >= precedence[token]
          ) {
            outputQueue.push(operatorStack.pop()!);
          }
          operatorStack.push(token);
        }
      });

      while (operatorStack.length) {
        outputQueue.push(operatorStack.pop()!);
      }

      const resultStack: number[] = []; // 用於計算後綴表結果
      outputQueue.forEach(token => {
        if (!isNaN(parseFloat(token))) {
          resultStack.push(parseFloat(token));
        } else {
          const b = resultStack.pop()!;
          const a = resultStack.pop()!;
          switch (token) {
            case '+':
              resultStack.push(a + b);
              break;
            case '-':
              resultStack.push(a - b);
              break;
            case '*':
              resultStack.push(a * b);
              break;
            case '/':
              resultStack.push(a / b);
              break;
          }
        }
      });

      return resultStack[0]; // 返回最終結果
    };

    // 註冊鍵盤事件監聽器
    onMounted(() => {
      window.addEventListener('keydown', handleKeyboardInput);
    });

    // 移除鍵盤事件監聽器
    onUnmounted(() => {
      window.removeEventListener('keydown', handleKeyboardInput);
    });

    // 返回響應式數據和方法
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
      history
    };
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

.layout {
  display: flex;
  width: 100%;
}

.history {
  width: 150px;
  max-height: 400px;
  overflow-y: auto;
  border: 1px solid #ccc;
  padding: 10px;
  background: #f9f9f9;
  border-radius: 5px;
}

.history-entry {
  font-size: 14px;
  color: #555;
  margin-bottom: 5px;
  text-align: left;
}

.main {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.display {
  margin-bottom: 10px;
  padding: 10px;
  font-size: 18px;
  border: 1px solid #ccc;
  border-radius: 5px;
  background: #fff;
  text-align: right;
  color: #000;
  height: 50px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.display .operation {
  font-size: 14px;
  color: #888;
}

.display .result {
  font-size: 18px;
  font-weight: bold;
  color: #000;
}

.keyboard {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
}

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

button.active {
  background: #0056b3;
}

button:hover {
  background: #0056b3;
}
</style>
