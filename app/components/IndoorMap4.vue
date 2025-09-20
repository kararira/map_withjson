<template>
  <div>
    <div id="indoor-map-container" style="height: 600px; width: 100%;"></div>
    
    <div class="mt-4 space-x-2">
      <button
        @click="setFloor(1)"
        :class="{
            'bg-blue-500 text-white': currentFloor === 1,
            'bg-gray-200 text-gray-800': currentFloor !== 1
        }"
        class="px-4 py-2 rounded-md font-bold transition-colors duration-200"
        >
        1F
      </button>
      <button
        @click="setFloor(2)"
        :class="{
            'bg-blue-500 text-white': currentFloor === 2,
            'bg-gray-200 text-gray-800': currentFloor !== 2
        }"
        class="px-4 py-2 rounded-md font-bold transition-colors duration-200"
        >
        2F
      </button>
      <button
        @click="setFloor(3)"
        :class="{
            'bg-blue-500 text-white': currentFloor === 3,
            'bg-gray-200 text-gray-800': currentFloor !== 3
        }"
        class="px-4 py-2 rounded-md font-bold transition-colors duration-200"
        >
        3F
      </button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { onMounted, onUnmounted, ref, watch } from 'vue';
import 'leaflet/dist/leaflet.css';
import type { Map, GeoJSON } from 'leaflet';

// データを直接インポート (ファイルパスと拡張子を.jsonに変更)
import indoorMapGeoJson from '../../assets/data/auto_generated2.json';

let mapInstance: Map | null = null;
let geoJsonLayer: GeoJSON | null = null;

// GeoJSONデータに合わせてデフォルトを3階に設定
const currentFloor = ref(3);

const getFeaturesByFloor = (floor: number) => {
  return {
    ...indoorMapGeoJson,
    features: indoorMapGeoJson.features.filter(
      (feature: any) => feature.properties && String(feature.properties.floor) === String(floor)
    ),
  };
};

const updateMap = async (floor: number) => {
  if (!mapInstance) return;

  if (geoJsonLayer) {
    mapInstance.removeLayer(geoJsonLayer);
  }

  const L = await import('leaflet');
  const floorGeoJson = getFeaturesByFloor(floor);

  if (!floorGeoJson || floorGeoJson.features.length === 0) {
    return;
  }

  geoJsonLayer = L.geoJSON(floorGeoJson as any, {
    style: (feature) => {
      switch (feature?.properties.category) {
        case '施設':
          return { color: '#4a90e2', weight: 2, fillColor: '#4a90e2', fillOpacity: 0.5 };
        default:
          return { color: '#333', weight: 1, fillColor: '#ccc', fillOpacity: 0.2 };
      }
    },
    onEachFeature: (feature, layer) => {
      if (feature.properties && feature.properties.name) {
        let popupContent = `<b>${feature.properties.name}</b><br>カテゴリ: ${feature.properties.category}`;
        layer.bindPopup(popupContent);
      }
    },
  }).addTo(mapInstance);

  const currentFloorBounds = geoJsonLayer.getBounds();
  if (currentFloorBounds.isValid()) {
    mapInstance.fitBounds(currentFloorBounds, { padding: [50, 50] });
  }
};

onMounted(async () => {
  // fetch処理を削除し、直接マップ初期化
  const L = await import('leaflet');
  
  mapInstance = L.map('indoor-map-container', {
    crs: L.CRS.Simple,
    minZoom: -2,
  }).setView([0, 0], 0);

  // インポートしたデータを使って初期フロアのマップを表示
  await updateMap(currentFloor.value);
});

onUnmounted(() => {
  if (mapInstance) {
    mapInstance.remove();
  }
});

const setFloor = (floorNumber: number) => {
  currentFloor.value = floorNumber;
};

watch(currentFloor, (newFloor) => {
  updateMap(newFloor);
});
</script>