# Technical Blueprint — Football Player Tracking & Possession Analytics Pipeline

## 1. Project Vision

**Semester scope (Phase 1–3):** Build an end-to-end Medallion (Bronze/Silver/Gold) data pipeline that ingests raw football broadcast footage and produces structured, analysis-ready tables describing player positions, tracking identities, and ball possession events — using Apache Spark on Databricks for the structured-data stages.

**Post-semester scope:** Use the Gold layer data to train/refine models for (a) more robust tracking and re-identification, and (b) jersey-number-based player naming, then package the result into a demo product — a match video with live player-tracking overlays and possession/pass analytics, potentially extended into a subscription tool for coaches, analysts, or content creators.

**Key design decision:** Player naming (jersey OCR → roster lookup) is explicitly **out of scope for the semester** and flagged as future work. This semester's deliverable is anonymous but persistent player tracking + possession analytics — genuinely difficult computer vision work, fully demoable, and honest about its limitations.

---

## 2. Tools & Where Each Stage Runs

| Stage | Tool/Environment | Why |
|---|---|---|
| Raw video storage | Google Drive (mounted in Colab) | Free, large storage, avoids re-uploading multi-GB files |
| Frame extraction | Python/OpenCV, on Colab or Kaggle | Needs to touch raw video/pixels |
| Player + ball detection | YOLO (pretrained, e.g. YOLOv8), on Colab/Kaggle **GPU** | Neural network inference is GPU-bound; free GPU tier is essential given the 8GB RAM/no-GPU laptop |
| Multi-object tracking | ByteTrack or DeepSORT, on Colab/Kaggle GPU | Runs alongside detection, assigns persistent track IDs per camera shot |
| Structured output (detections/tracks) | Written as Parquet/CSV/JSON from Colab | This is the bridge artifact — no more raw pixels past this point |
| Bronze layer ingestion | Databricks (Spark) | Ingests the structured detection files + raw video metadata |
| Silver layer cleaning | Databricks (Spark) | Casting, deduplication, camera-shot segmentation, filtering low-confidence detections |
| Gold layer aggregation | Databricks (Spark) | GroupBy/window functions: possession %, heatmaps, pass events |
| BI Dashboard | Power BI / Tableau (free tier) | Connects to Gold tables |
| Laptop (i5 8th-gen, 8GB RAM) | Code editing, Spark notebook authoring, light testing on short clips only | Not used for detection/tracking inference — too slow |

**Important principle:** OpenCV/YOLO never run *inside* Spark. Spark is not built for pixel-level processing. The pipeline is: **pixels → (Colab/Kaggle GPU) → structured rows → (Databricks/Spark) → Bronze → Silver → Gold**.

---

## 3. Data Source & Match Selection Plan

**Confirmed source: SoccerNet-Tracking** (soccer-net.org), an open academic dataset for multi-object tracking in soccer broadcast video, freely available for academic/non-commercial research use. It provides two kinds of raw video that map cleanly onto our full-load/incremental-load pattern:
- One **complete 45-minute half-time video** (continuous, long-form broadcast footage).
- A pool of **200 short (30-second) tracking sequences**, drawn from 12 different matches.

SoccerNet-Tracking also ships **ground-truth annotations** (tracklets, jersey numbers, team labels, camera/shot boundaries) for this footage. **We do not ingest these annotations as pipeline input.** Our pipeline processes only the raw video, independently, through our own detection/tracking/team-assignment stages — exactly as it would on any unlabeled real-world footage. The ground-truth annotations are used strictly downstream, as a **validation benchmark**, to measure how accurate our own pipeline's output is (see Gold layer, Section 6, and Limitations, Section 8). This preserves the project as genuine data engineering/CV work rather than reformatting an already-labeled dataset.

- **Do not upload full raw videos to Colab/Kaggle directly** — keep raw video in Google Drive; stream/mount it from there.
- **Frame sampling rate:** ~2 frames/second (not full 25–30fps) — cuts data volume by roughly 10–15x with minimal loss of tracking continuity for possession-level analysis.
- **Scope for the semester:**
  - **Full Load:** the complete 45-minute half video, sampled at 2fps, processed end-to-end as the primary proof of concept and historical baseline.
  - **Incremental Load:** a subset of the 30-second sequences (e.g. 5–10 clips to start, more added over time), each processed and appended as if newly arrived — see Section 4 for why this is a genuine incremental pattern and not just repeated full loads.
