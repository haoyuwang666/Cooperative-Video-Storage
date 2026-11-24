# Cooperative Video · Distributed Video Delivery Experiment Framework

TODO(Haoyu): Maintain the project entry point, build conventions, and feature status. Each owner should update the relevant documentation when completing a task.

The Go backend provides a Raft metadata service, Peer segment storage, and a playback Session API. The React + TypeScript frontend supports browser video playback and displays headless Session status and download metrics. The local quick start below uses **1 Controller + 3 Peers + Demo API + frontend**, with no Docker required.

## Recommended: Local Web Console

Storage and repair extensions include heartbeats, verified inventory, Raft-persisted repair tasks, resource-aware placement, and isolated paired experiments.
See [docs/reliability.md](docs/reliability.md) for operation and limitations. Use WSL on native Windows; browser playback has been manually verified in Edge.

Install Go ≥ 1.24, Node.js, pnpm 11.25.0, Python 3, and FFmpeg/FFprobe, then run from the repository root:

```bash
make console
```

Open **http://127.0.0.1:8090**, select 2–8 Peers and 1 or 3 Controllers, and click “Start cluster.” The status table queries actual health, Raft roles, ports, segment object counts, stored bytes, and cumulative bytes sent by each Peer. Wait until the cluster is ready, select a video file, and click “Upload, segment, and import.” You can optionally specify a start time in seconds and a duration to keep; leaving the duration blank keeps everything through the end. The upload limit is 512 MB. Processing displays actual stages and errors without fabricated encoding percentages. After import succeeds, click “Play” in the video library to load the video directly, without entering a video ID or parameters. Download strategies and buffer parameters are under the player's “Advanced settings”; the defaults are sufficient. The player displays segment sources, verified bytes, retries, and actual playback stalls.

Keep one terminal open after starting the management service. The web page does not directly execute user-entered commands. The management port is 8090, the Demo API port is 18080, Controller HTTP ports start at 18001, Raft ports start at 17001, and Peer ports start at 19001. This mode allows only local access and does not take over or stop existing manual services on 8001/8080/9001–9003. Cluster data is stored in `.data/console/p<peer-count>-c<controller-count>/`; restarting with the same cluster size preserves videos, while different sizes use separate data. New uploads receive unique IDs and do not overwrite existing records. Videos such as sample-5s from the original manual mode do not automatically appear in the new cluster; upload the original files again through the web page.

Stop the cluster before changing the number of nodes; live scaling is not supported. Stopping through the web page stops only child processes created by the console and preserves disk data. Exiting the management service terminates its child processes; closing the page does not stop the cluster. The cluster cannot be stopped while an upload is being processed. Unfinished tasks do not automatically resume after a restart; upload the file again. Video files should come from trusted local users.

For frontend development, run `./bin/local-console`, then run `pnpm dev` in frontend. Vite proxies to 8090 by default. For the traditional manual mode, use `API_TARGET=http://127.0.0.1:8080 pnpm dev` to connect to the original Demo API.

For an end-to-end workflow regression check, run `make backend`, then `python3 tests/system/console.py`. The test uses a separate temporary directory; ensure no console is using port 8090 before testing.

Run all commands from the repository root. The repository no longer has a `code/` directory layer. For an existing local checkout, enter the project directory directly. In a new environment, clone it first:

```bash
git clone https://github.com/haoyuwang666/CS512-Final-Project.git
cd CS512-Final-Project
```

## 1. Prepare the Environment

| Tool | Version / Purpose |
|---|---|
| Go | ≥ 1.24; backend builds and tests. Verified with 1.27.4 |
| Node.js | 24; verified with 24.22.1 |
| pnpm | 11.25.0; frontend dependency installation and Vite |
| Python | ≥ 3.10; test data generation, ingestion, and experiment scripts |
| make | Runs build commands from the root Makefile |
| FFmpeg / FFprobe | Optional; required only for generating or segmenting real videos |
| Docker + Compose | Optional; required only for starting a containerized cluster |

First confirm that these commands are available in your terminal:

