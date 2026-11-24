# Cooperative Video

Cooperative Video is a distributed video delivery framework with Raft-based metadata coordination, replicated segment storage, and browser playback. Its Go backend manages Controllers, Peers, and playback Sessions, while the React + TypeScript frontend provides a web console for uploading videos, managing a cluster, and running playback experiments.

## Repository Structure

```text
cmd/                 Controller, Peer, client, Demo API, and local console entry points
internal/domain/     Shared manifest, receipt, location, and task models
internal/protocol/   Shared HTTP protocols, DTOs, and error codes
internal/metadata/   Raft consensus, metadata storage, and metadata HTTP API
internal/control/    Task coordination
internal/storage/    Peer storage and HTTP API
internal/placement/  Segment replica placement
internal/repair/     Replica repair policies
internal/playback/   Download scheduling, buffering, metrics, and Sessions
frontend/            React + TypeScript + Vite web interface
scripts/media/       Video generation, segmentation, and manifest tools
scripts/ingest/      Segment upload and metadata registration
experiments/         Experiment runners, aggregation, and plots
deployments/         Dockerfiles and multi-node Compose configuration
tests/               Integration, system, and end-to-end tests
docs/                Architecture, protocols, guides, and reports
results/             Experiment output
report/              Report sections
```

## Prerequisites

| Tool | Version / Purpose |
|---|---|
| Go | ≥ 1.24; backend builds and tests |
| Node.js | 24; frontend tooling |
| pnpm | 11.25.0; frontend dependencies and builds |
| Python | ≥ 3.10; media, ingestion, and experiment scripts |
| make | Build and test commands |
| FFmpeg / FFprobe | Video generation and segmentation |
| Docker + Compose | Optional container-based deployment |

On Windows, use WSL. For browser playback, use an MSE-compatible browser with H.264/AAC support and access the application through localhost or HTTPS.

Install pnpm if needed:

```bash
npm install -g pnpm@11.25.0
```

Clone the repository and install dependencies:

```bash
git clone https://github.com/haoyuwang666/CS512-Final-Project.git
cd CS512-Final-Project
make setup
```

Run the following commands from the repository root unless instructed otherwise. Python scripts use the standard library.

## Quick Start: Web Console

Start the console:

```bash
make console
```

1. Open **http://127.0.0.1:8090**.
2. Select 2–8 Peers and 1 or 3 Controllers, then click **Start cluster**.
3. Wait for the cluster health indicators to show that the nodes are ready.
4. Select a video file and click **Upload, segment, and import**. Optionally set a start time and duration; leave the duration blank to keep the rest of the video. The upload limit is 512 MB.
5. Click **Play** in the video library. Use **Advanced settings** to adjust the download strategy and buffering parameters.

The console displays node health, Raft roles, storage usage, and transfer statistics. The player displays segment sources, verified bytes, retries, and playback stalls.

Keep the console terminal open while using the cluster. To change the number of nodes, stop the cluster first and then start it with the new configuration. Closing the browser leaves the cluster running; exiting the console service terminates its child processes.

### Ports and Persistent Data

| Service | Console Mode | Manual Mode |
|---|---|---|
| Management console | 8090 | — |
| Demo API | 18080 | 8080 |
| Controller HTTP | Starting at 18001 | 8001 |
| Controller Raft | Starting at 17001 | 7001 |
| Peers | Starting at 19001 | 9001–9003 |

Console cluster data is stored in `.data/console/p<peer-count>-c<controller-count>/`. Restarting with the same cluster size reuses its videos. Different cluster sizes use separate data directories. Each uploaded video receives a unique ID.

Use **Delete video** in the library to remove a video and its registered replicas. Confirm the selection carefully: deletion also removes the source video and processing outputs saved by the console. Keep the cluster running until deletion completes.

To remove extra replicas after a Peer recovers, expand **Clean up redundant replicas for the current video**, preview the plan, review the retained copies, and execute the plan. Track progress under **Replica maintenance tasks**.

### Frontend Development

Run the console service in one terminal, then start Vite in another:

```bash
make backend
./bin/local-console
```

```bash
cd frontend
pnpm dev
```

Open the URL printed by Vite. Its default development proxy connects to the console on port 8090.

## Run Services Manually

Build the backend:

```bash
make backend
```

Open five terminals in the repository root and run one service in each.

**Terminal 1: Peer 1**

```bash
HTTP_ADDR=127.0.0.1:9001 PEER_ID=peer-1 DATA_DIR=.data/peer-1 ./bin/peer
```

**Terminal 2: Peer 2**

```bash
HTTP_ADDR=127.0.0.1:9002 PEER_ID=peer-2 DATA_DIR=.data/peer-2 ./bin/peer
```

**Terminal 3: Peer 3**

```bash
HTTP_ADDR=127.0.0.1:9003 PEER_ID=peer-3 DATA_DIR=.data/peer-3 ./bin/peer
```

**Terminal 4: Controller**

```bash
PEERS='[{"peer_id":"peer-1","url":"http://127.0.0.1:9001"},{"peer_id":"peer-2","url":"http://127.0.0.1:9002"},{"peer_id":"peer-3","url":"http://127.0.0.1:9003"}]' ./bin/controller
```

Wait a few seconds for the Controller to elect a leader.

**Terminal 5: Demo API**

```bash
./bin/demo-api
```

Set configuration variables in the process environment when launching services. See [.env.example](.env.example) for available settings.

