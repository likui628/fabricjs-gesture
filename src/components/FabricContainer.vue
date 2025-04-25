<template>
  <div @dragover.prevent @drop="handleDrop" @touchmove.prevent>
    <canvas ref="canvasRef" style="border: solid 1px #eee"></canvas>
    <slot></slot>
  </div>
</template>
<script lang="ts" setup>
import * as fabric from "fabric";
import { ref, onMounted } from "vue";

const canvasRef = ref<HTMLCanvasElement | null>(null);
let canvas: fabric.Canvas;

const minZoom = 0.1;
const maxZoom = 5;
const zoomIntensity = 0.1;

const handleDrop = (e: DragEvent) => {
  e.stopPropagation();

  const data = JSON.parse(e.dataTransfer!.getData("text/plain"));

  fabric.FabricImage.fromURL(data.url).then((img) => {
    let scale = 1;
    if (data.height) {
      scale = data.height / img.height;
    }

    const viewportPoint = canvas.getViewportPoint(e);

    const zoom = canvas.getZoom();
    const vpt = canvas.viewportTransform;
    const left = (viewportPoint.x - vpt[4]) / zoom - (img.width / 2) * scale;
    const top = (viewportPoint.y - vpt[5]) / zoom - (img.height / 2) * scale;

    img.set({
      dirty: true,
      scaleX: scale,
      scaleY: scale,
      centeredScaling: true,
      left,
      top,
    });

    canvas.add(img);
    canvas.renderAll();
  });
};

const handleTouchEnd = (e: TouchEvent) => {
  e.preventDefault();
  const dragData = sessionStorage.getItem("dragData");
  if (!dragData) return;


  try {
    const touch = e.changedTouches[0];
    
    // Check if touch point is within canvas viewport
    const point = new fabric.Point(touch.clientX, touch.clientY);
    const viewportPoint = point.transform(fabric.util.invertTransform(canvas.viewportTransform));
    
    // Check if the point is within canvas boundaries
    if (
      viewportPoint.x < 0 ||
      viewportPoint.x > canvas.width ||
      viewportPoint.y < 0 ||
      viewportPoint.y > canvas.height
    ) {
      return; // Touch point is outside canvas viewport, do nothing
    }

    const dropEvent = new DragEvent("drop", {
      dataTransfer: new DataTransfer(),
      clientX: touch.clientX,
      clientY: touch.clientY,
    });
    dropEvent.dataTransfer?.setData("text/plain", dragData);
    handleDrop(dropEvent);
  } catch (error) {
    console.error("Error handling touch drop:", error);
  } finally {
    sessionStorage.removeItem("dragData");
  }
};

onMounted(() => {
  canvas = new fabric.Canvas(canvasRef.value!, {
    width: 500,
    height: 500,
    fireRightClick: true,
    stopContextMenu: true,
  });

  document.addEventListener("touchend", handleTouchEnd);

  const canvasElement = document.querySelector(".upper-canvas") as HTMLCanvasElement;
  if (!canvasElement) {
    throw new Error("Canvas element not found");
  }
  let isPanning = false;
  let lastTouchPoint: { x: number; y: number } | null = null;
  let lastPinchDistance: number | null = null;
  let wasSelectionEnabled = false;

  // Helper function to calculate distance between two touch points
  function getTouchDistance(touches: TouchList): number {
    const dx = touches[0].clientX - touches[1].clientX;
    const dy = touches[0].clientY - touches[1].clientY;
    return Math.sqrt(dx * dx + dy * dy);
  }

  // Helper function to get center point between two touches
  function getTouchCenter(touches: TouchList): { x: number; y: number } {
    return {
      x: (touches[0].clientX + touches[1].clientX) / 2,
      y: (touches[0].clientY + touches[1].clientY) / 2,
    };
  }

  canvasElement.addEventListener("touchstart", (e: TouchEvent) => {
    if (e.touches.length === 2) {
      // Disable selection and object interactions during panning/zooming
      wasSelectionEnabled = canvas.selection || false;
      canvas.selection = false;
      canvas.forEachObject((obj) => {
        obj.selectable = false;
        obj.evented = false;
      });

      isPanning = true;
      lastTouchPoint = getTouchCenter(e.touches);
      lastPinchDistance = getTouchDistance(e.touches);

      e.preventDefault();
      e.stopPropagation();
    }
  });

  canvasElement.addEventListener("touchmove", (e: TouchEvent) => {
    if (isPanning && e.touches.length === 2) {
      const currentTouchPoint = getTouchCenter(e.touches);

      // Handle panning
      if (lastTouchPoint) {
        const deltaX = currentTouchPoint.x - lastTouchPoint.x;
        const deltaY = currentTouchPoint.y - lastTouchPoint.y;

        const delta = new fabric.Point(deltaX, deltaY);
        canvas.relativePan(delta);
      }

      // Handle pinch zooming
      if (lastPinchDistance !== null) {
        const currentDistance = getTouchDistance(e.touches);
        const pinchRatio = currentDistance / lastPinchDistance;
        if (Math.abs(pinchRatio - 1) > 0.01) {
          const zoom = canvas.getZoom();
          const newZoom = zoom * pinchRatio;

          if (newZoom >= minZoom && newZoom <= maxZoom) {
            const point = new fabric.Point(currentTouchPoint.x, currentTouchPoint.y);
            const viewportPoint = point.transform(
              fabric.util.invertTransform(canvas.viewportTransform)
            );
            canvas.zoomToPoint(viewportPoint, newZoom);
          }
        }

        lastPinchDistance = currentDistance;
      }

      lastTouchPoint = currentTouchPoint;

      e.preventDefault();
      e.stopPropagation();
    }
  });

  canvasElement.addEventListener("touchend", (e: TouchEvent) => {
    if (e.touches.length < 2) {
      if (isPanning) {
        // Re-enable selection and object interactions
        canvas.selection = wasSelectionEnabled;
        canvas.forEachObject((obj) => {
          obj.selectable = true;
          obj.evented = true;
        });
      }

      isPanning = false;
      lastTouchPoint = null;
      lastPinchDistance = null;
    }
  });

  canvas.on("mouse:wheel", function (this: fabric.Canvas, evt) {
    const { viewportPoint, e } = evt;

    e.preventDefault();
    e.stopPropagation();

    if (e.ctrlKey) {
      const delta = e.deltaY > 0 ? -1 : 1;

      const zoom = canvas.getZoom();
      const newZoom = zoom * (1 + delta * zoomIntensity);
      if (newZoom < minZoom || newZoom > maxZoom) return;

      canvas.zoomToPoint(viewportPoint, newZoom);
    } else {
      const delta = new fabric.Point(-e.deltaX, -e.deltaY);
      canvas.relativePan(delta);
    }
  });

  // Add example text
  const textValue = "fabric.js sandbox";
  const text = new fabric.Textbox(textValue, {
    originX: "center",
    splitByGrapheme: true,
    width: 200,
    top: 20,
    styles: fabric.util.stylesFromArray(
      [
        {
          style: {
            fontWeight: "bold",
            fontSize: 64,
          },
          start: 0,
          end: 9,
        },
      ],
      textValue
    ),
  });
  canvas.add(text);
  canvas.centerObjectH(text);
});
</script>
