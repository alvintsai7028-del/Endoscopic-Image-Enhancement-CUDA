# Real-Time Endoscopic Image Enhancement System via CUDA & Multi-Threaded C++

## Project Overview
This repository details the architecture, optimization strategies, and performance benchmarks of a high-performance medical video processing pipeline designed for real-time endoscopic procedures. The core objective was to refactor a legacy, high-latency Python prototype into a production-ready C++ application capable of processing high-resolution video streams with zero perceptible lag.

By implementing custom mathematical algorithms, utilizing hardware acceleration via NVIDIA CUDA, and executing multi-threaded concurrency, the system's processing speed was accelerated significantly, achieving real-time performance of nearly 30 FPS for 1280x720 video streams.

***Due to intellectual property protections and non-disclosure agreements, the source code for this project is proprietary and cannot be shared publicly. This document serves as a comprehensive architectural and engineering summary of the implementation.***

---

## System Architecture & Pipeline Flowchart

The system ingests sequential video frames from an endoscopic camera device. To achieve maximum processing efficiency, the framework splits individual color channels and handles computationally intensive operations across parallel CPU threads and GPU kernels.

```mermaid
graph TD
    A[Endoscopic Camera Capture] --> B[Gaussian Pre-filtering GPU]
    B --> C[Split into R, G, B Channels]
    
    subgraph Parallel Channel Processing Multi-Threading
        C -->|Thread 1| D1[R Channel CLAHE GPU]
        C -->|Thread 2| D2[G Channel CLAHE GPU]
        C -->|Thread 3| D3[B Channel CLAHE GPU]
        
        D1 --> E1[Discrete Wavelet Decomposition CPU]
        D2 --> E2[Discrete Wavelet Decomposition CPU]
        D3 --> E3[Discrete Wavelet Decomposition CPU]
        
        E1 --> F1[Guided Filtering CUDA Kernel]
        E2 --> F2[Guided Filtering CUDA Kernel]
        E3 --> F3[Guided Filtering CUDA Kernel]
        
        F1 --> G1[Channel Reconstruction CPU]
        F2 --> G2[Channel Reconstruction CPU]
        F3 --> G3[Channel Reconstruction CPU]
        
        G1 --> H1[Laplacian Filter GPU]
        G2 --> H2[Laplacian Filter GPU]
        G3 --> H3[Laplacian Filter GPU]
    end
    
    H1 --> I[Merge Channels]
    H2 --> I
    H3 --> I
    
    I --> J[Alpha-Beta Blending & Contrast Tuning]
    J --> K[Real-Time Video Display / MP4 Output]
```
## Algorithmic Rationales & Clinical Significance

Endoscopic video streams present unique visualization challenges, including uneven illumination from internal light sources, severe specular reflections (surface glare) on wet tissue, and low contrast within sub-surface vascular structures. The pipeline's specific algorithmic sequence is meticulously structured to address these clinical artifacts without introducing artificial distortion:

* **Gaussian Pre-filtering:** Medical camera sensors inherently introduce high-frequency noise, especially in low-light cavities. Pre-filtering suppresses this noise early in the pipeline, preventing subsequent non-linear contrast enhancement stages from aggressively amplifying background artifacts.
* **CLAHE (Contrast Limited Adaptive Histogram Equalization):** Standard global histogram equalization would over-saturate bright reflections (specularities) and completely wash out structural details. CLAHE limits contrast amplification locally, bringing out subtle mucosal patterns and lesions without causing blooming or loss of details.
* **Discrete Wavelet Decomposition (DWT):** By breaking the image into multi-resolution sub-bands, the system isolates high-frequency structural details (such as tissue texturing and vascular boundaries) from low-frequency illumination shifts. This enables targeted enhancement of critical clinical features.
* **Guided Filtering:** Unlike conventional bilateral filters that often cause gradient reversal artifacts (making biological tissues look unnaturally blocky), the Guided Filter leverages a structural guide to smoothly suppress noise while strictly preserving sharp, critical boundaries of blood vessels.
* **Laplacian Filter:** Applied during the reconstruction stage to provide micro-edge sharpening. This enhances the visibility of fine capillary networks, making it easier to trace micro-vessel contours.
* **Alpha-Beta Blending & Contrast Tuning:** Serving as a final clinical safety checkpoint, this stage blends the highly enhanced texture map back with the original frames. It allows operators to dynamically adjust parameters to their personal visual comfort, ensuring the visual output remains anatomically accurate and customizable for the clinician.

