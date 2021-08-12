<svelte:options immutable={true} />

<script>
  import { onMount, createEventDispatcher } from "svelte";
  import { pannable } from "./utils/pannable.js";
  import { readAsArrayBuffer } from "./utils/asyncReader.js";
  export let payload;
  export let file;
  export let width;
  export let height;
  export let x;
  export let y;
  export let degree;
  export let pageScale = 1;
  export let canvasWidth = 0;
  const dispatch = createEventDispatcher();
  let startX;
  let startY;
  let canvas;
  let operation = "";
  let directions = [];
  let dx = 0;
  let dy = 0;
  let dw = 0;
  let dh = 0;
  async function render() {
    // use canvas to prevent img tag's auto resize
    canvas.width = width;
    canvas.height = height;
    canvas.getContext("2d").drawImage(payload, 0, 0);
    let scale = 1;
    const limit = 500;
    if (width > limit) {
      scale = limit / width;
    }
    if (height > limit) {
      scale = Math.min(scale, limit / height);
    }
    dispatch("update", {
      width: width * scale,
      height: height * scale,
    });
    if (!["image/jpeg", "image/png"].includes(file.type)) {
      canvas.toBlob((blob) => {
        dispatch("update", {
          file: blob,
        });
      });
    }
  }
  function handlePanMove(event) {
    if (operation === "rotate") {
      let centerX = x + width / 2;
      return;
    }

    const _dx = (event.detail.x - startX) / pageScale;
    const _dy = (event.detail.y - startY) / pageScale;
    if (operation === "move") {
      dx = _dx;
      dy = _dy;
    } else if (operation === "scale") {
      if (directions.includes("left")) {
        dx = _dx;
        dw = -_dx;
      }
      if (directions.includes("top")) {
        dy = _dy;
        dh = -_dy;
      }
      if (directions.includes("right")) {
        dw = _dx;
      }
      if (directions.includes("bottom")) {
        dh = _dy;
      }
    }
  }

  function handlePanEnd(event) {
    if (operation === "move") {
      dispatch("update", {
        x: x + dx,
        y: y + dy,
      });
      dx = 0;
      dy = 0;
    } else if (operation === "scale") {
      dispatch("update", {
        x: x + dx,
        y: y + dy,
        width: width + dw,
        height: height + dh,
      });
      dx = 0;
      dy = 0;
      dw = 0;
      dh = 0;
      directions = [];
    }
    operation = "";
  }
  function handlePanStart(event) {
    startX = event.detail.x;
    startY = event.detail.y;
    if (event.detail.target === event.currentTarget) {
      return (operation = "move");
    }
    operation = "scale";
    directions = event.detail.target.dataset.direction.split("-");
  }

  function find_angle(p0x, p0y, p1x, p1y, cx, cy) {
    var p0c = Math.sqrt(Math.pow(cx - p0x, 2) + Math.pow(cy - p0y, 2)); // p0->c (b)
    var p1c = Math.sqrt(Math.pow(cx - p1x, 2) + Math.pow(cy - p1y, 2)); // p1->c (a)
    var p0p1 = Math.sqrt(Math.pow(p1x - p0x, 2) + Math.pow(p1y - p0y, 2)); // p0->p1 (c)
    return Math.acos((p1c * p1c + p0c * p0c - p0p1 * p0p1) / (2 * p1c * p0c));
  }

  window.addEventListener("mouseup", (e) => {
    console.log('end');
    if (operation === "rotate") {
      
      dispatch("rotate", {
        degree: degree,
      });
      operation = "";
    }
  });

  window.addEventListener("mousemove", (e) => {
    console.log('rotate');
    if (operation === "rotate") {
      
      let centerX = (x + dx) + width / 2;
      let centerY = (y + dy) + height / 2;
      let offsetX = (window.innerWidth - canvasWidth) / 2;

      var angleEnd = find_angle(
        centerX,
        -1000,
        e.clientX - offsetX,
        e.clientY - 48 - 20 - 20,
        centerX,
        centerY
      ) * (180 / Math.PI);
      degree = angleEnd;
      if ((e.clientX - offsetX) < centerX)
        degree *= -1;
    }
  });

  function handleRotateStart(event) {
    startX = event.clientX;
    startY = event.clientY;
    operation = "rotate";
    console.log('start');
  }
  
  function onDelete() {
    dispatch("delete");
  }
  onMount(render);
</script>

<div
  class="absolute left-0 top-0 select-none"
  style="width: {width + dw}px; height: {height +
    dh}px; transform: translate({x + dx}px, 
  {y + dy}px) rotate({degree}deg);
  "
>
  <div
    use:pannable
    on:panstart={handlePanStart}
    on:panmove={handlePanMove}
    on:panend={handlePanEnd}
    class="absolute w-full h-full cursor-grab"
    class:cursor-grabbing={operation === "move"}
    class:operation
  >
    <div
      data-direction="left"
      class="resize-border h-full w-1 left-0 top-0 border-l cursor-ew-resize"
    />
    <div
      data-direction="top"
      class="resize-border w-full h-1 left-0 top-0 border-t cursor-ns-resize"
    />
    <div
      data-direction="bottom"
      class="resize-border w-full h-1 left-0 bottom-0 border-b cursor-ns-resize"
    />
    <div
      data-direction="right"
      class="resize-border h-full w-1 right-0 top-0 border-r cursor-ew-resize"
    />
    <div
      data-direction="left-top"
      class="resize-corner left-0 top-0 cursor-nwse-resize transform
      -translate-x-1/2 -translate-y-1/2 md:scale-25"
    />
    <div
      data-direction="right-top"
      class="resize-corner right-0 top-0 cursor-nesw-resize transform
      translate-x-1/2 -translate-y-1/2 md:scale-25"
    />
    <div
      data-direction="left-bottom"
      class="resize-corner left-0 bottom-0 cursor-nesw-resize transform
      -translate-x-1/2 translate-y-1/2 md:scale-25"
    />
    <div
      data-direction="right-bottom"
      class="resize-corner right-0 bottom-0 cursor-nwse-resize transform
      translate-x-1/2 translate-y-1/2 md:scale-25"
    />
  </div>
  <div
    on:click={onDelete}
    class="absolute left-0 right-0 w-12 h-12 m-auto rounded-full bg-white
    cursor-pointer transform -translate-y-1/2 md:scale-25"
  >
    <img class="w-full h-full" src="/delete.svg" alt="delete object" />
  </div>
  <div
    on:mousedown={handleRotateStart}
    style="top:-30px"
    class="absolute left-0 top-0 right-0 w-12 h-12 m-auto bg-white 
    cursor-pointer transform -translate-y-1/2 md:scale-25"
    class:cursor-grabbing={operation === "move"}
    class:operation
  >
    <img
      class="w-full h-full undraggable"
      src="/rotate.svg"
      alt="rotate object"
    />
  </div>
  <canvas class="w-full h-full" bind:this={canvas} />
</div>

<style>
  .operation {
    background-color: rgba(0, 0, 0, 0.3);
  }
  .resize-border {
    @apply absolute border-dashed border-gray-600;
  }
  .resize-corner {
    @apply absolute w-10 h-10 bg-blue-300 rounded-full;
  }

  .undraggable {
    user-drag: none;
    -webkit-user-drag: none;
    user-select: none;
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
  }
</style>
