<template>
  <div>

    <div class="position-relative">
      <!-- shape Hero -->
      <section class="section-shaped my-0">
        <div class="shape shape-style-1 shape-default shape-skew">
          <span></span>
          <span></span>
        </div>
        <div class="container shape-container d-flex">

          <div class="col text-white">

            <base-alert v-if="error !== null" type="danger">
              <h5 class="text-white">Error</h5>
              <p v-html="error"></p>
            </base-alert>

            <h1 class="display-3 text-white">{{ $t('visualize.title') }}</h1>

            <p>{{ $t('visualize.dataWarning') }}</p>

            <base-input addon-left-icon="ni ni-calendar-grid-58">
              <flat-picker v-if="dataLoaded"
                           slot-scope="{focus, blur}"
                           @on-open="focus"
                           @on-close="blur"
                           @on-change="buildLayers"
                           :config="datePickerFormat"
                           class="form-control datepicker"
                           v-model="dateFilter">
              </flat-picker>
            </base-input>

            <base-button v-for="(layerDefinition) in layersDefinifion" :key="layerDefinition.id"
                         class="mb-3"
                         size="sm"
                         :type="layerDefinition.buttonColor"
                         :icon="`fa fa-${layerEnabled(layerDefinition) ? 'check-square' : 'square-o'}`"
                         @click="toggleLayer(layerDefinition)">
              <span>{{ $t(layerDefinition.label) }}</span>
            </base-button>

            <div id="leaflet-map"></div>
            <p v-if="lastUpdate"><small>{{ $t('visualize.lastUpdate') }} {{ lastUpdate.toLocaleString() }}</small></p>
            <!--            <p>{{totalReports}} reports overall</p>-->
          </div>

        </div>
      </section>
    </div>

  </div>
</template>

