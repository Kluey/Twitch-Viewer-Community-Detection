# Twitch Viewer Community Detection at Scale

A GPU-accelerated, distributed pipeline that detects communities of Twitch viewers based on shared streamer-watching behavior, and benchmarks how the pipeline itself scales with data size and worker count.

## What this does

Given a log of which users watched which streamers, the pipeline:

1. **Loads and filters** the interaction log with Dask.
2. **Builds per-user "streamer sets"** - the set of streamers each user watched.
3. **Approximates pairwise user similarity** using MinHash + Locality-Sensitive Hashing (LSH), avoiding the O(n^2) cost of exact all-pairs Jaccard similarity.
4. **Filters candidate pairs** by a Jaccard similarity threshold to build a weighted similarity graph.
5. **Detects communities** with the Louvain algorithm, GPU-accelerated via RAPIDS cuGraph, and evaluates them with modularity.

Alongside the modeling question, the notebook benchmarks the pipeline's own performance: strong scaling (runtime vs. user count) and weak scaling (runtime vs. worker count at a fixed per-worker workload).

## Key result

Community modularity increases from **0.67 to 0.70** as the sample grows from 25k to 100k users, indicating the similarity graph has real, increasingly well-resolved community structure rather than noise.

The more interesting finding came from the scaling benchmarks: **MinHash signature generation, not GPU graph clustering, is the pipeline's bottleneck** - at 100k users it accounts for ~80% of total runtime while Louvain on the GPU takes under 2 seconds. Profiling also showed why weak-scaling efficiency drops to ~18% at 8 workers: only the initial data load is Dask-parallelized, while MinHash/LSH construction runs single-threaded regardless of cluster size. Full writeup in the notebook's Discussion section.

## Tech stack

- **Dask**: distributed data loading and filtering
- **datasketch (MinHash, MinHashLSH)**: approximate similarity search
- **RAPIDS cuDF / cuGraph**: GPU-accelerated graph construction and Louvain community detection
- **Google Colab (T4 GPU)**: execution environment

## Dataset

[Twitch Interactions dataset (UCSD)](https://cseweb.ucsd.edu/~jmcauley/datasets.html#twitch) - user-streamer viewing session logs. This notebook uses the 100k-user subset (`100k_a.csv`); the full dataset can be substituted by uncommenting the relevant cells.

## Running it

Open `twitch_community_detection.ipynb` in Google Colab with a GPU runtime (T4 or better). The first code cell installs RAPIDS (cuDF/cuGraph) and `datasketch`; everything else runs top to bottom.

## Possible next steps

- Vectorize or GPU-accelerate MinHash signature generation to remove the current bottleneck.
- Distribute the LSH/Jaccard candidate-generation stage across Dask workers instead of running it single-threaded on the driver.