- **Camera handling:** Only continuous single-camera segments are tracked continuously. SoccerNet's camera/shot annotations are used to validate our own shot-boundary detection logic (see Section 8), not to pre-segment the input.
- **Licensing:** SoccerNet-Tracking is open for academic/research use; confirm current license terms at soccer-net.org before pushing any derived clips to a public GitHub repo, and credit the dataset in the repo README.

---

## 4. Bronze Layer

**Contents (raw, unprocessed):**
- Raw match video files (or references/paths to them if stored outside the repo due to size) — the `.mp4`/`.mkv` source files.
- Match metadata: match name, date, teams, venue, source, resolution, frame rate, duration.
- Raw per-frame detection output as it comes straight out of YOLO/tracking (no cleaning applied yet): frame_id, timestamp, camera_shot_id (best guess), track_id (raw, unvalidated), bbox coordinates (x, y, width, height), class (`player`/`ball`), detection confidence score, dominant bbox color (raw RGB sample, pre-clustering).

**Ingestion pattern:**
- **Full Load:** Batch ingestion of the complete 45-minute half (all sampled frames) plus its metadata — this is the historical baseline, processed once.
- **Incremental Load:** Individual 30-second sequences from SoccerNet's pool of 200 clips are ingested **periodically, in batches, from the same source dataset** — each new batch of clips produces new detection rows that are appended to Bronze without reprocessing the baseline. This mirrors a real production pattern: a large historical backfill once, followed by new same-source records (new clips) arriving over time — the same insert-only pattern as new GitHub commits or new property listings, just applied to video segments instead of text records. Unlike processing unrelated matches, every incremental batch here comes from the identical source/dataset/schema, which is what makes this a genuine incremental load rather than repeated full loads.
- *Note on updates/deletes:* this domain is predominantly insert-only. "Updates" would occur only if a track ID is retroactively corrected during Silver-layer smoothing; "deletes" occur when low-confidence detections are filtered out during cleaning. Neither is central to ingestion, which is explicitly acknowledged rather than forced to fit.

---

## 5. Silver Layer

**Cleaning / transformation steps (in Spark):**
- **Casting:** timestamps to proper timestamp type, coordinates to numeric types, confidence to float, IDs to consistent integer/string types.
- **Filtering:** drop detections below a confidence threshold (e.g. <0.5) as noise.
- **Deduplication:** remove duplicate detections for the same object in the same frame (can happen with overlapping bounding boxes from the model).
- **Camera-shot segmentation:** apply/validate shot-boundary detection so that track IDs are only considered continuous *within* a single camera shot. Each shot gets a `shot_id`; track IDs are explicitly documented as **not** persistent across shots (known limitation, described in Section 8).
- **Team assignment:** cluster players into Team A / Team B / Referee per match based on dominant jersey color in each bounding box (k-means or similar, computed per match — no cross-match training needed, see Section 8).
- **Ball–player proximity computation:** compute distance between ball position and each player's bounding box per frame, to prepare for possession logic in Gold.

**High-level Silver data model (tables):**
- `silver_detections`: one row per detection (frame, shot_id, track_id, class, bbox, team, confidence).
- `silver_ball_positions`: one row per frame where the ball is detected (frame, shot_id, x, y, confidence).
- `silver_shots`: one row per camera shot (shot_id, match_id, start_frame, end_frame, camera_angle_guess).

---

## 6. Gold Layer

**Business-ready aggregated tables:**
- `gold_possession_by_track`: possession percentage per track_id per shot/match (based on ball-proximity frames attributed to that track).
- `gold_player_movement`: per-track_id summary — total distance covered (approximate, pixel-space or calibrated if a pitch-mapping step is added), average position (heatmap-ready x/y bins), time spent in each third of the pitch.
- `gold_pass_events`: candidate pass events — moments where ball proximity shifts from one track_id to a different track_id on the same team, with timestamp, from_track, to_track, shot_id.
- `gold_match_summary`: per-match rollup — total detected passes, possession split between the two teams (aggregated at team level, which *is* reliable even without player names), shot-segment count (i.e., how many camera cuts occurred).
- `gold_tracking_accuracy`: **validation table**, comparing our pipeline's own output against SoccerNet's ground-truth annotations for the same clips — e.g. % of frames where our track_id matched their labeled tracklet continuity, % agreement between our jersey-color team clustering and their labeled team field, and accuracy of our own shot-boundary detection against their camera-segmentation labels. This is not used as input anywhere upstream; it exists purely to demonstrate and quantify pipeline quality.

