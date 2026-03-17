<script>
  import TrustBar from "$lib/TrustBar.svelte";
  import ToolHeader from "$lib/ToolHeader.svelte";
  import ToolFooter from "$lib/ToolFooter.svelte";

  let canvas;
  let ctx;
  let image = $state(null);
  let imageEl = $state(null);
  let regions = $state([]);
  let drawing = $state(false);
  let startX = 0;
  let startY = 0;
  let currentRect = $state(null);
  let mode = $state("black");
  let blurRadius = $state(20);
  let copied = $state(false);
  let dragOver = $state(false);
  let canvasWidth = $state(0);
  let canvasHeight = $state(0);
  let scale = 1;

  function handleDrop(e) {
    e.preventDefault();
    dragOver = false;
    const file = e.dataTransfer.files[0];
    if (file && file.type.startsWith("image/")) loadImage(file);
  }

  function handleSelect(e) {
    const file = e.target.files[0];
    if (file) loadImage(file);
    e.target.value = "";
  }

  function handlePaste(e) {
    const items = e.clipboardData?.items;
    if (!items) return;
    for (const item of items) {
      if (item.type.startsWith("image/")) {
        loadImage(item.getAsFile());
        break;
      }
    }
  }

  function loadImage(file) {
    const url = URL.createObjectURL(file);
    const img = new Image();
    img.onload = () => {
      imageEl = img;
      const maxW = Math.min(720, window.innerWidth - 48);
      scale = img.width > maxW ? maxW / img.width : 1;
      canvasWidth = Math.round(img.width * scale);
      canvasHeight = Math.round(img.height * scale);
      regions = [];

      requestAnimationFrame(() => {
        if (!canvas) return;
        ctx = canvas.getContext("2d");
        render();
      });
    };
    img.src = url;
  }

  function getPos(e) {
    const rect = canvas.getBoundingClientRect();
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const clientY = e.touches ? e.touches[0].clientY : e.clientY;
    return {
      x: (clientX - rect.left) / scale,
      y: (clientY - rect.top) / scale,
    };
  }

  function onPointerDown(e) {
    e.preventDefault();
    const pos = getPos(e);
    startX = pos.x;
    startY = pos.y;
    drawing = true;
    currentRect = null;
  }

  function onPointerMove(e) {
    if (!drawing) return;
    e.preventDefault();
    const pos = getPos(e);
    currentRect = {
      x: Math.min(startX, pos.x),
      y: Math.min(startY, pos.y),
      w: Math.abs(pos.x - startX),
      h: Math.abs(pos.y - startY),
      mode: mode,
    };
    render();
  }

  function onPointerUp(e) {
    if (!drawing) return;
    drawing = false;
    if (currentRect && currentRect.w > 5 && currentRect.h > 5) {
      regions = [...regions, { ...currentRect }];
    }
    currentRect = null;
    render();
  }

  function render() {
    if (!ctx || !imageEl) return;
    canvas.width = imageEl.width;
    canvas.height = imageEl.height;
    ctx.drawImage(imageEl, 0, 0);

    const allRegions = currentRect ? [...regions, currentRect] : regions;

    for (const r of allRegions) {
      if (r.mode === "black") {
        ctx.fillStyle = "#000000";
        ctx.fillRect(r.x, r.y, r.w, r.h);
      } else {
        const tempCanvas = document.createElement("canvas");
        tempCanvas.width = r.w;
        tempCanvas.height = r.h;
        const tempCtx = tempCanvas.getContext("2d");
        tempCtx.drawImage(imageEl, r.x, r.y, r.w, r.h, 0, 0, r.w, r.h);
        tempCtx.filter = `blur(${blurRadius}px)`;
        tempCtx.drawImage(tempCanvas, 0, 0);
        ctx.drawImage(tempCanvas, r.x, r.y);
      }
    }
  }

  function undo() {
    if (regions.length === 0) return;
    regions = regions.slice(0, -1);
    render();
  }

  function clearAll() {
    regions = [];
    render();
  }

  function download() {
    if (!canvas) return;
    render();
    const a = document.createElement("a");
    a.href = canvas.toDataURL("image/png");
    a.download = "redacted.png";
    a.click();
  }

  function reset() {
    image = null;
    imageEl = null;
    regions = [];
    currentRect = null;
    canvasWidth = 0;
    canvasHeight = 0;
  }
</script>

<svelte:window on:paste={handlePaste} />

<TrustBar sourceUrl="https://github.com/Griffin2/localredact" />