```bash
go version
node -v
pnpm -v
python3 --version
make --version
```

If Node.js and npm are installed but you see `pnpm: command not found`, run:

```bash
npm install -g pnpm@11.25.0
pnpm -v
```

If `node` or `npm` is also unavailable, install Node.js 24, reopen the terminal, and then install pnpm.

All Python scripts currently use only the standard library, so `requirements.txt` contains explanatory notes and no third-party packages. It does not install Go, Node.js, or pnpm. To use a separate Python environment, optionally run:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
```

## 2. Install Dependencies and Build for the First Time

Run from the repository root:

```bash
make setup
make backend
```

`make setup` downloads Go dependencies and installs frontend dependencies using the lockfile. `make backend` generates `bin/controller`, `bin/peer`, `bin/client`, and `bin/demo-api`. If both commands have already completed successfully, proceed to the next step.

## 3. Start the Backend

Open **5 terminal tabs** and enter the root of your cloned `CS512-Final-Project` repository in each. Each service below runs continuously in its own terminal. Once its listening log appears, keep that terminal open and move to the next one.

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

The Controller's default HTTP address is `127.0.0.1:8001`, and its Raft address is `127.0.0.1:7001`. After startup, wait a few seconds for the single node to elect a leader.

**Terminal 5: Demo API**

```bash
./bin/demo-api
```

The Demo API listens on `127.0.0.1:8080` by default and queries the Controller for metadata so the frontend can create and inspect Sessions. Services read process environment variables directly and **do not automatically load `.env` or `.env.example`**.

This mode is suitable for local frontend/backend integration. For a three-Controller high-availability demonstration, use the Docker configuration below or see `docs/demo.md`.

## 4. Import a Video and Start the Frontend

### A. Play a Real Video (Default Mode)

Keep the backend from Section 3 running. Open **Terminal 6**, enter the repository root, and generate and import a 12-second test video with audio for the first time:

```bash
make media
python3 scripts/ingest/ingest.py \
  .data/media/segments/manifest.json \
  --controllers http://127.0.0.1:8001

cd frontend
pnpm dev
```

`make media` requires FFmpeg/FFprobe. It encodes H.264/AAC once and generates continuous fMP4 segments for seamless playback. The script refuses to overwrite an output directory that already contains `segment-*.mp4`. An existing manifest in the new format can be imported directly. For old independent MP4 segments, regenerate them in a new directory as described under “Upgrade and Migrate Older Videos to Seamless Playback” below.

Open **http://localhost:5173**, select **“Video playback (video and audio)”**, set the video ID to **`demo-video`**, and click **“Play video / Reload.”** Native controls support seeking across the entire video, pause, volume, and fullscreen; the progress bar below the page also supports seeking. If the page asks you to start manually, click the player's play button.

The player appends segments continuously through the same MediaSource/SourceBuffer without reloading the video source at segment boundaries. Use an MSE browser that supports the H.264/AAC codecs listed in the manifest, and open the page through `localhost` or HTTPS so the browser can perform SHA-256 verification. Playback may still stall if network delivery is insufficient or you seek to an unbuffered position.

Vite proxies `/api` to `http://127.0.0.1:8080`, so the Demo API must remain running. Vite's default port is 5173; if it is occupied, use the address printed in the terminal.

To play your own video, see “Use Your Own Video” below. The default input is `my-video`, which must be imported first; it is not included in a fresh environment.

### B. Headless Simulation Experiments (No FFmpeg Required)

Use deterministic byte fixtures to verify distributed delivery and simulated consumption. Run from the repository root:

```bash
python3 scripts/media/fixture.py
python3 scripts/ingest/ingest.py \
  .data/fixture/manifest.json \
  --controllers http://127.0.0.1:8001
```

Start the frontend with `cd frontend && pnpm dev`. Select **“Headless simulation experiment”**, enter **`fixture`**, and click **“Start simulated Session.”** `fixture` contains only 4 short segments, so it normally finishes quickly. It is not a video and cannot be decoded in video mode. This mode displays simulated metrics from the Go engine; real playback mode displays the browser's actual media state.

