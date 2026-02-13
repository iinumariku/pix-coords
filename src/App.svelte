<script lang="ts">
  import * as UTIF from "utif2";
  import * as XLSX from "xlsx";

  interface CoordPoint {
    id: number;
    x: number;
    y: number;
    color: string;
    name: string;
  }

  let nextId = 0;
  let imageLoaded = $state(false);
  let coordList = $state<CoordPoint[]>([]);
  let panelOpen = $state(true);
  let imageSrc = $state("");
  let imageWidth = $state(0);
  let imageHeight = $state(0);
  let mouseX = $state(0);
  let mouseY = $state(0);
  let pixelX = $state(0);
  let pixelY = $state(0);
  let pixelColor = $state({ r: 0, g: 0, b: 0, a: 255 });
  let isHovering = $state(false);
  let isDraggingOver = $state(false);
  let toastVisible = $state(false);
  let toastMessage = $state("");
  let zoom = $state(1);
  let panX = $state(0);
  let panY = $state(0);
  let isPanning = $state(false);
  let panStartX = 0;
  let panStartY = 0;
  let panStartPanX = 0;
  let panStartPanY = 0;

  let canvasEl: HTMLCanvasElement;
  let containerEl: HTMLDivElement;
  let magnifierCanvasEl: HTMLCanvasElement;
  let imgEl: HTMLImageElement | null = null;
  let fileInputEl: HTMLInputElement;

  const MAGNIFIER_SIZE = 140;
  const MAGNIFIER_ZOOM = 8;

  function isTiff(file: File): boolean {
    const name = file.name.toLowerCase();
    return (
      name.endsWith(".tif") ||
      name.endsWith(".tiff") ||
      file.type === "image/tiff"
    );
  }

  function loadImageFromElement(img: HTMLImageElement) {
    imgEl = img;
    imageWidth = img.naturalWidth;
    imageHeight = img.naturalHeight;
    imageSrc = img.src;
    imageLoaded = true;
    zoom = 1;
    panX = 0;
    panY = 0;
    requestAnimationFrame(() => drawCanvas());
  }

  function loadTiff(file: File) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const buf = e.target?.result as ArrayBuffer;
      const ifds = UTIF.decode(buf);
      if (ifds.length === 0) return;
      UTIF.decodeImage(buf, ifds[0]);
      const rgba = UTIF.toRGBA8(ifds[0]);
      const w = ifds[0].width;
      const h = ifds[0].height;

      const tempCanvas = document.createElement("canvas");
      tempCanvas.width = w;
      tempCanvas.height = h;
      const ctx = tempCanvas.getContext("2d")!;
      const imgData = ctx.createImageData(w, h);
      imgData.data.set(new Uint8ClampedArray(rgba.buffer));
      ctx.putImageData(imgData, 0, 0);

      const img = new Image();
      img.onload = () => loadImageFromElement(img);
      img.src = tempCanvas.toDataURL();
    };
    reader.readAsArrayBuffer(file);
  }

  function loadImage(file: File) {
    if (isTiff(file)) {
      loadTiff(file);
      return;
    }
    if (!file.type.startsWith("image/")) return;
    const reader = new FileReader();
    reader.onload = (e) => {
      const img = new Image();
      img.onload = () => loadImageFromElement(img);
      img.src = e.target?.result as string;
    };
    reader.readAsDataURL(file);
  }

  function drawCanvas() {
    if (!canvasEl || !imgEl) return;
    const ctx = canvasEl.getContext("2d");
    if (!ctx) return;

    const container = containerEl;
    if (!container) return;

    canvasEl.width = container.clientWidth;
    canvasEl.height = container.clientHeight;

    ctx.clearRect(0, 0, canvasEl.width, canvasEl.height);

    // Draw checkerboard background
    const tileSize = 16;
    for (let y = 0; y < canvasEl.height; y += tileSize) {
      for (let x = 0; x < canvasEl.width; x += tileSize) {
        const isLight = (x / tileSize + y / tileSize) % 2 === 0;
        ctx.fillStyle = isLight ? "#1e1e2a" : "#16161f";
        ctx.fillRect(x, y, tileSize, tileSize);
      }
    }

    // Calculate image placement (centered)
    const scale =
      Math.min(canvasEl.width / imageWidth, canvasEl.height / imageHeight) *
      zoom;

    const drawW = imageWidth * scale;
    const drawH = imageHeight * scale;
    const offsetX = (canvasEl.width - drawW) / 2 + panX;
    const offsetY = (canvasEl.height - drawH) / 2 + panY;

    ctx.imageSmoothingEnabled = zoom < 4;
    ctx.drawImage(imgEl, offsetX, offsetY, drawW, drawH);
  }

  function getImageCoords(
    clientX: number,
    clientY: number,
  ): { ix: number; iy: number } | null {
    if (!canvasEl || !imgEl) return null;
    const rect = canvasEl.getBoundingClientRect();
    const cx = clientX - rect.left;
    const cy = clientY - rect.top;

    const scale =
      Math.min(canvasEl.width / imageWidth, canvasEl.height / imageHeight) *
      zoom;

    const drawW = imageWidth * scale;
    const drawH = imageHeight * scale;
    const offsetX = (canvasEl.width - drawW) / 2 + panX;
    const offsetY = (canvasEl.height - drawH) / 2 + panY;

    const ix = Math.floor((cx - offsetX) / scale);
    const iy = Math.floor((cy - offsetY) / scale);

    if (ix < 0 || iy < 0 || ix >= imageWidth || iy >= imageHeight) return null;
    return { ix, iy };
  }

  function getPixelColor(x: number, y: number) {
    if (!imgEl) return { r: 0, g: 0, b: 0, a: 255 };
    const tempCanvas = document.createElement("canvas");
    tempCanvas.width = imageWidth;
    tempCanvas.height = imageHeight;
    const ctx = tempCanvas.getContext("2d")!;
    ctx.drawImage(imgEl, 0, 0);
    const data = ctx.getImageData(x, y, 1, 1).data;
    return { r: data[0], g: data[1], b: data[2], a: data[3] };
  }

  function drawMagnifier(ix: number, iy: number) {
    if (!magnifierCanvasEl || !imgEl) return;
    const ctx = magnifierCanvasEl.getContext("2d");
    if (!ctx) return;

    const size = MAGNIFIER_SIZE;
    magnifierCanvasEl.width = size;
    magnifierCanvasEl.height = size;

    ctx.clearRect(0, 0, size, size);

    // Draw checkerboard background for magnifier
    const tileSize = MAGNIFIER_ZOOM;
    for (let y = 0; y < size; y += tileSize) {
      for (let x = 0; x < size; x += tileSize) {
        const isLight = (x / tileSize + y / tileSize) % 2 === 0;
        ctx.fillStyle = isLight ? "#2a2a3a" : "#1e1e2a";
        ctx.fillRect(x, y, tileSize, tileSize);
      }
    }

    ctx.imageSmoothingEnabled = false;
    const srcSize = Math.floor(size / MAGNIFIER_ZOOM);
    const srcX = ix - Math.floor(srcSize / 2);
    const srcY = iy - Math.floor(srcSize / 2);
    ctx.drawImage(imgEl, srcX, srcY, srcSize, srcSize, 0, 0, size, size);

    // Draw grid lines
    ctx.strokeStyle = "rgba(255,255,255,0.08)";
    ctx.lineWidth = 0.5;
    for (let i = 0; i <= srcSize; i++) {
      const pos = i * MAGNIFIER_ZOOM;
      ctx.beginPath();
      ctx.moveTo(pos, 0);
      ctx.lineTo(pos, size);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(0, pos);
      ctx.lineTo(size, pos);
      ctx.stroke();
    }

    // Highlight center pixel
    const centerStart = Math.floor(srcSize / 2) * MAGNIFIER_ZOOM;
    ctx.strokeStyle = "rgba(108, 99, 255, 0.8)";
    ctx.lineWidth = 2;
    ctx.strokeRect(
      centerStart + 1,
      centerStart + 1,
      MAGNIFIER_ZOOM - 2,
      MAGNIFIER_ZOOM - 2,
    );
  }

  function drawCrosshair(clientX: number, clientY: number) {
    if (!canvasEl || !imgEl) return;
    drawCanvas();
    const ctx = canvasEl.getContext("2d");
    if (!ctx) return;

    const rect = canvasEl.getBoundingClientRect();
    const cx = clientX - rect.left;
    const cy = clientY - rect.top;

    ctx.setLineDash([6, 4]);
    ctx.strokeStyle = "var(--crosshair-color)";
    ctx.strokeStyle = "rgba(108, 99, 255, 0.5)";
    ctx.lineWidth = 1;

    ctx.beginPath();
    ctx.moveTo(cx, 0);
    ctx.lineTo(cx, canvasEl.height);
    ctx.stroke();

    ctx.beginPath();
    ctx.moveTo(0, cy);
    ctx.lineTo(canvasEl.width, cy);
    ctx.stroke();

    ctx.setLineDash([]);
  }

  function handleMouseMove(e: MouseEvent) {
    if (isPanning) {
      panX = panStartPanX + (e.clientX - panStartX);
      panY = panStartPanY + (e.clientY - panStartY);
      drawCanvas();
      return;
    }
    const coords = getImageCoords(e.clientX, e.clientY);
    if (coords) {
      isHovering = true;
      mouseX = e.clientX;
      mouseY = e.clientY;
      pixelX = coords.ix;
      pixelY = coords.iy;
      pixelColor = getPixelColor(coords.ix, coords.iy);
      drawCrosshair(e.clientX, e.clientY);
      drawMagnifier(coords.ix, coords.iy);
    } else {
      isHovering = false;
      drawCanvas();
    }
  }

  function handleMouseDown(e: MouseEvent) {
    if (e.button === 1 || (e.button === 0 && e.altKey)) {
      // Middle click or Alt+click to pan
      isPanning = true;
      panStartX = e.clientX;
      panStartY = e.clientY;
      panStartPanX = panX;
      panStartPanY = panY;
      e.preventDefault();
    }
  }

  function handleMouseUp(e: MouseEvent) {
    if (isPanning) {
      isPanning = false;
      return;
    }
  }

  function handleClick(e: MouseEvent) {
    if (!imageLoaded || e.altKey) return;
    const coords = getImageCoords(e.clientX, e.clientY);
    if (coords) {
      const color = getPixelColor(coords.ix, coords.iy);
      const hex = rgbToHex(color.r, color.g, color.b);
      coordList = [
        ...coordList,
        { id: nextId++, x: coords.ix, y: coords.iy, color: hex, name: "" },
      ];
      showToast(`Added (${coords.ix}, ${coords.iy})`);
    }
  }

  function removePoint(id: number) {
    coordList = coordList.filter((p) => p.id !== id);
  }

  function clearList() {
    coordList = [];
  }

  function exportCSV() {
    if (coordList.length === 0) return;
    const header = "#,Name,X,Y,Color";
    const rows = coordList.map((p, i) => {
      const name = p.name.includes(",") ? `"${p.name}"` : p.name;
      return `${i + 1},${name},${p.x},${p.y},${p.color}`;
    });
    const csv = [header, ...rows].join("\n");
    downloadFile(csv, "coordinates.csv", "text/csv");
  }

  function exportXLSX() {
    if (coordList.length === 0) return;
    const data = coordList.map((p, i) => ({
      "#": i + 1,
      Name: p.name,
      X: p.x,
      Y: p.y,
      Color: p.color,
    }));
    const ws = XLSX.utils.json_to_sheet(data);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, "Coordinates");
    XLSX.writeFile(wb, "coordinates.xlsx");
  }

  function downloadFile(content: string, filename: string, type: string) {
    const blob = new Blob([content], { type });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
  }

  function handleWheel(e: WheelEvent) {
    if (!imageLoaded) return;
    e.preventDefault();
    const delta = e.deltaY > 0 ? 0.9 : 1.1;
    const newZoom = Math.min(Math.max(zoom * delta, 0.1), 50);

    // Zoom towards mouse cursor position
    const rect = canvasEl.getBoundingClientRect();
    const cx = e.clientX - rect.left;
    const cy = e.clientY - rect.top;

    const ratio = newZoom / zoom;
    panX = ratio * panX + (cx - canvasEl.width / 2) * (1 - ratio);
    panY = ratio * panY + (cy - canvasEl.height / 2) * (1 - ratio);

    zoom = newZoom;
    drawCanvas();

    // Update coords
    const coords = getImageCoords(e.clientX, e.clientY);
    if (coords) {
      pixelX = coords.ix;
      pixelY = coords.iy;
      pixelColor = getPixelColor(coords.ix, coords.iy);
      drawCrosshair(e.clientX, e.clientY);
      drawMagnifier(coords.ix, coords.iy);
    }
  }

  function handleMouseLeave() {
    isHovering = false;
    if (imageLoaded) drawCanvas();
  }

  function showToast(msg: string) {
    toastMessage = msg;
    toastVisible = true;
    setTimeout(() => {
      toastVisible = false;
    }, 1500);
  }

  function handleDrop(e: DragEvent) {
    e.preventDefault();
    isDraggingOver = false;
    const file = e.dataTransfer?.files[0];
    if (file) loadImage(file);
  }

  function handleDragOver(e: DragEvent) {
    e.preventDefault();
    isDraggingOver = true;
  }

  function handleDragLeave() {
    isDraggingOver = false;
  }

  function handleFileSelect(e: Event) {
    const input = e.target as HTMLInputElement;
    const file = input.files?.[0];
    if (file) loadImage(file);
  }

  function handleResize() {
    if (imageLoaded) drawCanvas();
  }

  function resetView() {
    zoom = 1;
    panX = 0;
    panY = 0;
    drawCanvas();
  }

  function rgbToHex(r: number, g: number, b: number) {
    return "#" + [r, g, b].map((v) => v.toString(16).padStart(2, "0")).join("");
  }
