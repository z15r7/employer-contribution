<template>
    <pre style="text-align: left;">
        {{ tableData }}
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

                    console.log(convertedData[aIndex], convertedData[bIndex]);

                    const bodysData = (aIndex !== undefined && bIndex !== undefined)
                        ? convertedData.slice(aIndex, bIndex + 1).map((row, rIdx, headers) => {
                            let lastNonEmpty = '';
                            return row.map((cell, cIdx) => {
                                if (!cell) {
                                    const hasNextNonEmpty = headers.slice(rIdx + 1).some(row => row[cIdx]);
                                    return hasNextNonEmpty ? lastNonEmpty : '';
                                }
                                return (lastNonEmpty = cell);
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

                    console.log('headers:', headers);
                    // tableData.value = headers;

                    console.log('bodys:', bodys);
                    tableData.value = bodys;

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