Once data has been imported and the backend is still running, subsequent use only requires `cd frontend && pnpm dev` from the repository root. Viewing the page alone does not require a backend; creating playback or simulation Sessions requires imported data and running backend services.

### Serve the Frontend and Backend Together Without Vite

Stop the running Demo API, build the frontend from the repository root, and restart the API:

```bash
make frontend
./bin/demo-api
```

Then open **http://localhost:8080** directly, without `pnpm dev`. The Controller and Peers must still be running. Run `make frontend` again after modifying frontend source files.

## 5. Stop and Restart

Press `Ctrl+C` in each terminal running the frontend, Demo API, Controller, and 3 Peers. Data is saved in `.data/`; stopping services does not erase it. When restarting with the same configuration, you usually do not need to regenerate or reimport persisted videos in the new format or `fixture`.

Common issues:

| Symptom | Resolution |
|---|---|
| `pnpm: command not found` | Install pnpm as described in the environment setup section |
| `go: command not found` | Install Go and add it to PATH |
| `./bin/peer: No such file or directory` | Return to the repository root and run `make backend` first |
| The page opens, but Session creation fails / proxy connection is refused | Confirm that the Demo API runs on 8080, the Controller on 8001, and Peers on 9001–9003, and that test data has been imported |
| No leader is temporarily available during import | Wait for the Controller election to finish and retry; the ingestion tool also retries automatically |
| `metadata HTTP 404` / `video not found` | Ensure the input ID matches the manifest's video_id and has been successfully imported into the Controller used by the Demo API; copying the original video or generating segments alone is not an import |
| `REPACKAGE_REQUIRED` | Regenerate old independent MP4 segments as continuous fMP4; team examples use my-video, and existing data with that ID requires an isolated experiment environment. See the next section |
| `UNSUPPORTED_MEDIA` | `fixture` is for headless simulation only; video mode requires real continuous fMP4 media |
| `unknown field "media"` | The running Controller is an old binary; rebuild and restart all Controllers and the Demo API with their original configuration |
| The output directory already contains segments | Use a new directory; an existing valid manifest can be imported directly without segmenting again |
| Startup buffer exceeds the prefetch window | Reduce the startup buffer duration or increase the window; 0 starts playback as soon as the first segment is ready |
| Web Crypto / MSE format is unsupported | Use localhost or HTTPS and an MSE browser that supports the manifest's codecs |
| `address already in use` | Find and stop the previously started instance of the same service to avoid duplicate processes |

## Upgrade and Migrate Older Videos to Seamless Playback

Team documentation, frontend defaults, and experiment examples use the video ID **`my-video`**. The new manifest adds `media` information. Real playback requires `continuous-fmp4-h264-v1` or `continuous-fmp4-h264-aac-v1`. Old `independent-mp4-*` segments must be regenerated from the original file.

If the original cluster has already published `my-video` in the old format, its ID and Peer segment indexes cannot be overwritten directly. Preserve the original data and regenerate and import `my-video` in a separate, clean experiment environment. Isolate the Controller data directory, all Peer data directories, and the media output directory; clearing or replacing only the Controller is insufficient. The commands below assume an environment without old data under the same ID.

Steps 1–4 below are for **local mode**. For Docker mode, prepare new segments using Step 3, then use the Docker update and import commands at the end of this section.

1. Press `Ctrl+C` in the old Controller and Demo API terminals, then rebuild from the repository root:

   ```bash
   make backend frontend
   ```

2. Restart the Controller and Demo API using Section 3 or your original configuration. If no old data uses the same ID, continue using the original `DATA_DIR`, `PEERS`, and Raft configuration. Otherwise, use an isolated environment as described above. In a three-Controller cluster, update and restart all Controllers before writing the new manifest.

3. Generate continuous segments from the original video using the video ID `my-video` (the output directory should be empty):

   ```bash
   bash scripts/media/segment_video.sh \
     .data/uploads/my-video.mp4 \
     .data/media/my-video/segments
   python3 scripts/media/generate_manifest.py \
     .data/media/my-video/segments \
     --video-id my-video
   ```

   **If this directory already contains the new `manifest.json` generated for this run, skip this step and import it directly.** A fresh clone does not include `.data` media files.