## Multi-Mode Live Demo & Algorithmic Comparison

The enhancement engine features a dynamic runtime architecture supporting three modular visualization modes. The benchmarks below represent three identical frame samples captured from a continuous, real-time enhanced endoscopic video stream. Rather than a linear upgrade, these modes offer **diagnostic flexibility**, allowing clinicians to interactively toggle between views based on their personal visual comfort and specific clinical focus.

| Mode 0: Raw Clinical Input | Mode 1: RGB-Space CLAHE Engine | Mode 2: Dual-Stage Full Optimization |
| :---: | :---: | :---: |
| ![Mode 0 Raw](./mode0.jpg) | ![Mode 1 RGB CLAHE](./mode1.jpg) | ![Mode 2 Dual Stage](./mode2.jpg) |
| **Baseline Stream** | **Multi-Channel RGB CLAHE Pipeline** | **HSV V-Channel + Multi-Channel RGB CLAHE Pipeline** |
| Standard camera acquisition with muted sub-surface vascular details and uneven lighting. | Applies multi-threaded CLAHE independently across R, G, and B channels alongside DWT edge-preservation. | Executes an initial pre-processing CLAHE pass on the HSV V-channel before computing multi-channel RGB loops. |
| *Visual Characteristics:* Standard white-light illumination; high specular reflections (surface glare); deep capillaries remain soft and blurry. | *Visual Characteristics:* Noticeable amplification of capillary sharpness and localized contrast while maintaining a color spectrum relatively closer to white-light endoscopy. | *Visual Characteristics:* Aggressive accentuation of structural borders and micro-vessel topography, introducing a high-contrast hue that heavily isolates vascular networks. |

---

### Clinical Flexibility & Engineering Design Decisions

Medical imaging demands personalizability; different procedures and different clinicians require distinct visual cues. The system provides an interactive control panel for seamless runtime switching between Mode 1 and Mode 2 to accommodate different diagnostic scenarios:

* **Mode 1 (RGB-Space CLAHE): Balanced Full-Spectrum Visualization**
  By applying enhancement directly within the RGB channels, this mode delivers a uniform boost to local textures and edges while trying to preserve the overall organic tone of the tissue. This is ideal for clinicians who prefer a enhanced view that still closely mimics standard white-light endoscopy, avoiding radical color shifts during general screening.

* **Mode 2 (Dual-Stage HSV V-Channel + RGB): High-Contrast Structural Isolation**
  By mapping the frames to the **HSV color space** first, we isolate the **Luminance/Value (V)** channel. Running a primary CLAHE pass on the V-channel normalizes global light distribution and diffuses blinding specular glares *before* entering the multi-channel RGB enhancement loops. 
  
  This dual-stage approach forces the algorithm to focus heavily on the structural boundaries of blood vessels and mucosal texturing. The resulting image heavily emphasizes micro-vessel contours (yielding the distinct, high-contrast definition seen in the final frame), making it highly effective for targeted examinations where tracing fine capillary patterns is the primary objective.

## Technical Implementation Details

### 1. Algorithmic Refactoring & Mathematical Optimization
* **Hand-Crafted Mathematical Modules:** Because standard C++ libraries lack native support for Discrete Wavelet Transforms (DWT) and Guided Filtering, these components were engineered entirely from scratch.
* **Function-Level Acceleration:** The core algorithm's internal execution path was optimized using mathematical logic and structural consolidation to minimize computational redundancy and reduce per-frame execution time.

### 2. Hardware Acceleration via NVIDIA CUDA
* **Environment Architecture:** Configured a high-performance development environment from the ground up using Visual Studio, CMake, OpenCV (including `opencv_contrib`), and the NVIDIA CUDA toolkit.
* **GPU-Accelerated Matrix Operations:** Offloaded heavily repeated operations—specifically matrix computations within the Guided Filter and native OpenCV functions—directly onto GPU kernels to bypass sequential CPU execution bottlenecks.

### 3. Multi-Threaded Concurrency
* **Parallel Channel Processing:** Because sequential processing of individual image channels using standard loops was highly inefficient, the pipeline splits the frame into three discrete channels and processes them simultaneously.
* **Threaded Acceleration:** Utilizing `std::thread`, three concurrent CPU threads execute image enhancement tasks in parallel, maximizing multi-core processor efficiency and enabling high-throughput streaming.

