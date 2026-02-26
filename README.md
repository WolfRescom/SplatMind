# SplatMind

Official teaser repository for our ECCV 2026 submission. We introduce a novel Video World Model integrating 3D Gaussian Splatting (3DGS) as explicit, differentiable memory. Achieved high-quality memory and video generation even for long range video generation.

---

## Overview

Traditional video diffusion models struggle with long term spatial consistency. SplatMind bridges this gap by replacing implicit latent memory with an explicit 3D Gaussian Splatting backend. By maintaining a fully differentiable 3D map of the environment, SplatMind generates extended video sequences while strictly adhering to the geometric constraints of the physical world. 

By conditioning a Video Diffusion Transformer on static renders from a 3DGS spatial memory, our model achieves state of the art structural stability across extended generative horizons.

---

## Architecture Pipeline

Our training and dataset generation pipeline operates in four distinct stages:

1. **Data Curation & Preprocessing:** We process high quality monocular clips from the MiraData dataset. Frames are natively downscaled and cropped to a uniform 480x720 resolution to perfectly align with the 3D VAE latent space requirements while avoiding upscaling artifacts. 
2. **Dynamic Object Isolation (SAM 3):** To ensure our spatial memory does not hallucinate moving objects as permanent fixtures, we utilize SAM 3 to segment and mask out dynamic elements (riders, vehicles, etc.) across the context window.
3. **Explicit Spatial Memory (S3PO):** We feed 49 source frames and their corresponding masks into an S3PO monocular SLAM backend. This builds a persistent 3D Gaussian Splatting map of the static background. We then render 48 future frames from this map to serve as explicit structural guidance.
4. **Action Annotation (Qwen):** We run a Vision Language Model (Qwen) on the target video sequences to generate rich, descriptive text embeddings of the physical actions taking place.
5. **Video Diffusion (CogVideoX-5B):** The final Video World Model leverages the CogVideoX-5B-I2V architecture. It ingests the 49 source frames, the text annotations, and the 48 static 3DGS guidance frames to accurately predict and generate the next 48 frames of the video.

---

## Repository Structure

To avoid dependency conflicts between bleeding edge SLAM repositories and heavy Vision Language Models, the pipeline is orchestrated via isolated sub processes.

```text
SplatMind/
├── dry_run_pipeline.py          # Master orchestrator script
├── Qwen/                        # Action annotation module (runs in isolated environment)
│   └── annotate_actions.py      
├── S3PO-GS/                     # Spatial memory and SLAM backend
│   ├── slam.py
│   ├── generate_masks_SAM.py
│   └── configs/
│       └── mono/
│           └── dry_run.yaml     # RAM disk optimized config
└── video_world_dataset/         # Final packaged training data
    └── clip_001/
        ├── metadata.json
        ├── source_frames/       # Frames 0 to 48
        ├── target_frames/       # Frames 49 to 96
        └── static_guidance/     # 3DGS renders of target trajectory
```

---

## Hardware & Environment Setup

This pipeline is heavily optimized for High Performance Computing (HPC) clusters utilizing massive RAM disks (/dev/shm) and NVIDIA GH200 nodes.

Due to conflicting C++ library requirements, we recommend maintaining strictly separate Conda environments for the spatial memory (`s3po_env`) and the vision language models (`qwen_env`). Standard FFmpeg binaries should be compiled locally or loaded via system modules to prevent Conda Forge dependency overrides.

---

## Results

Integrating 3DGS as explicit memory yields significant improvements in structural consistency over long horizons. Our baseline evaluations on monocular driving and traversal clips demonstrate high fidelity reconstruction metrics (PSNR > 22.0, SSIM > 0.83) during the spatial mapping phase, translating directly to reduced geometric warping in the final diffusion outputs.
