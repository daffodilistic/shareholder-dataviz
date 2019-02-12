<template>
  <div class="hello">
    <div class="container">
      <h1>Shareholder Dataviz</h1>
      <div class="col s6">
        <form action="#">
          <div class="file-field input-field">
            <div class="btn">
              <span>File</span>
              <input ref="fileInput" v-on:change="onFileChanged" type="file" />
            </div>
            <div class="file-path-wrapper">
              <input class="file-path validate" type="text" />
            </div>
          </div>
        </form>
        <br><br>
        <div id="map"></div>
      </div>
    </div>
  </div>
</template>

<script>
import XLSX from 'xlsx';
import gmapsInit from '@/utils/gmaps';

export default {
  name: "HelloWorld",
  props: {
    msg: String
  },
  data() {
    return {
      data: null
    }
  },
  async mounted() {
    try {
      const google = await gmapsInit();
      const geocoder = new google.maps.Geocoder();
      const map = new google.maps.Map(document.getElementById('map'));

      geocoder.geocode({ address: 'Singapore 600332' }, (results, status) => {
        if (status !== 'OK' || !results[0]) {
          throw new Error(status);
        }

        map.setCenter(results[0].geometry.location);
        map.fitBounds(results[0].geometry.viewport);
      });
    } catch (error) {
      console.error(error);
    }
  },
  methods: {
    onFileChanged() {
      var file = event.target.files[0];
      const reader = new FileReader();
      reader.onload = e => {
        const bstr = e.target.result;
        const wb = XLSX.read(bstr, { type: "binary" });
        const wsname = wb.SheetNames[0];
        const ws = wb.Sheets[wsname];
        const data = XLSX.utils.sheet_to_json(ws, { header: 1 });
        this.data = data;

        console.log(this.data);
      };
      reader.readAsBinaryString(file);
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#map {
  height: 60vh;
}
</style>