This is the layer your dashboard and, later, your ML training will read from.

---

## 7. Business Intelligence Dashboard

Built in Power BI or Tableau, connected to the Gold tables. Planned visuals:
1. **Team possession over time** — a line/area chart showing possession % swinging between Team A and Team B across the match timeline (from `gold_match_summary` + time-windowed possession).
2. **Player movement heatmap** — using `gold_player_movement`, a pitch-overlay heatmap per track_id showing where that player spent most time.
3. **Pass network diagram** — using `gold_pass_events`, a node-edge diagram showing which track_ids passed to which other track_ids most frequently (anonymous "Player 7 → Player 11" style, upgradable to names later).
4. *(Stretch)* **Pipeline accuracy scorecard** — using `gold_tracking_accuracy`, a simple metric card/bar chart showing our tracker's agreement rate against SoccerNet's ground truth — a nice, low-effort way to demonstrate data quality rigor to graders.

Business questions answered: *Which team dominated possession, and when? Which areas of the pitch did a given player operate in? What does this team's passing network look like? How accurate is our own pipeline compared to a labeled benchmark?*

---

## 8. Known Limitations (state these explicitly and confidently in the proposal)

- **Track IDs do not persist across camera cuts/switches.** Each camera shot is tracked independently; cross-camera re-identification is a known hard research problem, flagged as future work. SoccerNet's camera annotations let us *measure* how often our own shot-boundary detection agrees with ground truth, even though we don't use their segmentation directly.
- **Occlusion during crowded scenes** (free kicks, penalties, goal celebrations) can cause track ID swaps or brief ID loss; tracker handles most cases via motion + appearance matching, but this is a documented, acceptable limitation, not a failure of the pipeline. SoccerNet's ground-truth tracklets let us quantify exactly how often this happens (`gold_tracking_accuracy`), rather than just asserting it as a risk.
- **No player naming this semester.** Track IDs remain anonymous; jersey-OCR-to-roster matching is explicitly deferred. Note: because SoccerNet already provides ground-truth jersey numbers for this footage, the *post-semester* naming task is meaningfully de-risked — it becomes matching our own detected track IDs to their existing labels (a spatial/temporal ID-matching problem) rather than building jersey OCR from zero.
- **Team assignment (not player naming) is per-match and lightweight** (color clustering), not a trained/reusable model — this is a strength, not a gap: it generalizes to any team combination for free, and its accuracy is directly checkable against SoccerNet's labeled team field.

---

## 9. Post-Semester Roadmap

1. **Acquire GPU access** (university lab, cloud credits, or continued free-tier Colab/Kaggle Pro if affordable).
2. **Jersey-to-name matching (de-risked by SoccerNet):** since SoccerNet-Tracking already provides ground-truth jersey numbers for this footage, this step becomes matching our own detected track IDs to their existing jersey-number labels (a spatial/temporal alignment problem) rather than building OCR from scratch — meaningfully lower-effort than originally scoped. For footage beyond SoccerNet (e.g. new matches without ground truth), OCR would still be needed and remains the fallback approach.
3. **Cross-camera re-identification:** explore appearance-embedding-based re-ID (e.g., using jersey color + body embeddings) to keep track continuity across camera switches.
4. **Video overlay demo:** render the original video with live bounding boxes + labels ("Player 7," later real names) and a possession trail — this is the visible, demoable product output.
5. **Product packaging:** wrap the pipeline + overlay into a small web app or desktop tool; potential audiences — coaches/analysts (tactical review), content creators (auto-highlight/pass-network graphics), or fans (enhanced match viewing). Explore a freemium/subscription model once naming + overlay are solid.

---

## 10. Cost / FinOps Notes

- All video/frame processing on **free-tier Colab or Kaggle GPU** — no paid compute.
- Databricks **Free/Community Edition** for Spark stages — structured data only, so storage/compute needs stay modest (detection tables, even for several matches, are megabytes-to-low-gigabytes, not raw video).
- Raw video kept in Google Drive, not duplicated into Databricks — only structured outputs cross that boundary, keeping Databricks storage well within free limits.
- Frame sampling (2fps) is the single biggest lever for staying inside free compute/storage limits.
