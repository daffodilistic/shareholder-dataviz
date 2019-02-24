<template>
  <div class="hello">
    <div class="container">
      <h1>Shareholder Dataviz</h1>
      <div class="col s6">
        <form action="#">
          <div class="file-field input-field">
            <div class="btn">
              <span>File</span>
              <input v-bind:disabled="isReady == false" ref="fileInput" v-on:change="onFileChanged" type="file" />
            </div>
            <div class="file-path-wrapper">
              <input v-bind:disabled="isReady == false" class="file-path validate" type="text" />
            </div>
          </div>
        </form>
        <br><br>
        <div id="map"></div>
      </div>
      <br>
      <div class="col s6">
        <div class="row">
          <div class="col s3 offset-s9">
            <div class="btn" v-on:click="saveFile">
              <span>Save raw data to file</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import XLSX from 'xlsx';
import L from '@/utils/leafletWrapper'
import { saveAs } from 'file-saver';

export default {
  name: "HelloWorld",
  props: {
    msg: String
  },
  data() {
    return {
      geocodeData: null,
      data: null,
      mapRef: null,
      fileData:null,
      isReady: false
    }
  },
  async mounted() {
    try {
      const MAPBOX_API_KEY = 'pk.eyJ1IjoiZGFmZm9kaWxpc3RpYyIsImEiOiJjam1kOGJ6M2YwbXRrM3Ztb2JwMHIzOTU5In0.mnbPSjRIinjgf8VBzPuHtg';
      const map = L.map('map').setView([1.3521, 103.8198], 12);

      L.tileLayer('https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token={accessToken}', {
          attribution: 'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
          maxZoom: 18,
          id: 'mapbox.streets',
          accessToken: MAPBOX_API_KEY
      }).addTo(map);
      
      this.mapRef = map;
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
        this.isReady = true;
      })
      .catch((error) => {
        // handle error
        console.log(error);
      })
  },
  methods: {
    saveFile() {
      if (this.data != null) {
        var blob = new Blob(JSON.stringify(this.data), {type: "text/plain;charset=utf-8"});
        saveAs(blob, "shareholder-dataviz.txt");
      }
    },
    generateData() {
      var userPostalCode = [];
      
      for (var i = 20; i < this.fileData.length; i++) {
        console.log(this.fileData[i]);
        var address = this.fileData[i];
        if (address[3] == "SINGAPORE") {
          var postalCode = this.fileData[i][7];
          
          var foundObject = this.geocodeData.find((e) => {
            return (
              e["POSTAL"] == postalCode);
          });

          if (foundObject != null || foundObject != undefined) {
            var gpsCoord = [
              foundObject.LATITUDE,
              foundObject.LONGITUDE
            ];
            userPostalCode.push(gpsCoord);
          }
        }
      }

      if (userPostalCode.length > 0) {
        this.data = userPostalCode;
        this.weighShareholdings();
        // console.log(this.data);

        L.heatLayer(this.data, {radius: 25}).addTo(this.mapRef);
      }
    },
    weighShareholdings() {
      // TODO assign heat intensity based on shareholdings
    },
    generateRandomData() {
      var userPostalCode = [];
      var users = 1500;

      for (var i = 1; i <= users; i++) {
        var index = this.getRandomIntInclusive(0,this.geocodeData.length);
        var gpsCoord = [
          this.geocodeData[index].LATITUDE,
          this.geocodeData[index].LONGITUDE
        ];

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
        this.fileData = data;

        // console.log(this.data);
        this.generateData();
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
@import "~leaflet/dist/leaflet.css";
#map {
  height: 60vh;
}
</style>