4. Import in local single-Controller mode:

   ```bash
   python3 scripts/ingest/ingest.py \
     .data/media/my-video/segments/manifest.json \
     --controllers http://127.0.0.1:8001
   ```

   Success should output `"status": "READY"`. Refresh the frontend, select “Video playback,” enter **`my-video`**, and start playback. In Vite mode, refresh the page; combined access on 8080 uses the frontend built in Step 1.

In Docker mode, use `make up` to rebuild and update services, then run:

```bash
make ingest MANIFEST=.data/media/my-video/segments/manifest.json
```

Import and playback must use the same Controllers. Audio removed from old segments can only be restored from an original video containing an audio track; editing the manifest or renaming segments cannot restore it.

## Validation and Tests

Run from the repository root:

```bash
make vet test test-race
make build
make smoke
make test-player
make test-media
python3 scripts/check_scaffold.py
```

`make smoke` starts a temporary cluster of 3 Controllers + 3 Peers, generates short byte fixtures, imports them, simulates playback, stops one Peer, stops the leader, restarts a Controller, and verifies that writes fail without a majority. It shuts down its processes afterward. The temporary directory is automatically cleaned up. It does not depend on an existing cluster or leave long-running frontend/backend services. `make test-raft` and `make test-e2e` currently reuse this check; they are not yet a complete partition-testing suite.

`make test-player` checks bounded client prefetching, seek cancellation, serialized SourceBuffer updates, and media source reuse. `make test-media` requires FFmpeg/FFprobe and starts an isolated temporary cluster. It verifies importing video with audio, source failover, segment retrieval after the leader goes offline during a playback session, hashes, audio/video timestamp continuity, and decoding. It does not modify an existing cluster.

### Optional: Real Browser Acceptance Tests

Install the separate test tools and run from the repository root:

```bash
npm install --prefix .data/browser-tools playwright
export PLAYWRIGHT_MODULE="$PWD/.data/browser-tools/node_modules/playwright/index.mjs"
export PLAYWRIGHT_BROWSERS_PATH="$PWD/.data/browser-tools/browsers"
.data/browser-tools/node_modules/.bin/playwright install chromium
make test-media-browser
```

If Playwright reports missing system dependencies, install them as instructed and retry. These tools are for testing only and are not application runtime dependencies. The acceptance test starts a temporary cluster and verifies in Chromium that full playback loads the media source only once, introduces no new stalls at segment boundaries, and supports pause, backward seeking after completion, forward seeking, and stop. It then clears the gateway cache and separately verifies that killing the leader during playback allows recovery and completion without reloading.

The playback path retries Controller connection failures, 5xx responses, and 429 responses with backoff, allowing up to 8 seconds per metadata query. Once retries are exhausted, it returns a retryable 503. The frontend retains the current MediaSource and buffer and makes up to two additional requests. Stop or seek cancels the associated waits. Responses with 404, 409, invalid content, or segment verification failures are not retried as election failures. A short stall may occur if an election takes longer than the buffered duration. Prolonged loss of a majority still results in an error after retries are exhausted.

To check the short `my-video` with audio generated in this guide, keep the environment variables above and run:

```bash
BROWSER_TEST=1 \
MEDIA_MANIFEST=.data/media/my-video/segments/manifest.json \
python3 tests/system/media.py
```

This command imports the specified manifest into a temporary cluster for testing and does not depend on currently running services.

## Delete Videos from the Console

Click “Delete video” in the video library. After confirmation, the console stops its current playback and submits a deletion request. Deletion is irreversible and includes all registered Peer segment replicas for that video, plus the source video, transcoding outputs, and library record stored by the console.

