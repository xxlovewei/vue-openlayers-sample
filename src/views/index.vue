<template>
  <div class="content">
    <div id="map">
    </div>

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
  import {transform} from 'ol/proj';
  import Axios from 'axios';

  import {CanvasScalarControl} from '../assets/plugins/weather/scalar-ol';
  let spotData = require('../assets/file/spot/t2.json');

  export default {
    data() {
      return {
        map: null,
        overlayStyles: [{text: '粒子动画', key: 'flow'}, {text: '填色', key: 'spot'}, {
          text: '方向标',
          key: 'arrow'
        }, {text: '等值线', key: 'isoline'}],
        checkedOverlayStyles: ['flow'],
        SpotLayer: null,
      };
    },
    watch: {
      checkedOverlayStyles: function (newVal, oldVal) {
        this.SpotLayer.update();
      }
    },
    mounted() {
      this.map = new Map({
        target: "map",
        layers: [
          new TileLayer({
            source: new XYZ({url: 'http://webst0{1-4}.is.autonavi.com/appmaptile?lang=zh_cn&size=1&scale=1&style=7&x={x}&y={y}&z={z}'})
          })
        ],
        view: new View({
          projection: "EPSG:3857",    //使用这个坐标系
          center: transform([115.43621, 30.22617], 'EPSG:4326', 'EPSG:3857'),  //深圳
          zoom: 5
        })
      });

      let vm = this;
      let SpotLayer = function () {
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
              return;
            }
          }
        };
      }();
      SpotLayer.update();
      vm.SpotLayer = SpotLayer;

      // let WindLayer = function() {
      //   let windy;
      //   let windCanvas = document.getElementById('windyMap');
      //
      //   function refreshWindy(windJSON, overlay) {
      //     if(!windy) {
      //       windy = new Windy({canvas: windCanvas});
      //     }
      //
      //     let mapSize = map.getSize();
      //     extent = ol.proj.transformExtent(view.calculateExtent(mapSize), EPSG3857, EPSG4326);
      //
      //     windCanvas.width = mapSize[0];
      //     windCanvas.height = mapSize[1];
      //
      //     let zoom = view.getZoom();
      //     let minZoom = view.getMinZoom();
      //
      //     windy.start(
      //       [[0,0], [mapSize[0], mapSize[1]]],
      //       extent, overlay, windJSON, zoom, minZoom
      //     );
      //   }
      //
      //   function updateWind() {
      //     let curOverlay = null;
      //     let fileName = null, filePath = null;
      //     let checkedFlowFactor = vueModel.LayerTool.checkedFactor['flow'];
      //     if(! checkedFlowFactor) {
      //       WindLayer.stop();
      //       return;
      //     }
      //
      //     metricConfig.getMetricFile(checkedFlowFactor, 'flow', function(data) {
      //       if(data) {
      //         refreshWindy(data, checkedFlowFactor);
      //       } else {
      //         WindLayer.stop();
      //       }
      //     });
      //   }
      //
      //   return {
      //     'update': updateWind,
      //     'stop': function() {
      //       windy && windy.stop();
      //     }
      //   };
      // }();
      //
      // let IsolineLayer = function() {
      //   let isoline = null;
      //   let isolineCanvas = document.getElementById('isolineCanvas');
      //
      //   function drawIsoline(isolineJSON, productType) {
      //     if(!isoline) {
      //       isoline = new Isoline({'canvas': isolineCanvas, 'map': map});
      //     }
      //     isoline.draw({'srcData': isolineJSON, 'productType': productType});
      //   }
      //
      //   function update() {
      //     let isolineTypes = ['isoline', 'heightIsoline', 'periodIsoline'];
      //     let checkedIsolineProductType = null;
      //     let isolineType = null;
      //     for(let i = 0; i < isolineTypes.length; i++) {
      //       if(vueModel.LayerTool.checkedFactor[isolineTypes[i]]) {
      //         checkedIsolineProductType = vueModel.LayerTool.checkedFactor[isolineTypes[i]];
      //         isolineType = isolineTypes[i];
      //         break;
      //       }
      //     }
      //     if(! checkedIsolineProductType) {
      //       IsolineLayer.clear();
      //       return;
      //     }
      //
      //     metricConfig.getMetricFile(checkedIsolineProductType, isolineType, function(data) {
      //       if(data) {
      //         drawIsoline(data, checkedIsolineProductType);
      //       } else {
      //         IsolineLayer.clear();
      //       }
      //     });
      //   }
      //
      //   return {
      //     'update': update,
      //     'clear': function() {
      //       isoline && isoline.clear();
      //     }
      //   };
      // }();
      //
      // let ArrowLayer = function() {
      //   let arrow = new Arrow({'canvas': $("#arrowCanvas")[0]});
      //
      //   function draw(data, productType, key) {
      //     arrow.setMap(map);
      //     arrow.draw({'srcData': data, 'productType': productType});
      //   }
      //
      //   function update() {
      //     let checkedArrowProductType = vueModel.LayerTool.checkedFactor['arrow'];
      //     if(! checkedArrowProductType) {
      //       ArrowLayer.clear();
      //       return;
      //     }
      //     metricConfig.getMetricFile(checkedArrowProductType, 'arrow', function(data) {
      //       if(data) {
      //         draw(data, checkedArrowProductType);
      //       } else {
      //         ArrowLayer.clear();
      //       }
      //     });
      //   }
      //
      //   return {
      //     'update': update,
      //     'clear': function() {
      //       arrow && arrow.clear();
      //     }
      //   };
      // }();
    },
    methods: {}
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

  #control {
    position: absolute;
    bottom: 20px;
    right: 20px;
  }
</style>