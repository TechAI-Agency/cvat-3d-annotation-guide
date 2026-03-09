# 3D Point Cloud Annotation on Self-Hosted CVAT

A practical guide for teams running 3D LiDAR annotation on self-hosted CVAT. Based on real production experience annotating KITTI-format point clouds for autonomous vehicle and robotics perception systems.

## Who This Is For

- Teams self-hosting CVAT who need to annotate 3D point cloud data
- ML engineers setting up LiDAR annotation pipelines for the first time
- Annotation managers transitioning from 2D to 3D workflows

## KITTI Format Upload Structure

CVAT expects a specific folder structure for 3D point cloud tasks. Getting this wrong is the #1 reason uploads fail silently.

```
project_folder/
├── velodyne_points/
│   └── data/
│       ├── 0000000000.bin
│       ├── 0000000001.bin
│       ├── 0000000002.bin
│       └── ...
└── image_00/
    └── data/
        ├── 0000000000.png
        ├── 0000000001.png
        ├── 0000000002.png
        └── ...
```

**Critical details:**
- Filenames must be 10-digit zero-padded (e.g., `0000000000.bin`)
- `.bin` files contain raw point cloud data (x, y, z, intensity as float32)
- `.png` files are the corresponding camera images for reference
- Both folders must have matching filenames

## Upload Workflow (Docker + API)

### Step 1: Zip your data
```bash
zip -r pointcloud_batch.zip velodyne_points/ image_00/
```

### Step 2: Copy to CVAT server
```bash
docker cp pointcloud_batch.zip cvat_server:/tmp/
```

### Step 3: Create task via API
```bash
curl -X POST "http://your-server:8080/api/tasks" \
  -H "Authorization: Token YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "LiDAR Batch 001",
    "labels": [
      {"name": "Car"},
      {"name": "Pedestrian"},
      {"name": "Cyclist"},
      {"name": "Truck"}
    ],
    "data_type": "3d"
  }'
```

### Step 4: Upload data to task
```bash
curl -X POST "http://your-server:8080/api/tasks/{task_id}/data" \
  -H "Authorization: Token YOUR_TOKEN" \
  -F "client_files[0]=@/tmp/pointcloud_batch.zip"
```

## Common Issues and Fixes

| Problem | Cause | Fix |
|---------|-------|-----|
| Upload succeeds but no frames appear | Wrong folder structure | Verify `velodyne_points/data/` path exactly |
| Point cloud renders but no image overlay | Mismatched filenames | Ensure .bin and .png files have identical names |
| "Task creation failed" error | File too large for default config | Increase CVAT_CHUNK_SIZE in docker-compose |
| Cuboids don't persist across frames | Object tracking not enabled | Enable "Track" mode before drawing cuboids |
| Slow rendering on large point clouds | Browser memory limits | Reduce points per frame or use Chrome with increased memory flag |

## Annotation Best Practices for 3D Cuboids

### Edge Cases That Break Models

The hardest part of 3D annotation isn't the tooling — it's the judgment calls:

- **Sparse points at range:** Objects beyond 40m may have only 10-15 points. Annotators need clear guidelines on minimum point thresholds for labeling vs. skipping.
- **Overlapping objects:** Two objects close together with interleaved points. Define whose points belong to which cuboid.
- **Partially occluded objects:** A car behind a truck with only the roof visible. Annotate the full estimated cuboid or only the visible portion? Document your decision.
- **Ground plane ambiguity:** Cuboid bottom face should sit on the ground plane, but uneven terrain makes this inconsistent. Establish a tolerance.

### QA Process for 3D Annotation

1. **Gold standard frames:** Create 50-100 expert-annotated frames as ground truth
2. **Random sampling:** QA review 10% of all production frames against gold standard
3. **IoU threshold:** 3D cuboids should achieve >0.7 IoU with ground truth
4. **Cross-frame consistency:** Track IDs must persist across consecutive frames — check for ID switches
5. **Per-annotator metrics:** Track individual accuracy to identify retraining needs early

## Export Formats

CVAT supports multiple export formats for 3D annotations:

- **CVAT XML 3D** — native format, preserves all metadata
- **KITTI** — standard for autonomous driving benchmarks
- **nuScenes** — required for nuScenes-compatible pipelines
- **Custom JSON** — via CVAT API for custom pipeline integration

## Infrastructure Notes

For production 3D annotation at scale:

- **Self-host on dedicated hardware** — cloud CVAT works for small projects but latency kills productivity on large point clouds
- **UFW firewall** — restrict to ports 22 (SSH) and 8080 (CVAT) only
- **Disable public registration** — manage annotator accounts manually for security
- **Backup strategy** — CVAT stores annotations in PostgreSQL; schedule daily pg_dump

## About

This guide is maintained by [TechAI Remote](https://techairemote.com), a Nairobi-based annotation team specializing in 3D LiDAR and edge case annotation for computer vision, robotics, and autonomous vehicles.

Questions or feedback: contact@techairemote.com
