# Example Model System

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examplemodel/MLproject](examplemodel/MLproject)
- [examplemodel/README.md](examplemodel/README.md)
- [examplemodel/src/train.py](examplemodel/src/train.py)

</details>



## Purpose and Scope

The Example Model System is a complete end-to-end machine learning pipeline for refugee camp detection in satellite imagery. This system demonstrates a production-ready implementation of a geospatial AI model with standardized metadata, multiple deployment formats, and reproducible training workflows.

This document covers the ML pipeline components, training and inference workflows, model architecture, output artifact generation, and MLflow orchestration. For infrastructure setup and deployment, see [Infrastructure System](#4). For specific deployment strategies to different platforms, see [Model Deployment Options](#6.2).

**Sources:** [examplemodel/README.md:1-52](), [examplemodel/src/train.py:1-519](), [examplemodel/MLproject:1-63]()

---

## System Overview

The Example Model System implements a semantic segmentation pipeline using U-Net architecture to identify refugee camps in satellite imagery. The system is structured around five distinct entry points orchestrated by MLflow, each handling a specific phase of the ML lifecycle.

### High-Level Architecture

```mermaid
graph TB
    subgraph "MLflow Entry Points"
        EP_PREPROCESS["preprocess<br/>Entry Point"]
        EP_TRAIN["train<br/>Entry Point"]
        EP_INFERENCE["inference<br/>Entry Point"]
        EP_VALIDATE["validate_stac<br/>Entry Point"]
        EP_STAC2ESRI["stac2esri<br/>Entry Point"]
    end
    
    subgraph "Core Python Modules"
        PREPROCESS_PY["preprocess.py"]
        TRAIN_PY["train.py"]
        INFERENCE_PY["inference.py"]
        MODEL_PY["model.py"]
        STAC2ESRI_PY["stac2esri.py"]
        VALIDATE_PY["validate_stac_mlm.py"]
    end
    
    subgraph "Model Components"
        CAMP_DATA["CampDataModule<br/>PyTorch Lightning DataModule"]
        LIT_MODEL["LitRefugeeCamp<br/>PyTorch Lightning Module"]
        UNET["U-Net Neural Network"]
    end
    
    subgraph "ESRI Integration"
        DETECTOR["RefugeeCampDetector<br/>Inference Class"]
        EMD["model.emd<br/>Esri Model Definition"]
    end
    
    subgraph "Output Artifacts"
        PTH["best_model.pth<br/>State Dictionary"]
        PT["best_model.pt<br/>TorchScript Traced"]
        ONNX["best_model.onnx<br/>ONNX Format"]
        DLPK["best_model.dlpk<br/>ESRI Package"]
        STAC["stac_item.json<br/>STAC-MLM Metadata"]
    end
    
    EP_PREPROCESS -->|executes| PREPROCESS_PY
    EP_TRAIN -->|executes| TRAIN_PY
    EP_INFERENCE -->|executes| INFERENCE_PY
    EP_VALIDATE -->|executes| VALIDATE_PY
    EP_STAC2ESRI -->|executes| STAC2ESRI_PY
    
    TRAIN_PY -->|imports| MODEL_PY
    TRAIN_PY -->|imports| INFERENCE_PY
    TRAIN_PY -->|imports| STAC2ESRI_PY
    
    MODEL_PY -->|defines| CAMP_DATA
    MODEL_PY -->|defines| LIT_MODEL
    LIT_MODEL -->|contains| UNET
    
    TRAIN_PY -->|instantiates| CAMP_DATA
    TRAIN_PY -->|trains| LIT_MODEL
    
    TRAIN_PY -->|produces| PTH
    TRAIN_PY -->|produces| PT
    TRAIN_PY -->|produces| ONNX
    TRAIN_PY -->|produces| STAC
    TRAIN_PY -->|produces| DLPK
    
    STAC2ESRI_PY -->|creates| DLPK
    STAC2ESRI_PY -->|uses| DETECTOR
    STAC2ESRI_PY -->|generates| EMD
```

**Sources:** [examplemodel/MLproject:1-63](), [examplemodel/src/train.py:1-519]()

---

## MLflow Entry Points

The system defines five entry points in the `MLproject` file, each representing a distinct phase of the ML workflow.

### Entry Point Configuration

| Entry Point | Module | Purpose | Key Parameters |
|-------------|--------|---------|----------------|
| `preprocess` | `src/preprocess.py` | Data acquisition from TMS and OSM | `zoom`, `bbox`, `tms`, `train_dir` |
| `train` | `src/train.py` | Model training and artifact generation | `epochs`, `batch_size`, `chips_dir`, `labels_dir`, `lr` |
| `inference` | `src/inference.py` | Model prediction on new images | `image_path`, `model_path`, `output_dir`, `mlflow_tracking` |
| `validate_stac` | `validate_stac_mlm.py` | STAC-MLM metadata validation | `stac_file` |
| `stac2esri` | `src/stac2esri.py` | DLPK package generation for ArcGIS | `stac_path`, `onnx_path`, `out_dir`, `dlpk_name` |

**Sources:** [examplemodel/MLproject:5-63]()

### Entry Point Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant MLflow as "MLflow CLI"
    participant UV as "uv Package Manager"
    participant Module as "Python Module"
    participant Artifacts as "Artifact Storage"
    
    User->>MLflow: "mlflow run . -e preprocess"
    MLflow->>UV: "uv run python src/preprocess.py --zoom 19 --bbox ..."
    UV->>Module: "Execute with parameters"
    Module->>Module: "Fetch TMS imagery"
    Module->>Module: "Fetch OSM labels"
    Module->>Artifacts: "Save chips/ and labels/"
    
    User->>MLflow: "mlflow run . -e train"
    MLflow->>UV: "uv run python src/train.py --epochs 1 ..."
    UV->>Module: "Execute train_model()"
    Module->>Module: "Call get_system_info()"
    Module->>Module: "Call calculate_dataset_statistics()"
    Module->>Module: "Initialize CampDataModule"
    Module->>Module: "Train LitRefugeeCamp"
    Module->>Module: "Call create_stac_mlm_item()"
    Module->>Module: "Call create_dlpk()"
    Module->>Artifacts: "Save .pth, .pt, .onnx, .dlpk, stac_item.json"
    
    User->>MLflow: "mlflow run . -e inference --image_path=test.jpg"
    MLflow->>UV: "uv run python src/inference.py test.jpg"
    UV->>Module: "Load model and predict"
    Module->>Artifacts: "Save prediction outputs"
```

**Sources:** [examplemodel/MLproject:5-63](), [examplemodel/src/train.py:370-519]()

---

## Training Pipeline Components

The training pipeline is orchestrated by the `train_model()` function in `train.py`, which coordinates multiple subsystems.

### Training Pipeline Architecture

```mermaid
graph LR
    subgraph "train_model Function"
        START["train_model(args)"]
        SYSINFO["get_system_info()"]
        STATS["calculate_dataset_statistics()"]
        DATAMOD["CampDataModule instantiation"]
        TRAINER["pl.Trainer configuration"]
        TRAINING["trainer.fit()"]
        CKPT["ModelCheckpoint callback"]
        CONVERT["Model format conversions"]
        STAC["create_stac_mlm_item()"]
        DLPK_CREATE["create_dlpk()"]
        LOG["MLflow logging"]
    end
    
    subgraph "PyTorch Lightning"
        LITMODEL["LitRefugeeCamp"]
        UNET_MODEL["self.model<br/>U-Net"]
    end
    
    subgraph "Artifacts Generated"
        PTH_FILE["best_model.pth"]
        PT_FILE["best_model.pt<br/>TorchScript"]
        ONNX_FILE["best_model.onnx"]
        STAC_FILE["stac_item.json"]
        DLPK_FILE["best_model.dlpk"]
        EMD_FILE["model.emd"]
    end
    
    START --> SYSINFO
    START --> STATS
    START --> DATAMOD
    START --> TRAINER
    TRAINER --> CKPT
    TRAINER --> TRAINING
    TRAINING --> LITMODEL
    LITMODEL --> UNET_MODEL
    TRAINING --> CONVERT
    CONVERT --> PTH_FILE
    CONVERT --> PT_FILE
    CONVERT --> ONNX_FILE
    START --> STAC
    START --> DLPK_CREATE
    STAC --> STAC_FILE
    DLPK_CREATE --> DLPK_FILE
    DLPK_CREATE --> EMD_FILE
    CONVERT --> LOG
    STAC --> LOG
    DLPK_CREATE --> LOG
```

**Sources:** [examplemodel/src/train.py:370-519]()

### Key Functions in train.py

#### get_system_info()

Collects comprehensive system and hardware information for reproducibility tracking.

**Function signature:** `def get_system_info() -> Dict[str, Any]:`

**Returns:** Dictionary containing:
- Platform information (`platform`, `architecture`, `processor`)
- Python version
- Hardware specs (`cpu_count`, `memory_total`, `memory_available`)
- PyTorch and CUDA versions
- GPU information via `pynvml` (name, memory stats)

**Sources:** [examplemodel/src/train.py:25-61]()

#### calculate_dataset_statistics()

Computes pixel-level statistics across the training dataset for normalization.

**Function signature:** `def calculate_dataset_statistics(data_module: CampDataModule) -> Dict[str, Any]:`

**Process:**
1. Iterates through training dataloader
2. Accumulates channel sums and squared sums
3. Calculates mean and standard deviation per channel
4. Counts samples in train/val/test splits

**Returns:** Dictionary with dataset statistics including `train_samples`, `val_samples`, `test_samples`, `pixel_mean`, `pixel_std`

**Sources:** [examplemodel/src/train.py:63-99]()

#### create_stac_mlm_item()

Generates STAC-MLM compliant metadata describing the trained model.

**Function signature:** `def create_stac_mlm_item(model, dataset_stats, system_info, model_performance, checkpoint_path) -> Dict[str, Any]:`

**STAC Item Structure:**
- **Core properties:** `id`, `geometry`, `bbox`, `datetime`
- **MLM properties:** Model architecture, framework, parameters, I/O specifications
- **Processing properties:** Training environment details
- **Assets:** Links to all model artifacts (PyTorch, ONNX, DLPK, etc.)

**Key STAC Extensions Used:**
- `mlm/v1.5.0` - Machine Learning Model extension
- `file/v1.0.0` - File information
- `processing/v1.1.0` - Processing provenance

**Sources:** [examplemodel/src/train.py:101-368]()

#### train_model()

Main training orchestration function that coordinates all pipeline components.

**Function signature:** `def train_model(args):`

**Execution Flow:**
1. Start MLflow run
2. Collect system info
3. Initialize `CampDataModule` with data directories
4. Calculate dataset statistics
5. Configure `ModelCheckpoint` callback (monitors `val_loss`)
6. Initialize `pl.Trainer` with auto-accelerator detection
7. Train `LitRefugeeCamp` model
8. Convert best checkpoint to multiple formats:
   - `.pth` - Raw state dictionary
   - `.pt` - TorchScript traced model
   - `.onnx` - ONNX format
9. Generate STAC-MLM metadata
10. Create ESRI DLPK package via `create_dlpk()`
11. Log all artifacts to MLflow
12. Log model with signature for MLflow Model Registry

**Sources:** [examplemodel/src/train.py:370-519]()

---

## Model Format Conversions

The training pipeline produces four distinct model formats for different deployment scenarios.

### Conversion Process

```mermaid
graph TB
    subgraph "Source Model"
        CHECKPOINT["best_model_path<br/>PyTorch Lightning Checkpoint"]
    end
    
    subgraph "Loaded Model"
        LOADED["LitRefugeeCamp.load_from_checkpoint()"]
        STATE_DICT["model.state_dict()"]
        CLEAN_MODEL["clean_model = LitRefugeeCamp()"]
        TORCH_MODEL["torch_model = clean_model.model"]
    end
    
    subgraph "Format Conversions"
        SAVE_PTH["torch.save(state_dict)"]
        TRACE["torch.jit.trace()"]
        ONNX_EXPORT["torch.onnx.export()"]
    end
    
    subgraph "Output Files"
        PTH["meta/best_model.pth<br/>State Dictionary"]
        PT["meta/best_model.pt<br/>TorchScript Traced"]
        ONNX["meta/best_model.onnx<br/>ONNX Format"]
    end
    
    subgraph "ESRI Package Creation"
        DLPK_FUNC["create_dlpk()"]
        DLPK["meta/best_model.dlpk<br/>Deep Learning Package"]
    end
    
    CHECKPOINT --> LOADED
    LOADED --> STATE_DICT
    STATE_DICT --> CLEAN_MODEL
    CLEAN_MODEL --> TORCH_MODEL
    
    STATE_DICT --> SAVE_PTH
    SAVE_PTH --> PTH
    
    TORCH_MODEL --> TRACE
    TRACE --> PT
    
    TORCH_MODEL --> ONNX_EXPORT
    ONNX_EXPORT --> ONNX
    
    PT --> DLPK_FUNC
    ONNX --> DLPK_FUNC
    DLPK_FUNC --> DLPK
```

**Sources:** [examplemodel/src/train.py:433-469]()

### Format Specifications

| Format | File Extension | Purpose | Generation Code |
|--------|---------------|---------|-----------------|
| State Dictionary | `.pth` | Raw PyTorch weights for flexible loading | `torch.save(model.state_dict(), "meta/best_model.pth")` |
| TorchScript | `.pt` | Traced model for optimized inference | `torch.jit.trace(torch_model, torch.randn(1,3,256,256))` |
| ONNX | `.onnx` | Cross-platform interoperability | `torch.onnx.export(torch_model, ...)` with opset 11 |
| ESRI DLPK | `.dlpk` | ArcGIS Pro deployment package | `create_dlpk(emd_path, pt_path, esri_inference_path, dlpk_path)` |

**Key Implementation Details:**

**TorchScript Export:** Uses `torch.jit.trace()` on the inner neural network (`clean_model.model`) rather than the Lightning wrapper to ensure compatibility.

**ONNX Export Parameters:**
- `opset_version=11` for broad compatibility
- Dynamic axes for batch dimension: `{"input": {0: "batch_size"}, "output": {0: "batch_size"}}`
- Input/output names: `["input"]` and `["output"]`

**Sources:** [examplemodel/src/train.py:438-469]()

---

## STAC-MLM Metadata Generation

The system generates comprehensive STAC (SpatioTemporal Asset Catalog) metadata following the Machine Learning Model (MLM) extension specification v1.5.0.

### STAC Item Structure

```mermaid
graph TB
    subgraph "STAC Item Core"
        TYPE["type: Feature"]
        VERSION["stac_version: 1.1.0"]
        ID["id: refugee-camp-detector-v1.0.0"]
        GEOMETRY["geometry: Polygon"]
    end
    
    subgraph "STAC Extensions"
        MLM_EXT["mlm/v1.5.0<br/>Machine Learning Model"]
        FILE_EXT["file/v1.0.0<br/>File Information"]
        PROC_EXT["processing/v1.1.0<br/>Processing Provenance"]
    end
    
    subgraph "Properties"
        META["Descriptive metadata<br/>title, description, keywords"]
        MLM_PROPS["mlm:* properties<br/>architecture, tasks, framework"]
        INPUT_SPEC["mlm:input<br/>shape, normalization, bands"]
        OUTPUT_SPEC["mlm:output<br/>shape, classes, data_type"]
        PROC_PROPS["processing:* properties<br/>facility, software, expression"]
    end
    
    subgraph "Assets"
        PYTORCH_CKPT["pytorch-checkpoint<br/>Lightning checkpoint"]
        PYTORCH_SD["pytorch-state-dict<br/>TorchScript model"]
        PYTORCH_RAW["pytorch-state-dict-raw<br/>Raw .pth file"]
        ONNX_ASSET["onnx-model<br/>ONNX format"]
        DLPK_ASSET["esri-package<br/>DLPK file"]
        CODE_ASSETS["source-code, inference-script,<br/>training-script"]
        VIZ_ASSETS["confusion-matrix, example-input,<br/>example-prediction"]
    end
    
    TYPE --> GEOMETRY
    VERSION --> MLM_EXT
    MLM_EXT --> MLM_PROPS
    MLM_PROPS --> INPUT_SPEC
    MLM_PROPS --> OUTPUT_SPEC
    PROC_EXT --> PROC_PROPS
    
    ID --> PYTORCH_CKPT
    ID --> PYTORCH_SD
    ID --> PYTORCH_RAW
    ID --> ONNX_ASSET
    ID --> DLPK_ASSET
    ID --> CODE_ASSETS
    ID --> VIZ_ASSETS
```

**Sources:** [examplemodel/src/train.py:101-368]()

### Key STAC-MLM Properties

#### Model Architecture Properties

```
mlm:name: "RefugeeCampDetector"
mlm:architecture: "U-Net"
mlm:tasks: ["semantic-segmentation"]
mlm:framework: "PyTorch"
mlm:framework_version: torch.__version__
mlm:total_parameters: sum(p.numel() for p in model.parameters())
mlm:memory_size: sum(p.numel() * p.element_size() for p in model.parameters())
```

**Sources:** [examplemodel/src/train.py:172-180]()

#### Input Specification

The input specification defines expected image format and normalization:

```json
{
  "name": "satellite_image",
  "bands": ["red", "green", "blue"],
  "input": {
    "shape": [-1, 3, 256, 256],
    "dim_order": ["batch", "bands", "height", "width"],
    "data_type": "float32"
  },
  "value_scaling": [
    {"type": "z-score", "mean": 0.485, "stddev": 0.229},
    {"type": "z-score", "mean": 0.456, "stddev": 0.224},
    {"type": "z-score", "mean": 0.406, "stddev": 0.225}
  ]
}
```

Uses ImageNet normalization statistics for transfer learning compatibility.

**Sources:** [examplemodel/src/train.py:190-206]()

#### Output Specification

Defines the segmentation mask output format:

```json
{
  "name": "segmentation_mask",
  "result": {
    "shape": [-1, 1, 256, 256],
    "dim_order": ["batch", "channel", "height", "width"],
    "data_type": "float32"
  },
  "classification:classes": [
    {"value": 0, "name": "background"},
    {"value": 1, "name": "refugee_camp"}
  ]
}
```

**Sources:** [examplemodel/src/train.py:207-230]()

### Asset Catalog

The STAC item catalogs 15 distinct assets across multiple categories:

| Asset Key | Type | Role | File Path |
|-----------|------|------|-----------|
| `pytorch-checkpoint` | Lightning checkpoint | `mlm:model`, `mlm:checkpoint` | Path from checkpoint callback |
| `pytorch-state-dict` | TorchScript | `mlm:model`, `mlm:weights` | `meta/best_model.pt` |
| `pytorch-state-dict-raw` | State dict | `mlm:model`, `mlm:weights` | `meta/best_model.pth` |
| `onnx-model` | ONNX | `mlm:model`, `mlm:inference` | `meta/best_model.onnx` |
| `esri-package` | DLPK ZIP | `mlm:model` | `meta/best_model.dlpk` |
| `source-code` | Repository link | `mlm:source_code`, `code` | GitHub URL |
| `inference-script` | Python script | `mlm:source_code` | `src/inference.py` |
| `training-script` | Python script | `mlm:training`, `code` | `src/train.py` |
| `container-image` | OCI image | `mlm:container`, `runtime` | GHCR URL |
| `confusion-matrix` | PNG | `metadata`, `overview` | `meta/confusion_matrix.png` |
| `example-input` | PNG | `metadata`, `overview` | `meta/example_input.png` |
| `example-prediction` | PNG | `metadata`, `overview` | `meta/example_pred.png` |
| `example-target` | PNG | `metadata`, `overview` | `meta/example_target.png` |
| `model-metadata` | EMD JSON | `metadata` | `meta/model.emd` |
| `requirements` | Text file | `runtime`, `metadata` | `requirements.txt` |

**Sources:** [examplemodel/src/train.py:246-364]()

---

## MLflow Integration

The training pipeline integrates with MLflow for experiment tracking, artifact logging, and model registry.

### MLflow Logging Architecture

```mermaid
graph LR
    subgraph "Training Process"
        RUN_START["mlflow.start_run()"]
        LOG_PARAMS["mlflow.log_params()"]
        LOG_METRICS["mlflow.log_metrics()"]
        LOG_ARTIFACTS["mlflow.log_artifact()"]
        LOG_MODEL["mlflow.pytorch.log_model()"]
    end
    
    subgraph "Logged Parameters"
        DATASET_PARAMS["train_samples, val_samples,<br/>test_samples, num_classes"]
    end
    
    subgraph "Logged Metrics"
        PERFORMANCE["best_val_loss,<br/>epochs_trained"]
    end
    
    subgraph "Logged Artifacts"
        METADATA_DIR["metadata/<br/>stac_item.json"]
        MODELS_DIR["models/<br/>pth, pt, onnx, dlpk"]
        CHECKPOINTS_DIR["checkpoints/<br/>Lightning checkpoint"]
        DATASETS_DIR["datasets/train/<br/>chips/, labels/"]
        ESRI_DIR["esri/<br/>dlpk, emd, pt, RefugeeCampDetector.py"]
    end
    
    subgraph "Model Registry"
        MODEL_ARTIFACT["model/<br/>PyTorch model with signature"]
        SIGNATURE["infer_signature()"]
    end
    
    RUN_START --> LOG_PARAMS
    LOG_PARAMS --> DATASET_PARAMS
    RUN_START --> LOG_METRICS
    LOG_METRICS --> PERFORMANCE
    RUN_START --> LOG_ARTIFACTS
    LOG_ARTIFACTS --> METADATA_DIR
    LOG_ARTIFACTS --> MODELS_DIR
    LOG_ARTIFACTS --> CHECKPOINTS_DIR
    LOG_ARTIFACTS --> DATASETS_DIR
    LOG_ARTIFACTS --> ESRI_DIR
    RUN_START --> LOG_MODEL
    LOG_MODEL --> SIGNATURE
    LOG_MODEL --> MODEL_ARTIFACT
```

**Sources:** [examplemodel/src/train.py:373-507]()

### Artifact Organization

The system logs artifacts in a structured directory hierarchy:

```
mlflow_run/
├── metadata/
│   └── stac_item.json
├── models/
│   ├── best_model.pth
│   ├── best_model.pt
│   ├── best_model.onnx
│   └── best_model.dlpk
├── checkpoints/
│   └── epoch={epoch}-step={step}.ckpt
├── datasets/
│   └── train/
│       ├── chips/
│       └── labels/
├── esri/
│   ├── best_model.dlpk
│   ├── model.emd
│   ├── best_model.pt
│   └── RefugeeCampDetector.py
└── model/
    └── [PyTorch model with signature]
```

**Implementation:**
```python
mlflow.log_artifact(stac_output_path, artifact_path="metadata")
mlflow.log_artifact("meta/best_model.pth", artifact_path="models")
mlflow.log_artifact("meta/best_model.pt", artifact_path="models")
mlflow.log_artifact("meta/best_model.onnx", artifact_path="models")
mlflow.log_artifact("meta/best_model.dlpk", artifact_path="models")
mlflow.log_artifact(best_checkpoint, artifact_path="checkpoints")
mlflow.log_artifact(args.chips_dir, artifact_path="datasets/train/chips")
mlflow.log_artifact(args.labels_dir, artifact_path="datasets/train/labels")
```

**Sources:** [examplemodel/src/train.py:479-496]()

### Model Signature

The system uses `infer_signature()` to capture input/output schemas:

```python
dummy_input = torch.randn(1, 3, 256, 256)
signature = infer_signature(
    dummy_input.numpy(), 
    clean_model(dummy_input).detach().numpy()
)
mlflow.pytorch.log_model(
    clean_model, 
    "model", 
    signature=signature, 
    extra_files=[stac_output_path]
)
```

This enables MLflow's model serving capabilities with schema validation.

**Sources:** [examplemodel/src/train.py:498-503]()

---

## System Dependencies and Configuration

### Dependency Management

The system uses `uv` package manager with dependencies defined in `pyproject.toml`. All entry points execute via `uv run` to ensure consistent dependency resolution.

**Example from MLproject:**
```
command: >
  uv run python src/train.py 
  --epochs {epochs}
  --batch_size {batch_size}
  --chips_dir {chips_dir}
  --labels_dir {labels_dir}
  --lr {lr}
```

**Sources:** [examplemodel/MLproject:26-32]()

### Training Parameters

The training entry point accepts five configurable parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `epochs` | int | 1 | Number of training epochs |
| `batch_size` | int | 32 | Batch size for training |
| `chips_dir` | str | `data/train/sample/chips` | Directory containing image chips |
| `labels_dir` | str | `data/train/sample/labels` | Directory containing label masks |
| `lr` | float | 1e-3 | Learning rate for optimizer |

**Command-line usage:**
```bash
uv run mlflow run . -e train \
  -P epochs=10 \
  -P batch_size=16 \
  -P chips_dir=data/custom/chips \
  -P labels_dir=data/custom/labels \
  -P lr=0.001
```

**Sources:** [examplemodel/MLproject:19-32](), [examplemodel/src/train.py:510-518]()

---

## Data Flow Through the System

```mermaid
flowchart TD
    subgraph "External Data Sources"
        TMS["OpenAerialMap TMS<br/>Satellite Imagery"]
        OSM["OpenStreetMap<br/>Label Polygons"]
    end
    
    subgraph "Preprocessing"
        PREPROCESS["preprocess.py"]
        CHIPS["chips/<br/>256x256 image tiles"]
        LABELS["labels/<br/>Binary masks"]
    end
    
    subgraph "Data Loading"
        DATAMODULE["CampDataModule"]
        TRAIN_LOADER["train_dataloader()"]
        VAL_LOADER["val_dataloader()"]
        TEST_LOADER["test_dataloader()"]
    end
    
    subgraph "Training"
        TRAINER["pl.Trainer"]
        LITMODEL["LitRefugeeCamp"]
        FORWARD["forward() pass"]
        LOSS["Loss computation"]
        BACKWARD["Backpropagation"]
    end
    
    subgraph "Evaluation"
        VAL_STEP["validation_step()"]
        TEST_STEP["test_step()"]
        CONFUSION["log_confusion_matrix()"]
        INFERENCE_EX["log_inference_example()"]
    end
    
    subgraph "Model Export"
        CHECKPOINT["Best checkpoint"]
        CONVERSIONS["Format conversions"]
        PTH_OUT["best_model.pth"]
        PT_OUT["best_model.pt"]
        ONNX_OUT["best_model.onnx"]
    end
    
    subgraph "Metadata & Packaging"
        STAC_GEN["create_stac_mlm_item()"]
        DLPK_GEN["create_dlpk()"]
        STAC_JSON["stac_item.json"]
        DLPK_OUT["best_model.dlpk"]
    end
    
    subgraph "MLflow Tracking"
        MLFLOW_LOG["mlflow.log_*()"]
        MINIO["MinIO Storage"]
        POSTGRES["PostgreSQL Metadata"]
    end
    
    TMS --> PREPROCESS
    OSM --> PREPROCESS
    PREPROCESS --> CHIPS
    PREPROCESS --> LABELS
    
    CHIPS --> DATAMODULE
    LABELS --> DATAMODULE
    DATAMODULE --> TRAIN_LOADER
    DATAMODULE --> VAL_LOADER
    DATAMODULE --> TEST_LOADER
    
    TRAIN_LOADER --> TRAINER
    VAL_LOADER --> TRAINER
    TRAINER --> LITMODEL
    LITMODEL --> FORWARD
    FORWARD --> LOSS
    LOSS --> BACKWARD
    
    VAL_LOADER --> VAL_STEP
    TEST_LOADER --> TEST_STEP
    TEST_STEP --> CONFUSION
    TEST_LOADER --> INFERENCE_EX
    
    TRAINER --> CHECKPOINT
    CHECKPOINT --> CONVERSIONS
    CONVERSIONS --> PTH_OUT
    CONVERSIONS --> PT_OUT
    CONVERSIONS --> ONNX_OUT
    
    LITMODEL --> STAC_GEN
    CHECKPOINT --> STAC_GEN
    STAC_GEN --> STAC_JSON
    PT_OUT --> DLPK_GEN
    ONNX_OUT --> DLPK_GEN
    DLPK_GEN --> DLPK_OUT
    
    PTH_OUT --> MLFLOW_LOG
    PT_OUT --> MLFLOW_LOG
    ONNX_OUT --> MLFLOW_LOG
    DLPK_OUT --> MLFLOW_LOG
    STAC_JSON --> MLFLOW_LOG
    CONFUSION --> MLFLOW_LOG
    INFERENCE_EX --> MLFLOW_LOG
    
    MLFLOW_LOG --> MINIO
    MLFLOW_LOG --> POSTGRES
```

**Sources:** [examplemodel/src/train.py:1-519](), [examplemodel/MLproject:1-63]()

---

## Output Artifacts Summary

The training pipeline generates a comprehensive set of outputs for different use cases:

### Model Artifacts

1. **best_model.pth** - Raw PyTorch state dictionary for flexible loading and fine-tuning
2. **best_model.pt** - TorchScript traced model for production inference with JIT compilation
3. **best_model.onnx** - ONNX format for cross-platform deployment (TensorRT, ONNX Runtime, etc.)
4. **best_model.dlpk** - ESRI Deep Learning Package for ArcGIS Pro integration

### Metadata and Documentation

1. **stac_item.json** - STAC-MLM compliant metadata describing model capabilities, requirements, and provenance
2. **model.emd** - ESRI Model Definition file with ArcGIS-specific configuration

### Evaluation Artifacts

1. **confusion_matrix.png** - Model performance visualization on test set
2. **example_input.png** - Sample input satellite imagery
3. **example_pred.png** - Model prediction on sample
4. **example_target.png** - Ground truth for sample

### Source Code

1. **RefugeeCampDetector.py** - ESRI inference class bundled in DLPK
2. Links to training and inference scripts in STAC metadata

**Sources:** [examplemodel/src/train.py:436-496]()

---

## Integration with Infrastructure

The Example Model System integrates with the Infrastructure System through MLflow tracking:

```mermaid
graph LR
    subgraph "Example Model System"
        TRAIN["train.py"]
        INFERENCE["inference.py"]
    end
    
    subgraph "Infrastructure System"
        MLFLOW_SERVER["MLflow Tracking Server"]
        POSTGRES_DB["PostgreSQL Database"]
        MINIO_S3["MinIO Object Storage"]
    end
    
    subgraph "Environment Variables"
        MLFLOW_URI["MLFLOW_TRACKING_URI"]
        AWS_KEY["AWS_ACCESS_KEY_ID"]
        AWS_SECRET["AWS_SECRET_ACCESS_KEY"]
        S3_ENDPOINT["MLFLOW_S3_ENDPOINT_URL"]
    end
    
    TRAIN -->|"mlflow.start_run()"| MLFLOW_SERVER
    INFERENCE -->|"mlflow.log_*()"| MLFLOW_SERVER
    
    MLFLOW_SERVER -->|"stores metrics"| POSTGRES_DB
    MLFLOW_SERVER -->|"stores artifacts"| MINIO_S3
    
    MLFLOW_URI -.->|"configures"| TRAIN
    AWS_KEY -.->|"authenticates"| MINIO_S3
    AWS_SECRET -.->|"authenticates"| MINIO_S3
    S3_ENDPOINT -.->|"routes to"| MINIO_S3
```

**Configuration Example (from README):**
```bash
export AWS_ACCESS_KEY_ID=mlflow
export AWS_SECRET_ACCESS_KEY=mlflow123
export MLFLOW_S3_ENDPOINT_URL=http://your_remote_server_minio:9000
```

**Sources:** [examplemodel/README.md:40-48](), [examplemodel/src/train.py:8-22]()

---

## Related Documentation

- For model architecture details and PyTorch Lightning components, see [Model Overview and Architecture](#3.1)
- For detailed training process and hyperparameters, see [Training Pipeline](#3.2)
- For inference workflows and prediction generation, see [Inference System](#3.3)
- For ESRI package creation and ArcGIS deployment, see [ESRI Integration and DLPK Generation](#3.4)
- For MLproject entry point specifications, see [MLflow Project Structure](#3.5)
- For dependency management and configuration, see [Dependencies and Configuration](#3.6)
- For infrastructure setup and MLflow server deployment, see [Infrastructure System](#4)