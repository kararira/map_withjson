<template>
  <div id="indoor-map-container" style="height: 600px; width: 100%;"></div>
</template>

<script lang="ts" setup>
import { onMounted, onUnmounted, ref } from 'vue';
import 'leaflet/dist/leaflet.css';
// GeoJSONデータを直接インポート
import indoorMapGeoJson from '../../assets/data/indoor-map.json';

// Leafletマップインスタンスを保持する変数
let mapInstance: L.Map | null = null;

onMounted(async () => {
  // 動的にLeafletをインポートすることで、クライアントサイドでのみコードが実行されることを保証
  const L = await import('leaflet'); 

  mapInstance = L.map('indoor-map-container', {
    crs: L.CRS.Simple,
    minZoom: -2,
    maxZoom: 2,
    zoomControl: true,
    attributionControl: false
  }).setView([0, 0], 0);

  L.geoJSON(indoorMapGeoJson as any, {
    style: (feature) => {
      switch (feature?.properties.type) {
        case 'classroom':
          return { color: '#4a90e2', weight: 1, fillColor: '#4a90e2', fillOpacity: 0.5 };
        case 'restroom':
          return { color: '#e74c3c', weight: 1, fillColor: '#e74c3c', fillOpacity: 0.5 };
        default:
          return { color: '#333', weight: 1, fillColor: '#ccc', fillOpacity: 0.2 };
      }
    },
    onEachFeature: (feature, layer) => {
      if (feature.properties && feature.properties.name) {
        layer.bindPopup(`<b>${feature.properties.name}</b>`);
      }
    }
  }).addTo(mapInstance);

  const bounds = L.geoJSON(indoorMapGeoJson as any).getBounds();
  mapInstance.fitBounds(bounds);
});

onUnmounted(() => {
  if (mapInstance) {
    mapInstance.remove();
  }
});
</script>