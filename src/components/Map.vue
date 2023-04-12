<template>
  <div class="container" id="map">
    <div v-if="mapData.length">
      <div
        style="width: 20px; height: 20px"
        v-for="(item, index) in mapData"
        :key="index"
        :id="'marker-' + index"
        type="coffee"
        @click="console.log('牛逼')"
      >
        <a-icon style="font-size: 20px" type="environment" theme="filled" />
      </div>
    </div>
    <a-icon
      style="font-size: 20px"
      type="smile"
      theme="twoTone"
      two-tone-color="#eb2f96"
      id="heart"
    />
  </div>
</template>

<script>
import Point from "ol/geom/Point";
import Icon from "ol/style/Icon";
import CircleStyle from "ol/style/Circle";
import Fill from "ol/style/Fill";
import { map as mapImages } from "@/components/image";
import cloneDeep from "lodash/cloneDeep";
// 导入转换坐标工具
import gcoord from "gcoord";
// 导入样式
import "ol/ol.css";
// 导入Map
import { Map, View, Feature } from "ol";
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
import { Style, Stroke } from "ol/style";
import { LineString } from "ol/geom";
import { Vector as VectorLayer } from "ol/layer";
import { Vector as VectorSource } from "ol/source";
// import VectorSource from 'ol/source/Vector';

