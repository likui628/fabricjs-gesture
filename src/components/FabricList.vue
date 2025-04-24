<template>
  <div class="sidebar">
    <div class="card" v-for="(item, index) of list" :key="index">
      <img :src="item.url" draggable="true" :data-height="item.renderHeight"
      @dragstart="handleDragStart" @dragend="handleDragEnd" @touchstart="handleTouchStart"
      @touchend="handleTouchEnd"
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref } from "vue";

const list = ref([
  {
    url: "/model.png",
    renderHeight: 384,
  },
  {
    url: "/top.png",
    renderHeight: 120,
  },
  {
    url: "/skirt.png",
    renderHeight: 100,
  },
  {
    url: "/shoes.png",
    renderHeight: 90,
  },
]);

let touchStartTime: number | null = null;
const LONG_PRESS_DURATION = 300;

const handleTouchStart = (e: TouchEvent) => {
  const target = e.target as HTMLImageElement | null;
  if (!target) return;

  touchStartTime = Date.now();

  setTimeout(() => {
    if (touchStartTime && Date.now() - touchStartTime >= LONG_PRESS_DURATION) {
      target.classList.add("img_dragging");

      const dragData = JSON.stringify({
        height: target.dataset.height,
        url: target.src,
      });
      sessionStorage.setItem("dragData", dragData);
    }
  }, LONG_PRESS_DURATION);
};

const handleTouchEnd = (e: TouchEvent) => {
  const target = e.target as HTMLImageElement | null;
  if (!target) return;

  touchStartTime = null;
  target.classList.remove("img_dragging");
};

const handleDragStart = (e: DragEvent) => {
  if (!e.target || !(e.target instanceof HTMLImageElement)) return;
  e.target.classList.add("img_dragging");
  e.dataTransfer?.setData(
    "text/plain",
    JSON.stringify({
      height: e.target.dataset.height,
      url: e.target.src,
    })
  );
};

const handleDragEnd = (e: DragEvent) => {
  if (!e.target || !(e.target instanceof HTMLImageElement)) return;
  e.target.classList.remove("img_dragging");
};
</script>

<style scoped>
.sidebar {
  height: 120px;
  width: 500px;
  display: flex;
  overflow-x: auto;
  overflow-y: hidden;
  gap: 10px;
  padding: 10px;
}

.sidebar .card {
  flex-shrink: 0;
  width: 100px;
  height: 100px;
  border: solid 1px #ccc;
  border-radius: 20px;
  padding: 5px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.sidebar img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.sidebar img.img_dragging {
  opacity: 0.4;
}
</style>
