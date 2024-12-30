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
    const ContributionData = ref([])
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
            // if (isNaN(inputValue) || inputValue < 1 || inputValue > 12) {
            //   showError('無效的月份輸入')
            //   return
            // }
            months.value = inputValue
            calculate()
          }
          break
        case 'C':
          clearInput()
          break
        default:
          // console.log(value, input.value + value)
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
      updateEmployerContribution()
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

    //   
    const openEmployerContributionTable = async () => {
      alert('打開雇主負擔表格編輯界面 (提供新增/刪除/修改/保存功能)');

      try {
        // Fetch the XLSX file from the Vue public folder
        const response = await fetch('/contribution.xlsx');

        if (!response.ok) {
          throw new Error('Failed to fetch the XLSX file.');
        }

        // Read the file as a Blob
        const fileBlob = await response.blob();

        // Read the Blob content as ArrayBuffer
        const reader = new FileReader();
        reader.onload = (e) => {
          const data = new Uint8Array(e.target.result);

          // Parse the workbook using XLSX library
          const workbook = XLSX.read(data, { type: 'array' });

          // Extract the first sheet's data
          const sheetName = workbook.SheetNames[0];
          const sheet = workbook.Sheets[sheetName];
          const jsonData = XLSX.utils.sheet_to_json(sheet);

          // console.log('Sheet Data:', jsonData);

          // Provide an interface to edit and save the data
          editTable(jsonData, workbook, sheetName);
        };

        reader.readAsArrayBuffer(fileBlob);
      } catch (error) {
        console.error('Error:', error);
        alert('無法載入文件。請檢查文件是否存在於 public 資料夾中。');
      }
    };

    const editTable = (data, workbook, sheetName) => {
      // Create a simple editable table dynamically
      const container = document.createElement('div');
      const table = document.createElement('table');
      const saveButton = document.createElement('button');

      // Generate table headers
      const headers = Object.keys(data[0] || {});
      const thead = document.createElement('thead');
      const headerRow = document.createElement('tr');

      headers.forEach(header => {
        const th = document.createElement('th');
        th.textContent = header;
        headerRow.appendChild(th);
      });
      thead.appendChild(headerRow);
      table.appendChild(thead);

      // Generate table body
      const tbody = document.createElement('tbody');
      data.forEach(row => {
        const tr = document.createElement('tr');
        headers.forEach(header => {
          const td = document.createElement('td');
          td.contentEditable = 'true';
          td.textContent = row[header] || '';
          tr.appendChild(td);
        });
        tbody.appendChild(tr);
      });
      table.appendChild(tbody);

      // Save button functionality
      saveButton.textContent = '保存';
      saveButton.onclick = () => {
        const updatedData = [];
        Array.from(tbody.children).forEach(row => {
          const rowData = {};
          Array.from(row.children).forEach((cell, index) => {
            rowData[headers[index]] = cell.textContent;
          });
          updatedData.push(rowData);
        });

        // Convert updated data back to XLSX and download
        const newSheet = XLSX.utils.json_to_sheet(updatedData);
        workbook.Sheets[sheetName] = newSheet;
        const newWorkbookBlob = XLSX.write(workbook, { bookType: 'xlsx', type: 'blob' });

        const link = document.createElement('a');
        link.href = URL.createObjectURL(newWorkbookBlob);
        link.download = 'updated_employer_contributions.xlsx';
        link.click();
      };

      container.appendChild(table);
      container.appendChild(saveButton);
      document.body.appendChild(container);
    };



    const loadContributionData = () => {
      ContributionData.value = [
        { min: 0, max: 3000, percentage: 0.1 },
        { min: 3001, max: 6000, percentage: 0.15 }
      ]
    }

    const updateEmployerContribution = () => {
      if (!ContributionData.value.length || !salary.value) return
      const item = ContributionData.value.find((d) => salary.value >= d.min && salary.value <= d.max)
      employerContribution.value = item ? item.percentage * salary.value : 0
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
      loadContributionData()
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
      ContributionData,
      error,
      keys,
      resultDisplay,
      formatNumber,
      handleInput,
      clearInput,
      deleteLast,
      calculate,
      openEmployerContributionTable
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
