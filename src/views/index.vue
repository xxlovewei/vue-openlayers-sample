<template>
  <div class="content">
    <div id="map">
    </div>
    <canvas ref="windyCanvas" class="canvasMap"></canvas>
    <canvas ref="isolineCanvas" class="canvasMap"></canvas>
    <canvas ref="arrowCanvas" class="canvasMap"></canvas>

    <div id="control">
      <el-checkbox-group v-model="checkedOverlayStyles" size="small">
        <el-checkbox-button v-for="overlayStyle in overlayStyles" :label="overlayStyle.key"
                            :key="overlayStyle.key">{{overlayStyle.text}}
        </el-checkbox-button>
      </el-checkbox-group>
    </div>
  </div>
</template>

<script>
  import 'ol/ol.css';
  import {Map, View,} from "ol";
  import TileLayer from "ol/layer/Tile";
  import XYZ from "ol/source/XYZ";
  import {transform, transformExtent} from 'ol/proj';
  import {defaults as defaultControls} from 'ol/control.js';
  import MousePosition from 'ol/control/MousePosition';

  import {CanvasScalarControl} from '../assets/plugins/weather/scalar-ol';
  import {Windy} from '../assets/plugins/weather/wind-flow';
  import {Isoline} from '../assets/plugins/weather/isoline';
  import {Arrow} from '../assets/plugins/weather/windbarb-arrow';
  import {EPSG3857, EPSG4326} from '../assets/js/ol-util'

  let spotData = require('../assets/file/spot/t2');
  let flowData = require('../assets/file/flow/uv10');
  let isolineData = require('../assets/file/isoline/t2');
  let arrowDataRtofs = require('../assets/file/arrow/rtofs_uv');
  let arrowDataWind = require('../assets/file/arrow/uv10');

  export default {
    data() {
      return {
        map: null,
        overlayStyles: [{text: '粒子动画', key: 'flow'}, {text: '填色', key: 'spot'}, {
          text: '方向标',
          key: 'arrow'
        }, {text: '等值线', key: 'isoline'}],
        checkedOverlayStyles: [],
        SpotLayer: null,
        WindLayer: null,
        IsolineLayer: null,
        ArrowLayer: null,
        mapFirstLoadComplete: false,
      };
    },
    watch: {
      checkedOverlayStyles: function (newVal, oldVal) {
        this.updateWeatherLayers('show');
      }
    },
    mounted() {
      let vm = this;
      let map = new Map({
        target: "map",
        controls: defaultControls().extend([new MousePosition({
          undefinedHTML: '经纬度显示',
          projection: EPSG4326,
          coordinateFormat: function (coordinate) {
            return " 纬度:" + coordinate[1] + " 经度:" + coordinate[0];
          }
        })]),
        layers: [
          new TileLayer({
            source: new XYZ({url: 'http://webst0{1-4}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}'})
          })
        ],
        view: new View({
          projection: EPSG3857,    //使用这个坐标系
          center: transform([115.43621, 30.22617], EPSG4326, EPSG3857),  //深圳
          zoom: 5
        })
      });
      map.on('movestart', function (evt) {
        vm.updateWeatherLayers('clear');
      });
      map.on('moveend', function (evt) {
        vm.mapFirstLoadComplete = true;
        vm.updateWeatherLayers('show');
      });
      vm.map = map;

      vm.SpotLayer = function () {
        let lastestData = spotData;
        return {
          update: function () {
            let checkedSpot = vm.checkedOverlayStyles.indexOf('spot') != -1;
            if (!checkedSpot) {
              CanvasScalarControl.hide();
              return;
            }
            CanvasScalarControl.show();
            if (lastestData) {
              CanvasScalarControl.draw(vm.map, lastestData, 'temp');
            }
          }
        };
      }();

      vm.WindLayer = function () {
        let windy;
        let windCanvas = vm.$refs.windyCanvas;

        function refreshWindy(windJSON, overlay) {
          if (!windy) {
            windy = new Windy({canvas: windCanvas});
          }

          let mapSize = vm.map.getSize();
          let view = vm.map.getView();
          let extent = transformExtent(view.calculateExtent(mapSize), EPSG3857, EPSG4326);

          windCanvas.width = mapSize[0];
          windCanvas.height = mapSize[1];

          let zoom = view.getZoom();
          let minZoom = view.getMinZoom();

          windy.start(
            [[0, 0], [mapSize[0], mapSize[1]]],
            extent, overlay, windJSON, zoom, minZoom
          );
        }

        function updateWind() {
          let curOverlay = null;
          let fileName = null, filePath = null;
          let checkedFlow = vm.checkedOverlayStyles.indexOf('flow') != -1;
          if (!checkedFlow) {
            stop();
            return;
          }

          if (flowData) {
            refreshWindy(flowData, 'wind');
          } else {
            stop();
          }
        }

        function stop() {
          windy && windy.stop();
        }

        return {
          'update': updateWind,
          'stop': stop
        };
      }();

      vm.IsolineLayer = function () {
        let isoline = null;
        let isolineCanvas = vm.$refs.isolineCanvas;

        function clear() {
          isoline && isoline.clear();
        }

        function drawIsoline(isolineJSON, productType) {
          if (!isoline) {
            isoline = new Isoline({'canvas': isolineCanvas, 'map': map});
          }
          isoline.draw({'srcData': isolineJSON, 'productType': productType});
        }

        function update() {
          let checkedIsoline = vm.checkedOverlayStyles.indexOf('isoline') != -1;
          if (!checkedIsoline) {
            clear();
            return;
          }

          if (isolineData) {
            drawIsoline(isolineData, 'temp');
          } else {
            clear();
          }
        }

        return {
          'update': update,
          'clear': clear
        };
      }();

      vm.ArrowLayer = function () {
        let arrow = new Arrow({'canvas': vm.$refs.arrowCanvas, 'map': vm.map});
        let productType = 'wind';
        let lastestData = arrowDataWind;

        function draw(data, productType, key) {
          arrow.draw({'srcData': data, 'productType': productType});
        }

        function update() {
          let checked = vm.checkedOverlayStyles.indexOf('arrow') != -1;
          if (!checked) {
            clear();
            swtich();// 关闭时切换一次数据
            return;
          }

          draw(lastestData, productType);
        }

        function clear() {
          arrow && arrow.clear();
        }

        function swtich() {
          if (productType == 'wind') {
            productType = 'oceanCurrent';
            lastestData = arrowDataRtofs;
          } else {
            productType = 'wind';
            lastestData = arrowDataWind;
          }
        }

        return {
          'update': update,
          'clear': clear,
          'swtich': swtich
        };
      }();

      vm.checkedOverlayStyles = ['flow', 'spot', 'isoline', 'arrow'];
    },
    methods: {
      updateWeatherLayers: function (operation) {
        if (this.mapFirstLoadComplete) {
          if (operation == 'show') {
            this.SpotLayer.update();
            this.WindLayer.update();
            this.IsolineLayer.update();
            this.ArrowLayer.update();

          } else if (operation == 'clear') {
            this.WindLayer.stop();
            this.IsolineLayer.clear();
            this.ArrowLayer.clear();
          }
        }
      }
    }
  };
</script>

<style>
  .content {
    height: 100%;
    width: 100%;
    padding: 0;
    margin: 0;
  }

  #map {
    height: 100%;
  }

  .canvasMap {
    position: absolute;
    z-index: 2;
    pointer-events: none;
    top: 0;
    left: 0;
  }

  #control {
    position: absolute;
    bottom: 20px;
    right: 20px;
  }
</style>