### 4. Dynamic Parameter Control & User Interface
* **Interactive Control Panel:** Interactive Control Panel: Designed an enhanced real-time application featuring an integrated graphical user interface (GUI) with sliders, enabling operators to dynamically adjust algorithm parameters in real time.
* **Configurable Optimization Pipelines:** Users can interactively tune parameters such as thresholding limits, filter kernels, and visualization modes (e.g., side-by-side comparison or dual-lens stream views).
* **V-Channel Contrast Enhancement:** Integrated an optional pipeline expansion that applies Contrast Limited Adaptive Histogram Equalization (CLAHE) specifically to the V (Value) channel within the HSV color space for enhanced detail visualization.

## Quantitative Performance Indicators

Through structural algorithmic refactoring, hardware acceleration, and concurrency tuning, the system achieved a massive reduction in frame processing latency, successfully enabling smooth, real-time visualization.

| Optimization Phase | Execution Time Per Frame (1280x720) | Equivalent Frame Rate (FPS) | Performance Gain |
| :--- | :--- | :--- | :--- |
| **Python Prototype (Baseline)** | 3.52 seconds | ~0.28 FPS | Baseline |
| **Naive C++ Port (Logic Tweaks)** | 1.48 seconds | ~0.68 FPS | 2.38x Acceleration |
| **CUDA + Multi-Threaded C++** | 0.035 seconds | ~28.6 FPS | **100.57x Acceleration** |

### Core Benchmarks & Key Metrics
* **Latency Reduction:** Frame processing time for a standard 1280x720 resolution stream dropped from 3.52 seconds to 1.48 seconds after basic C++ logic modifications, and was ultimately minimized to 0.035 seconds through advanced acceleration.
* **Real-Time Throughput:** By shifting critical matrix computations to the GPU and implementing multi-threaded processing while omitting non-essential Super-Resolution overhead, the pipeline maintains a consistent throughput of nearly 30 FPS.
* **Dynamic Resolution Scalability:** The framework delivers lag-free, real-time processing across various endoscopic instruments by dynamically adjusting the pre-processing magnification scale based on the target camera's input resolution.

## Engineering Challenges & Key Learnings

### Technical Challenges & Bottleneck Resolution

* **Low-Level Algorithmic Reconstruction:** While the initial prototype relied heavily on high-level Python built-in functions, standard C++ libraries lacked native modules for Discrete Wavelet Transforms (DWT) and Guided Filtering. I engineered these core mathematical functions entirely from scratch, overcoming the obstacle of sparse online reference documentation.
* **Complex Dependency & Version Compatibility:** Establishing the high-performance development ecosystem presented significant configuration challenges. I successfully built a unified runtime pipeline integrating Visual Studio, OpenCV, and the NVIDIA CUDA toolkit from the ground up, resolving complex version compatibility conflicts and overcoming environment disparities between desktop and laptop hardware that caused standard tutorials to fail.
* **Cross-Platform Dynamic Library Deployment:** To prepare the product for commercial vendor delivery, the framework had to be compiled on a desktop workstation as a Dynamic Link Library (DLL) and successfully deployed on a target testing laptop. This required managing strict environment synchronization and OpenCV version parity across devices to prevent critical linking failures during execution.

### Key Learnings & Skill Acquisition

* **Hardware Acceleration & Concurrency:** Gained hands-on expertise in GPU-accelerated computing via CUDA kernels and multi-threaded parallel processing using `std::thread` to maximize hardware processing throughput.
* **Custom Source Compilation:** Mastered low-level build configurations by utilizing CMake to build and compile OpenCV and `opencv_contrib` from source, unlocking advanced GPU and hardware-optimization modules.
* **Real-Time Signal Ingestion:** Acquired practical engineering experience interfacing with specialized medical hardware, successfully capturing, reading, and streaming live endoscopic video input.
* **High-Performance Image Enhancement:** Deployed production-grade video processing pipelines optimized for real-time visualization and boundary/edge reinforcement.
* **Modular Software Architecture:** Mastered the concepts of dynamic linking by building custom DLLs for cross-environment integration and explored GUI application workflows.
