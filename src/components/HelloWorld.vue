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
      geocodeData: null,
      data: null,
      mapRef: null
    }
  },
  async mounted() {
    try {
      const google = await gmapsInit();
      const geocoder = new google.maps.Geocoder();
      const map = new google.maps.Map(document.getElementById('map'));

      this.mapRef = map;

      geocoder.geocode({ address: 'Singapore' }, (results, status) => {
        if (status !== 'OK' || !results[0]) {
          throw new Error(status);
        }

        map.setCenter(results[0].geometry.location);
        map.fitBounds(results[0].geometry.viewport);
      });
    } catch (error) {
      console.error(error);
    }

    const axios = require('axios');

    // Make a request for a user with a given ID
    await axios.get('https://raw.githubusercontent.com/xkjyeah/singapore-postal-codes/master/buildings.json')
      .then((response) => {
        // handle success
        // console.log(response.data);
        this.geocodeData = response.data;
        this.generateData();

        var heatmap = new google.maps.visualization.HeatmapLayer({
          data: this.data,
          map: this.mapRef
        });
        // heatmap.setMap(heatmap.getMap() ? null : this.mapRef);

        // console.log("mapRef is " + this.mapRef);

        // heatmap.setMap(this.mapRef);

        // delete this.geocodeData;
      })
      .catch((error) => {
        // handle error
        console.log(error);
      })
  },
  methods: {
    generateData() {
      var userPostalCode = [];
      var users = 1500;

      for (var i = 1; i <= users; i++) {
        var index = this.getRandomIntInclusive(0,this.geocodeData.length);
        var gpsCoord = new google.maps.LatLng(
          this.geocodeData[index].LATITUDE,
          this.geocodeData[index].LONGITUDE
        );

        // console.log(this.geocodeData[index].LATITUDE + ", " + this.geocodeData[index].LONGITUDE);
        userPostalCode.push(gpsCoord);
      }

      this.data = userPostalCode;
      // console.log(this.data);
    },
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
    },
    getRandomIntInclusive(min, max) {
      min = Math.ceil(min);
      max = Math.floor(max);
      return Math.floor(Math.random() * (max - min + 1)) + min; //The maximum is inclusive and the minimum is inclusive 
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