<div class="page-shell">
  <ToolHeader
    title="Screenshot Redactor"
    description="Black out sensitive info in screenshots — nothing leaves your browser."
  />

  {#if !imageEl}
    <div
      class="drop-zone"
      class:drag-over={dragOver}
      ondragover={(e) => {
        e.preventDefault();
        dragOver = true;
      }}
      ondragleave={() => (dragOver = false)}
      ondrop={handleDrop}
      role="button"
      tabindex="0"
    >
      <div class="drop-content">
        <span class="drop-icon">🖼️</span>
        <p class="drop-text">
          Drop a screenshot, click to select, or paste from clipboard
        </p>
        <p class="drop-hint">
          Ctrl+V / Cmd+V works too — draw rectangles to redact
        </p>
        <label class="btn btn-primary drop-btn">
          Select Image
          <input type="file" accept="image/*" onchange={handleSelect} hidden />
        </label>
      </div>
    </div>
  {:else}
    <div class="controls">
      <div class="mode-toggle">
        <button
          class="btn"
          class:btn-active={mode === "black"}
          onclick={() => {
            mode = "black";
          }}>■ Black</button
        >
        <button
          class="btn"
          class:btn-active={mode === "blur"}
          onclick={() => {
            mode = "blur";
          }}>◌ Blur</button
        >
      </div>
      {#if mode === "blur"}
        <label class="blur-control">
          <span class="blur-label">Radius</span>
          <input
            type="range"
            min="5"
            max="50"
            bind:value={blurRadius}
            class="range-input"
          />
          <span class="blur-value">{blurRadius}</span>
        </label>
      {/if}
      <button class="btn" onclick={undo} disabled={regions.length === 0}
        >Undo</button
      >
      <button class="btn" onclick={clearAll} disabled={regions.length === 0}
        >Clear</button
      >
      <button class="btn btn-primary" onclick={download}>Download</button>
      <button class="btn btn-reset" onclick={reset}>New Image</button>
      <div class="spacer"></div>
      {#if regions.length > 0}
        <span class="pill pill-ok">
          <span class="pill-dot"></span>
          {regions.length} region{regions.length !== 1 ? "s" : ""}
        </span>
      {/if}
    </div>

    <div class="canvas-container">
      <canvas
        bind:this={canvas}
        style="width: {canvasWidth}px; height: {canvasHeight}px;"
        onmousedown={onPointerDown}
        onmousemove={onPointerMove}
        onmouseup={onPointerUp}
        ontouchstart={onPointerDown}
        ontouchmove={onPointerMove}
        ontouchend={onPointerUp}
      ></canvas>
    </div>
  {/if}

  <ToolFooter />
</div>

<style>
  .drop-zone {
    border: 2px dashed var(--border-light);
    border-radius: var(--radius-md);
    padding: 48px 24px;
    text-align: center;
    cursor: pointer;
    transition:
      border-color 0.15s,
      background 0.15s;
    background: var(--bg-surface);
  }

  .drop-zone:hover,
  .drag-over {
    border-color: var(--accent);
    background: var(--accent-bg);
  }

  .drop-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
  }

  .drop-icon {
    font-size: 32px;
  }

  .drop-text {
    font-size: 14px;
    color: var(--text-primary);
    margin: 0;
  }

  .drop-hint {
    font-family: var(--font-mono);
    font-size: 11px;
    color: var(--text-muted);
    margin: 0;
  }

  .drop-btn {
    margin-top: 8px;
    cursor: pointer;
  }

  .canvas-container {
    margin-top: 12px;
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    overflow: hidden;
    background: var(--bg-surface);
    display: flex;
    justify-content: center;
  }

  canvas {
    cursor: crosshair;
    display: block;
    max-width: 100%;
  }

  .mode-toggle {
    display: flex;
    gap: 0;
  }

  .mode-toggle .btn {
    border-radius: 0;
    font-size: 12px;
    padding: 8px 14px;
  }

  .mode-toggle .btn:first-child {
    border-radius: var(--radius-sm) 0 0 var(--radius-sm);
  }

  .mode-toggle .btn:last-child {
    border-radius: 0 var(--radius-sm) var(--radius-sm) 0;
    border-left: none;
  }

  .btn-active {
    background: var(--accent);
    color: white;
    border-color: var(--accent);
  }

  .blur-control {
    display: flex;
    align-items: center;
    gap: 6px;
    font-family: var(--font-mono);
    font-size: 11px;
    color: var(--text-muted);
  }

  .blur-label {
    white-space: nowrap;
  }

  .blur-value {
    width: 20px;
    text-align: right;
  }

  .range-input {
    width: 80px;
    accent-color: var(--accent);
    cursor: pointer;
  }

  .btn-reset {
    font-size: 12px;
    color: var(--text-muted);
  }

  .spacer {
    margin-left: auto;
  }

  @media (max-width: 640px) {
    .blur-control {
      display: none;
    }

    .controls {
      flex-wrap: wrap;
    }
  }
</style>
