<template>
  <div class="hello">
    <div class="container">
      <h1>Shareholder Data Visualizer</h1>
      <div class="col s6">
        <form action="#">
          <div class="file-field input-field">
            <div class="btn">
              <span>File</span>
              <input
                v-bind:disabled="isReady == false"
                ref="fileInput"
                v-on:change="onFileChanged"
                type="file"
              />
            </div>
            <div class="file-path-wrapper">
              <input
                v-bind:disabled="isReady == false"
                class="file-path validate"
                type="text"
              />
            </div>
          </div>
        </form>
        <br /><br />
        <div id="map"></div>
      </div>
      <br />
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
import XLSX from "xlsx";
import L from "@/utils/leafletWrapper";
import { saveAs } from "file-saver";
import lodash from "lodash";

export default {
  name: "HelloWorld",
  props: {},
  data() {
    return {
      geocodeData: null,
      data: null,
      mapRef: null,
      fileData: null,
      isReady: false
    };
  },
  async mounted() {
    const axios = require("axios");

    try {
      const MAPBOX_API_KEY =
        "pk.eyJ1IjoiZGFmZm9kaWxpc3RpYyIsImEiOiJjam1kOGJ6M2YwbXRrM3Ztb2JwMHIzOTU5In0.mnbPSjRIinjgf8VBzPuHtg";
      const map = L.map("map").setView([1.3521, 103.8198], 12);

      L.tileLayer(
        "https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token={accessToken}",
        {
          attribution:
            'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, ' +
            '<a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, ' +
            'Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
          maxZoom: 18,
          id: "mapbox.streets",
          accessToken: MAPBOX_API_KEY
        }
      ).addTo(map);

      this.mapRef = map;
    } catch (error) {
      console.error(error);
    }

    // Make a request for a user with a given ID
    await axios
      .get(
        "https://raw.githubusercontent.com/xkjyeah/singapore-postal-codes/master/buildings.json"
      )
      .then(response => {
        // handle success
        // console.log(response.data);
        this.geocodeData = lodash.keyBy(response.data, "POSTAL");
        this.isReady = true;
      })
      .catch(error => {
        // handle error
        console.log(error);
      });
  },
  beforeDestroy() {
    this.geocodeData = null;
    this.data = null;
    this.fileData = null;
  },
  methods: {
    saveFile() {
      if (this.data != null) {
        var blob = new Blob([JSON.stringify(this.data)], {
          type: "text/plain;charset=utf-8"
        });
        saveAs(blob, "shareholder-dataviz.txt");
      }
    },
    generateData() {
      var userList = [];
      var sharesCounter = 0;

      for (var i = 0; i < this.fileData.length; i++) {
        var userEntry = this.fileData[i];
        if (
          userEntry[3] != "DEP AGENT" &&
          userEntry[7] != "" &&
          userEntry[7] != null &&
          userEntry[7] != undefined
        ) {
          var postalCode = userEntry[7];

          var foundObject = this.geocodeData[postalCode];
          if (foundObject != null || foundObject != undefined) {
            var gpsCoord = [foundObject.LATITUDE, foundObject.LONGITUDE];

            var shareholdings = parseInt(userEntry[8].split(",").join(""));
            if (shareholdings > sharesCounter) {
              sharesCounter = shareholdings;
            }
            var userObject = {
              gpsCoord: gpsCoord,
              shares: shareholdings
            };

            // console.log(userObject);
            userList.push(userObject);
          }
        }
      }

      if (userList.length > 0) {
        this.fileData.length = 0;
        this.data = userList;
        this.weighShareholdings(sharesCounter);

        L.heatLayer(this.data, {
          maxZoom: 12
        }).addTo(this.mapRef);
      }
    },
    weighShareholdings(totalShares) {
      // console.log("totalShares is " + totalShares);
      var weightedList = [];

      this.data.forEach(e => {
        var heatPoint = [
          e.gpsCoord[0],
          e.gpsCoord[1],
          (1.0 * e.shares) / totalShares
        ];
        //console.log(heatPoint);
        weightedList.push(heatPoint);
      });
      this.data = weightedList;
      // console.log(this.data);
    },
    generateRandomData() {
      this.fileData.length = 0;
      const users = 5000;
      const keys = Object.keys(this.geocodeData);

      var userList = [];
      for (var i = 1; i <= users; i++) {
        var index = this.getRandomIntInclusive(0, keys.length);
        var postalCode = keys[index];
        var gpsCoord = [
          this.geocodeData[postalCode].LATITUDE,
          this.geocodeData[postalCode].LONGITUDE
        ];

        // console.log(this.geocodeData[index].LATITUDE + ", " + this.geocodeData[index].LONGITUDE);
        var userObject = [gpsCoord[0], gpsCoord[1], Math.random() * users];

        userList.push(userObject);
      }

      this.data = userList;
      // console.log(this.data);

      L.heatLayer(this.data, {
        // maxZoom: 12,
        max: users
      }).addTo(this.mapRef);
    },
    onFileChanged() {
      // console.log(event.target.files);
      var file = event.target.files[0];
      const reader = new FileReader();
      reader.onload = e => {
        // const startTime = Date.now();

        const bstr = e.target.result;
        const wb = XLSX.read(bstr, { type: "binary" });
        const wsname = wb.SheetNames[0];
        const ws = wb.Sheets[wsname];

        var data = XLSX.utils.sheet_to_json(ws, {
          header: 1,
          blankrows: false
        });
        // Skip first 6 rows since its garbage data
        for (var i = 1; i <= 6; i++) {
          data.shift();
        }
        this.fileData = data;
        // console.log(this.fileData);

        // const endTime = Date.now();
        this.generateData();
        // this.generateRandomData();
        // console.log("Time taken to generate data: " + (endTime - startTime) + "ms");
      };
      reader.readAsBinaryString(file);
    },
    getRandomIntInclusive(min, max) {
      min = Math.ceil(min);
      max = Math.floor(max);
      // The maximum is inclusive and the minimum is inclusive
      return Math.floor(Math.random() * (max - min + 1)) + min;
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
