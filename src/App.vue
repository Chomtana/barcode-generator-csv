<template>
  <div id="app" class="container-fluid py-2">
    <div class="mb-3" v-if="!fileUrl">
      <label class="mb-1">Choose CSV</label>
      <input type="file" class="form-control" @change="fileOnChange($event)" />
    </div>

    <template v-if="fileLoaded">
      <div class="no-print">
        <div class="mb-3">
          <label class="mb-1">Table column (JSON Array)</label>
          <div class="input-group">
            <input class="form-control" v-model="columnsJson" />
            <div class="input-group-append">
              <button class="btn btn-primary">เลือก</button>
            </div>
          </div>
        </div>

        <div class="row mb-3">
          <div class="col-6">
            <label>Width</label>
            <input type="number" v-model="codeWidth" class="form-control" />
          </div>

          <div class="col-6">
            <label>Height</label>
            <input type="number" v-model="codeHeight" class="form-control" />
          </div>
        </div>

        <div class="mb-3">
          <label class="mb-1">Barcode column</label>
          <select class="form-control" v-model="codeColumn">
            <option value="" disabled selected hidden></option>
            <option v-for="column in columns" :key="column">
              {{ column }}
            </option>
          </select>
        </div>

        <div class="mb-3">
          <button class="btn btn-primary me-2" @click="downloadZip()">Download images</button>
          <button class="btn btn-warning me-2" onclick="window.print()">
            Print
          </button>
        </div>
      </div>

      <table class="table table-bordered table-striped">
        <thead>
          <tr>
            <th v-for="column in columns" :key="column">{{ column }}</th>
            <th v-if="codeColumn">Barcode: {{ codeColumn }}</th>
          </tr>
        </thead>

        <tbody>
          <tr v-for="(row, i) in data" :key="i">
            <td v-for="column in columns" :key="column">{{ row[column] }}</td>
            <td v-if="codeColumn">
              <barcode
                class="code-container"
                :data-value="row[codeColumn]"
                v-bind:value="row[codeColumn]"
                :displayValue="barcodeDisplayValue"
                :width="barcodeWidth(row[codeColumn])"
                :height="codeHeight"
                :margin="0"
                elementTag="img"
                >FAILED</barcode
              >
            </td>
          </tr>
        </tbody>
      </table>
    </template>

    <div class="mt-4 text-center no-print">
      Barcode generator from CSV by Chomtana :)
    </div>
  </div>
</template>

<script>
import VueBarcode from "vue-barcode";
import JSZip from "jszip";
import { saveAs } from "file-saver";

const urlParams = new URLSearchParams(window.location.search);
const fileUrl = urlParams.get("fileUrl");
const codeColumn = urlParams.get("codeColumn");

function numberOfBars(value) {
  // from https://www.adams1.com/128code.html
  // this is correct as long as there aren't consecutive numbers.
  value = value.toString();
  let length = 0;
  for (let c of value) {
    if (isNaN(c)) {
      length += 11;
    } else {
      length += 5.5;
    }
  }
  return length + 35;
}

export default {
  name: "App",
  data() {
    return {
      data: [],
      columns: [],
      columnsJson: "[]",
      fileLoaded: false,
      fileUrl,
      codeColumn,
      barcodeDisplayValue: false,
      codeWidth: 121,
      codeHeight: 75,
    };
  },
  async mounted() {
    if (fileUrl) {
      await this.loadFile(fileUrl);
    }
  },
  components: {
    barcode: VueBarcode,
  },
  computed: {},
  methods: {
    loadFile(file) {
      return new Promise((resolve, reject) => {
        let download = false;

        if (typeof file === "string" && file.startsWith("http")) {
          download = true;
        }

        //console.log("LOAD", file);

        window.Papa.parse(file, {
          download: download,
          header: true,
          complete: (results) => {
            console.log(results);

            if (results.data.length > 0) {
              this.data = results.data;

              this.data = this.data.filter(
                (c) =>
                  Object.values(c)
                    .map((x) => (x ? 1 : 0))
                    .reduce((acc, curr) => acc + curr, 0) > 0
              );

              this.columns = Object.keys(this.data[0]);
              this.fileLoaded = true;
              //console.log("SUCCESS", file);
              resolve(results.data);
            } else {
              let message = "ไม่พบข้อมูล";
              alert(message);
              reject({
                err: new Error(message),
              });
            }
          },
          error: (err) => {
            alert(
              "การโหลดข้อมูลเกิดข้อผิดพลาด " +
                (typeof file === "string" ? file : "")
            );
            reject(err);
          },
        });
      });
    },
    changeColumns(columnsJson) {
      try {
        this.columns = JSON.parse(columnsJson);
      } catch (err) {
        console.error(err);
        alert("ไม่สามารถอ่าน Table column JSON ได้ กรุณาเขียนให้ถูก Syntax");
      }
    },
    async fileOnChange(event) {
      let fileList = event.target.files;

      if (fileList.length > 0) {
        let file = fileList[0];
        await this.loadFile(file);
      }
    },
    numberOfBars,
    barcodeWidth(data) {
      return this.codeWidth / numberOfBars(data);
    },
    downloadZip() {
      var zip = new JSZip();

      document.querySelectorAll(".code-container").forEach((container) => {
        zip.file(
          container.dataset.value + ".png",
          container.querySelector("img").src.replace("data:image/png;base64,", ""),
          { base64: true }
        );
      });

      zip.generateAsync({ type: "blob" }).then(function (content) {
        // see FileSaver.js
        saveAs(content, "barcode.zip");
      });
    },
  },
  watch: {
    columns: {
      deep: true,
      handler(val) {
        this.columnsJson = JSON.stringify(val);
      },
    },
    codeColumn(val) {
      if (val) {
        setTimeout(() => {}, 1000);
      }
    },
  },
};
</script>

<style>
#app {
}

@media print {
  .no-print {
    display: none;
  }
}
</style>