<script>
  import L from 'leaflet';
  import 'leaflet/dist/leaflet.css';

  import * as Papa from 'papaparse';

  import flatPicker from 'vue-flatpickr-component';
  import 'flatpickr/dist/flatpickr.css';

  var _map = null;
  var _data = [];
  var _tileLayers = [];

  var _today = new Date();
  var _yesterday = new Date();
  _yesterday.setDate(_yesterday.getDate() - 1);

  export default {
    name: "visualize",
    components: {flatPicker},
    data() {
      return {
        dataLoaded: false,
        mapBaseLayerUrl: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
        dataSourceBaseUrl: process.env.VUE_APP_VISU_DATA_SOURCE_URL,
        geocodeFileUrl: process.env.VUE_APP_VISU_GEOCODE_URL,
        lastUpdateFileUrl: process.env.VUE_APP_VISU_LAST_UPDATE_URL,

        showMerged: true,
        scaleFactor: 100,

        dateData: 0,
        geocoding: null,
        lastUpdate: null,

        dateFilter: _today.toISOString().split('T')[0],
        allowedDates: [],
        datePickerFormat: {
          allowInput: true,
          dateFormat: 'Y-m-d',
          minDate: '2020-03-25',
          maxDate: _today.toISOString(),
        },

        minBubbleSize: 1000,
        maxBubbleSize: 3000,

        error: null,

        activeLayers: [],
        layersDefinifion: [
          {
            id: 'healthy',
            label: 'visualize.layerHealthy',
            value: (entry) => entry.total_healthy,
            sizeRatio: 1,
            color: 'blue',
            buttonColor: 'info',
            opacity: 0.2,
            defaultEnabled: false,
          },
          {
            id: 'sick_guess_no_corona',
            label: 'visualize.layerSickNoCovid',
            value: (entry) => entry.total_sick_guess_no_corona,
            sizeRatio: 1,
            color: 'blue',
            buttonColor: 'info',
            opacity: 0.1,
            defaultEnabled: false,
          },
          {
            id: 'sick_guess_corona',
            label: 'visualize.layerSickCovid',
            value: (entry) => entry.total_sick_guess_corona,
            sizeRatio: 1,
            color: 'orangered',
            buttonColor: 'warning',
            opacity: 0.3,
            defaultEnabled: true,
          },
          {
            id: 'sick_corona_confirmed',
            label: 'visualize.layerSickCovidConfirmed',
            value: (entry) => entry.total_sick_corona_confirmed,
            sizeRatio: 1,
            color: 'red',
            buttonColor: 'danger',
            opacity: 0.3,
            defaultEnabled: true,
          },
          {
            id: 'recovered_not_confirmed',
            label: 'visualize.layerRecovered',
            value: (entry) => entry.total_recovered_not_confirmed,
            sizeRatio: 1,
            color: 'green',
            buttonColor: 'success',
            opacity: 0.3,
            defaultEnabled: false,
          },
          {
            id: 'recovered_confirmed',
            label: 'visualize.layerRecoveredConfirmed',
            value: (entry) => entry.total_recovered_confirmed,
            sizeRatio: 1,
            color: 'green',
            buttonColor: 'success',
            opacity: 0.3,
            defaultEnabled: false,
          },
        ]
      };
    },
    async mounted() {

      try {
        this.geocoding = await this.loadGeocode(this.geocodeFileUrl);
      } catch (error) {
        this.error = error;
      }

      try {
        await this.loadLeaflet();
      } catch (error) {
        this.error = error;
      }

      try {
        const lu = await this.loadLastUpdate(this.lastUpdateFileUrl);
        this.lastUpdate = new Date(Object.keys(lu[0])[0]);
      } catch (error) {

      }
    },
    computed: {
      dataSourceUrl() {
        return this.showMerged ? `${this.dataSourceBaseUrl}/merge-all-days.csv` : `${this.dataSourceBaseUrl}/daily-reports/ch-covid-19-${this.dateFilter}.csv`;
      },
    },
    methods: {
      isDateAllowed: function (date) {
        return this.allowedDates.includes(date);
      },
      layerEnabled: function (layerDefinition) {
        return this.activeLayers.includes(layerDefinition);
      },
      toggleLayer: function (layerDefinition) {
        if (this.layerEnabled(layerDefinition)) {
          this.removeLayer(layerDefinition);
        } else {
          this.addLayer(layerDefinition);
        }
      },
      addLayer: function (layerDefinition) {
        layerDefinition.data.layer.addTo(_map);
        this.activeLayers.push(layerDefinition);
      },
      removeLayer: function (layerDefinition) {
        _map.removeLayer(layerDefinition.data.layer);
        this.activeLayers = this.activeLayers.filter((layer) => layer !== layerDefinition);
      },
      loadGeocode: function (url) {
        return new Promise(function (resolve, reject) {
          Papa.parse(url, {
            download: true,
            header: true,
            error: (err, file, inputElem, reason) => reject(`Could not get geo-coding data<br>${err}`),
            complete: (content, file) => {

              const geocoding = {};
              for (const location of content.data) {
                location.coordinates = [+location.latitude, +location.longitude];
                geocoding[location.npa_plz] = location;
              }

              resolve(geocoding)
            },
          });
        });
      },
      loadData: async function (dataFileUrl) {
        return new Promise(function (resolve, reject) {
          Papa.parse(`${dataFileUrl}?timestamp=${new Date()}`, {
            download: true,
            header: true,
            error: (err, file, inputElem, reason) => reject(`Could not get map data<br>${err}`),
            complete: (content, file) => resolve(content.data),
          });
        });
      },
      loadLastUpdate: async function (lastUpdateFileUrl) {
        return new Promise(function (resolve, reject) {
          Papa.parse(`${lastUpdateFileUrl}?timestamp=${new Date()}`, {
            download: true,
            header: true,
            error: (err, file, inputElem, reason) => reject(`Could not get map data<br>${err}`),
            complete: (content, file) => resolve(content.data),
          });
        });
      },
      computeAllowedDates: function (data) {
        const flags = {};
        const allowedDates = [];
        for (const entry of data) {
          if (entry.date !== '' && !flags[entry.date]) {
            flags[entry.date] = true;
            allowedDates.push(entry.date);
          }
        }
        return allowedDates;
      },
      loadLeaflet: async function () {

        _map = L.map('leaflet-map', {
          preferCanvas: true,
        }).setView([46.7, 8.2], 8);

        _tileLayers.push(L.tileLayer(this.mapBaseLayerUrl, {
          attribution: `&copy; <a href="http://osm.org/copyright">OpenStreetMap</a> contributors`,
        }).addTo(_map));

        try {
          _data = await this.loadData(this.dataSourceUrl);
          this.dataLoaded = true;

          this.allowedDates = this.computeAllowedDates(_data);

          this.buildLayers(null, this.dateFilter);

        } catch (error) {
          console.error(error);
        }
      },
      buildLayers: function (selectedDates, dateStr, instance) {

        const activeLayersBackup = this.activeLayers.map(layerDefinition => layerDefinition.id);

        for (const layerDefinition of this.activeLayers) {
          _map.removeLayer(layerDefinition.data.layer);
        }

        let allLayersMax = 0;

        for (const layerDefinition of this.layersDefinifion) {
          layerDefinition.data = {
            max: 0,
            markers: [],
          };
        }

        let totalReports = 0;
        for (const entry of _data) {

          if (entry.date !== dateStr) {
            continue;
          }

          entry.total_healthy = +entry.total_healthy;
          entry.total_sick_guess_no_corona = +entry.total_sick_guess_no_corona;
          entry.total_sick_guess_corona = +entry.total_sick_guess_corona;
          entry.total_sick_corona_confirmed = +entry.total_sick_corona_confirmed;
          entry.total_recovered_not_confirmed = +entry.total_recovered_not_confirmed;
          entry.total_recovered_confirmed = +entry.total_recovered_confirmed;
          totalReports += entry.total_healthy +
            entry.total_sick_guess_no_corona +
            entry.total_sick_guess_corona +
            entry.total_sick_corona_confirmed +
            entry.total_recovered_not_confirmed +
            entry.total_recovered_confirmed;

          for (const layerDefinition of this.layersDefinifion) {
            const markerSize = layerDefinition.value(entry) * layerDefinition.sizeRatio;
            if (markerSize > layerDefinition.data.max) {
              layerDefinition.data.max = markerSize;

              if (layerDefinition.data.max > allLayersMax) {
                allLayersMax = layerDefinition.data.max;
              }
            }
          }
        }

        this.totalReports = totalReports;
        console.log(totalReports);

        let ignored = 0;
        for (const entry of _data) {

          if (entry.date !== dateStr) {
            continue;
          }

          const geocoding = this.geocoding[entry.npa_plz];

          if (!geocoding) {
            ignored++;
            continue;
          }

          let popup = `
            <h4>${entry.npa_plz} ${geocoding.town} (${geocoding.state})</h4>
            <table class="data">`;

          for (const layerDefinition of this.layersDefinifion) {
            popup += `
              <tr>
                <td style="color: ${layerDefinition.color}">${layerDefinition.value(entry)}</td>
                <td>${this.$i18n.t(layerDefinition.label)}</td>
              </tr>
            `;
          }

          popup += `</table>
          `;

          for (const layerDefinition of this.layersDefinifion) {
            try {
              let markerSize = layerDefinition.value(entry) * layerDefinition.sizeRatio;
              // markerSize = markerSize / allLayersMax * this.maxBubbleSize;
              markerSize = markerSize / layerDefinition.data.max * this.maxBubbleSize;

              layerDefinition.data.markers.push(L.circle(geocoding.coordinates, {
                weight: 0,
                fillColor: layerDefinition.color,
                fillOpacity: layerDefinition.opacity,
                radius: markerSize > 0 ? Math.max(markerSize, this.minBubbleSize) : 0,
              }).bindPopup(popup));
            } catch (error) {
              console.log(entry, error);
            }
          }
        }

        const options = {
          attribution: `<a href="${this.dataSourceUrl}" target="_blank">Data source</a>`,
        };

        const overlays = {};
        for (const layerDefinition of this.layersDefinifion) {
          layerDefinition.data.layer = L.layerGroup(layerDefinition.data.markers, options);

          if (activeLayersBackup.length > 0) {
            if (activeLayersBackup.includes(layerDefinition.id)) {
              this.addLayer(layerDefinition);
            }
          } else {
            if (layerDefinition.defaultEnabled) {
              this.addLayer(layerDefinition);
            }
          }

          const label = `<span style="color: ${layerDefinition.color}">${this.$i18n.t(layerDefinition.label)}</span>`;
          overlays[label] = layerDefinition.data.layer;
        }
      }
    }
  };

</script>

<style>

  #leaflet-map {
    height: 500px;
    z-index: 1;
    position: relative;
  }

  .data td:first-child {
    font-weight: bold;
    text-align: right;
    margin-right: 5px;
  }

</style>
