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

// Leafletの型をインポート
import type { Map, GeoJSON } from 'leaflet';

// GeoJSONデータを保持するリアクティブな変数
const indoorMapGeoJson = ref<any>(null);

let mapInstance: Map | null = null;
let geoJsonLayer: GeoJSON | null = null;

// 現在表示するフロア番号 (GeoJSONデータに合わせてデフォルトを3階に変更)
const currentFloor = ref(3);

// 特定のフロアのフィーチャーをフィルタリングする関数
const getFeaturesByFloor = (floor: number) => {
  if (!indoorMapGeoJson.value) return null;
  
  return {
    ...indoorMapGeoJson.value,
    features: indoorMapGeoJson.value.features.filter(
      // feature.properties.floorが文字列"3"、floorが数値3でも比較できるようString()で統一
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

  // 表示するデータがない場合は何もしない
  if (!floorGeoJson || floorGeoJson.features.length === 0) {
    console.warn(`No features found for floor ${floor}`);
    return;
  }

  geoJsonLayer = L.geoJSON(floorGeoJson as any, {
    style: (feature) => {
      // スタイル定義は変更なし
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
  try {
    // 1. publicディレクトリからGeoJSONをfetchで読み込む
    const response = await fetch('/auto_generated.geojson');
    if (!response.ok) throw new Error('Failed to fetch GeoJSON');
    indoorMapGeoJson.value = await response.json();
    console.log("GeoJSON data loaded successfully.");

    // 2. Leafletを動的にインポート
    const L = await import('leaflet');
    
    // 3. マップを初期化
    mapInstance = L.map('indoor-map-container', {
      crs: L.CRS.Simple, // 非地理座標系を使用
      minZoom: -2,
    }).setView([0, 0], 0);

    // 4. 初期フロアのマップを表示
    await updateMap(currentFloor.value);

  } catch (error) {
    console.error("Failed to initialize map:", error);
  }
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