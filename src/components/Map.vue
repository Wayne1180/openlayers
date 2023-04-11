<template>
  <div class="container" id="map">
    <div v-if="mapData.length">
    <div style="width: 20px; height: 20px" v-for="(item, index) in mapData" :key="index" :id="'marker-' + index" type="coffee" @click="console.log('牛逼')" ><a-icon style="font-size: 20px;" type="environment" theme="filled" /></div>
    </div>
    <a-icon style="font-size: 20px" type="smile" theme="twoTone" two-tone-color="#eb2f96" id="heart" />
  </div>
</template>

<script>
// 导入转换坐标工具
import gcoord from "gcoord";
// 导入样式
import "ol/ol.css";
// 导入Map
import {Map, View, Feature} from "ol";
// 导入控件
import { defaults as defaultControls } from "ol/control";
// 导入图层
import TileLayer from "ol/layer/Tile";
//
// import View from "ol/View";
import WMTS from "ol/source/WMTS";
import WMTSTileGrid from "ol/tilegrid/WMTS";
import { getTopLeft, getWidth } from "ol/extent";
import { get as getProjection } from "ol/proj";
// 导入点
import Overlay from "ol/Overlay";
// 导入要素相关的样式工具
import { Style, Stroke } from 'ol/style'
import { LineString } from "ol/geom";
import { Vector as VectorLayer } from 'ol/layer'
import { Vector as VectorSource } from 'ol/source'
// import VectorSource from 'ol/source/Vector';

export default {
  props: {
    mapData: {
      type: Array,
      default: () => []
    },
    lineData: {
      type: Array,
      default: () => []
    },
    mm: {
      type: Array,
      default: () => []
    }
  },
  mounted() {
    this.initMap();
    console.log(this.mapData);
  },
  methods: {
    initMap() {
      // 先计算出投影，再根据投影计算出分辨率和矩阵ID
      const _projection = getProjection("EPSG:3857");
      const projectionExtent = _projection.getExtent();
      const size = getWidth(projectionExtent) / 256;
      const _resolutions = new Array(18);
      const matrixIds = new Array(18);
      for (let z = 1; z < 19; ++z) {
        _resolutions[z] = size / Math.pow(2, z);
        matrixIds[z] = z;
      }

      // 初始化背景图层
      const tiandiVec = new TileLayer({
        visible: true,
        title: '天地图',
        // 使用WMTS来加载瓦片数据
        source: new WMTS({
          // 瓦片的URL
          url: 'http://t0.tianditu.gov.cn/vec_w/wmts?tk=ca0d54bde130e0e4984b58c8fad41a44',
          // 图层名称
          layer: 'vec',
          // 样式
          style: 'default',
          // 矩阵集
          matrixSet: 'w',
          // 输出格式
          format: 'tiles',
          wrapX: true,
          // 投影
          projection: _projection,
          // 瓦片网格
          tileGrid: new WMTSTileGrid({
            // 指定了左上角瓦片的原点坐标
            origin: getTopLeft(projectionExtent),
            // 设置分辨率
            resolutions: _resolutions,
            // 设置矩阵ID
            matrixIds: matrixIds
          })
        })
      })
      // 图层：文字信息
      const tiandiCva = new TileLayer({
        visible: true,
        title: '天地图',
        // className: 'tiandiCvaDark',
        source: new WMTS({
          url: 'http://t0.tianditu.gov.cn/cva_w/wmts?tk=ca0d54bde130e0e4984b58c8fad41a44',
          layer: 'cva',
          style: 'default',
          matrixSet: 'w',
          format: 'tiles',
          wrapX: true,
          projection: _projection,
          tileGrid: new WMTSTileGrid({
            origin: getTopLeft(projectionExtent),
            resolutions: _resolutions,
            matrixIds: matrixIds
          })
        })
      })
      // 初始化地图
      const map = new Map({
        layers: [tiandiVec, tiandiCva],
        target: 'map',
        controls: defaultControls({
          zoom: true,
          rotate: true,
          attribution: true
        }),
        view: new View({
          center: gcoord.transform([118.78414381954954,32.04185158563585], gcoord.WGS84, gcoord.EPSG3857),
          zoom: 14
        }),
        // interactions: defaults({
        //   doubleClickZoom: false // 屏蔽双击放大事件
        // })
      })
      this.map = map
      this.addSnapMarkers(this.mapData)
      this.addPath(this.lineData)
      this.jumpTo(this.mm)
    },

    // 添加点要素
    addSnapMarkers (data) {
      const markers =  data.map((i, index) => {
        return new Overlay({
          position: gcoord.transform([i.siteX, i.siteY], gcoord.WGS84, gcoord.EPSG3857),
          element: document.getElementById('marker-' + index),
          stopEvent: false,
          positioning: 'center-center',
          id: index
        })
      })
      markers.forEach((i) => {
        this.map.addOverlay(i)
      })
    },
    // 添加线要素
    addPath (data) {
      data = data.map((item) => {
        const geo = gcoord.transform([item[0], item[1]], gcoord.WGS84, gcoord.EPSG3857)
        return geo
      })

      // 创建线集合对象
      const lineGeom = new LineString(data)

      // 创建线要素对象
      const lineFeature = new Feature({
        geometry: lineGeom,
        name: 'line'
      })

      const lineStyle = new Style({
        stroke: new Stroke({
          color: '#2B8EF3',
          width: 10
        })
      })

      // 创建矢量数据源
      const source = new VectorSource({
        features: [lineFeature]
      })

      // 创建矢量图层
      const vectorLayer = new VectorLayer({
        source: source,
        style: lineStyle
      })

      // 添加矢量图层到地图中
      this.map.addLayer(vectorLayer)
    },
    // 跳转
    jumpTo (site) {
      // 跳转
      if(!site[0] || !site[1]) {
        return
      }
      console.log('跳转中');
      this.map.getView().animate({
        center: gcoord.transform([site[0], site[1]], gcoord.WGS84, gcoord.EPSG3857),
        zoom: 16
      })
      // 添加点要素
      const markers = new Overlay({
        position: gcoord.transform([site[0], site[1]], gcoord.WGS84, gcoord.EPSG3857),
        element: document.getElementById('heart'),
        stopEvent: false,
        positioning: 'center-center',
        id: '1'
      })
      this.map.addOverlay(markers)
    }
  },
};
</script>

<style lang="scss" scoped>
.container {
  width: 100%;
  height: 1289px;
  // background-color: pink;
}
</style>
