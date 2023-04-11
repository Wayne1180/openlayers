<template>
  <div id="app">
    <Map :mapData="mapData" :lineData="lineData" :mm="jumpToMv"></Map>
  </div>
</template>

<script>
import Map from './components/Map.vue';
export default {
  name: 'App',
  components: {
    Map
  },
  data() {
    return {
      mapData: [
        {
          name: '夫子庙平江府店',
          phone: '(025)68216777',
          siteX: 118.799052,
          siteY: 32.027263
        },
        {
          name: '南京中山陵店',
          phone: '(025)52335777',
          siteX: 118.842222,
          siteY: 32.046835
        },
        {
          name: '1912总统府店',
          phone: '(025)86272077',
          siteX: 118.802462,
          siteY: 32.051314
        }
      ],
      lineData: [
        [118.78834303820417, 31.919733629663907],
        [118.7862777372627, 31.919697202804955],
        [118.78610071146772, 31.928771598184594],
        [118.78464695418165, 31.93229092951254],
        [118.78779586756514, 31.932869125010087]
      ],
      jumpToMv: [118.79092139933393, 32.04107783039368]
    }
  },
}
</script>

<style lang="scss">
body {
  margin: 0;
}
</style>