Through Raft, the Controller marks the video as `DELETING`, registers per-replica deletion tasks, and stops publishing its manifest and segment locations. If a Peer is offline, the page retains the video and shows deletion progress; processing continues when the node recovers. If the video already has repair or redundant-replica cleanup tasks, deletion waits for them to finish before submission. Once all replicas are confirmed deleted, the status becomes `DELETED`, and console polling removes local files and the library entry. After restarting the console, start the original cluster configuration and open the page to continue processing local deletion records.

Repeated requests do not cause duplicate deletion. Shared files are retained while other videos reference them. Raft retains deletion markers and task history; the original video ID cannot be reused for import. Buffered playback content may remain visible briefly. Unregistered orphan objects from failed imports are outside this feature's cleanup scope.

```bash
make test-delete-video
```

## Manually Clean Up Redundant Replicas

After a Peer recovers, select a video in the library, expand “Clean up redundant replicas for the current video,” and click “Preview cleanup plan.” Review the two replicas to retain and the locations to delete, then click “Execute this cleanup plan.” Progress appears under “Replica maintenance tasks” in the fault-testing panel.

Each plan removes at most one replica per segment. If more redundant replicas remain, preview another plan after completion. Cleanup estimates node load using the logical byte totals of confirmed replicas across all videos. It processes larger segments first and prefers deleting redundant replicas from more heavily loaded nodes, subtracting the expected load as each plan entry is generated. Ties are resolved by Peer ID, and both retained replicas must be verified. This strategy attempts to balance existing replicas without migrating data; segment sizes, existing placement, and node health may prevent perfectly equal loads. Displayed sizes are logical sizes. A shared hash file releases disk space only when its last reference is removed.

Cleanup tasks are persisted through Raft, and targets are immediately removed from locations. Before deletion, a Peer queries its configured Controllers for authorization and re-verifies the two retained replicas. Other repair/cleanup tasks for the same segment wait during this process. If the target is offline, no majority is available, or retained replicas cannot be verified, the task stays pending and retries after node recovery. There is no force-delete or cancellation option. Old playback caches may still reference deleted replicas; existing playback failover retries other locations.

Peers must configure `CONTROLLERS` to accept deletion; `make console` and Compose already do so. Replaying old uploads/acknowledgments does not restore deleted replicas. Cleanup logs and deletion markers are not yet automatically garbage-collected. This feature assumes a trusted experiment network and provides no authentication.

```bash
make test-cleanup
```

This test uses a temporary three-node cluster and covers an offline target, leader changes, Peer restarts after deletion, and verification of retained replicas. It does not operate on an existing cluster.

## Existing Foundation and Limitations

| Capability | Current Status |
|---|---|
| Real HashiCorp Raft / BoltDB / FileSnapshotStore | Basic integration implemented; supports a single-node or fixed three-node cluster |
| Draft → two-replica acknowledgment → READY | Implemented; all mutations go through the Raft FSM, and Controllers verify persisted object metadata with configured Peers |
| Peer disk storage / SHA-256 / quota / idempotent uploads | Implemented with immutable content-addressed data and a persistent index; orphan files are conservatively retained |
| Shared Peer upload rate limiting / bytes-written metrics | Basic implementation; measured at the application ResponseWriter layer, not bytes actually received over TCP |
| Static two-replica placement / Python ingestion | Implemented; placement recommendations deduct expected space for the current plan, but there are no persistent reservations across requests |
| Headless round robin / concurrent prefetch / verification / source failover | Implemented using real HTTP and simulated media consumption; starts when the first segment is ready by default |
| Session REST / SSE / React single-page app | Live Summary/Buffer/source estimates, SSE resumption, persistent Sessions, and bounded retention |
| Heartbeat / inventory / task API | Implemented; reports are transient observations, while repair tasks and completion states are persisted through Raft |
| Deadline / throughput prediction / startup buffer / metadata cache | Implemented in the headless engine; see docs/playback.md for configuration and metric limitations |
| Repair / adaptive placement | Implemented: repair to a minimum of two replicas, task recovery across leader changes, and resource-aware ingestion placement; popularity learning and migration are not implemented |
| Browser video playback | Implemented: MSE continuous fMP4, continuous audio/video timestamps, volume/pause/fullscreen, full-video seeking, bounded prefetch, and hash verification |
| Paired experiments / hedging / coding | Entry points exist for independent resets, real seeded traces, and comparisons under equal budgets; a complete formal study, hedging, and coding remain unfinished |

