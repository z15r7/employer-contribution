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
import { defineComponent, ref, computed, onMounted, onUnmounted } from 'vue';

export default defineComponent({
  setup() {
    const currentInput = ref('0');
    const currentOperation = ref('');
    const activeKey = ref('');
    const resultDisplayed = ref(false);
    const history = ref<string[]>([]);
    const keys = [
      '7', '8', '9', '/',
      '4', '5', '6', '*',
      '1', '2', '3', '-',
      '0', '.', 'C', '+',
      '='
    ];
    const keyMap: { [key: string]: string } = {
      '7': '7', '8': '8', '9': '9', '/': '/',
      '4': '4', '5': '5', '6': '6', '*': '*',
      '1': '1', '2': '2', '3': '3', '-': '-',
      '0': '0', '.': '.', '=': '=', '+': '+',
      'C': 'C',
      'Enter': '=',
      'Delete': 'C'
    };

    const formattedInput = computed(() => {
      if (!currentInput.value || currentInput.value === 'Error') {
        return currentInput.value;
      }
      const parsedValue = parseFloat(currentInput.value.replace(/,/g, ''));
      return parsedValue.toLocaleString('en', { maximumFractionDigits: 20 });
    });

    const formattedOperation = computed(() => {
      return currentOperation.value.replace(/\*/g, '×').replace(/\//g, '÷');
    });

    const stack = ref<string[]>([]);

    const handleKeyPress = (key: string): void => {
      activeKey.value = key;
      setTimeout(() => (activeKey.value = ''), 200);

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

    const handleOperator = (operator: string): void => {
      if (currentInput.value !== '') {
        if (stack.value.length === 0) {
          stack.value.push(currentInput.value);
          currentOperation.value = currentInput.value + operator;
        } else {
          stack.value.push(currentInput.value);
          const result = evaluateStack(stack.value);
          currentInput.value = result.toString();
          currentOperation.value += currentInput.value + operator;
          stack.value = [result.toString(), operator];
        }
        currentInput.value = '0';
      } else {
        if (stack.value.length > 0) {
          stack.value[stack.value.length - 1] = operator;
          currentOperation.value = currentOperation.value.slice(0, -1) + operator;
        }
      }
      resultDisplayed.value = false;
    };

    const handleEquals = (): void => {
      if (currentInput.value !== '') {
        stack.value.push(currentInput.value);
      }
      if (['+', '-', '*', '/'].includes(stack.value[stack.value.length - 1])) {
        stack.value.pop();
      }
      try {
        const result = evaluateStack(stack.value);
        history.value.push(`${currentOperation.value}${currentInput.value} = ${result}`);
        currentInput.value = result.toString();
        currentOperation.value = ''; 
        stack.value = [];
        resultDisplayed.value = true;
      } catch (error) {
        currentInput.value = 'Error';
        currentOperation.value = '';
        stack.value = [];
        resultDisplayed.value = false;
      }
    };

    const handleClear = (): void => {
      currentInput.value = '0';
      currentOperation.value = '';
      stack.value = [];
      resultDisplayed.value = false;
    };

    const handleNumber = (number: string): void => {
      if (resultDisplayed.value) {
        currentInput.value = number;
        resultDisplayed.value = false;
      } else {
        if (currentInput.value === 'Error') {
          currentInput.value = '';
          currentOperation.value = ''; 
        }
        if (currentInput.value === '0' && number !== '.') {
          currentInput.value = number;
        } else {
          currentInput.value += number;
        }
      }
    };

    const handleKeyboardInput = (event: KeyboardEvent): void => {
      const key = keyMap[event.key];
      if (key) {
        handleKeyPress(key);
      }
    };

    const evaluateStack = (stack: string[]): number => {
      const outputQueue: string[] = [];
      const operatorStack: string[] = [];
      const precedence: { [key: string]: number } = {
        '+': 1,
        '-': 1,
        '*': 2,
        '/': 2
      };

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

      const resultStack: number[] = [];
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

      return resultStack[0];
    };

    onMounted(() => {
      window.addEventListener('keydown', handleKeyboardInput);
    });

    onUnmounted(() => {
      window.removeEventListener('keydown', handleKeyboardInput);
    });

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
