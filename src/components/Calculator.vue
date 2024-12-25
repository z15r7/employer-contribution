<template>
  <div id="app">
    <!-- Display Area -->
    <div class="display-area">
      <div class="current-operation">
        薪資：{{ salary ? formatNumber(salary) : '--' }}，月份：{{ months || '--' }}
      </div>
      <div class="result" ref="resultDisplay">
        {{ display }}
      </div>
    </div>

    <!-- Button Area -->
    <div class="button-area">
      <button v-for="key in keys" :key="key" @click="handleInput(key)">
        {{ key }}
      </button>
    </div>

    <button class="edit-button" @click="openEmployerBurdenTable">編輯雇主負擔</button>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import * as XLSX from 'xlsx'

export default {
  name: 'PersonnelCostCalculator',
  setup() {
    const input = ref('')
    const mode = ref('salary')
    const salary = ref(null)
    const months = ref(null)
    const employerBurden = ref(0)
    const display = ref('請輸入薪資金額')
    const burdenData = ref([])
    const error = ref(null)

    const resultDisplay = ref(null)
    const keys = [
      '7', '8', '9', '←',
      '4', '5', '6', '月份',
      '1', '2', '3', '薪資',
      '0', '.', 'C', '='
    ]

    const formatNumber = (val) => {
      if (!val && val !== 0) return ''
      return parseFloat(val).toLocaleString()
    }

    const handleInput = (value) => {
      if (error.value) error.value = null

      switch (value) {
        case '薪資':
          if (isNaN(parseFloat(input.value)) || parseFloat(input.value) <= 0) {
            showError('無效的薪資輸入')
            return
          }
          (mode.value === 'salary' ? salary : months).value = parseFloat(input.value)
          salary.value = null
          input.value = ''
          mode.value = 'salary'
          display.value = '請輸入薪資金額'
          break
        case '月份':
          if (isNaN(parseFloat(input.value)) || parseFloat(input.value) <= 0) {
            showError('無效的薪資輸入')
            return
          }
          (mode.value === 'salary' ? salary : months).value = parseFloat(input.value)
          months.value = null
          input.value = ''
          mode.value = 'months'
          display.value = '請輸入月份(1~12)'
          break
        case '=':
          if (mode.value === 'salary') {
            const inputValue = parseFloat(input.value)
            if (isNaN(inputValue) || inputValue <= 0) {
              showError('無效的薪資輸入')
              return
            }
            salary.value = inputValue
            months.value = 1
            calculate()
          } else if (mode.value === 'months') {
            const inputValue = parseInt(input.value)
            if (isNaN(inputValue) || inputValue < 1 || inputValue > 12) {
              showError('無效的月份輸入')
              return
            }
            months.value = inputValue
            calculate()
          }
          break
        case 'C':
          clearInput()
          break
        default:
          input.value += value
          display.value = formatNumber(input.value)
          scrollToEnd()
          break
      }
    }

    const clearInput = () => {
      input.value = ''
      salary.value = null
      months.value = null
      error.value = null
      mode.value = 'salary'
      display.value = '請輸入薪資金額'
    }

    const deleteLast = () => {
      input.value = input.value.slice(0, -1)
      display.value = input.value || '輸入中...'
    }

    const calculate = () => {
      updateEmployerBurden()
      const totalCost = (salary.value + employerBurden.value) * months.value
      display.value = `總人事費用：${formatNumber(totalCost)} 
        (公式：(${formatNumber(salary.value)} + ${formatNumber(employerBurden.value)}) × ${months.value})`
      input.value = ''
      mode.value = 'salary'
    }

    const showError = (msg) => {
      error.value = msg
      display.value = msg
      input.value = ''
    }

    // https://posman.nccu.edu.tw/download.php?dir=rule&filename=f68439116016d3d4d09629e1b696391b.xlsx&title=6-1.%E5%8B%9E%E5%81%A5%E4%BF%9D%E3%80%81%E5%8B%9E%E9%80%80%E6%8A%95%E4%BF%9D%E5%88%86%E7%B4%9A%E8%A1%A8%EF%BC%88114.1.1%E8%B5%B7%E9%81%A9%E7%94%A8%EF%BC%89.xlsx
    const openEmployerBurdenTable = () => {
      alert('打開雇主負擔表格編輯界面 (提供新增/刪除/修改/保存功能)')
    }

    const loadBurdenData = () => {
      burdenData.value = [
        { min: 0, max: 3000, percentage: 0.1 },
        { min: 3001, max: 6000, percentage: 0.15 }
      ]
    }

    const updateEmployerBurden = () => {
      if (!burdenData.value.length || !salary.value) return
      const item = burdenData.value.find((d) => salary.value >= d.min && salary.value <= d.max)
      employerBurden.value = item ? item.percentage * salary.value : 0
    }

    const scrollToEnd = () => {
      nextTick(() => {
        if (resultDisplay.value) {
          resultDisplay.value.scrollLeft = resultDisplay.value.scrollWidth
        }
      })
    }

    const handleKeyboardInput = (event) => {
      const key = event.key
      switch (key) {
        case 'Backspace':
          deleteLast()
          break
        case 'Enter':
          handleInput('=')
          break
        case 'Delete':
          clearInput()
          break
        case '*':
          handleInput('月份')
          break
        default:
          if (keys.includes(key) || key === '.') {
            handleInput(key)
          }
          break
      }
    }

    onMounted(() => {
      window.addEventListener('keydown', handleKeyboardInput)
      loadBurdenData()
    })

    onUnmounted(() => {
      window.removeEventListener('keydown', handleKeyboardInput)
    })

    return {
      input,
      mode,
      salary,
      months,
      employerBurden,
      display,
      burdenData,
      error,
      keys,
      resultDisplay,
      formatNumber,
      handleInput,
      clearInput,
      deleteLast,
      calculate,
      openEmployerBurdenTable
    }
  }
}
</script>

<style scoped>
#app {
  font-family: Arial, sans-serif;
  text-align: center;
  margin: 20px;
}

.display-area {
  margin-bottom: 20px;
}

.current-operation {
  font-size: 16px;
  color: #555;
  margin-bottom: 10px;
}

.result {
  font-size: 24px;
  font-weight: bold;
  margin-bottom: 20px;
  overflow-x: auto;
  white-space: nowrap;
}

.button-area {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 10px;
}

button {
  padding: 15px;
  font-size: 18px;
  cursor: pointer;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #ddd;
}

.edit-button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}
</style>