</script>

<svelte:window onresize={handleResize} />

<div class="app-layout">
  <!-- Header -->
  <header class="header">
    <div class="header-left">
      <div class="logo">
        <svg
          width="20"
          height="20"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <circle cx="12" cy="12" r="3" />
          <line x1="12" y1="2" x2="12" y2="6" />
          <line x1="12" y1="18" x2="12" y2="22" />
          <line x1="2" y1="12" x2="6" y2="12" />
          <line x1="18" y1="12" x2="22" y2="12" />
        </svg>
      </div>
      <h1 class="title">Pix Coords</h1>
    </div>
    <div class="header-center">
      {#if imageLoaded}
        <div class="image-info">
          <span class="info-badge">{imageWidth} × {imageHeight}</span>
          <span class="info-badge">{Math.round(zoom * 100)}%</span>
        </div>
      {/if}
    </div>
    <div class="header-right">
      {#if imageLoaded}
        {#if !panelOpen}
          <button
            class="btn-icon"
            onclick={() => (panelOpen = true)}
            title="パネルを開く"
          >
            <svg
              width="16"
              height="16"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              ><rect x="3" y="3" width="18" height="18" rx="2" /><line
                x1="15"
                y1="3"
                x2="15"
                y2="21"
              /></svg
            >
          </button>
        {/if}
        <button class="btn-icon" onclick={resetView} title="Reset View">
          <svg
            width="16"
            height="16"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
          >
            <path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8" />
            <path d="M3 3v5h5" />
          </svg>
        </button>
      {/if}
      <button class="btn-subtle" onclick={() => fileInputEl.click()}>
        <svg
          width="15"
          height="15"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
        >
          <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
          <polyline points="17,8 12,3 7,8" />
          <line x1="12" y1="3" x2="12" y2="15" />
        </svg>
        Open
      </button>
    </div>
    <input
      bind:this={fileInputEl}
      type="file"
      accept="image/*,.tif,.tiff"
      onchange={handleFileSelect}
      hidden
    />
  </header>

  <!-- Main Content -->
  <div class="main-content" class:with-panel={imageLoaded && panelOpen}>
    {#if !imageLoaded}
      <!-- Drop Zone -->
      <div
        class="drop-zone"
        class:drag-active={isDraggingOver}
        ondrop={handleDrop}
        ondragover={handleDragOver}
        ondragleave={handleDragLeave}
        role="button"
        tabindex="0"
        onclick={() => fileInputEl.click()}
        onkeydown={(e) => e.key === "Enter" && fileInputEl.click()}
      >
        <div class="drop-zone-content">
          <div class="drop-icon">
            <svg
              width="48"
              height="48"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="1.5"
            >
              <rect x="3" y="3" width="18" height="18" rx="2" ry="2" />
              <circle cx="8.5" cy="8.5" r="1.5" />
              <polyline points="21,15 16,10 5,21" />
            </svg>
          </div>
          <p class="drop-text">画像をドロップ、またはクリックして選択</p>
          <p class="drop-hint">PNG, JPG, GIF, WebP, SVG, TIFF</p>
        </div>
      </div>
    {:else}
      <!-- Canvas Area -->
      <div
        class="canvas-container"
        bind:this={containerEl}
        role="application"
        aria-label="Image pixel coordinate viewer"
      >
        <canvas
          bind:this={canvasEl}
          onmousemove={handleMouseMove}
          onmousedown={handleMouseDown}
          onmouseup={handleMouseUp}
          onclick={handleClick}
          onwheel={handleWheel}
          onmouseleave={handleMouseLeave}
          ondrop={handleDrop}
          ondragover={handleDragOver}
          ondragleave={handleDragLeave}
          class:panning={isPanning}
        ></canvas>

        <!-- Coordinate HUD -->
        {#if isHovering}
          <div class="coord-hud">
            <div class="coord-row">
              <span class="coord-label">X</span>
              <span class="coord-value">{pixelX}</span>
            </div>
            <div class="coord-row">
              <span class="coord-label">Y</span>
              <span class="coord-value">{pixelY}</span>
            </div>
            <div class="coord-divider"></div>
            <div class="color-info">
              <div
                class="color-swatch"
                style="background-color: rgb({pixelColor.r},{pixelColor.g},{pixelColor.b})"
              ></div>
              <div class="color-values">
                <span class="color-hex"
                  >{rgbToHex(pixelColor.r, pixelColor.g, pixelColor.b)}</span
                >
                <span class="color-rgb"
                  >rgb({pixelColor.r}, {pixelColor.g}, {pixelColor.b})</span
                >
              </div>
            </div>
          </div>

          <!-- Magnifier -->
          <div class="magnifier">
            <canvas bind:this={magnifierCanvasEl}></canvas>
          </div>
        {/if}
      </div>
    {/if}

    <!-- Coordinate List Panel -->
    {#if imageLoaded && panelOpen}
      <aside class="panel">
        <div class="panel-header">
          <h2 class="panel-title">座標リスト</h2>
          <span class="panel-count">{coordList.length}</span>
          <div class="panel-actions">
            <button
              class="btn-icon-sm"
              onclick={clearList}
              title="リストをクリア"
              disabled={coordList.length === 0}
            >
              <svg
                width="14"
                height="14"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                ><polyline points="3,6 5,6 21,6" /><path
                  d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"
                /></svg
              >
            </button>
            <button
              class="btn-icon-sm"
              onclick={() => (panelOpen = false)}
              title="パネルを閉じる"
            >
              <svg
                width="14"
                height="14"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                ><line x1="18" y1="6" x2="6" y2="18" /><line
                  x1="6"
                  y1="6"
                  x2="18"
                  y2="18"
                /></svg
              >
            </button>
          </div>
        </div>

        <div class="panel-list">
          {#if coordList.length === 0}
            <div class="panel-empty">
              <p>画像をクリックして座標を追加</p>
            </div>
          {:else}
            {#each coordList as point, i (point.id)}
              <div class="point-row">
                <div class="point-top">
                  <span class="point-index">{i + 1}</span>
                  <input
                    class="point-name"
                    type="text"
                    placeholder="名前を入力..."
                    value={point.name}
                    oninput={(e) => {
                      coordList[i].name = (e.target as HTMLInputElement).value;
                    }}
                  />
                </div>
                <div class="point-bottom">
                  <div
                    class="point-color-dot"
                    style="background-color: {point.color}"
                  ></div>
                  <span class="point-coord">{point.x}</span>
                  <span class="point-comma">,</span>
                  <span class="point-coord">{point.y}</span>
                  <span class="point-hex">{point.color}</span>
                </div>
                <button
                  class="point-delete"
                  onclick={() => removePoint(point.id)}
                  title="削除"
                >
                  <svg
                    width="12"
                    height="12"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    ><line x1="18" y1="6" x2="6" y2="18" /><line
                      x1="6"
                      y1="6"
                      x2="18"
                      y2="18"
                    /></svg
                  >
                </button>
              </div>
            {/each}
          {/if}
        </div>

        <div class="panel-footer">
          <button
            class="btn-export"
            onclick={exportCSV}
            disabled={coordList.length === 0}
          >
            <svg
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              ><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" /><polyline
                points="7,10 12,15 17,10"
              /><line x1="12" y1="15" x2="12" y2="3" /></svg
            >
            CSV
          </button>
          <button
            class="btn-export btn-export-accent"
            onclick={exportXLSX}
            disabled={coordList.length === 0}
          >
            <svg
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              ><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" /><polyline
                points="7,10 12,15 17,10"
              /><line x1="12" y1="15" x2="12" y2="3" /></svg
            >
            XLSX
          </button>
        </div>
      </aside>
    {/if}
  </div>

  <!-- Footer hint -->
  {#if imageLoaded}
    <div class="footer-hint">
      <span>クリックで座標を追加</span>
      <span class="separator">·</span>
      <span>スクロールでズーム</span>
      <span class="separator">·</span>
      <span>Alt+ドラッグでパン</span>
    </div>
  {/if}

  <!-- Toast -->
  {#if toastVisible}
    <div class="toast" class:visible={toastVisible}>
      <svg
        width="14"
        height="14"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2.5"
      >
        <polyline points="20,6 9,17 4,12" />
      </svg>
      {toastMessage}
    </div>
  {/if}
</div>

<style>
  .app-layout {
    height: 100%;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  /* Header */
  .header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 16px;
    height: 48px;
    min-height: 48px;
    background: var(--bg-secondary);
    border-bottom: 1px solid var(--border-color);
    z-index: 10;
  }
  .header-left {
    display: flex;
    align-items: center;
    gap: 10px;
  }
  .header-center {
    display: flex;
    align-items: center;
  }
  .header-right {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .logo {
    display: flex;
    align-items: center;
    color: var(--accent);
  }
  .title {
    font-size: 15px;
    font-weight: 600;
    letter-spacing: -0.02em;
    color: var(--text-primary);
  }
  .image-info {
    display: flex;
    gap: 6px;
  }
  .info-badge {
    font-family: var(--font-mono);
    font-size: 11px;
    font-weight: 500;
    color: var(--text-secondary);
    background: var(--bg-tertiary);
    padding: 3px 8px;
    border-radius: var(--radius-sm);
    border: 1px solid var(--border-color);
  }
  .btn-icon {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    border-radius: var(--radius-sm);
    border: 1px solid var(--border-color);
    background: var(--bg-tertiary);
    color: var(--text-secondary);
    cursor: pointer;
    transition: all var(--transition-fast);
  }
  .btn-icon:hover {
    background: var(--bg-elevated);
    color: var(--text-primary);
    border-color: var(--border-hover);
  }
  .btn-subtle {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 14px;
    border-radius: var(--radius-sm);
    border: 1px solid var(--border-color);
    background: var(--bg-tertiary);
    color: var(--text-secondary);
    cursor: pointer;
    font-family: var(--font-sans);
    font-size: 13px;
    font-weight: 500;
    transition: all var(--transition-fast);
  }
  .btn-subtle:hover {
    background: var(--bg-elevated);
    color: var(--text-primary);
    border-color: var(--border-hover);
  }

  /* Main Content */
  .main-content {
    flex: 1;
    overflow: hidden;
    position: relative;
    display: flex;
  }
  .main-content.with-panel .canvas-container,
  .main-content.with-panel .drop-zone {
    flex: 1;
    min-width: 0;
  }

  /* Drop Zone */
  .drop-zone {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    background: radial-gradient(
      circle at 50% 50%,
      rgba(108, 99, 255, 0.03) 0%,
      transparent 70%
    );
    transition: all var(--transition-normal);
    position: relative;
    outline: none;
  }
  .drop-zone::before {
    content: "";
    position: absolute;
    inset: 24px;
    border: 2px dashed var(--border-color);
    border-radius: var(--radius-xl);
    transition: all var(--transition-normal);
    pointer-events: none;
  }
  .drop-zone:hover::before,
  .drop-zone.drag-active::before {
    border-color: var(--accent);
    box-shadow: inset 0 0 60px var(--accent-soft);
  }
  .drop-zone.drag-active {
    background: radial-gradient(
      circle at 50% 50%,
      rgba(108, 99, 255, 0.08) 0%,
      transparent 70%
    );
  }
  .drop-zone-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
  }
  .drop-icon {
    color: var(--text-muted);
    opacity: 0.6;
    transition: all var(--transition-normal);
  }
  .drop-zone:hover .drop-icon,
  .drop-zone.drag-active .drop-icon {
    color: var(--accent);
    opacity: 1;
    transform: scale(1.05);
  }
  .drop-text {
    font-size: 16px;
    font-weight: 500;
    color: var(--text-secondary);
  }
  .drop-hint {
    font-size: 12px;
    color: var(--text-muted);
    letter-spacing: 0.04em;
  }

  /* Canvas */
  .canvas-container {
    flex: 1;
    min-width: 0;
    height: 100%;
    position: relative;
    overflow: hidden;
  }
  canvas {
    width: 100%;
    height: 100%;
    display: block;
    cursor: crosshair;
  }
  canvas.panning {
    cursor: grabbing;
  }

  /* Coordinate List Panel */
  .panel {
    width: 280px;
    min-width: 280px;
    height: 100%;
    background: var(--bg-secondary);
    border-left: 1px solid var(--border-color);
    display: flex;
    flex-direction: column;
    z-index: 5;
  }
  .panel-header {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 14px;
    border-bottom: 1px solid var(--border-color);
    min-height: 44px;
  }
  .panel-title {
    font-size: 13px;
    font-weight: 600;
    color: var(--text-primary);
    letter-spacing: -0.01em;
  }
  .panel-count {
    font-family: var(--font-mono);
    font-size: 10px;
    font-weight: 600;
    color: var(--accent);
    background: var(--accent-soft);
    padding: 1px 6px;
    border-radius: 10px;
    min-width: 20px;
    text-align: center;
  }
  .panel-actions {
    margin-left: auto;
    display: flex;
    gap: 4px;
  }
  .btn-icon-sm {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border-radius: var(--radius-sm);
    border: none;
    background: transparent;
    color: var(--text-muted);
    cursor: pointer;
    transition: all var(--transition-fast);
  }
  .btn-icon-sm:hover {
    background: var(--bg-tertiary);
    color: var(--text-primary);
  }
  .btn-icon-sm:disabled {
    opacity: 0.3;
    cursor: default;
  }
  .panel-list {
    flex: 1;
    overflow-y: auto;
    padding: 4px 0;
  }
  .panel-empty {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 100%;
    color: var(--text-muted);
    font-size: 12px;
  }
  .point-row {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: 3px;
    padding: 8px 14px;
    transition: background var(--transition-fast);
    font-family: var(--font-mono);
    font-size: 12px;
    border-bottom: 1px solid var(--border-color);
  }
  .point-row:hover {
    background: var(--bg-tertiary);
  }
  .point-top {
    display: flex;
    align-items: center;
    gap: 6px;
  }
  .point-bottom {
    display: flex;
    align-items: center;
    gap: 6px;
    padding-left: 24px;
  }
  .point-index {
    color: var(--text-muted);
    font-size: 10px;
    min-width: 18px;
    text-align: right;
  }
  .point-name {
    flex: 1;
    min-width: 0;
    background: transparent;
    border: 1px solid transparent;
    border-radius: 4px;
    padding: 2px 6px;
    font-family: var(--font-sans);
    font-size: 12px;
    font-weight: 500;
    color: var(--text-primary);
    outline: none;
    transition: all var(--transition-fast);
  }
  .point-name::placeholder {
    color: var(--text-muted);
    font-weight: 400;
    opacity: 0.6;
  }
  .point-name:hover {
    border-color: var(--border-color);
  }
  .point-name:focus {
    border-color: var(--accent);
    background: var(--bg-primary);
  }
  .point-color-dot {
    width: 12px;
    height: 12px;
    border-radius: 3px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    flex-shrink: 0;
  }
  .point-coord {
    color: var(--text-primary);
    font-weight: 500;
    min-width: 32px;
    text-align: right;
  }
  .point-comma {
    color: var(--text-muted);
    margin: 0 -2px;
  }
  .point-hex {
    color: var(--text-secondary);
    font-size: 10px;
    margin-left: auto;
  }
  .point-delete {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 20px;
    height: 20px;
    border: none;
    background: transparent;
    color: var(--text-muted);
    cursor: pointer;
    border-radius: 4px;
    opacity: 0;
    transition: all var(--transition-fast);
    flex-shrink: 0;
  }
  .point-row:hover .point-delete {
    opacity: 1;
  }
  .point-delete:hover {
    background: rgba(239, 68, 68, 0.15);
    color: #ef4444;
  }
  .panel-footer {
    padding: 10px 14px;
    border-top: 1px solid var(--border-color);
    display: flex;
    gap: 8px;
  }
  .btn-export {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    padding: 7px 0;
    border-radius: var(--radius-sm);
    border: 1px solid var(--border-color);
    background: var(--bg-tertiary);
    color: var(--text-secondary);
    cursor: pointer;
    font-family: var(--font-mono);
    font-size: 12px;
    font-weight: 500;
    transition: all var(--transition-fast);
  }
  .btn-export:hover {
    background: var(--bg-elevated);
    color: var(--text-primary);
    border-color: var(--border-hover);
  }
  .btn-export:disabled {
    opacity: 0.35;
    cursor: default;
  }
  .btn-export-accent {
    background: var(--accent-soft);
    border-color: rgba(108, 99, 255, 0.2);
    color: var(--accent);
  }
  .btn-export-accent:hover {
    background: rgba(108, 99, 255, 0.2);
    border-color: rgba(108, 99, 255, 0.35);
    color: #8b83ff;
  }

  /* Coordinate HUD */
  .coord-hud {
    position: absolute;
    top: 16px;
    left: 16px;
    background: rgba(10, 10, 15, 0.88);
    backdrop-filter: blur(16px);
    border: 1px solid var(--border-color);
    border-radius: var(--radius-md);
    padding: 12px 16px;
    min-width: 160px;
    box-shadow: var(--shadow-lg);
    pointer-events: none;
  }
  .coord-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 24px;
    padding: 2px 0;
  }
  .coord-label {
    font-family: var(--font-mono);
    font-size: 11px;
    font-weight: 500;
    color: var(--accent);
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }
  .coord-value {
    font-family: var(--font-mono);
    font-size: 16px;
    font-weight: 600;
    color: var(--text-primary);
    letter-spacing: -0.02em;
  }
  .coord-divider {
    height: 1px;
    background: var(--border-color);
    margin: 8px 0;
  }
  .color-info {
    display: flex;
    align-items: center;
    gap: 10px;
  }
  .color-swatch {
    width: 28px;
    height: 28px;
    border-radius: var(--radius-sm);
    border: 1px solid rgba(255, 255, 255, 0.12);
    box-shadow: var(--shadow-sm);
    flex-shrink: 0;
  }
  .color-values {
    display: flex;
    flex-direction: column;
    gap: 1px;
  }
  .color-hex {
    font-family: var(--font-mono);
    font-size: 12px;
    font-weight: 600;
    color: var(--text-primary);
    text-transform: uppercase;
  }
  .color-rgb {
    font-family: var(--font-mono);
    font-size: 10px;
    color: var(--text-muted);
  }

  /* Magnifier */
  .magnifier {
    position: absolute;
    top: 16px;
    right: 16px;
    border-radius: var(--radius-md);
    border: 1px solid var(--border-color);
    overflow: hidden;
    box-shadow: var(--shadow-lg);
    pointer-events: none;
    background: var(--bg-secondary);
  }
  .magnifier canvas {
    display: block;
    width: 140px;
    height: 140px;
  }

  /* Footer Hint */
  .footer-hint {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 8px;
    font-size: 11px;
    color: var(--text-muted);
    background: var(--bg-secondary);
    border-top: 1px solid var(--border-color);
  }
  .separator {
    color: var(--border-color);
  }

  /* Toast */
  .toast {
    position: fixed;
    bottom: 48px;
    left: 50%;
    transform: translateX(-50%) translateY(20px);
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 20px;
    background: rgba(10, 10, 15, 0.9);
    backdrop-filter: blur(16px);
    border: 1px solid var(--success);
    border-radius: var(--radius-lg);
    color: var(--success);
    font-family: var(--font-mono);
    font-size: 13px;
    font-weight: 500;
    box-shadow: 0 0 24px var(--success-glow);
    opacity: 0;
    transition: all var(--transition-smooth);
    pointer-events: none;
    z-index: 100;
  }
  .toast.visible {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
</style>