This is a foundational framework, not a complete project delivery. The current fixed three-node cluster is an actual consensus service; it does not imitate Raft with an in-memory Map. The FSM's in-memory view is driven by real log replication and persistent recovery.

## Project Structure

```text
cmd/                 Four entry points: controller / peer / client / demo-api
internal/domain/     Shared Manifest, Receipt, Location, and Task models
internal/protocol/   DTOs, error codes, HTTP, and lifecycle utilities
internal/metadata/   FSM, Raft, and metadata HTTP
internal/control/    Task coordination extension points
internal/storage/    Peer persistent storage and HTTP
internal/placement/  Static placement and adaptive extension points
internal/repair/     Repair decision interfaces
internal/playback/   Downloading, scheduling, logical Buffer, clocks, Metrics, Sessions
frontend/            React + TypeScript + Vite
scripts/media/       Binary fixtures, real video generation, segmentation, manifests
scripts/ingest/      Ingestion tools using the shared protocol
experiments/         Baseline runner, fault entry points, aggregation, diagnostic plots
deployments/         Multi-node Compose and Dockerfiles
tests/integration/   Real HTTP integration tests
tests/system/        Real multi-process system tests without Docker
tests/e2e/           Demo acceptance extension notes
docs/                Protocols, architecture, invariants, ownership, development guides
results/             Generated run data (ignored by default)
report/              Report sections split by owner
```

Compared with the plan, shared API utilities are consolidated in `internal/protocol/`; no overlapping `internal/api/` package is created. Scripts are split between `scripts/media/` and `scripts/ingest/` by responsibility. `control` handles composition and keeps policy packages from modifying the FSM.

## Docker Demo Entry Point (Configuration Provided; Pending Verification on a Docker-Enabled Machine)

```bash
make up
make fixture
make ingest
# Open http://localhost:8080, select Headless simulation experiment, and use fixture
make experiment-baseline
make plot
make down
```

Exposed Controller ports are 8001–8003, Peer ports are 9001–9003, and the Dashboard port is 8080. They bind only to the host loopback interface. Raft ports are used only within the container network. Initial leader election takes a few seconds; the ingestion tool retries.

Peer addresses returned by containers use container DNS, so `make ingest` and `make experiment-baseline` run inside the same Compose network. Returned addresses cannot be used directly for downloads from the host. The experiment container's mounted directory must be writable; on Linux, set `LOCAL_UID` and `LOCAL_GID` to the current user's IDs if needed.

For real video media:

```bash
make media
make ingest MANIFEST=.data/media/segments/manifest.json
# In the Dashboard, select Video playback and enter demo-video
```

`fixture` is deterministic binary protocol test data, not a video. `demo-video` consists of continuous fMP4 segments generated by a single FFmpeg encoding pass. Use headless simulation for `fixture`; browsers can decode video and audio from `demo-video` in “Video playback” mode. The new player appends continuous segments through the same MediaSource/SourceBuffer without resetting the video source or decoder between segments. Playback can still buffer if network delivery is insufficient.

See `docs/demo.md` for local manual service configuration. For frontend development with an existing Demo API, run `cd frontend && pnpm dev`; the default port is 5173, proxying to 8080.

## Use Your Own Video (Local / Docker)

Place original videos in `.data/uploads/` under the project root. `.data` is a hidden directory; in macOS Finder, use `Command + Shift + .` to show hidden files. All paths are relative to the repository root.

**This workflow currently has no automatic directory monitoring or frontend upload button. After copying a file, manually segment and import it.** The example below uses `my-video.mp4` and the video ID `my-video`. FFmpeg, FFprobe, and Python 3 must be installed locally.

1. Create the input directory in the project root, copy your video into it, and name it `my-video.mp4`:

   ```bash
   mkdir -p .data/uploads
   # Copy the video to .data/uploads/my-video.mp4
   ```

