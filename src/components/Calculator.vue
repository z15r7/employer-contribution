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

    <button class="edit-button" @click="openEmployerContributionTable">編輯雇主負擔</button>
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
    const employerContribution = ref(0)
    const display = ref('請輸入薪資金額')
    const contributionData = ref([])
    const error = ref(null)

    // const headers = ref([]);
    // const bodys = ref([]);

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
            months.value = inputValue
            calculate()
          }
          break
        case 'C':
          clearInput()
          break
        default:
          if (mode.value === 'months') {
            const inputValue = parseInt(input.value + value)
            if (isNaN(inputValue) || inputValue < 1 || inputValue > 12) {
              input.value = value
              display.value = formatNumber(input.value)
              return
            }
          }
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
      if (!contributionData.value.length || !salary.value) {return}
      const item = contributionData.value.find((d) => salary.value >= d['月薪總額-(含)以上'] && salary.value <= d['月薪總額-迄'])
      employerContribution.value = item ? item['勞保費用']['雇主負擔']['雇主負擔合計'] + item['健保費用']['雇主負擔'] : 0
      const totalCost = (salary.value + employerContribution.value) * months.value
      display.value = `總人事費用：${formatNumber(totalCost)} 
        (公式：(${formatNumber(salary.value)} + ${formatNumber(employerContribution.value)}) × ${months.value})`
      input.value = ''
      mode.value = 'salary'
    }

    const showError = (msg) => {
      error.value = msg
      display.value = msg
      input.value = ''
    }

    const loadContributionData = async () => {
      try {
        const response = await fetch('/contribution.xlsx');

        if (!response.ok) {
          throw new Error('Failed to fetch the XLSX file.');
        }

        const fileBlob = await response.blob();

        const reader = new FileReader();
        reader.onload = (e) => {
          const data = new Uint8Array(e.target.result);
          const workbook = XLSX.read(data, { type: 'array' });
          const sheetName = workbook.SheetNames[0];
          const sheet = workbook.Sheets[sheetName];
          const csvData = XLSX.utils.sheet_to_csv(sheet);

          // console.log('Loaded Data:', csvData);

          let convertedData = csvData.replace(/"(?!.*?,)([^"]*?\r\n[^"]*?)"/g, (match, p1) => {
            return p1.replace(/\r\n/g, '');
          }).replace(/"(.*?)"/g, (match, p1) => {
            return p1.replace(/,/g, '[comma]');
          }).split('\n').map(row => row
            .split(',').map(cell => cell
              .trim()
              .replace(/\[comma\]/g, ',')
              .replace(/ /g, '')
              .replace(/^(\d+,\d+)$/g, (match, p1) => {
                return p1.replace(/,/g, '');
              })
            )
          );

          // console.log('convertedData:', convertedData);

          const findIndexes = (data, keyword) =>
            data.reduce((acc, row, idx) => row.some(cell => cell === keyword) ? [...acc, idx] : acc, []);

          const [startIndex, endIndex] = [
            findIndexes(convertedData, '級數').at(-1),
            findIndexes(convertedData, '雇主負擔合計').at(-1)
          ];

          const headers = (startIndex !== undefined && endIndex !== undefined)
            ? convertedData.slice(startIndex, endIndex + 1).map((row, rIdx, headers) => {
              let lastNonEmpty = '';
              return row.map((cell, cIdx) => {
                if (!cell) {
                  const hasNextNonEmpty = headers.slice(rIdx + 1).some(row => row[cIdx]);
                  return hasNextNonEmpty ? lastNonEmpty : '';
                }
                return (lastNonEmpty = cell);
              });
            }).reduce((acc, row, rIdx, allHeaders) => {
              row.forEach((cell, cIdx) => {
                if (!cell) return;

                let currentLevel = acc;

                [...Array(rIdx)].forEach((_, i) => {
                  const ancestorCell = allHeaders[i][cIdx];
                  if (!ancestorCell) return;

                  if (typeof currentLevel[ancestorCell] !== 'object') {
                    currentLevel[ancestorCell] = {};
                  }
                  currentLevel = currentLevel[ancestorCell];
                });

                currentLevel[cell] = currentLevel[cell] || cIdx;
              });
              return acc;
            }, {})
            : {};

          const [aIndex, bIndex] = [
            findIndexes(convertedData, '第1級').at(-1),
            findIndexes(convertedData, '第59級').at(-1)
          ];

          // console.log(convertedData[aIndex], convertedData[bIndex]);

          const bodysData = (aIndex !== undefined && bIndex !== undefined)
            ? convertedData.slice(aIndex, bIndex + 1).map((row, rIdx, headers) => {
              let lastNonEmpty = '';
              return row.map((cell, cIdx) => {
                if (!cell) {
                  const hasNextNonEmpty = headers.slice(rIdx + 1).some(row => row[cIdx]);
                  return hasNextNonEmpty ? lastNonEmpty : '';
                }
                return (lastNonEmpty = (!isNaN(cell) ? parseInt(cell) : cell));
              });
            }) : {};

          const bodys = bodysData.map(row => {
            const buildNestedObject = (nestedHeaders, row) => {
              return Object.entries(nestedHeaders).reduce((acc, [key, value]) => {
                if (typeof value === 'object') {
                  acc[key] = buildNestedObject(value, row);
                } else {
                  acc[key] = row[value];
                }
                return acc;
              }, {});
            };

            return buildNestedObject(headers, row);
          });

          // console.log('headers:', headers);
          // tableData.value = headers;

          // console.log('bodys:', bodys);
          contributionData.value = bodys;

        };

        reader.readAsArrayBuffer(fileBlob);
      } catch (error) {
        console.error('Error:', error);
        alert('Unable to load the file. Please check if it exists in the public folder.');
      }
    };

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
      loadContributionData();
    })

    onUnmounted(() => {
      window.removeEventListener('keydown', handleKeyboardInput)
    })

    return {
      input,
      mode,
      salary,
      months,
      employerContribution,
      display,
      ContributionData: contributionData,
      error,
      keys,
      resultDisplay,
      formatNumber,
      handleInput,
      clearInput,
      deleteLast,
      calculate
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
