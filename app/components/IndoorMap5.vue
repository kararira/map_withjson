<template>
  <div>
    <div id="indoor-map-container" style="height: 600px; width: 100%;"></div>
    
    <div class="mt-4 space-x-2">
      <button
        @click="setFloor('Frame 52')"
        :class="{
            'bg-blue-500 text-white': currentFloor === 'Frame 52',
            'bg-gray-200 text-gray-800': currentFloor !== 'Frame 52'
        }"
        class="px-4 py-2 rounded-md font-bold transition-colors duration-200"
        >
        1F
      </button>
      <button
        @click="setFloor('Frame 53')"
        :class="{
            'bg-blue-500 text-white': currentFloor === 'Frame 53',
            'bg-gray-200 text-gray-800': currentFloor !== 'Frame 53'
        }"
        class="px-4 py-2 rounded-md font-bold transition-colors duration-200"
        >
        2F
      </button>
      <button
        @click="setFloor('Frame 54')"
        :class="{
            'bg-blue-500 text-white': currentFloor === 'Frame 54',
            'bg-gray-200 text-gray-800': currentFloor !== 'Frame 54'
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
import indoorMapGeoJson from '../../assets/data/test_20250922_2.json';

let mapInstance: Map | null = null;
let geoJsonLayer: GeoJSON | null = null;

// GeoJSONデータに合わせてデフォルトを3階に設定
const currentFloor = ref("Frame 52");

const getFeaturesByFloor = (floor: string) => {
  return {
    ...indoorMapGeoJson,
    features: indoorMapGeoJson.features.filter(
      (feature: any) => feature.properties && String(feature.properties.floor) === String(floor)
    ),
  };
};

const updateMap = async (floor: string) => {
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

  //const currentFloorBounds = geoJsonLayer.getBounds();
  //if (currentFloorBounds.isValid()) {
    //mapInstance.fitBounds(currentFloorBounds, { padding: [50, 50] });
  //}
};

onMounted(async () => {
  // fetch処理を削除し、直接マップ初期化
  const L = await import('leaflet');
  
  mapInstance = L.map('indoor-map-container', {
    crs: L.CRS.Simple,
    minZoom: -2,
  });//.setView([0, 0], 0)

  // ★★★ 変更点: 全フロアのデータを使って初期表示範囲を設定 ★★★
  const allFeaturesLayer = L.geoJSON(indoorMapGeoJson as any);
  const allBounds = allFeaturesLayer.getBounds();
  if (allBounds.isValid()) {
    mapInstance.fitBounds(allBounds, { padding: [50, 50] });
  }


  // インポートしたデータを使って初期フロアのマップを表示
  await updateMap(currentFloor.value);
});

onUnmounted(() => {
  if (mapInstance) {
    mapInstance.remove();
  }
});

const setFloor = (floorNumber: string) => {
  currentFloor.value = floorNumber;
};

watch(currentFloor, (newFloor) => {
  updateMap(newFloor);
});
</script>