### Import a Sample Video

In another terminal, generate and import a 12-second video with audio:

```bash
make media
python3 scripts/ingest/ingest.py \
  .data/media/segments/manifest.json \
  --controllers http://127.0.0.1:8001
```

Start the frontend with the manual Demo API as its target:

```bash
cd frontend
API_TARGET=http://127.0.0.1:8080 pnpm dev
```

Open the Vite URL, select **Video playback**, enter **`demo-video`**, and click **Play video / Reload**. Use the player's controls for seeking, pause, volume, and fullscreen. If prompted, click play to start audio and video.

### Import Your Own Video

For the web console, use the upload workflow above. For manual or Docker deployments, place your source file at `.data/uploads/my-video.mp4` and prepare it as follows:

```bash
mkdir -p .data/uploads
# Copy your video to .data/uploads/my-video.mp4
bash scripts/media/segment_video.sh \
  .data/uploads/my-video.mp4 \
  .data/media/my-video/segments
python3 scripts/media/generate_manifest.py \
  .data/media/my-video/segments \
  --video-id my-video
```

Use an empty output directory for segmentation. The tools create continuous H.264/AAC fMP4 segments, a media timeline, upload files, and `manifest.json`. Choose a unique video ID for each video; IDs may contain letters, digits, underscores, and hyphens and must be 1–96 characters long.

Import into the running manual cluster:

```bash
python3 scripts/ingest/ingest.py \
  .data/media/my-video/segments/manifest.json \
  --controllers http://127.0.0.1:8001
```

After the import returns `"status": "READY"`, select **Video playback** in the frontend, enter **`my-video`**, and start playback. The importer uploads segments to two different Peers and registers their locations with the Controller.

### Run a Headless Simulation

Generate deterministic test data and import it:

```bash
python3 scripts/media/fixture.py
python3 scripts/ingest/ingest.py \
  .data/fixture/manifest.json \
  --controllers http://127.0.0.1:8001
```

In the frontend, select **Headless simulation experiment**, enter **`fixture`**, and click **Start simulated Session**. This mode uses byte fixtures to measure distributed downloads and simulated media consumption. Use the video workflow for browser audio/video playback.

### Serve the Built Frontend

Stop the Demo API, build the frontend, and restart the API:

```bash
make frontend
./bin/demo-api
```

Open **http://localhost:8080**. Keep the Controller and Peers running. Rebuild the frontend after changing its source files.

### Stop and Restart

Press `Ctrl+C` in each service terminal to stop the manual deployment. Data remains in `.data/`. Restart with the same configuration to reuse imported videos.

## Docker Deployment

Start the cluster and import the headless fixture:

```bash
make up
make fixture
make ingest
```

Open **http://localhost:8080**, select **Headless simulation experiment**, and use video ID **`fixture`**. Docker exposes Controller ports 8001–8003, Peer ports 9001–9003, and the Dashboard on 8080.

For real video playback:

```bash
make media
make ingest MANIFEST=.data/media/segments/manifest.json
```

Select **Video playback** and enter **`demo-video`**. To import your own prepared video instead, run:

```bash
make ingest MANIFEST=.data/media/my-video/segments/manifest.json
```

Run baseline experiments and generate plots:

```bash
make experiment-baseline
make plot
```

Ingestion and experiment commands run within the Compose network to resolve container Peer addresses. On Linux, set `LOCAL_UID` and `LOCAL_GID` to your user and group IDs when needed for writable experiment output.

Stop the cluster with:

```bash
make down
```

Docker stores Peer replicas in separate named volumes. `make down` preserves those volumes for subsequent runs.

## Tests and Experiments

Run the main validation commands:

```bash
make vet test test-race
make build
make smoke
make test-player
make test-media
python3 scripts/check_scaffold.py
```

The smoke test starts a temporary multi-process cluster and exercises ingestion, playback, Peer failure, leader failure, recovery, and quorum checks. Media tests verify audio/video ingestion, segment integrity, timestamp continuity, decoding, and playback failover.

Additional system checks:

```bash
python3 tests/system/console.py
make test-repair
make test-cleanup
make test-delete-video
```

Stop any console using port 8090 before running the console test.

For browser playback tests, install Playwright in the project's local data directory:

```bash
npm install --prefix .data/browser-tools playwright
export PLAYWRIGHT_MODULE="$PWD/.data/browser-tools/node_modules/playwright/index.mjs"
export PLAYWRIGHT_BROWSERS_PATH="$PWD/.data/browser-tools/browsers"
.data/browser-tools/node_modules/.bin/playwright install chromium
make test-media-browser
```

To run the paired experiment suite:

```bash
make experiment-main
```

## Data and Documentation

| Content | Location |
|---|---|
| Source videos | `.data/uploads/` |
| Prepared sample segments | `.data/media/segments/` |
| Prepared custom video segments | `.data/media/my-video/segments/` |
| Manual Peer data | `.data/peer-1/`, `.data/peer-2/`, `.data/peer-3/` |
| Console cluster data | `.data/console/` |
| Experiment output | `results/` |

Keep media assets and generated data in `.data/`, which is ignored by Git. In macOS Finder, press `Command + Shift + .` to show hidden directories.

Further reading:

- [Service configuration and demo guide](docs/demo.md)
- [Storage, repair, and reliability guide](docs/reliability.md)
- [Playback configuration and metrics](docs/playback.md)
- [Final report](docs/final-report/README.md)
