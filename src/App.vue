<script>
import * as turf from "@turf/turf";

import('@maptiler/sdk/dist/maptiler-sdk.css')
import points from './assets/points_for_map.json'

  import * as maptilersdk from '@maptiler/sdk';
  export default {
  name: 'MapComponent',
    data(){
      return {
        map: null
      }
    },
  async mounted() {
    maptilersdk.config.apiKey = 'BvrtwMrSBaJInDrAfqu9';

    this.map = new maptilersdk.Map({
      container: 'map', // container's id or the HTML element in which SDK will render the map
      style: `https://api.maptiler.com/maps/87bc86b0-be50-42e3-8c05-f78258a31d48/style.json?key=BvrtwMrSBaJInDrAfqu9`,
      center: [31, 49], // starting position [lng, lat]
      zoom: 4.7, // starting zoom
      compassControl: false,
      attributionControl: false,
      navigationControl: false,
      geolocateControl: true,
      language: 'ua',
      preserveDrawingBuffer: true,
      smoothInterpolationResult: null
    })

    this.smoothInterpolationResult = this.smoothInterpolation(this.transformCircleToSquares(points), 2)
    // this.smoothInterpolationResult = this.transformCircleToSquares(points)

    this.map.on('load', async (map) => {
      this.map.addSource('dots-source', {
        type: 'geojson',
        data: points
      });

      this.map.addSource('squares-source', {
        type: 'geojson',
        data: this.smoothInterpolationResult
      });

      console.log('!!', this.smoothInterpolationResult)

      this.map.addLayer({
        id: 'squares-layer',
        type: 'fill',
        source: 'squares-source',
        layout: {},
        paint: {
          'fill-color': [
            'interpolate',
            ['linear'],
            ['get', 'value'],
            0, '#F2F12D',
            6, '#EED322',
            8, '#E6B71E',
            10, '#DA9C20',
            12, '#CA8323',
            14, '#B86B25',
            16, '#A25626',
            18, '#8B4225',
            20, '#723122'
          ],
          'fill-opacity': 0.75,
          'fill-outline-color': 'rgba(255, 255, 255, 0)'
        }
      })

    })
  },
    methods: {
      transformCircleToSquares(data, size=0.05){
        const newData = {
          type: "FeatureCollection",
          features: data.features.map((item) => {
            return {
              ...item,
              geometry: {
                "type": "Polygon",
                coordinates: [[
                  [parseFloat(item.geometry.coordinates[0]) - size, parseFloat(item.geometry.coordinates[1]) + size],
                  [parseFloat(item.geometry.coordinates[0]) + size, parseFloat(item.geometry.coordinates[1]) + size],
                  [parseFloat(item.geometry.coordinates[0]) + size, parseFloat(item.geometry.coordinates[1]) - size],
                  [parseFloat(item.geometry.coordinates[0]) - size, parseFloat(item.geometry.coordinates[1]) - size],
                  [parseFloat(item.geometry.coordinates[0]) - size, parseFloat(item.geometry.coordinates[1]) + size],
                ]]
              }
            }
          })
        }
        return newData
      },
      diffValues(data){
        const newData = {
          ...data,
          features: data.features.map((item) => {
            return {
              ...item,
              properties: {
                value: item.properties.value < 12 ? item.properties.value / 2 : item.properties.value * 2
              }
            }
          })
        }

        return newData
      },
      hashGeoJSON(geojson) {
        // Простая хэш-функция для GeoJSON объекта
        return JSON.stringify(geojson).split('').reduce((a, b) => {
          a = ((a << 5) - a) + b.charCodeAt(0);
          return a & a;
        }, 0);
      },
      divideAndInterpolatePolygon(geojson, divisionFactor) {
        const hashKey = 'geojson_' + this.hashGeoJSON(geojson) + '_divFactor_' + divisionFactor;

        const cachedData = localStorage.getItem(hashKey);
        if (cachedData) {
          return JSON.parse(cachedData); // Возвращаем кэшированный результат, если он есть
        }

        const newFeatures = [];

        geojson.features.forEach(feature => {
          const coords = feature.geometry.coordinates[0];
          const properties = feature.properties;

          const latStep = (coords[2][0] - coords[0][0]) / divisionFactor;
          const lonStep = (coords[2][1] - coords[0][1]) / divisionFactor;
          const baseValue = properties.value;

          // Параметры интерполяции
          const minValue = baseValue * 0.8; // Минимальное значение
          const maxValue = baseValue * 1.2; // Максимальное значение

          for (let i = 0; i < divisionFactor; i++) {
            for (let j = 0; j < divisionFactor; j++) {
              const newCoords = [
                [
                  [coords[0][0] + i * latStep, coords[0][1] + j * lonStep],
                  [coords[0][0] + (i + 1) * latStep, coords[0][1] + j * lonStep],
                  [coords[0][0] + (i + 1) * latStep, coords[0][1] + (j + 1) * lonStep],
                  [coords[0][0] + i * latStep, coords[0][1] + (j + 1) * lonStep],
                  [coords[0][0] + i * latStep, coords[0][1] + j * lonStep]
                ]
              ];

              // Интерполяция значения
              const factorX = i / (divisionFactor - 1);
              const factorY = j / (divisionFactor - 1);
              const interpolationFactor = (factorX + factorY) / 2;
              const interpolatedValue = this.interpolateValue(baseValue, minValue, maxValue, interpolationFactor);

              const newProperties = {
                value: interpolatedValue
              };

              newFeatures.push({
                type: 'Feature',
                properties: newProperties,
                geometry: {
                  type: 'Polygon',
                  coordinates: newCoords
                }
              });
            }
          }
        });

        return {
          type: 'FeatureCollection',
          features: newFeatures
        };
      },
      interpolateValue(baseValue, min, max, factor) {
        return baseValue + (max - min) * factor;
      },
      smoothInterpolation(geojson, divisionFactor) {
        const dividedGeojson = this.divideAndInterpolatePolygon(geojson, divisionFactor);
        const bufferedFeatures = dividedGeojson.features.map(feature => {
          return {
            feature: feature,
            buffer: turf.buffer(feature, 0.01, { units: 'kilometers' }) // небольшой буфер вокруг каждого фича
          };
        });

        const smoothedFeatures = dividedGeojson.features.map(feature => {
          let baseValue = feature.properties.value;
          let totalWeight = 1;
          let weightedSum = baseValue;

          // Определение соседей через пересечение буферов
          bufferedFeatures.forEach(buff => {
            if (feature !== buff.feature && turf.intersect(buff.buffer, feature)) {
              const weight = 1; // простой вес, можно использовать расстояние между центроидами для веса
              weightedSum += buff.feature.properties.value * weight;
              totalWeight += weight;
            }
          });

          const smoothedValue = weightedSum / totalWeight - divisionFactor * 0.7;
          return turf.feature(feature.geometry, {
            ...feature.properties,
            value: smoothedValue
          });
        });

        return turf.featureCollection(smoothedFeatures);
      }
    }
};
</script>

<template>
  <div id="map"></div>
</template>

<style scoped>
#map {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 100%;
  height: 100vh;
  z-index: -1;
}
</style>
