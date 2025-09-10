<template>
  <div id="indoor-map-container" style="height: 600px; width: 100%;"></div>
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
</template>

<script lang="ts" setup>
import { onMounted, onUnmounted, ref, watch } from 'vue';
import 'leaflet/dist/leaflet.css';
// GeoJSONデータを直接インポート
import indoorMapGeoJson from '../../assets/data/indoor-map2.json';

// Leafletマップインスタンスを保持する変数
let mapInstance: L.Map | null = null;
let geoJsonLayer: L.GeoJSON | null = null;
// 現在表示するフロア番号
const currentFloor = ref(1); // デフォルトは1階

// GeoJSONデータから特定のフロアのフィーチャーをフィルタリングする関数
const getFeaturesByFloor = (floor: number) => {
  return {
    ...indoorMapGeoJson, // GeoJSONのメタデータを保持
    features: indoorMapGeoJson.features.filter(
      (feature) => feature.properties && feature.properties.floor === floor
    ),
  };
};


const updateMap = async (floor: number) => {
  if (!mapInstance) return;

  // 既存のGeoJSONレイヤーがあれば削除
  if (geoJsonLayer) {
    mapInstance.removeLayer(geoJsonLayer);
  }

  const L = await import('leaflet'); // 動的インポート

  const floorGeoJson = getFeaturesByFloor(floor);

  geoJsonLayer = L.geoJSON(floorGeoJson as any, {
    style: (feature) => {
      switch (feature?.properties.type) {
        case 'classroom':
        case 'laboratory':
        case 'meeting_room':
          return { color: '#4a90e2', weight: 1, fillColor: '#4a90e2', fillOpacity: 0.5 };
        case 'restroom_male':
        case 'restroom_female':
          return { color: '#e74c3c', weight: 1, fillColor: '#e74c3c', fillOpacity: 0.5 };
        case 'stairs':
        case 'elevator':
          return { color: '#f39c12', weight: 1, fillColor: '#f39c12', fillOpacity: 0.4 };
        case 'cafe':
          return { color: '#27ae60', weight: 1, fillColor: '#27ae60', fillOpacity: 0.5 };
        case 'corridor': // 廊下は線で、少し薄い色
          return { color: '#aaa', weight: 2, opacity: 0.6, fillOpacity: 0 };
        case 'floor_boundary': // フロア境界は見えないように
          return { weight: 0, opacity: 0 };
        default:
          return { color: '#333', weight: 1, fillColor: '#ccc', fillOpacity: 0.2 };
      }
    },
    onEachFeature: (feature, layer) => {
      if (feature.properties && feature.properties.name) {
        let popupContent = `<b>${feature.properties.name}</b>`;
        // その他のプロパティもポップアップに表示
        for (const key in feature.properties) {
          if (key !== 'id' && key !== 'name' && key !== 'type' && key !== 'floor') {
            popupContent += `<br>${key}: ${JSON.stringify(feature.properties[key])}`;
          }
        }
        layer.bindPopup(popupContent);
      }
    },
  }).addTo(mapInstance);

  // マップの境界をGeoJSONデータに合わせて調整
  // 現在のフロアのフィーチャーのみで境界を計算
  const currentFloorBounds = L.geoJSON(floorGeoJson as any).getBounds();
  if (currentFloorBounds.isValid()) { // 有効な境界がある場合のみfitBoundsを呼び出す
    mapInstance.fitBounds(currentFloorBounds);
  } else {
    // 境界がない（データがない）場合はデフォルトのビューに設定
    mapInstance.setView([0,0], 0);
  }
};

onMounted(async () => {
  // 動的にLeafletをインポートすることで、クライアントサイドでのみコードが実行されることを保証
  const L = await import('leaflet'); 

  mapInstance = L.map('indoor-map-container', {
    crs: L.CRS.Simple,
    minZoom: -2,
    maxZoom: 2,
    zoomControl: true,
    attributionControl: false,
  }).setView([0, 0], 0);

  // 初期ロード時にマップを更新
  await updateMap(currentFloor.value);
});

onUnmounted(() => {
  if (mapInstance) {
    mapInstance.remove();
  }
});

// フロアを変更するハンドラ関数
const setFloor = (floorNumber: number) => {
  currentFloor.value = floorNumber;
};

// currentFloorの値が変更されたらマップを更新
watch(currentFloor, (newFloor) => {
  updateMap(newFloor);
});
</script>