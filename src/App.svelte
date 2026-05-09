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
  import { save } from "@tauri-apps/plugin-dialog";
  import { writeTextFile } from "@tauri-apps/plugin-fs";

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

  type LocateDraft = {
    x: string;
    y: string;
    z: string;
  };

  type Transform = {
    a: number;
    b: number;
    c: number;
    d: number;
    e: number;
    f: number;
  };

  type ErrorRow = {
    label: string;
    actualPixelX: number;
    actualPixelY: number;
    predictedPixelX: number;
    predictedPixelY: number;
    errorPixels: number;
  };

  type FloorResult = {
    floorId: string;
    floorLabel: string;
    pointCount: number;
    transform: Transform;
    originPixelX: number;
    originPixelY: number;
    averageErrorPixels: number;
    maxErrorPixels: number;
    pointErrors: ErrorRow[];
  };

  type AffineIssue = {
    floorId: string;
    floorLabel: string;
    message: string;
  };

  type AffineReport = {
    results: FloorResult[];
    issues: AffineIssue[];
  };

  type ReplayShot = {
    playerIndex?: number;
    weaponRef?: number;
    x: number;
    y: number;
    z: number;
    yaw?: number;
    pitch?: number;
    timeSecs: number;
    binOffset?: number;
    seq?: number;
  };

  type ReplayFile = {
    header?: {
      mapName?: string;
    };
    libraryShots?: unknown;
    analysis?: {
      libraryShots?: unknown;
    };
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
  let replayShots: ReplayShot[] = [];
  let activeShotIndex = 0;
  let locateDraft: LocateDraft = { x: "", y: "", z: "" };
  let locatedPixel: { x: number; y: number; game: LocateDraft } | null = null;

  $: activeFloor =
    floors.find((floor) => floor.id === activeFloorId) ?? floors[0] ?? null;
  $: visiblePoints = activeFloor
    ? points.filter((point) => point.floorId === activeFloor.id)
    : [];
  $: selectedPoint =
    points.find((point) => point.id === selectedId) ?? null;
  $: missingFloorImageCount = floors.filter((floor) => !floor.url).length;
  $: activeShot = replayShots[activeShotIndex] ?? null;
  $: affineReport = buildAffineReport();
  $: activeFloorTransform = activeFloor
    ? affineReport.results.find(
        (result) => floorGroupKey(result.floorLabel) === floorGroupKey(activeFloor.label),
      ) ?? null
    : null;
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
        if (floors.length > 0) {
          statusMessage =
            "Saved project restored. Re-add this map's floor images to show the map backgrounds.";
        }
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
        if (floors.length > 0) {
          statusMessage =
            "Saved project restored. Re-add this map's floor images to show the map backgrounds.";
        }
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

  function formatTime(value: number) {
    if (!Number.isFinite(value)) return "0:00.00";
    const minutes = Math.floor(value / 60);
    const seconds = value - minutes * 60;
    return `${minutes}:${seconds.toFixed(2).padStart(5, "0")}`;
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

  function sanitizeFilename(value: string) {
    return value.replace(/[<>:"/\\|?*\x00-\x1f]/g, "-").trim();
  }

  function downloadJson(filename: string, json: string) {
    const blob = new Blob([json], {
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

  async function saveJsonFile(filename: string, json: string, label: string) {
    const tauriWindow = window as Window & { __TAURI_INTERNALS__?: unknown };

    if (!tauriWindow.__TAURI_INTERNALS__) {
      downloadJson(filename, json);
      statusMessage = `Exported ${label}. Check your browser downloads.`;
      return;
    }

    try {
      const selectedPath = await save({
        defaultPath: filename,
        filters: [{ name: "JSON", extensions: ["json"] }],
      });

      if (!selectedPath) {
        statusMessage = `${label} export canceled.`;
        return;
      }

      const outputPath = selectedPath.toLowerCase().endsWith(".json")
        ? selectedPath
        : `${selectedPath}.json`;
      await writeTextFile(outputPath, json);
      statusMessage = `Exported ${label} to ${outputPath}.`;
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      statusMessage = `${label} export failed: ${message}`;
    }
  }

  async function exportBindings() {
    const filename = `${sanitizeFilename(mapName) || "r6-map"}-layered-bindings.json`;
    await saveJsonFile(filename, JSON.stringify(exportPayload, null, 2), "bindings");
  }

  function isReplayShot(value: unknown): value is ReplayShot {
    if (!value || typeof value !== "object") return false;
    const shot = value as Partial<ReplayShot>;
    return (
      Number.isFinite(shot.x) &&
      Number.isFinite(shot.y) &&
      Number.isFinite(shot.z) &&
      Number.isFinite(shot.timeSecs)
    );
  }

  async function importReplayShots(file: File) {
    const text = await file.text();
    const parsed = JSON.parse(text) as ReplayFile;
    const rawShots = parsed.analysis?.libraryShots ?? parsed.libraryShots;

    if (!Array.isArray(rawShots)) {
      throw new Error("Replay JSON does not contain analysis.libraryShots.");
    }

    const importedShots = rawShots
      .filter(isReplayShot)
      .sort((a, b) => a.timeSecs - b.timeSecs);

    if (importedShots.length === 0) {
      throw new Error("No usable shots were found in libraryShots.");
    }

    replayShots = importedShots;
    activeShotIndex = 0;
    pendingPixel = null;
    selectedId = null;
    if (!mapName && parsed.header?.mapName) {
      mapName = parsed.header.mapName;
    }
    statusMessage = `Imported ${importedShots.length} replay shot(s). Select a floor, then click each shot position on the map.`;
  }

  function selectShot(index: number) {
    activeShotIndex = Math.max(0, Math.min(replayShots.length - 1, index));
    pendingPixel = null;
    selectedId = null;
  }

  function clearReplayShots() {
    replayShots = [];
    activeShotIndex = 0;
    pendingPixel = null;
    statusMessage = "Replay shots cleared.";
  }

  function locateGamePosition() {
    if (!activeFloor || !imageSize) {
      statusMessage = "Load and select a floor before locating an in-game position.";
      return;
    }

    if (!activeFloorTransform) {
      statusMessage = `Need a valid affine transform for ${activeFloor.label} before locating positions.`;
      return;
    }

    const gameX = Number.parseFloat(locateDraft.x);
    const gameY = Number.parseFloat(locateDraft.y);

    if (!Number.isFinite(gameX) || !Number.isFinite(gameY)) {
      statusMessage = "Enter valid in-game X and Y coordinates before locating.";
      return;
    }

    const pixel = gameToPixel(gameX, gameY, activeFloorTransform.transform);
    locatedPixel = {
      x: Number(pixel.x.toFixed(4)),
      y: Number(pixel.y.toFixed(4)),
      game: { ...locateDraft },
    };
    selectedId = null;
    pendingPixel = null;
    statusMessage = `Located ${activeFloor.label}: pixel ${formatPixel(pixel.x)}, ${formatPixel(pixel.y)}.`;
  }

  function solve3x3(a: number[][], b: number[]) {
    const matrix = [
      [a[0][0], a[0][1], a[0][2], b[0]],
      [a[1][0], a[1][1], a[1][2], b[1]],
      [a[2][0], a[2][1], a[2][2], b[2]],
    ];

    for (let col = 0; col < 3; col += 1) {
      let pivot = col;
      for (let row = col + 1; row < 3; row += 1) {
        if (Math.abs(matrix[row][col]) > Math.abs(matrix[pivot][col])) {
          pivot = row;
        }
      }

      if (Math.abs(matrix[pivot][col]) < 1e-12) {
        throw new Error("matrix is singular; points may be collinear or too clustered");
      }

      if (pivot !== col) {
        [matrix[col], matrix[pivot]] = [matrix[pivot], matrix[col]];
      }

      const divisor = matrix[col][col];
      for (let j = col; j < 4; j += 1) {
        matrix[col][j] /= divisor;
      }

      for (let row = 0; row < 3; row += 1) {
        if (row === col) continue;

        const factor = matrix[row][col];
        for (let j = col; j < 4; j += 1) {
          matrix[row][j] -= factor * matrix[col][j];
        }
      }
    }

    return [matrix[0][3], matrix[1][3], matrix[2][3]];
  }

  function computeLeastSquaresAffine(floorPoints: BindingPoint[]): Transform {
    if (floorPoints.length < 3) {
      throw new Error("need at least 3 points");
    }

    const ata = [
      [0, 0, 0],
      [0, 0, 0],
      [0, 0, 0],
    ];
    const atx = [0, 0, 0];
    const aty = [0, 0, 0];

    for (const point of floorPoints) {
      const row = [point.game.x, point.game.y, 1];

      for (let i = 0; i < 3; i += 1) {
        for (let j = 0; j < 3; j += 1) {
          ata[i][j] += row[i] * row[j];
        }

        atx[i] += row[i] * point.pixel.x;
        aty[i] += row[i] * point.pixel.y;
      }
    }

    const xCoefficients = solve3x3(ata, atx);
    const yCoefficients = solve3x3(ata, aty);

    return {
      a: xCoefficients[0],
      b: xCoefficients[1],
      c: xCoefficients[2],
      d: yCoefficients[0],
      e: yCoefficients[1],
      f: yCoefficients[2],
    };
  }

  function gameToPixel(gameX: number, gameY: number, transform: Transform) {
    return {
      x: transform.a * gameX + transform.b * gameY + transform.c,
      y: transform.d * gameX + transform.e * gameY + transform.f,
    };
  }

  function evaluateFloor(floorId: string, floorPoints: BindingPoint[], transform: Transform) {
    let totalError = 0;
    let maxError = 0;
    const pointErrors = floorPoints.map((point) => {
      const predicted = gameToPixel(point.game.x, point.game.y, transform);
      const dx = predicted.x - point.pixel.x;
      const dy = predicted.y - point.pixel.y;
      const errorPixels = Math.sqrt(dx * dx + dy * dy);

      totalError += errorPixels;
      maxError = Math.max(maxError, errorPixels);

      return {
        label: point.label,
        actualPixelX: point.pixel.x,
        actualPixelY: point.pixel.y,
        predictedPixelX: predicted.x,
        predictedPixelY: predicted.y,
        errorPixels,
      };
    });
    const origin = gameToPixel(0, 0, transform);

    return {
      floorId,
      floorLabel: floorPoints[0]?.floorLabel ?? "",
      pointCount: floorPoints.length,
      transform,
      originPixelX: origin.x,
      originPixelY: origin.y,
      averageErrorPixels: totalError / floorPoints.length,
      maxErrorPixels: maxError,
      pointErrors,
    };
  }

  function floorGroupKey(label: string) {
    return label.trim().toLowerCase();
  }

  function buildAffineReport(): AffineReport {
    const pointGroups = new Map<
      string,
      { floorId: string; floorLabel: string; floorPoints: BindingPoint[] }
    >();
    const results: FloorResult[] = [];
    const issues: AffineIssue[] = [];

    for (const point of points) {
      const floor = floors.find((item) => item.id === point.floorId);
      const floorLabel = floor?.label || point.floorLabel || point.floorId;
      const key = floorGroupKey(floorLabel);
      const existing = pointGroups.get(key);

      if (existing) {
        existing.floorPoints.push({ ...point, floorLabel: existing.floorLabel });
      } else {
        pointGroups.set(key, {
          floorId: floor?.id ?? point.floorId,
          floorLabel,
          floorPoints: [{ ...point, floorLabel }],
        });
      }
    }

    for (const { floorId, floorLabel, floorPoints } of pointGroups.values()) {
      if (floorPoints.length === 0) continue;

      if (floorPoints.length < 3) {
        issues.push({
          floorId,
          floorLabel,
          message: `Need at least 3 points, got ${floorPoints.length}.`,
        });
        continue;
      }

      try {
        results.push(
          evaluateFloor(floorId, floorPoints, computeLeastSquaresAffine(floorPoints)),
        );
      } catch (error) {
        issues.push({
          floorId,
          floorLabel,
          message: error instanceof Error ? error.message : String(error),
        });
      }
    }

    return {
      results: results.sort((a, b) => a.floorLabel.localeCompare(b.floorLabel)),
      issues: issues.sort((a, b) => a.floorLabel.localeCompare(b.floorLabel)),
    };
  }

  async function exportAffineTransforms() {
    if (affineReport.results.length === 0) {
      statusMessage = "Add at least 3 non-collinear bindings on a floor before exporting affine transforms.";
      return;
    }

    const filename = `${sanitizeFilename(mapName) || "r6-map"}-affine-transforms.json`;
    await saveJsonFile(
      filename,
      JSON.stringify(affineReport.results, null, 2),
      "affine transforms",
    );
  }

  async function loadFloorImages(fileList: FileList | null) {
    const files = Array.from(fileList ?? []).sort((a, b) =>
      a.name.localeCompare(b.name, undefined, { numeric: true }),
    );
    if (files.length === 0) return;

    let nextFloors = [...floors];
    const reconnectedFloorIds: string[] = [];
    let addedCount = 0;
    let reconnectedCount = 0;
    let rejected = 0;
    let baseSize = imageSize;

    for (const [fileIndex, file] of files.entries()) {
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

        const missingIndexByName = nextFloors.findIndex(
          (floor) => !floor.url && floor.name === file.name,
        );
        const missingIndexByOrder = nextFloors.findIndex(
          (floor) => !floor.url && floor.order === fileIndex,
        );
        const existingIndex =
          missingIndexByName >= 0 ? missingIndexByName : missingIndexByOrder;

        if (existingIndex >= 0) {
          const existingFloor = nextFloors[existingIndex];
          nextFloors = nextFloors.map((floor, index) =>
            index === existingIndex
              ? {
                  ...floor,
                  name: file.name,
                  url,
                  width: size.width,
                  height: size.height,
                }
              : floor,
          );
          reconnectedFloorIds.push(existingFloor.id);
          reconnectedCount += 1;
        } else {
          const order = nextFloors.length;
          nextFloors = [
            ...nextFloors,
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
          addedCount += 1;
        }
      } catch {
        URL.revokeObjectURL(url);
        rejected += 1;
      }
    }

    if (addedCount + reconnectedCount === 0) {
      statusMessage = rejected
        ? "No images were added because their dimensions did not match."
        : "";
      return;
    }

    floors = nextFloors
      .map((floor, index) => ({ ...floor, order: index }))
      .sort((a, b) => a.order - b.order);
    imageSize = baseSize;
    if (!activeFloor?.url) {
      activeFloorId =
        reconnectedFloorIds[0] ?? floors.find((floor) => floor.url)?.id ?? null;
    }
    pendingPixel = null;
    selectedId = null;
    locatedPixel = null;
    const loadedMessage =
      reconnectedCount > 0 && addedCount > 0
        ? `Reconnected ${reconnectedCount} saved floor image(s) and added ${addedCount} new floor image(s).`
        : reconnectedCount > 0
          ? `Reconnected ${reconnectedCount} saved floor image(s).`
          : `Added ${addedCount} floor image(s).`;
    statusMessage =
      rejected > 0
        ? `${loadedMessage} Skipped ${rejected} dimension mismatch.`
        : loadedMessage;
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
    locatedPixel = null;
  }

  function removeFloor(floorId: string) {
    const floor = floors.find((item) => item.id === floorId);
    if (!floor) return;

    if (floor.url) URL.revokeObjectURL(floor.url);

    const remainingFloors = floors
      .filter((item) => item.id !== floorId)
      .map((item, index) => ({ ...item, order: index }));

    floors = remainingFloors;
    points = points.filter((point) => point.floorId !== floorId);
    activeFloorId =
      activeFloorId === floorId
        ? remainingFloors[0]?.id ?? null
        : activeFloorId;
    imageSize = remainingFloors.length > 0 ? imageSize : null;
    pendingPixel = null;
    selectedId = null;
    locatedPixel = null;
    statusMessage = `Removed ${floor.label}.`;
  }

  function clearMap() {
    const hasWork = floors.length > 0 || points.length > 0 || mapName.trim() !== "";
    if (hasWork && !window.confirm("Clear this map and remove all images and bindings?")) {
      return;
    }

    for (const floor of floors) {
      if (floor.url) URL.revokeObjectURL(floor.url);
    }

    localStorage.removeItem(storageKey);
    imageElement = null;
    floors = [];
    activeFloorId = null;
    imageSize = null;
    mapName = "";
    points = [];
    selectedId = null;
    pendingPixel = null;
    locateDraft = { x: "", y: "", z: "" };
    locatedPixel = null;
    draft = emptyDraft();
    resetView();
    statusMessage = "Map cleared.";
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
    locatedPixel = null;
    draft = {
      ...draft,
      label:
        draft.label ||
        (activeShot
          ? `Shot ${activeShotIndex + 1} @ ${formatTime(activeShot.timeSecs)}`
          : `${activeFloor.label} point ${visiblePoints.length + 1}`),
      x: activeShot ? String(activeShot.x) : draft.x,
      y: activeShot ? String(activeShot.y) : draft.y,
      z: activeShot ? String(activeShot.z) : draft.z,
      notes: activeShot
        ? [
            `Replay shot ${activeShotIndex + 1} of ${replayShots.length}`,
            `time ${formatTime(activeShot.timeSecs)}`,
            activeShot.seq === undefined ? "" : `seq ${activeShot.seq}`,
            activeShot.playerIndex === undefined ? "" : `player ${activeShot.playerIndex}`,
          ]
            .filter(Boolean)
            .join("; ")
        : draft.notes,
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
    if (activeShot && activeShotIndex < replayShots.length - 1) {
      activeShotIndex += 1;
    }
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
    locatedPixel = null;
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

    <label class="icon-button" title="Import replay shots JSON">
      <Upload size={18} />
      <span>Shots</span>
      <input
        hidden
        type="file"
        accept="application/json,.json"
        on:change={(event) => {
          const input = event.currentTarget;
          const file = input.files?.[0];
          if (file) {
            void importReplayShots(file).catch((error) => {
              const message = error instanceof Error ? error.message : String(error);
              statusMessage = `Shot import failed: ${message}`;
            });
          }
          input.value = "";
        }}
      />
    </label>

    <button
      class="icon-button"
      type="button"
      title="Export JSON bindings"
      on:click={() => void exportBindings()}
    >
      <Download size={18} />
      <span>Export</span>
    </button>

    <button
      class="icon-button danger"
      type="button"
      title="Clear current map"
      on:click={clearMap}
    >
      <Trash2 size={18} />
      <span>Exit Map</span>
    </button>
  </section>

  <section class="workspace">
    <aside class="side-panel">
      <div class="field-grid single">
        <label>
          <span>Map</span>
          <input bind:value={mapName} placeholder="Map name" />
        </label>
      </div>

      <div class="image-meta">
        <FileJson size={17} />
        <span>
          {#if activeFloor?.url}
            {activeFloor.name}
          {:else if activeFloor}
            Missing image: {activeFloor.name}
          {:else}
            No floor image loaded
          {/if}
        </span>
        {#if imageSize}
          <strong>{imageSize.width} x {imageSize.height}</strong>
        {/if}
      </div>

      {#if missingFloorImageCount > 0}
        <div class="status-message">
          {missingFloorImageCount} saved floor image{missingFloorImageCount === 1 ? "" : "s"}
          need to be re-added. Click Floors and select the map images again.
        </div>
      {/if}

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
              <button
                class="delete-floor"
                type="button"
                title={`Remove ${floor.label}`}
                aria-label={`Remove ${floor.label}`}
                on:click={() => removeFloor(floor.id)}
              >
                <Trash2 size={16} />
              </button>
            </div>
          {/each}

          {#if floors.length === 0}
            <div class="empty-list">Add this map's floor images together.</div>
          {/if}
        </div>
      </div>

      <div class="panel-section">
        <h2>New Binding</h2>
        {#if activeShot}
          <div class="shot-card">
            <div class="shot-card-header">
              <strong>Shot {activeShotIndex + 1} of {replayShots.length}</strong>
              <span>{formatTime(activeShot.timeSecs)}</span>
            </div>
            <div class="shot-grid">
              <span>x {activeShot.x}</span>
              <span>y {activeShot.y}</span>
              <span>z {activeShot.z}</span>
            </div>
            <div class="shot-controls">
              <button
                type="button"
                disabled={activeShotIndex === 0}
                on:click={() => selectShot(activeShotIndex - 1)}
              >
                Previous
              </button>
              <button
                type="button"
                disabled={activeShotIndex >= replayShots.length - 1}
                on:click={() => selectShot(activeShotIndex + 1)}
              >
                Next
              </button>
              <button type="button" on:click={clearReplayShots}>Clear</button>
            </div>
          </div>
        {/if}

        <div class="pixel-readout">
          <MousePointer2 size={17} />
          <span>
            {#if pendingPixel}
              {activeFloor?.label}: pixel {formatPixel(pendingPixel.x)}, {formatPixel(
                pendingPixel.y,
              )}
            {:else if activeShot}
              Click the active floor image to bind this replay shot
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
          {activeShot ? "Save Shot Binding" : "Save Binding"}
        </button>
      </div>

      <div class="panel-section">
        <h2>Locate Position</h2>
        <div class="pixel-readout">
          <Crosshair size={17} />
          <span>
            {#if activeFloorTransform}
              {activeFloorTransform.floorLabel}: transform ready
            {:else}
              Add 3+ bindings on this floor to locate positions
            {/if}
          </span>
        </div>

        <div class="coordinate-grid">
          <label>
            <span>X</span>
            <input bind:value={locateDraft.x} inputmode="decimal" />
          </label>
          <label>
            <span>Y</span>
            <input bind:value={locateDraft.y} inputmode="decimal" />
          </label>
          <label>
            <span>Z</span>
            <input bind:value={locateDraft.z} inputmode="decimal" />
          </label>
        </div>

        {#if locatedPixel}
          <div class="pixel-readout compact">
            <span>
              Pixel {formatPixel(locatedPixel.x)}, {formatPixel(locatedPixel.y)}
            </span>
          </div>
        {/if}

        <div class="button-row">
          <button
            class="wide-button secondary"
            type="button"
            disabled={!activeFloorTransform}
            on:click={locateGamePosition}
          >
            Show On Map
          </button>
          <button
            class="wide-button secondary"
            type="button"
            disabled={!locatedPixel}
            on:click={() => (locatedPixel = null)}
          >
            Clear
          </button>
        </div>
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

            {#if locatedPixel}
              <div
                class="located-marker"
                style={`left: ${locatedPixel.x}px; top: ${locatedPixel.y}px;`}
                title={`Located: ${locatedPixel.game.x}, ${locatedPixel.game.y}, ${locatedPixel.game.z}`}
              >
                <Crosshair size={16} />
              </div>
            {/if}
          </div>
        {:else}
          <div class="empty-state">
            <ImagePlus size={42} />
            <h2>Add aligned floor images</h2>
            <p>Select the floor images for one map at once. Same-size images share the same pixel grid across floors.</p>
          </div>
        {/if}
      </div>
    </section>

    <aside class="bindings-panel">
      <div class="bindings-header">
        <h2>Bindings</h2>
        <span>{points.length}</span>
      </div>

      <div class="affine-panel">
        <div class="bindings-header">
          <h2>Affine</h2>
          <span>{affineReport.results.length}</span>
        </div>
        <button
          class="wide-button secondary"
          type="button"
          disabled={affineReport.results.length === 0}
          on:click={() => void exportAffineTransforms()}
        >
          Export Affine JSON
        </button>

        {#if affineReport.results.length > 0}
          <div class="affine-list">
            {#each affineReport.results as result (result.floorLabel)}
              <div class="affine-row">
                <strong>{result.floorLabel}</strong>
                <span>{result.pointCount} points</span>
                <span>avg {formatPixel(result.averageErrorPixels)} px</span>
                <span>max {formatPixel(result.maxErrorPixels)} px</span>
              </div>
            {/each}
          </div>
        {/if}

        {#if affineReport.issues.length > 0}
          <div class="affine-list">
            {#each affineReport.issues as issue (issue.floorLabel)}
              <div class="affine-row warning">
                <strong>{issue.floorLabel}</strong>
                <span>{issue.message}</span>
              </div>
            {/each}
          </div>
        {/if}
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