2. Encode the original video once, split it into continuous fMP4 segments of approximately 2 seconds, and generate the import manifest (the output directory must be empty):

   ```bash
   bash scripts/media/segment_video.sh \
     .data/uploads/my-video.mp4 \
     .data/media/my-video/segments
   python3 scripts/media/generate_manifest.py \
     .data/media/my-video/segments \
     --video-id my-video
   ```

   The current segmentation script encodes H.264/yuv420p and AAC only once, splits at keyframes, and preserves continuous tfdt timestamps. This avoids repeated padding caused by restarting the AAC encoder for every segment. Videos without audio tracks are also supported. The output directory contains `segment-*.mp4`, `stream.json` recording the continuous media timeline, `my-video--*.bin` for uploading, and `manifest.json`. Each `.bin` contains the bytes of its corresponding MP4 segment. This guide uses the fixed example ID `my-video`. If a re-encoded version has the same ID as data in the original cluster, reimport it in an isolated experiment environment to avoid overwriting immutable objects. If this example directory already contains a valid continuous fMP4 manifest, skip segmentation and import it directly. IDs may contain only letters, digits, underscores, and hyphens, and must be 1–96 characters long.

3. For a Docker cluster, ensure Docker is running, start the cluster (skip if already started), and import:

   ```bash
   make up
   make ingest MANIFEST=.data/media/my-video/segments/manifest.json
   ```

   For the local single-Controller mode described above, keep all services running and use:

   ```bash
   python3 scripts/ingest/ingest.py \
     .data/media/my-video/segments/manifest.json \
     --controllers http://127.0.0.1:8001
   ```

4. A successful import outputs JSON with `status: READY`. Enter the video ID `my-video` in the frontend, select “Video playback,” and click “Play video / Reload” to play video and any existing audio track. Native controls support full-video seeking, pause, volume, and fullscreen. The progress bar below also supports seeking across the entire video. If the browser blocks autoplay with sound, click the player's play button.

File locations:

| Content | Location |
|---|---|
| Original video | `.data/uploads/my-video.mp4` in the local project directory; ingestion does not move it |
| Prepared segments and manifest | `.data/media/my-video/segments/` in the local project directory |
| Segment replicas in Docker mode | Each Peer's separate named volume, mounted at `/data/` in its container; stored as `<sha256>.blob` and indexed by `index.json` |
| Segment replicas in local mode | `.data/peer-1/`, `.data/peer-2/`, and `.data/peer-3/`, using the configuration above |
| Video manifests, publication status, and replica locations | Controller persistent data |

The ingestion tool uploads each segment to two different Peers and registers metadata. Do not copy MP4 files directly into a Peer's `/data`; this does not complete indexing or registration. The three Docker `/data` directories are independent and do not correspond to local `.data/peer-*` directories. `make down` preserves volumes; deleting volumes loses the replicas they contain. `.data/` is ignored by Git. Team members must create it on first use; do not commit video assets to the repository.

## Team Development Entry Points

- Haoyu: `internal/metadata/`, `internal/control/`, shared protocols, deployment, and integration. Next: persistent capacity reservations and task lifecycles.
- Yuquan: storage reliability, inventory reporting, the repair loop, and resource-aware placement. Next: partition/power-loss coverage, global reservations, and popularity-based policies. The current implementation spans directories beyond the original ownership split; review ownership against actual changes.
- Tian: `internal/playback/`, `frontend/`, and `experiments/`. Completed MSE seamless video playback, browser acceptance tests, headless Deadline/throughput estimation, live snapshots, and event persistence. Next: formal experiment traces, resource budgets, and environment resets.

Before taking over code, run the existing validation checks and then claim a TODO. Do not rewrite a separate Raft implementation, importer, or metrics system. See `docs/ownership.md` for directory owners.

## Scope of Use

For trusted LAN/VPN or local experiments only. There is no authentication, TLS, Byzantine fault protection, or internet-facing deployment configuration. There is no automatic deletion; insufficient space is explicitly rejected. Do not present baseline results without the proposed optimizations as research gains.
