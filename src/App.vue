<template>
  <div id="app" class="container-fluid py-2">
    <div class="mb-3" v-if="!fileUrl">
      <label class="mb-1">Choose CSV</label>
      <input type="file" class="form-control" @change="fileOnChange($event)" />
    </div>

    <template v-if="fileLoaded">
      <div class="mb-3">
        <label class="mb-1">Table column (JSON Array)</label>
        <div class="input-group">
          <input class="form-control" v-model="columnsJson" />
          <div class="input-group-append">
            <button class="btn btn-primary">เลือก</button>
          </div>
        </div>
      </div>

      <div class="mb-3">
        <label class="mb-1">Barcode column</label>
        <select class="form-control" v-model="barcodeColumn">
          <option value="" disabled selected hidden></option>
          <option v-for="column in columns" :key="column">{{ column }}</option>
        </select>
      </div>

      <table class="table table-bordered table-striped">
        <thead>
          <tr>
            <th v-for="column in columns" :key="column">{{ column }}</th>
            <th v-if="barcodeColumn">Barcode: {{ barcodeColumn }}</th>
          </tr>
        </thead>

        <tbody>
          <tr v-for="(row, i) in data" :key="i">
            <td v-for="column in columns" :key="column">{{ row[column] }}</td>
            <td v-if="barcodeColumn">
              <barcode v-bind:value="row[barcodeColumn]">FAILED</barcode>
            </td>
          </tr>
        </tbody>
      </table>
    </template>

    <div class="mt-4 text-center">
      Barcode generator from CSV by Chomtana :)
    </div>
  </div>
</template>

<script>
import VueBarcode from "vue-barcode";

const urlParams = new URLSearchParams(window.location.search);
const fileUrl = urlParams.get("fileUrl");
const barcodeColumn = urlParams.get("barcodeColumn");

export default {
  name: "App",
  data() {
    return {
      data: [],
      columns: [],
      columnsJson: "[]",
      fileLoaded: false,
      fileUrl,
      barcodeColumn,
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
              this.columns = Object.keys(results.data[0]);
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
  },
  watch: {
    columns: {
      deep: true,
      handler(val) {
        this.columnsJson = JSON.stringify(val);
      },
    },
    barcodeColumn(val) {
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
</style>