export default {
  props: {
    mapData: {
      type: Array,
      default: () => [],
    },
    lineData: {
      type: Array,
      default: () => [],
    },
    mm: {
      type: Array,
      default: () => [],
    },
    arrData: {
      type: Array,
      default: () => [],
    },
    flowData: {
      type: Array,
      default: () => [],
    },
    riverData: {
      type: Array,
      default: () => [],
    },
  },
  mounted() {
    this.initMap();
    // console.log(this.mapData);
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
        title: "天地图",
        // 使用WMTS来加载瓦片数据
        source: new WMTS({
          // 瓦片的URL
          url: "http://t0.tianditu.gov.cn/vec_w/wmts?tk=ca0d54bde130e0e4984b58c8fad41a44",
          // 图层名称
          layer: "vec",
          // 样式
          style: "default",
          // 矩阵集
          matrixSet: "w",
          // 输出格式
          format: "tiles",
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
            matrixIds: matrixIds,
          }),
        }),
      });
      // 图层：文字信息
      const tiandiCva = new TileLayer({
        visible: true,
        title: "天地图",
        // className: 'tiandiCvaDark',
        source: new WMTS({
          url: "http://t0.tianditu.gov.cn/cva_w/wmts?tk=ca0d54bde130e0e4984b58c8fad41a44",
          layer: "cva",
          style: "default",
          matrixSet: "w",
          format: "tiles",
          wrapX: true,
          projection: _projection,
          tileGrid: new WMTSTileGrid({
            origin: getTopLeft(projectionExtent),
            resolutions: _resolutions,
            matrixIds: matrixIds,
          }),
        }),
      });
      // 初始化地图
      const map = new Map({
        layers: [tiandiVec, tiandiCva],
        target: "map",
        controls: defaultControls({
          zoom: true,
          rotate: true,
          attribution: true,
        }),
        view: new View({
          center: gcoord.transform(
            [118.78414381954954, 32.04185158563585],
            gcoord.WGS84,
            gcoord.EPSG3857
          ),
          zoom: 14,
        }),
        // interactions: defaults({
        //   doubleClickZoom: false // 屏蔽双击放大事件
        // })
      });
      this.map = map;
      this.addSnapMarkers(this.mapData);
      this.addPath(this.lineData);
      this.jumpTo(this.mm);
      // this.arrowPath(this.arrData);
      // this.flowPath(this.flowData);
      this.riverPath(this.riverData);
    },

    // 添加点要素
    addSnapMarkers(data) {
      const markers = data.map((i, index) => {
        return new Overlay({
          position: gcoord.transform(
            [i.siteX, i.siteY],
            gcoord.WGS84,
            gcoord.EPSG3857
          ),
          element: document.getElementById("marker-" + index),
          stopEvent: false,
          positioning: "center-center",
          id: index,
        });
      });
      markers.forEach((i) => {
        this.map.addOverlay(i);
      });
    },
    // 添加线要素
    addPath(data) {
      data = data.map((item) => {
        const geo = gcoord.transform(
          [item[0], item[1]],
          gcoord.WGS84,
          gcoord.EPSG3857
        );
        return geo;
      });

      // 创建线集合对象
      const lineGeom = new LineString(data);

      // 创建线要素对象
      const lineFeature = new Feature({
        geometry: lineGeom,
        name: "line",
      });

      const lineStyle = new Style({
        stroke: new Stroke({
          color: "#2B8EF3",
          width: 10,
        }),
      });

      // 创建矢量数据源
      const source = new VectorSource({
        features: [lineFeature],
      });

      // 创建矢量图层
      const vectorLayer = new VectorLayer({
        source: source,
        style: lineStyle,
      });

      // 添加矢量图层到地图中
      this.map.addLayer(vectorLayer);
    },
    // 跳转
    jumpTo(site) {
      // 跳转
      if (!site[0] || !site[1]) {
        return;
      }
      // console.log("跳转中");
      this.map.getView().animate({
        center: gcoord.transform(
          [site[0], site[1]],
          gcoord.WGS84,
          gcoord.EPSG3857
        ),
        zoom: 16,
      });
      // 添加点要素
      const markers = new Overlay({
        position: gcoord.transform(
          [site[0], site[1]],
          gcoord.WGS84,
          gcoord.EPSG3857
        ),
        element: document.getElementById("heart"),
        stopEvent: false,
        positioning: "center-center",
        id: "1",
      });
      this.map.addOverlay(markers);
    },
    // 添加箭头样式
    arrowPath(data) {
      const map = this.map;
      let points = data.map((item) => {
        const geo = gcoord.transform(
          [item[0], item[1]],
          gcoord.WGS84,
          gcoord.EPSG3857
        );
        return { geo, item: { geo, ...item } };
      });
      /* 
        将坐标转换格式，结果为[{geo: [1, 2], item: {0: 1, 1: 2, geo }},{}]
      */
      // console.log(points, "points");
      // 定义线的样式（是否包含箭头）
      const arrowStyles = function (feature, res) {
        const geometry = feature.getGeometry();
        const styles = [
          new Style({
            stroke: new Stroke({
              color: "red",
              width: 6,
            }),
          }),
        ];
        // 如果像素高于某个范围，直接显示红线
        if (!(geometry instanceof LineString) || res > 10) return styles;
        // 计算箭头的间隔与旋转角度
        const length = geometry.getLength();
        const step = 60;
        const geoStep = step * res;
        const arrowNum = Math.floor(length / geoStep);
        const rotations = [];
        const distances = [0];
        geometry.forEachSegment(function (start, end) {
          const dx = end[0] - start[0];
          const dy = end[1] - start[1];
          const rotation = Math.atan2(dy, dx);
          distances.unshift(Math.sqrt(dx ** 2 + dy ** 2) + distances[0]);
          rotations.push(rotation);
        });
        // 添加箭头
        for (let i = 1; i < arrowNum; ++i) {
          const arrowCoord = geometry.getCoordinateAt(i / arrowNum);
          const d = i * geoStep;
          const grid = distances.findIndex((x) => x <= d);
          styles.push(
            new Style({
              geometry: new Point(arrowCoord),
              image: new Icon({
                src: mapImages.arrow,
                opacity: 1,
                anchor: [0.5, 0.5],
                rotateWithView: true,
                rotation: -rotations[distances.length - grid - 1] + Math.PI / 2,
                scale: 0.1,
              }),
            })
          );
        }
        return styles;
      };
      // 关键点
      const snapPointFeatures = data.map((item) => {
        var pointFeature = new Feature({
          geometry: new Point(
            gcoord.transform([item[0], item[1]], gcoord.WGS84, gcoord.EPSG3857)
          ),
          data: item,
        });
        pointFeature.setStyle((feature, resolution) => {
          const zoom = map.getView().getZoom();
          // console.log('🚀 ~ file: mapFunction.js ~ line 394 ~ pointFeature.setStyle ~ resolution', resolution)
          if (zoom < 12) {
            return null;
          } else {
            return [
              new Style({
                image: new Icon({
                  scale: 0.4,
                  anchor: [0.5, 1.15],
                  src: mapImages.camera,
                }),
              }),
              new Style({
                image: new CircleStyle({
                  radius: 5,
                  stroke: new Stroke({
                    color: "#169ef5",
                  }),
                  fill: new Fill({
                    color: "#169ef5",
                  }),
                }),
              }),
            ];
          }
        });
        return pointFeature;
      });
      // 关键点结束
      var layerLines = new VectorLayer({
        source: new VectorSource({
          features: [
            new Feature({
              geometry: new LineString(
                points.map((item) => item.geo),
                "XY"
              ),
              name: "Line",
            }),
            ...snapPointFeatures,
            // ...timePointFeatures
          ],
        }),
        style: arrowStyles,
      });
      layerLines.setZIndex(999999);
      // this.layerLines && this.map.removeLayer(this.layerLines);
      this.map.addLayer(layerLines);
      this.layerLines = layerLines;
    },
    // 添加船舶轨迹
    flowPath(data) {
      const map = this.map;
      let points = data.map((item) => {
        const geo = gcoord.transform(
          [item[0], item[1]],
          gcoord.WGS84,
          gcoord.EPSG3857
        );
        return { geo, item: { geo, ...item } };
      });
      /* 
        将坐标转换格式，结果为[{geo: [1, 2], item: {0: 1, 1: 2, geo }},{}]
      */
      // console.log(points, "points");
      // 定义线的样式（是否包含箭头）
      const arrowStyles = function (feature, res) {
        const geometry = feature.getGeometry();
        const styles = [
          new Style({
            stroke: new Stroke({
              color: "black",
              width: 1,
            }),
          }),
        ];
        // 如果像素高于某个范围，直接显示红线
        if (!(geometry instanceof LineString) || res > 10) return styles;
        // 计算箭头的间隔与旋转角度
        const length = geometry.getLength();
        const step = 60;
        const geoStep = step * res;
        const arrowNum = Math.floor(length / geoStep);
        const rotations = [];
        const distances = [0];
        geometry.forEachSegment(function (start, end) {
          const dx = end[0] - start[0];
          const dy = end[1] - start[1];
          const rotation = Math.atan2(dy, dx);
          distances.unshift(Math.sqrt(dx ** 2 + dy ** 2) + distances[0]);
          rotations.push(rotation);
        });
        // 添加箭头
        for (let i = 1; i < arrowNum; ++i) {
          const arrowCoord = geometry.getCoordinateAt(i / arrowNum);
          const d = i * geoStep;
          const grid = distances.findIndex((x) => x <= d);
          styles.push(
            new Style({
              geometry: new Point(arrowCoord),
              image: new Icon({
                src: mapImages.arrow,
                opacity: 1,
                anchor: [0.5, 0.5],
                rotateWithView: true,
                rotation: -rotations[distances.length - grid - 1] + Math.PI / 2,
                scale: 0.2,
              }),
            })
          );
        }
        return styles;
      };
      // 关键点
      const snapPointFeatures = data.map((item) => {
        var pointFeature = new Feature({
          geometry: new Point(
            gcoord.transform([item[0], item[1]], gcoord.WGS84, gcoord.EPSG3857)
          ),
          data: item,
        });
        pointFeature.setStyle((feature, resolution) => {
          const zoom = map.getView().getZoom();
          // console.log('🚀 ~ file: mapFunction.js ~ line 394 ~ pointFeature.setStyle ~ resolution', resolution)
          if (zoom < 12) {
            return null;
          } else {
            return [
              // new Style({
              //   image: new Icon({
              //     scale: 0.4,
              //     anchor: [0.5, 1.15],
              //     src: mapImages.camera,
              //   }),
              // }),
              new Style({
                image: new CircleStyle({
                  radius: 5,
                  stroke: new Stroke({
                    color: "#169ef5",
                  }),
                  fill: new Fill({
                    color: "white",
                  }),
                }),
              }),
            ];
          }
        });
        return pointFeature;
      });

      // 起始点
      var startPoint = new Feature({
        geometry: new Point(
          gcoord.transform(
            [data[0][0], data[0][1]],
            gcoord.WGS84,
            gcoord.EPSG3857
          )
        ),
      });
      startPoint.setStyle((feature, resolution) => {
        const zoom = map.getView().getZoom();
        // console.log('🚀 ~ file: mapFunction.js ~ line 394 ~ pointFeature.setStyle ~ resolution', resolution)
        if (zoom < 12) {
          return null;
        } else {
          return [
            // new Style({
            //   image: new Icon({
            //     scale: 0.4,
            //     anchor: [0.5, 1.15],
            //     src: mapImages.camera,
            //   }),
            // }),
            new Style({
              image: new CircleStyle({
                radius: 7,
                stroke: new Stroke({
                  color: "red",
                }),
                fill: new Fill({
                  color: "red",
                }),
              }),
            }),
          ];
        }
      });

      // 中止点
      var endPoint = new Feature({
        geometry: new Point(
          gcoord.transform(
            [data[data.length - 1][0], data[data.length - 1][1]],
            gcoord.WGS84,
            gcoord.EPSG3857
          )
        ),
      });
      endPoint.setStyle((feature, resolution) => {
        const zoom = map.getView().getZoom();
        // console.log('🚀 ~ file: mapFunction.js ~ line 394 ~ pointFeature.setStyle ~ resolution', resolution)
        if (zoom < 12) {
          return null;
        } else {
          return [
            // new Style({
            //   image: new Icon({
            //     scale: 0.4,
            //     anchor: [0.5, 1.15],
            //     src: mapImages.camera,
            //   }),
            // }),
            new Style({
              image: new CircleStyle({
                radius: 7,
                stroke: new Stroke({
                  color: "black",
                }),
                fill: new Fill({
                  color: "black",
                }),
              }),
            }),
          ];
        }
      });
      // 关键点结束
      var layerLines = new VectorLayer({
        source: new VectorSource({
          features: [
            new Feature({
              geometry: new LineString(
                points.map((item) => item.geo),
                "XY"
              ),
              name: "Line",
            }),
            ...snapPointFeatures,
            startPoint,
            endPoint,
            // ...timePointFeatures
          ],
        }),
        style: arrowStyles,
      });
      layerLines.setZIndex(999999);
      // this.layerLines && this.map.removeLayer(this.layerLines);
      this.map.addLayer(layerLines);
      this.layerLines = layerLines;
    },
    // 添加船舶时间
    riverPath(data) {
      const map = this.map;
      // 保存数据，之后添加文字使用
      const dataForTime = cloneDeep(data);
      // 转换坐标格式
      let points = data.map((item) => {
        const geo = gcoord.transform(
          [item.siteX, item.siteY],
          gcoord.WGS84,
          gcoord.EPSG3857
        );
        return { geo, item: { geo, ...item } };
      });

      // 添加关键点
      const snapPointFeatures = points.map((item) => {
        let pointFeature = new Feature({
          geometry: new Point([item.geo[0], item.geo[1]]),
          data: item,
        });
        pointFeature.setStyle((feature, resolution) => {
          const zoom = map.getView().getZoom();
          if (zoom < 12) {
            return null;
          } else {
            return [
              new Style({
                image: new CircleStyle({
                  radius: 5,
                  stroke: new Stroke({
                    color: "#169ef5",
                  }),
                  fill: new Fill({
                    color: "white",
                  }),
                }),
              }),
            ];
          }
        });
        return pointFeature;
      });
      // 定义线的样式（是否包含箭头）
      // const arrowStyles = function (feature, res) {
      //   const geometry = feature.getGeometry();
      //   const styles = [
      //     new Style({
      //       stroke: new Stroke({
      //         color: "black",
      //         width: 1,
      //       }),
      //     }),
      //   ];
      //   // 如果像素高于某个范围，直接显示红线
      //   if (!(geometry instanceof LineString) || res > 10) return styles;
      //   // 计算箭头的间隔与旋转角度
      //   const length = geometry.getLength();
      //   const step = 60;
      //   const geoStep = step * res;
      //   const arrowNum = Math.floor(length / geoStep);
      //   console.log(arrowNum, 'arrowNum');
      //   const rotations = [];
      //   const distances = [0];
      //   geometry.forEachSegment(function (start, end) {
      //     const dx = end[0] - start[0];
      //     const dy = end[1] - start[1];
      //     const rotation = Math.atan2(dy, dx);
      //     distances.unshift(Math.sqrt(dx ** 2 + dy ** 2) + distances[0]);
      //     rotations.push(rotation);
      //   });
      //   // 添加箭头
      //   for (let i = 0; i < arrowNum; ++i) {
      //     const arrowCoord = geometry.getCoordinateAt(i / arrowNum);
      //     const d = i * geoStep;
      //     const grid = distances.findIndex((x) => x <= d);
      //     styles.push(
      //       new Style({
      //         geometry: new Point(arrowCoord),
      //         image: new Icon({
      //           src: mapImages.arrow,
      //           opacity: 1,
      //           anchor: [0.5, 0.5],
      //           rotateWithView: true,
      //           rotation: -rotations[distances.length - grid - 1] + Math.PI / 2,
      //           scale: 0.2,
      //         }),
      //       })
      //     );
      //   }
      //   return styles;
      // };

      // 定义线的样式（只在每两个关键点的中心添加箭头）
      const arrowStyles = function (feature, res) {
        const geometry = feature.getGeometry();
        const styles = [
          new Style({
            stroke: new Stroke({
              color: "black",
              width: 1,
            }),
          }),
        ];
        // 如果像素高于某个范围，直接显示红线
        if (!(geometry instanceof LineString) || res > 10) return styles;
        // 添加箭头
        const coordinates = geometry.getCoordinates();
        for (let i = 0; i < coordinates.length - 1; ++i) {
          const start = coordinates[i];
          const end = coordinates[i + 1];
          const midPoint = [(start[0] + end[0]) / 2, (start[1] + end[1]) / 2];
          const dx = end[0] - start[0];
          const dy = end[1] - start[1];
          const rotation = Math.atan2(dy, dx);
          styles.push(
            new Style({
              geometry: new Point(midPoint),
              image: new Icon({
                src: mapImages.arrow,
                opacity: 1,
                anchor: [0.5, 0.5],
                rotateWithView: true,
                rotation: -rotation + Math.PI / 2,
                scale: 0.2,
              }),
            })
          );
        }
        return styles;
      };

      // const arrowStyles = function (feature, res) {
      //   const geometry = feature.getGeometry();
      //   const styles = [
      //     new Style({
      //       stroke: new Stroke({
      //         color: "black",
      //         width: 1,
      //       }),
      //     }),
      //   ];

      //   if (!(geometry instanceof LineString) || res > 10) return styles;

      //   // 计算箭头数量
      //   const keyPointNum = geometry.getCoordinates().length;
      //   const arrowNum = keyPointNum > 1 ? keyPointNum - 1 : 0;

      //   // 计算箭头的间隔与旋转角度
      //   const length = geometry.getLength();
      //   const step = 60;
      //   const geoStep = step * res;
      //   const rotations = [];
      //   const distances = [0];

      //   geometry.forEachSegment(function (start, end) {
      //     const dx = end[0] - start[0];
      //     const dy = end[1] - start[1];
      //     const rotation = Math.atan2(dy, dx);
      //     distances.unshift(Math.sqrt(dx ** 2 + dy ** 2) + distances[0]);
      //     rotations.push(rotation);
      //   });

      //   // 添加箭头
      //   for (let i = 0; i < arrowNum; ++i) {
      //     const arrowCoord = geometry.getCoordinateAt((i + 1) / (arrowNum + 1));
      //     const d = (i + 1) * (length / (arrowNum + 1));
      //     const grid = distances.findIndex((x) => x <= d);
      //     styles.push(
      //       new Style({
      //         geometry: new Point(arrowCoord),
      //         image: new Icon({
      //           src: mapImages.arrow,
      //           opacity: 1,
      //           anchor: [0.5, 0.5],
      //           rotateWithView: true,
      //           rotation: -rotations[distances.length - grid - 1] + Math.PI / 2,
      //           scale: 0.2,
      //         }),
      //       })
      //     );
      //   }

      //   return styles;
      // };
      // 关键点结束
      var layerLines = new VectorLayer({
        source: new VectorSource({
          features: [
            new Feature({
              geometry: new LineString(
                points.map((item) => item.geo),
                "XY"
              ),
              name: "Line",
            }),
            ...snapPointFeatures,
            // ...timePointFeatures
          ],
        }),
        style: arrowStyles,
      });

      const shipNameFeatures = this.getShipNameFeatures(dataForTime);
      const nameLayer = new VectorLayer({
        source: new VectorSource({
          features: shipNameFeatures,
        }),
        declutter: true,
      });

      // this.nameLayer && map.removeLayer(this.nameLayer);
      map.addLayer(nameLayer);
      this.nameLayer = nameLayer;

      layerLines.setZIndex(999999);
      // this.layerLines && this.map.removeLayer(this.layerLines);
      this.map.addLayer(layerLines);
      this.layerLines = layerLines;
    },
    getPixelRatio (context) {
  var backingStore =
    context.backingStorePixelRatio ||
    context.webkitBackingStorePixelRatio ||
    context.mozBackingStorePixelRatio ||
    context.msBackingStorePixelRatio ||
    context.oBackingStorePixelRatio ||
    context.backingStorePixelRatio ||
    1
  return (window.devicePixelRatio || 1) / backingStore
},
    // 添加船舶时间需要的函数
    getShipNameFeatures(data) {
      const map = this.map;
      return data.map((item) => {
        console.log(item, 'item');
        let pointFeature = new Feature({
          geometry: new Point(gcoord.transform([item.siteX, item.siteY], gcoord.WGS84, gcoord.EPSG3857)),
          data: item,
        });
        console.log(pointFeature, 'pointFeature');
        pointFeature.setStyle((feature, resolution) => {
          const name = String(item.time);
          const zoom = map.getView().getZoom();
          console.log('🚀 ~ file: mapFunctions.js ~ line 337 ~ name', name)
          if (zoom < 14) {
            return null;
          } else {
            //  const _offset = offset(20, (item.courseOverGround && (item.courseOverGround * Math.PI) / 180) || 0)
            // 创建一个空的canvas元素
            var canvas = document.createElement("canvas");
            // 获取绘图上下文对象
            var ctx = canvas.getContext("2d");
            // 获取设备像素比
            var ratio = this.getPixelRatio(ctx);
            // 根据船名计算标签的宽度
            // const cw = getStrFullLength(name) * 9
            const cw = ctx.measureText(name).width;
            // 设置标签的高度
            const ch = 20;
            /*
          根据标签的宽度、高度和设备像素比设置canvas的实际大小。其中加上15的额外长度是为了留出标签文字的空白边距
        */
            canvas.width = (cw + 15) * ratio;
            canvas.height = (ch + 15) * ratio;
            // 设置填充色和描边色为半透明白色
            ctx.fillStyle = "rgba(255,255,255,0.5)";
            ctx.strokeStyle = "rgba(255,255,255,0.5)";
            // 使用填充色填充整个canvas元素
            ctx.fillRect(0, 0, cw * ratio, ch * ratio);
            // 设置文字对齐中心对齐
            ctx.textAlign = "center";
            // 设置文字填充色为黑色
            ctx.fillStyle = "#000";
            // 设置文字的字体、字号和字体名称
            ctx.font = `${11 * ratio}px "Source Han Sans CN"`;
            // 在canvas元素的中心位置绘制名字文字
            ctx.fillText(name, (cw * ratio) / 2, (ch * ratio) / 2 + 4 * ratio);

            // 绘制黑线
            // 设置绘图起点
            ctx.beginPath();
            // 从起点处绘制一条线段，连接标签宽高两个顶点
            ctx.moveTo(cw * ratio, ch * ratio);
            ctx.lineTo(canvas.width, canvas.height);
            // 封闭绘图路径
            ctx.closePath();
            // 设置线条宽度和颜色
            ctx.lineWidth = 2;
            ctx.strokeStyle = "black";
            // 绘制边框
            ctx.stroke();
            return [
              new Style({
                image: new Icon({
                  img: canvas,
                  // 图标的缩放比例
                  scale: 1 / ratio,
                  imgSize: [canvas.width, canvas.height],
                  // 指定了图标的锚点，即图标相对于其所在位置的偏移量，这里是将锚点设置为图标的右下角
                  anchor: [1, 1],
                }),
              }),
            ];
          }
        });
        return pointFeature;
      });
    },
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
