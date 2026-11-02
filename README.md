<div align="center">

# AFTERLIGHT

### Explore the changing infrared universe — one observation at a time

**A NASA Space Apps Challenge 2026 project · Planet X and SPHEREx**

AFTERLIGHT is an interactive astronomical exploration workbench for studying how
celestial objects move and change brightness across observation epochs and infrared
wavelengths. It brings a sky viewer, a 102-band SPHEREx reference model, public
archive lookups, and a transparent moving-object candidate pipeline into one
browser-based experience.

[Explore the architecture](#system-architecture) ·
[Run locally](#run-locally) ·
[Data and scientific integrity](#data-and-scientific-integrity)

</div>

> **Project status:** Challenge prototype / research and education interface.
> Several sky plates, spectra, and epochs are deterministic demonstrations rather
> than downloaded science products. Candidate detections are unconfirmed and are
> not astronomical discoveries.

---

## What it does

- **Explore a sky region.** Search curated targets, enter J2000 coordinates in
  decimal degrees or sexagesimal form, or resolve an object name with CDS Sesame.
- **Inspect an interactive sky plate.** Pan and zoom, inspect cursor coordinates,
  toggle a WCS-style grid and target markers, and adjust infrared wavelength.
- **Travel through time.** Step or play through curated epochs to see modeled proper
  motion and brightness changes.
- **Compare epochs.** View two frames side by side, blink between them, or inspect a
  signed difference image with alignment and display controls.
- **Explore SPHEREx wavelengths.** Browse 102 modeled channels across six detector
  arrays from 0.75 to 5.00 µm, with reference spectral energy distributions and
  molecular-feature annotations.
- **Review moving-object candidates.** Extract point sources from two rendered
  epochs, match static sources, rank displaced sources, and set a review status.
- **Query public archives.** When network access is available, request CDS HiPS
  survey cutouts and NASA/IPAC IRSA AllWISE or SPHEREx TAP results.
- **Export products.** Download the displayed canvas as PNG or request a
  standards-aligned demonstration FITS image from the local API.

## System architecture

The diagram follows a user action from the React interface through the local
astronomy pipeline and, where applicable, the Express API to public services. Solid
paths represent the primary runtime flow; the dashed path is the explicit
demonstration/reference fallback.

```mermaid
flowchart LR
  visitor(["Observer / learner"])

  subgraph browser["1 · Browser experience"]
    direction TB
    entry["src/main.tsx<br/>React entry + global styles"]
    app["src/App.tsx<br/>in-memory route + selected target / region"]
    routes["Pages<br/>Home · Explorer · Observations<br/>Object detail · Discoveries · About"]
    ui["Instrument components<br/>SkyCanvasViewer · TimeMachinePanel<br/>ImageComparisonPanel · WavelengthExplorerPanel<br/>DiscoveryRadarPanel · SpherexTutorialPanel"]
    entry --> app --> routes --> ui
  end

  subgraph science["2 · Client-side astronomy & rendering"]
    direction TB
    coords["lib/astronomy.ts<br/>RA/Dec parsing + formatting<br/>proper motion · angular separation<br/>TAN-style pixel / sky transforms"]
    targets["data/objects.ts<br/>curated targets + epoch metadata<br/>target search + reference field"]
    bands["lib/spherex.ts<br/>six arrays · 102 bands<br/>reference SED model"]
    renderer["lib/skyRenderer.ts<br/>coordinate-seeded field<br/>synthetic PSFs + epoch / band response<br/>asinh stretch + infrared color mapping"]
    compare["Image comparison<br/>side-by-side · blink · B − A residual"]
    detector["lib/detection.ts<br/>background / noise estimate<br/>local peaks + weighted centroids<br/>static matches + candidate ranking"]
    coords --> renderer
    targets --> renderer
    bands --> renderer
    renderer --> ui
    renderer --> compare
    renderer -->|"two epoch buffers"| detector
    coords --> detector
    detector --> ui
  end

  subgraph api["3 · Same-origin Express API · server.ts"]
    direction TB
    health["GET /api/health"]
    resolve["GET /api/archive/resolve<br/>parse coordinates or query CDS Sesame"]
    cutout["GET /api/archive/cutout<br/>proxy allow-listed HiPS2FITS survey"]
    wise["GET /api/archive/irsa-catalog<br/>AllWISE cone-search TAP"]
    spherex["GET /api/archive/spherex-cutouts<br/>SPHEREx artifact / plane TAP"]
    fits["GET /api/archive/fits-export<br/>128 × 128 float32 demonstration FITS"]
    serverStatic["Development: Vite middleware<br/>Production: static dist + SPA fallback"]
  end

  subgraph data["4 · Local reference data"]
    direction TB
    epochData["Curated target and epoch metadata<br/>WISE / NEOWISE reference epochs<br/>modeled SPHEREx epochs"]
    m101["IRSA SPHEREx tutorial reference<br/>M101 DR1 / QR3 cutout URIs<br/>17 listed records: 12 DR1 + 5 QR3"]
    spectral["SPHEREx instrument reference<br/>102 modeled channels · six arrays<br/>reference—not live—spectra"]
  end

  subgraph external["5 · Optional public services"]
    direction TB
    sesame["CDS Sesame<br/>name resolution"]
    hips["CDS HiPS2FITS<br/>AllWISE · 2MASS · DSS2 survey plates"]
    irsa["NASA/IPAC IRSA TAP<br/>AllWISE catalog · SPHEREx cutout metadata"]
    ibe["NASA/IPAC IRSA IBE<br/>linked SPHEREx Level-2 FITS cutout URIs"]
  end

  visitor --> entry
  ui -->|"same-origin fetch"| resolve
  ui -->|"archive overlay"| cutout
  ui -->|"catalog / cutout metadata"| wise
  ui -->|"catalog / cutout metadata"| spherex
  ui -->|"download link"| fits
  routes --> epochData
  routes --> m101
  ui --> spectral
  resolve -->|"when query is not coordinates"| sesame
  cutout --> hips
  wise --> irsa
  spherex --> irsa
  m101 -.->|"reference fallback when TAP is unavailable<br/>and field is near M101"| spherex
  spherex -->|"returned record links"| ibe

  classDef actor fill:#0b3d91,color:#ffffff,stroke:#07295f,stroke-width:2px
  classDef frontend fill:#eaf2ff,color:#142d4e,stroke:#4d7fb8,stroke-width:1.5px
  classDef science fill:#e9f8f5,color:#123d36,stroke:#348b79,stroke-width:1.5px
  classDef api fill:#fff3df,color:#4b3310,stroke:#cc8b21,stroke-width:1.5px
  classDef data fill:#f1edff,color:#30254d,stroke:#8066b3,stroke-width:1.5px
  classDef external fill:#fcebf0,color:#4c2030,stroke:#b75d78,stroke-width:1.5px
  class visitor actor
  class entry,app,routes,ui frontend
  class coords,targets,bands,renderer,compare,detector science
  class health,resolve,cutout,wise,spherex,fits,serverStatic api
  class epochData,m101,spectral data
  class sesame,hips,irsa,ibe external
```

### Runtime flow

1. The React app keeps route, target, region, selected epoch, and active band in
   component state; it does not use a client-side router or a database.
2. The browser builds a deterministic float sky buffer from the selected target,
   epoch, and band. Canvas rendering, epoch comparison, and candidate extraction
   happen locally.
3. Optional live archive actions go to the same-origin Express endpoints. The
   server makes bounded-time requests to public services and returns normalized
   results or a clearly labeled fallback.
4. Production mode serves Vite's `dist` output from Express; development mode
   mounts Vite's middleware on that same server.

## Science and analysis features

### Sky coordinates and motion

`src/lib/astronomy.ts` parses RA and declination in decimal degrees or sexagesimal
formats, formats coordinate readouts, computes great-circle angular separation,
propagates proper motion from a reference epoch, and converts between sky
coordinates and viewer pixels using a small-field tangent-plane approximation.

### Synthetic observation renderer

`src/lib/skyRenderer.ts` seeds a reproducible background field from sky coordinates,
then renders stars, faint galaxy-like sources, a target, wavelength-dependent
brightness, a point-spread function, and deterministic sky grain into a `Float32Array`.
The viewer maps that buffer to an asinh-stretched, false-color canvas. It is an
interactive **demonstration renderer**, not a calibrated SPHEREx instrument simulator
or a substitute for archival image data.

### SPHEREx reference model

`src/lib/spherex.ts` defines six detector arrays and generates 102 wavelength
channels over 0.75–5.00 µm. The wavelength explorer computes deterministic reference
SED curves from blackbody-like shapes and illustrative molecular or dust features
for supported target classes. These curves are labeled as reference models, not
observed SPHEREx spectra.

### Discovery Radar

`src/lib/detection.ts` estimates background level and noise from sampled pixels,
detects local maxima above an SNR threshold, computes intensity-weighted centroids,
matches near-stationary sources between frames, and ranks remaining displaced
sources as candidates. Results include positions, displacement, estimated motion,
SNR, confidence score, and review status. The pipeline operates on the app's
rendered demonstration frames; it is an educational prototype, not a validated
survey discovery pipeline.

### Archive and export API

All endpoints are implemented in `server.ts`:

| Endpoint | Purpose | Live source / behavior |
| --- | --- | --- |
| `GET /api/health` | Service health and timestamp | Local server |
| `GET /api/archive/resolve?q=...` | Accept coordinate pairs or resolve object names | Local parser, then CDS Sesame |
| `GET /api/archive/cutout?ra=...&dec=...&fov=...&survey=...` | Retrieve an archival sky cutout | CDS HiPS2FITS; allow-listed AllWISE, 2MASS, DSS2 |
| `GET /api/archive/irsa-catalog?ra=...&dec=...&radiusArcsec=...` | Query nearby AllWISE sources | NASA/IPAC IRSA TAP, `allwise_p3as_psd` |
| `GET /api/archive/spherex-cutouts?ra=...&dec=...&sizeDeg=...&bandpass=...` | Find matching SPHEREx cutout metadata and URIs | NASA/IPAC IRSA TAP, `spherex.artifact` joined with `spherex.plane`; M101 tutorial references can be used as fallback |
| `GET /api/archive/fits-export?ra=...&dec=...&fov=...&epoch=...&band=...&targetName=...` | Download a FITS-format product | Locally generated demonstration Gaussian field; **not an archive cutout** |

Public-service calls have short timeouts. Some endpoints return a demonstration
status or reference rows when the upstream is unavailable; the UI also surfaces
provenance labels so those results are not silently presented as live observations.
The SPHEREx tutorial data is based on the
[Caltech/IPAC IRSA SPHEREx cutouts tutorial](https://caltech-ipac.github.io/irsa-tutorials/spherex-cutouts/).

## Project structure

```text
.
├── server.ts                     # Express API, archive proxies, FITS export, Vite host
├── index.html                    # Document metadata and React mount point
├── vite.config.ts                # React + Tailwind Vite plugins and aliases
├── package.json                  # Scripts and dependencies
├── src/
│   ├── main.tsx                  # React entry
│   ├── App.tsx                   # App state, navigation, target resolution
│   ├── index.css                 # Tailwind import and global styles
│   ├── assets/images/            # Illustrative plates and M101 tutorial preview
│   ├── components/               # Viewer, comparison, timeline, radar, layout panels
│   ├── data/
│   │   ├── objects.ts            # Curated objects, epoch metadata, target lookup
│   │   └── spherexTutorialDataset.ts # M101 DR1/QR3 reference records and ADQL
│   ├── lib/
│   │   ├── astronomy.ts          # Coordinate parsing, motion, separation, WCS helpers
│   │   ├── detection.ts          # Point-source extraction and candidate matching
│   │   ├── skyRenderer.ts        # Deterministic sky-field renderer and color mapping
│   │   ├── spherex.ts            # 102-band reference model and SED calculations
│   │   └── lenisContext.tsx      # Smooth-scroll provider
│   └── pages/                    # Home, Explorer, Observations, Detail, Discoveries, About
└── README.md
```

## Run locally

### Requirements

- Node.js compatible with Vite 8 (Node.js 20.19+ or 22.12+)
- npm
- Network access is optional; live name resolution and archive queries require
  external services to be reachable.

### Install and start

```bash
npm install
npm run dev
```

Open the URL printed by the server (by default, `http://localhost:3000`). The
development script runs `server.ts` with `tsx`; Vite is mounted as middleware.

### Build and serve production output

```bash
npm run build
```

To serve the generated `dist/` directory through Express, start the app with
`NODE_ENV=production` set in the environment. The server listens on port `3000` by
default; set `PORT` to override it.

### Checks

```bash
npm run lint
npx tsx --test src/lib/astronomy.test.ts src/data/spherexTutorialDataset.test.ts
```

`lint` runs the TypeScript compiler in no-emit mode. The test files cover coordinate
parsing and transforms, proper motion, SPHEREx band generation, the demonstration
moving-source pipeline, and the M101 tutorial dataset and query builder.

## Data and scientific integrity

- **Live archive data and modeled data are different.** The app can request select
  public archival products and catalog metadata, but its canvas sky plates and
  reference SED curves are generated locally.
- **Epoch metadata is not proof of downloaded pixels.** Curated WISE/NEOWISE and
  SPHEREx epoch entries provide context for the demonstration timeline; do not treat
  every rendered epoch as an actual retrieved survey exposure.
- **SPHEREx integration has boundaries.** The app models the six-array, 102-channel
  wavelength layout and queries public SPHEREx cutout metadata. It does not currently
  ingest live SPHEREx FITS pixel arrays into the interactive sky renderer.
- **FITS export is a demo product.** The export endpoint writes a small generated
  float32 field with coordinate and band metadata. It is not downloaded science data
  or a calibrated instrument observation.
- **Candidates need independent confirmation.** Offsets can arise from rendering,
  noise, source blending, detector artifacts, or catalog mismatches. Validate any
  real target against astrometric catalogs and appropriate follow-up; this interface
  does not establish a planet or other discovery.
- **External dependencies can be unavailable.** Public endpoints may be inaccessible
  due to network, service, or runtime conditions. The app is designed to report
  demonstration/reference behavior in those cases.

AFTERLIGHT was developed for the NASA Space Apps Challenge 2026. It is an independent
challenge project and is not an official NASA product or an endorsement by NASA,
JPL-Caltech, Caltech/IPAC, or CDS.

## Acknowledgements

The project references public astronomy services and mission/archive material from
NASA and JPL-Caltech, NASA/IPAC IRSA, AllWISE/NEOWISE, 2MASS, ESA Gaia, CDS Strasbourg,
and the Caltech/IPAC SPHEREx cutouts tutorial. See the in-app About page and
individual dataset references for more context.
