<script lang="ts">
  import {
    Crosshair,
    Download,
    FileJson,
    ImagePlus,
    Minus,
    MousePointer2,
    Plus,
    RotateCcw,
    Trash2,
    Upload,
  } from "lucide-svelte";

  type FloorImage = {
    id: string;
    name: string;
    label: string;
    url: string;
    width: number;
    height: number;
    order: number;
  };

  type ExportFloor = Omit<FloorImage, "url">;

  type BindingPoint = {
    id: string;
    floorId: string;
    floorLabel: string;
    label: string;
    pixel: {
      x: number;
      y: number;
    };
    game: {
      x: number;
      y: number;
      z: number;
    };
    notes: string;
    createdAt: string;
  };

  type LegacyBindingPoint = Omit<BindingPoint, "floorId" | "floorLabel">;

  type ExportFile = {
    version: 2;
    mapName: string;
    activeFloorId: string | null;
    imageSize: {
      width: number;
      height: number;
    } | null;
    floors: ExportFloor[];
    points: BindingPoint[];
  };

  type LegacyExportFile = {
    version: 1;
    mapName: string;
    floorName: string;
    image: {
      name: string;
      width: number;
      height: number;
    } | null;
    points: LegacyBindingPoint[];
  };

  type Draft = {
    label: string;
    x: string;
    y: string;
    z: string;
    notes: string;
  };

  const storageKey = "r6-coordinate-binder-state";
  const defaultFloorNames = ["Basement", "1F", "2F", "3F", "Roof"];

  const emptyDraft = (): Draft => ({
    label: "",
    x: "",
    y: "",
    z: "",
    notes: "",
  });

  let imageElement: HTMLImageElement | null = null;
  let floors: FloorImage[] = [];
  let activeFloorId: string | null = null;
  let imageSize: { width: number; height: number } | null = null;
  let mapName = "";
  let points: BindingPoint[] = [];
  let selectedId: string | null = null;
  let pendingPixel: { x: number; y: number } | null = null;
  let draft = emptyDraft();
  let zoom = 1;
  let pan = { x: 24, y: 24 };
  let isPanning = false;
  let dragStart = { x: 0, y: 0, panX: 0, panY: 0 };
  let hydrated = false;
  let statusMessage = "";

  $: activeFloor =
    floors.find((floor) => floor.id === activeFloorId) ?? floors[0] ?? null;
  $: visiblePoints = activeFloor
    ? points.filter((point) => point.floorId === activeFloor.id)
    : [];
  $: selectedPoint =
    points.find((point) => point.id === selectedId) ?? null;
  $: exportPayload = {
    version: 2 as const,
    mapName,
    activeFloorId,
    imageSize,
    floors: floors.map(({ url: _url, ...floor }) => floor),
    points,
  };

  $: if (hydrated) {
    localStorage.setItem(storageKey, JSON.stringify(exportPayload));
  }

  function hydrateSavedState() {
    const saved = localStorage.getItem(storageKey);
    if (!saved) {
      hydrated = true;
      return;
    }

    try {
      const parsed = JSON.parse(saved) as Partial<ExportFile | LegacyExportFile>;
      mapName = parsed.mapName ?? "";

      if (parsed.version === 2) {
        const v2 = parsed as Partial<ExportFile>;
        imageSize = v2.imageSize ?? null;
        floors = Array.isArray(v2.floors)
          ? v2.floors.map((floor) => ({ ...floor, url: "" }))
          : [];
        points = Array.isArray(v2.points) ? v2.points : [];
        activeFloorId = v2.activeFloorId ?? floors[0]?.id ?? null;
      } else {
        const legacy = parsed as Partial<LegacyExportFile>;
        const floorId = crypto.randomUUID();
        if (legacy.image) {
          imageSize = {
            width: legacy.image.width,
            height: legacy.image.height,
          };
          floors = [
            {
              id: floorId,
              name: legacy.image.name,
              label: legacy.floorName || "Floor 1",
              url: "",
              width: legacy.image.width,
              height: legacy.image.height,
              order: 0,
            },
          ];
          activeFloorId = floorId;
        }
        points = Array.isArray(legacy.points)
          ? legacy.points.map((point) => ({
              ...point,
              floorId,
              floorLabel: legacy.floorName || "Floor 1",
            }))
          : [];
      }
    } catch {
      localStorage.removeItem(storageKey);
    } finally {
      hydrated = true;
    }
  }

  function numberOrZero(value: string) {
    const parsed = Number.parseFloat(value);
    return Number.isFinite(parsed) ? parsed : 0;
  }

  function formatPixel(value: number) {
    return Number.isInteger(value) ? String(value) : value.toFixed(2);
  }

  function floorLabelForFile(file: File, index: number) {
    const match = file.name.match(/blueprint[-_ ]?(\d+)/i);
    const numericIndex = match ? Number.parseInt(match[1], 10) - 1 : index;
    return defaultFloorNames[numericIndex] ?? `Floor ${index + 1}`;
  }

  function readImageSize(url: string) {
    return new Promise<{ width: number; height: number }>((resolve, reject) => {
      const image = new Image();
      image.onload = () =>
        resolve({ width: image.naturalWidth, height: image.naturalHeight });
      image.onerror = () => reject(new Error("Could not read image size."));
      image.src = url;
    });
  }

  function downloadJson(filename: string, payload: ExportFile) {
    const blob = new Blob([JSON.stringify(payload, null, 2)], {
      type: "application/json",
    });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = filename;
    document.body.appendChild(link);
    link.click();
    link.remove();
    URL.revokeObjectURL(url);
  }

  async function loadFloorImages(fileList: FileList | null) {
    const files = Array.from(fileList ?? []).sort((a, b) =>
      a.name.localeCompare(b.name, undefined, { numeric: true }),
    );
    if (files.length === 0) return;

    let accepted: FloorImage[] = [];
    let rejected = 0;
    let baseSize = imageSize;

    for (const file of files) {
      const url = URL.createObjectURL(file);
      try {
        const size = await readImageSize(url);
        if (!baseSize) {
          baseSize = size;
        }

        if (size.width !== baseSize.width || size.height !== baseSize.height) {
          URL.revokeObjectURL(url);
          rejected += 1;
          continue;
        }

        const order = floors.length + accepted.length;
        accepted = [
          ...accepted,
          {
            id: crypto.randomUUID(),
            name: file.name,
            label: floorLabelForFile(file, order),
            url,
            width: size.width,
            height: size.height,
            order,
          },
        ];
      } catch {
        URL.revokeObjectURL(url);
        rejected += 1;
      }
    }

    if (accepted.length === 0) {
      statusMessage = rejected
        ? "No images were added because their dimensions did not match."
        : "";
      return;
    }

    floors = [...floors, ...accepted].sort((a, b) => a.order - b.order);
    imageSize = baseSize;
    activeFloorId = activeFloorId ?? accepted[0].id;
    pendingPixel = null;
    selectedId = null;
    statusMessage =
      rejected > 0
        ? `Added ${accepted.length} floor image(s). Skipped ${rejected} dimension mismatch.`
        : `Added ${accepted.length} floor image(s).`;
  }

  function updateFloorLabel(floorId: string, label: string) {
    floors = floors.map((floor) =>
      floor.id === floorId ? { ...floor, label } : floor,
    );
    points = points.map((point) =>
      point.floorId === floorId ? { ...point, floorLabel: label } : point,
    );
  }

  function switchFloor(floorId: string) {
    activeFloorId = floorId;
    pendingPixel = null;
    selectedId = null;
  }

  function capturePixel(event: MouseEvent) {
    if (!imageElement || !imageSize || !activeFloor || isPanning) return;

    const rect = imageElement.getBoundingClientRect();
    const x = ((event.clientX - rect.left) / rect.width) * imageSize.width;
    const y = ((event.clientY - rect.top) / rect.height) * imageSize.height;

    pendingPixel = {
      x: Math.max(0, Math.min(imageSize.width, x)),
      y: Math.max(0, Math.min(imageSize.height, y)),
    };
    selectedId = null;
    draft = {
      ...draft,
      label: draft.label || `${activeFloor.label} point ${visiblePoints.length + 1}`,
    };
  }

  function savePoint() {
    if (!pendingPixel || !activeFloor) return;

    const point: BindingPoint = {
      id: crypto.randomUUID(),
      floorId: activeFloor.id,
      floorLabel: activeFloor.label,
      label: draft.label.trim() || `${activeFloor.label} point ${visiblePoints.length + 1}`,
      pixel: {
        x: Number(pendingPixel.x.toFixed(4)),
        y: Number(pendingPixel.y.toFixed(4)),
      },
      game: {
        x: numberOrZero(draft.x),
        y: numberOrZero(draft.y),
        z: numberOrZero(draft.z),
      },
      notes: draft.notes.trim(),
      createdAt: new Date().toISOString(),
    };

    points = [...points, point];
    selectedId = point.id;
    pendingPixel = null;
    draft = emptyDraft();
  }

  function updateSelectedPoint(updates: Partial<BindingPoint>) {
    if (!selectedPoint) return;
    points = points.map((point) =>
      point.id === selectedPoint?.id ? { ...point, ...updates } : point,
    );
  }

  function updateSelectedGame(axis: "x" | "y" | "z", value: string) {
    if (!selectedPoint) return;
    updateSelectedPoint({
      game: {
        ...selectedPoint.game,
        [axis]: numberOrZero(value),
      },
    });
  }

  function removePoint(pointId: string) {
    points = points.filter((point) => point.id !== pointId);
    if (selectedId === pointId) selectedId = null;
  }

  async function importBindings(file: File) {
    const text = await file.text();
    const parsed = JSON.parse(text) as ExportFile;

    if (!Array.isArray(parsed.points)) {
      throw new Error("Imported JSON does not contain a points array.");
    }

    mapName = parsed.mapName ?? "";
    imageSize = parsed.imageSize ?? null;
    floors = Array.isArray(parsed.floors)
      ? parsed.floors.map((floor) => ({ ...floor, url: "" }))
      : [];
    points = parsed.points;
    activeFloorId = parsed.activeFloorId ?? floors[0]?.id ?? null;
    selectedId = null;
    pendingPixel = null;
    statusMessage =
      "Imported bindings. Re-add floor images to view the map backgrounds.";
  }

  function resetView() {
    zoom = 1;
    pan = { x: 24, y: 24 };
  }

  function startPan(event: PointerEvent) {
    if (event.button !== 1 && !event.altKey) return;
    isPanning = true;
    dragStart = { x: event.clientX, y: event.clientY, panX: pan.x, panY: pan.y };
    (event.currentTarget as HTMLDivElement).setPointerCapture(event.pointerId);
  }

  function movePan(event: PointerEvent) {
    if (!isPanning) return;
    pan = {
      x: dragStart.panX + event.clientX - dragStart.x,
      y: dragStart.panY + event.clientY - dragStart.y,
    };
  }

  function stopPan() {
    isPanning = false;
  }

  hydrateSavedState();
