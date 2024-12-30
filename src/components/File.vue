<template>
    <pre>
        {{ JSON.stringify(tableData, null, 4) }}
    </pre>
    <!-- <vue-excel-editor v-model="tableData">
        <vue-excel-column v-for="(column, index) in headers" :key="index" :field="column.value" :label="column.text"
            :type="column.type" :width="column.width" />
    </vue-excel-editor> -->
</template>

<script>
import { ref, onMounted } from 'vue';
import * as XLSX from 'xlsx';

export default {
    name: "App",
    setup() {
        const tableData = ref([]);
        const headers = ref([]);

        const loadXlsx = async () => {
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

                    const startIndex = convertedData.map((row, idx) => row.filter(cell => cell.includes('級數')).length > 0 ? idx : -1).filter(idx => idx !== -1);
                    const endIndex = convertedData.map((row, idx) => row.filter(cell => cell.includes('雇主負擔合計')).length > 0 ? idx : -1).filter(idx => idx !== -1);

                    let headers = [];

                    if (startIndex !== -1 && endIndex !== -1) {
                        headers = convertedData.slice(startIndex[startIndex.length - 1], endIndex[endIndex.length - 1] + 1);
                    }

                    console.log('headers:', headers);

                    let headers2 = [];

                    // add all column headers to headers2

                    // headers.map(row =>)


                    if (convertedData.length > 0) {
                        headers.value = Object.keys(convertedData[0]).map(key => ({ text: key, value: key }));
                    }

                    // tableData.value = convertedData;
                    tableData.value = headers;
                };

                reader.readAsArrayBuffer(fileBlob);
            } catch (error) {
                console.error('Error:', error);
                alert('Unable to load the file. Please check if it exists in the public folder.');
            }
        };

        onMounted(() => {
            loadXlsx();
        });

        return { tableData, headers, loadXlsx };
    }
};
</script>
