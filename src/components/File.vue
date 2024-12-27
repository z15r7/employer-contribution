<template>
    <v-row>
        <v-col cols="8">
            <template>
                <v-file-input accept=".xlsx" label="File input(xlsx)" outlined v-model="selectXlsx" show-size>
                </v-file-input>
            </template>
        </v-col>

        <v-col cols="4">
            <v-btn color="primary" @click="uploadXlsx" class="mt-3">
                Upload
            </v-btn>
        </v-col>
    </v-row>
</template>

<script>
import XLSX from "xlsx";

export default {
    name: "App",
    data() {
        return {
            selectXlsx: null,
        }
        methods: {
            uploadXlsx() {
                if (!this.selectXlsx) {
                    console.log("Please upload a xlsx file")
                    return;
                }
                if (this.selectXlsx) {
                    const reader = new FileReader();
                    reader.onload = (e) => {
                        /* Parse data */
                        const bstr = e.target.result;
                        const wb = XLSX.read(bstr, { type: "binary" });
                        /* Get first worksheet */
                        const wsname = wb.SheetNames[0];
                        const ws = wb.Sheets[wsname];
                        /* Convert array of arrays */
                        const data = XLSX.utils.sheet_to_json(ws, { header: 1 });

                        console.log(data);
                    };

                    reader.readAsBinaryString(this.selectXlsx);
                }
                this.selectXlsx = null;
            },
        }
    }
}
</script>