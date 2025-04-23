<template>
  <div>
    <canvas ref="canvasRef"></canvas>
  </div>
</template>

<script lang="ts" setup>
import * as fabric from 'fabric';
import { ref, onMounted } from 'vue';

const canvasRef = ref<HTMLCanvasElement | null>(null);

const minZoom = 0.1
const maxZoom = 5
const zoomIntensity = 0.1

onMounted(() => {
  const canvas = new fabric.Canvas(canvasRef.value, {
    width: 500,
    height: 500,
  });

  canvas.on('mouse:wheel', function (this: fabric.Canvas, evt) {
    const { viewportPoint, e } = evt

    e.preventDefault()
    e.stopPropagation()

    if (e.ctrlKey) {
      const delta = e.deltaY > 0 ? -1 : 1

      const zoom = canvas.getZoom()
      const newZoom = zoom * (1 + delta * zoomIntensity)
      if (newZoom < minZoom || newZoom > maxZoom) return

      canvas.zoomToPoint(viewportPoint, newZoom)
    } else {
      const vpt = this.viewportTransform
      vpt[4] -= e.deltaX
      vpt[5] -= e.deltaY
      this.requestRenderAll()
    }
  })

  const textValue = 'fabric.js sandbox';
  const text = new fabric.Textbox(textValue, {
    originX: 'center',
    splitByGrapheme: true,
    width: 200,
    top: 20,
    styles: fabric.util.stylesFromArray(
      [
        {
          style: {
            fontWeight: 'bold',
            fontSize: 64,
          },
          start: 0,
          end: 9,
        },
      ],
      textValue,
    ),
  });
  canvas.add(text);
  canvas.centerObjectH(text);
})
</script>
