# R6 Map Coordinate Binder

A Tauri desktop app for binding Rainbow Six Siege map blueprint pixels to in-game
`x`, `y`, and `z` coordinates.

The app is built for layered map sets where each floor image uses the same
canvas size and alignment. If all floor images are the same dimensions, the same
pixel coordinate can be treated as the same horizontal map position across
floors, while the selected floor and in-game `z` coordinate identify height.

## Features

- Load multiple aligned floor images for one map.
- Switch between floors with tabs.
- Click a pixel on the active floor image.
- Save a label, notes, and in-game `x`, `y`, `z` coordinate for that pixel.
- Edit and delete saved bindings.
- Export bindings as JSON.
- Import a saved JSON project.
- Zoom and pan the map view.

## Requirements

- Node.js
- npm
- Rust and Cargo
- Tauri system prerequisites for Windows

## Install

```powershell
npm install
```

## Development

Run the Tauri app in development mode:

```powershell
npm run tauri -- dev
```

Run only the web frontend:

```powershell
npm run dev
```

## Build

Create a release desktop build:

```powershell
npm run tauri -- build
```

The Windows release executable is generated at:

```text
src-tauri/target/release/map-coordinate-binder.exe
```

Installers are generated under:

```text
src-tauri/target/release/bundle/
```

## Workflow

1. Click `Floors`.
2. Select all floor images for a map at the same time.
3. Rename the floor labels if needed.
4. Select the floor you want to plot.
5. Click a pixel on the map image.
6. Enter the in-game `x`, `y`, and `z` coordinate.
7. Save the binding.
8. Repeat for other floors.
9. Export the project JSON.

## Image Alignment

Floor images must have the same width and height. The app rejects images that do
not match the first loaded floor image.

For example, if all floor images for a map are `1600 x 900` and share the same
world alignment, pixel `(800, 450)` refers to the same horizontal location on
every floor. The selected floor and in-game `z` value distinguish basement,
middle-floor, and upper-floor points.

## JSON Export

Exports include:

- Map name
- Active floor id
- Shared image size
- Floor metadata
- Saved binding points
- Pixel coordinates
- In-game `x`, `y`, `z` coordinates

Image bytes are not embedded in the JSON export. After importing a project JSON,
add the floor images again to view the map backgrounds.