</script>

<main class="app-shell">
  <section class="toolbar" aria-label="Project controls">
    <div class="identity">
      <Crosshair size={22} />
      <div>
        <h1>R6 Coordinate Binder</h1>
        <p>Plot aligned floor images against shared pixel coordinates.</p>
      </div>
    </div>

    <label class="icon-button primary" title="Add floor images">
      <ImagePlus size={18} />
      <span>Floors</span>
      <input
        hidden
        multiple
        type="file"
        accept="image/png,image/jpeg,image/webp"
        on:change={(event) => {
          const input = event.currentTarget;
          void loadFloorImages(input.files);
          input.value = "";
        }}
      />
    </label>

    <label class="icon-button" title="Import JSON bindings">
      <Upload size={18} />
      <span>Import</span>
      <input
        hidden
        type="file"
        accept="application/json,.json"
        on:change={(event) => {
          const input = event.currentTarget;
          const file = input.files?.[0];
          if (file) void importBindings(file);
          input.value = "";
        }}
      />
    </label>

    <button
      class="icon-button"
      type="button"
      title="Export JSON bindings"
      on:click={() =>
        downloadJson(`${mapName || "r6-map"}-layered-bindings.json`, exportPayload)}
    >
      <Download size={18} />
      <span>Export</span>
    </button>
  </section>

  <section class="workspace">
    <aside class="side-panel">
      <div class="field-grid single">
        <label>
          <span>Map</span>
          <input bind:value={mapName} placeholder="Oregon" />
        </label>
      </div>

      <div class="image-meta">
        <FileJson size={17} />
        <span>{activeFloor?.name || "No floor image loaded"}</span>
        {#if imageSize}
          <strong>{imageSize.width} x {imageSize.height}</strong>
        {/if}
      </div>

      {#if statusMessage}
        <div class="status-message">{statusMessage}</div>
      {/if}

      <div class="panel-section">
        <h2>Floors</h2>
        <div class="floor-list">
          {#each floors as floor, index (floor.id)}
            <div class:active={floor.id === activeFloorId} class="floor-row">
              <button type="button" on:click={() => switchFloor(floor.id)}>
                {index + 1}
              </button>
              <input
                value={floor.label}
                aria-label={`Floor ${index + 1} label`}
                on:input={(event) =>
                  updateFloorLabel(floor.id, event.currentTarget.value)}
              />
            </div>
          {/each}

          {#if floors.length === 0}
            <div class="empty-list">Add the Oregon floor images together.</div>
          {/if}
        </div>
      </div>

      <div class="panel-section">
        <h2>New Binding</h2>
        <div class="pixel-readout">
          <MousePointer2 size={17} />
          <span>
            {#if pendingPixel}
              {activeFloor?.label}: pixel {formatPixel(pendingPixel.x)}, {formatPixel(
                pendingPixel.y,
              )}
            {:else}
              Click the active floor image to capture a pixel
            {/if}
          </span>
        </div>

        <label>
          <span>Label</span>
          <input bind:value={draft.label} placeholder="SE corner of B Laundry Stairs" />
        </label>

        <div class="coordinate-grid">
          <label>
            <span>X</span>
            <input bind:value={draft.x} inputmode="decimal" />
          </label>
          <label>
            <span>Y</span>
            <input bind:value={draft.y} inputmode="decimal" />
          </label>
          <label>
            <span>Z</span>
            <input bind:value={draft.z} inputmode="decimal" />
          </label>
        </div>

        <label>
          <span>Notes</span>
          <textarea bind:value={draft.notes} rows="3"></textarea>
        </label>

        <button
          class="wide-button"
          type="button"
          disabled={!pendingPixel || !activeFloor}
          on:click={savePoint}
        >
          Save Binding
        </button>
      </div>

      {#if selectedPoint}
        <div class="panel-section">
          <h2>Edit Selected</h2>
          <div class="selected-floor">{selectedPoint.floorLabel}</div>
          <label>
            <span>Label</span>
            <input
              value={selectedPoint.label}
              on:input={(event) =>
                updateSelectedPoint({
                  label: event.currentTarget.value,
                })}
            />
          </label>
          <div class="coordinate-grid">
            <label>
              <span>X</span>
              <input
                value={selectedPoint.game.x}
                inputmode="decimal"
                on:input={(event) => updateSelectedGame("x", event.currentTarget.value)}
              />
            </label>
            <label>
              <span>Y</span>
              <input
                value={selectedPoint.game.y}
                inputmode="decimal"
                on:input={(event) => updateSelectedGame("y", event.currentTarget.value)}
              />
            </label>
            <label>
              <span>Z</span>
              <input
                value={selectedPoint.game.z}
                inputmode="decimal"
                on:input={(event) => updateSelectedGame("z", event.currentTarget.value)}
              />
            </label>
          </div>
          <label>
            <span>Notes</span>
            <textarea
              value={selectedPoint.notes}
              rows="3"
              on:input={(event) =>
                updateSelectedPoint({
                  notes: event.currentTarget.value,
                })}
            ></textarea>
          </label>
        </div>
      {/if}
    </aside>

    <section class="map-area">
      <div class="map-controls" aria-label="Map view controls">
        <button
          type="button"
          title="Zoom out"
          on:click={() => (zoom = Math.max(0.2, zoom - 0.1))}
        >
          <Minus size={17} />
        </button>
        <span>{Math.round(zoom * 100)}%</span>
        <button
          type="button"
          title="Zoom in"
          on:click={() => (zoom = Math.min(6, zoom + 0.1))}
        >
          <Plus size={17} />
        </button>
        <button type="button" title="Reset view" on:click={resetView}>
          <RotateCcw size={17} />
        </button>
      </div>

      {#if floors.length > 0}
        <div class="floor-tabs" aria-label="Floor tabs">
          {#each floors as floor (floor.id)}
            <button
              class:active={floor.id === activeFloorId}
              type="button"
              on:click={() => switchFloor(floor.id)}
            >
              {floor.label}
            </button>
          {/each}
        </div>
      {/if}

      <div
        class:is-panning={isPanning}
        class="map-viewport"
        role="application"
        on:pointerdown={startPan}
        on:pointermove={movePan}
        on:pointerup={stopPan}
        on:pointercancel={stopPan}
      >
        {#if activeFloor && imageSize && activeFloor.url}
          <div
            class="map-layer"
            role="button"
            tabindex="0"
            style={`width: ${imageSize.width}px; height: ${imageSize.height}px; transform: translate(${pan.x}px, ${pan.y}px) scale(${zoom});`}
            on:click={capturePixel}
            on:keydown={(event) => {
              if (event.key === "Enter" && imageSize) {
                pendingPixel = {
                  x: imageSize.width / 2,
                  y: imageSize.height / 2,
                };
              }
            }}
          >
            <img
              bind:this={imageElement}
              src={activeFloor.url}
              alt={`${activeFloor.label} blueprint`}
              draggable="false"
            />

            {#each visiblePoints as point, index (point.id)}
              <button
                class:selected={point.id === selectedId}
                class="marker"
                style={`left: ${point.pixel.x}px; top: ${point.pixel.y}px;`}
                type="button"
                title={`${point.label}: ${point.game.x}, ${point.game.y}, ${point.game.z}`}
                on:click|stopPropagation={() => {
                  selectedId = point.id;
                  pendingPixel = null;
                }}
              >
                {index + 1}
              </button>
            {/each}

            {#if pendingPixel}
              <div
                class="pending-marker"
                style={`left: ${pendingPixel.x}px; top: ${pendingPixel.y}px;`}
              ></div>
            {/if}
          </div>
        {:else}
          <div class="empty-state">
            <ImagePlus size={42} />
            <h2>Add aligned floor images</h2>
            <p>Select all five Oregon images at once. Same-size images share the same pixel grid across floors.</p>
          </div>
        {/if}
      </div>
    </section>

    <aside class="bindings-panel">
      <div class="bindings-header">
        <h2>Bindings</h2>
        <span>{points.length}</span>
      </div>

      <div class="bindings-list">
        {#each points as point, index (point.id)}
          <button
            class:selected={point.id === selectedId}
            class="binding-row"
            type="button"
            on:click={() => {
              selectedId = point.id;
              activeFloorId = point.floorId;
              pendingPixel = null;
            }}
          >
            <strong>{index + 1}. {point.label}</strong>
            <span>{point.floorLabel}</span>
            <span>px {formatPixel(point.pixel.x)}, {formatPixel(point.pixel.y)}</span>
            <span>xyz {point.game.x}, {point.game.y}, {point.game.z}</span>
            <span
              class="delete-icon"
              role="button"
              tabindex="0"
              aria-label={`Delete ${point.label}`}
              on:click|stopPropagation={() => removePoint(point.id)}
              on:keydown|stopPropagation={(event) => {
                if (event.key === "Enter" || event.key === " ") {
                  event.preventDefault();
                  removePoint(point.id);
                }
              }}
            >
              <Trash2 size={17} />
            </span>
          </button>
        {/each}

        {#if points.length === 0}
          <div class="empty-list">Saved bindings will appear here.</div>
        {/if}
      </div>
    </aside>
  </section>
</main>
