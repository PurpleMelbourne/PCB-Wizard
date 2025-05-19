**PCB-Wizard** is an offline, Python 3/Tkinter application that automates the *first mile* of reverse-engineering two-layer printed-circuit boards from a pair of ordinary photographs.  Its goal is to hand you a rectified, millimetre-true board image, a rough vector outline of every copper trace, a drill/via catalogue, and a small bundle of CAD-ready files that drop straight into KiCad or other EDA packages—-all in under two minutes on a laptop, no cloud, no paid libraries.

---

### 1  What it does

| Stage             | Key output files                  | Purpose                                                                                                                 |
| ----------------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Pre-process**   | `front_rect.png`, `back_rect.png` | Remove the blue/green background sheet, deskew & warp each photo so the PCB becomes a perfectly axis-aligned rectangle. |
| **Calibrate**     | (scalar) `px_per_mm`              | Detect a known pitch (header, SOIC, QFP) and turn pixels into millimetres.                                              |
| **Align**         | (in-RAM matrix)                   | Mirror the back image, match ORB key-points, and affine-align it to the front to < 1 px RMS.                            |
| **Analyse**       | `analysis.json`                   | Skeleton-thin copper mask, convert skeleton to polylines, cluster widths, detect drills/vias.                           |
| **Vector export** | `rough.svg`, `trace_mask.png`     | SVG polyline + circle vectors, plus a QA raster mask showing the extracted geometry.                                    |

Everything is driven by a *six-stage* pipeline that runs each heavy operation in a `concurrent.futures.ProcessPoolExecutor`, emitting progress events back to the GUI so the interface never freezes.

---

### 2  Pipeline internals (Milestones 0-1)

1. **Load** – read two JPEG/PNG files into NumPy BGR arrays, warn if they exceed \~20 MP.
2. **Pre-process** –

   * CIEDE2000 ΔE threshold → binary board mask
   * Morphological bridge to close IDC notch gaps
   * Largest contour → minimum-area rectangle (optionally refined by Hough lines)
   * `cv2.getPerspectiveTransform` + `warpPerspective` → axis crop
3. **Calibrate** – heuristic search for a 2 × 25 header, SOIC pins, or QFP pads; fallback 300 DPI.
4. **Align** – detect up to 4 k ORB key-points per side, brute-force Hamming match, RANSAC affine, sub-pixel tweak.
5. **Analyse** –

   * Binary copper mask → Zhang–Suen skeleton (`utils.fast_thin`, SSE-optimised C ext if installed)
   * DFS converts skeleton pixels to ordered polylines
   * Distance-transform samples width every 6 px; DBSCAN clusters to canonical widths (0.15/0.20/0.30 mm…)
   * `cv2.HoughCircles` finds circular drill hits (0.2–2.0 mm) and records centres + diameters
6. **Vector export** –

   * `vectorise.svg_writer` builds `<polyline>`/`<circle>` elements inside a physical-unit viewBox
   * `BoardAnalysis.to_json()` serialises traces, drills, widths, and px/mm scale
   * `trace_mask.png` paints the vector back into a one-pixel QA mask for visual sanity check

---

### 3  Flat-Eight code layout

```text
pcbwizard/
├ gui.py        # Tkinter front-end: drag-drop, progress bus, manual-crop dialog
├ pipeline.py   # Manager thread & ProcessPool, emits Stage events
├ preprocess.py # Mask, quad detect, warp, dpi estimate
├ analysis.py   # Align, skeleton-to-polyline, width cluster, drill detect
├ vectorise.py  # Writers for SVG/JSON/PNG + preview helper
├ models.py     # Dataclasses: BoardImage, BoardAnalysis, Event(Stage, payload)
├ utils.py      # Colour utils, fast_thin, DnD helpers, image I/O, logging
└ plugins.py    # Decorator registry, auto-discovery for UNet/GNN extensions
```

Each heavy module is pure-Python + OpenCV, so it runs on Windows, macOS, or Linux; CUDA is detected automatically but never required.

---

### 4  Road-map snapshot (May 2025)

| M-ID | Codename             | Status     | Highlights                                           |
| ---- | -------------------- | ---------- | ---------------------------------------------------- |
| 0    | **Geometry Bedrock** | ✅ Complete | Isolation, warp, px/mm scale, layer alignment        |
| 1    | **Rough Vectors**    | ✅ Complete | Skeleton→polyline, drill map, SVG/JSON/PNG export    |
| 2    | **DRC Inference**    | 🔜         | Histograms → `min_width`, `min_clearance`, `annulus` |
| 3    | **Net-Graph**        | ℹ️ Planned | Pad/via graph, KiCad netlist stub                    |
| 4    | **Polished CAD**     | ℹ️ Planned | Gerber, DXF, IPC-2581, native `.kicad_pcb`, STL      |
| 5    | **GUI v2**           | ℹ️ Planned | Live corner tweak, batch queue, plugin options       |
| 6    | **AI Segmentation**  | ℹ️ Planned | UNet/GNN copper mask, hidden-layer inference         |
| 7    | **Multilayer + CT**  | ℹ️ Planned | X-ray stack, 3-D net-graph, ODB++                    |
| 8    | **Docs & CI/CD**     | ℹ️ Planned | Sphinx docs, sample boards, GitHub Actions           |

---

### 5  Extensibility & plug-ins

`plugins.py` exposes a `@pcbwizard.plugin` decorator.  Any third party can drop `*.py` in `~/.pcbwizard/plugins/`, and the GUI auto-loads:

* **AI segmentation** (UNet)
* **Footprint classifiers** (ML-based pad grouping)
* **Export filters** (custom JSON → proprietary CAD)

A Y-up CAD coordinate space is preserved end-to-end, so add-ons never need to guess units or orientation.

---

### 6  Typical performance

| Image (Front+Back)  | Time (AMD R7-5800U, 8-core) |
| ------------------- | --------------------------- |
| 12 MP (4000 × 3000) | 1.4 – 1.9 s total           |
| 24 MP DSLR          | 3 – 4 s                     |
| RAM peak            | ≈ 3 × image size            |

Skeleton thinning and HoughCircles dominate runtime; both are swappable for faster C++ or GPU kernels without touching the rest of the code.

---

### 7  Getting started

```bash
git clone https://github.com/<you>/pcb-wizard.git
cd pcb-wizard
python -m venv .venv && source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python gui.py
```

Drag two photos, click **Run**, watch the progress thumbnails, and collect your SVG, JSON, and PNG assets in the output folder.
