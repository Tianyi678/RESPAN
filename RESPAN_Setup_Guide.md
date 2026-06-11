# RESPAN Environment Setup Guide
**Version 1.5.00 | Windows 10/11 | CUDA 11.8**

---

## Prerequisites

- Windows 10 or 11 (64-bit)
- NVIDIA GPU with CUDA support
- Internet connection
- PowerShell

---

## Step 1 — Install Miniconda

Open PowerShell and run:

```powershell
Invoke-WebRequest -Uri "https://repo.anaconda.com/miniconda/Miniconda3-latest-Windows-x86_64.exe" -OutFile "$env:USERPROFILE\Downloads\miniconda.exe"
```

```powershell
Start-Process -FilePath "$env:USERPROFILE\Downloads\miniconda.exe" -ArgumentList "/S /AddToPath=1 /RegisterPython=1 /D=C:\Users\<USERNAME>\miniconda3" -Wait
```

> Replace `<USERNAME>` with your actual Windows username.

Initialize conda for PowerShell:

```powershell
C:\Users\<USERNAME>\miniconda3\Scripts\conda.exe init powershell
```

Set execution policy:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**Close and reopen PowerShell**, then verify:

```powershell
conda --version
```

---

## Step 2 — Install Mamba

```powershell
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/msys2
```

```powershell
conda install -n base -c conda-forge mamba -y
```

---

## Step 3 — Create the `respan_gpu` Environment

This environment handles TensorFlow (CARE restoration), CuPy, and core image processing.

```powershell
mamba create -n respan_gpu python=3.9 -y
conda activate respan_gpu
```

Install core scientific packages:

```powershell
mamba install scikit-image pandas "numpy>=1.23,<2" nibabel ipython pyyaml pynvml numba zarr memory_profiler trimesh psutil -c conda-forge -y
```

Install GPU and deep learning packages:

```powershell
pip install "cupy-cuda11x>=13.2" "scipy>=1.13"
pip install "tensorflow<2.11" csbdeep
pip install pyqt5
pip install "patchify>=0.2.3" tifffile
```

Install PyTorch with CUDA 11.8:

```powershell
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

Install CUDA toolkit and cuDNN 8 (required for TensorFlow):

```powershell
mamba install -c conda-forge cudatoolkit=11.8 -y
mamba install -c conda-forge cudnn=8.9.2.26 -y
```

Make the cuDNN PATH permanent:

```powershell
conda env config vars set PATH="C:\Users\<USERNAME>\miniconda3\envs\respan_gpu\Library\bin;%PATH%"
conda activate respan_gpu
```

Verify GPU detection:

```powershell
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

Both should show your GPU name / `[PhysicalDevice(...)]`.

---

## Step 4 — Create the `respan_nnunet` Environment

This environment handles nnU-Net v2 segmentation (PyTorch-based).

```powershell
mamba create -n respan_nnunet python=3.9 -y
conda activate respan_nnunet
```

Install PyTorch with CUDA 11.8:

```powershell
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
```

Install build tools needed for nnU-Net dependencies:

```powershell
mamba install -c conda-forge cmake cxx-compiler -y
pip install hatchling scikit-build-core
```

Clone nnU-Net v2.3.1:

```powershell
cd C:\Users\<USERNAME>\Desktop\Respan\RESPAN
git clone -b v2.3.1 https://github.com/MIC-DKFZ/nnUNet.git
cd nnUNet
```

Install nnU-Net without the problematic `python-gdcm` dependency:

```powershell
pip install -e . --no-deps
pip install torch acvl-utils dynamic-network-architectures batchgenerators scikit-learn "scikit-image>=0.19.3" SimpleITK pandas graphviz nibabel seaborn imagecodecs yacs tqdm scipy numpy matplotlib requests tifffile fsspec sympy pydicom
pip install dicom2nifti --no-deps
```

> **Note:** `python-gdcm` is only needed for compressed DICOM formats (JPEG2000, JPEG-LS). If your data uses standard DICOM or NIfTI, the pipeline will work without it.

Verify PyTorch GPU:

```powershell
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

---

## Step 5 — Create the `respan99` Environment

This is the GUI launcher environment that runs `RESPAN_GUI_DIST.py`.

```powershell
mamba create -n respan99 python=3.9 -y
conda activate respan99
```

Install all required packages:

```powershell
pip install pynvml pyyaml PyQt5 "tensorflow<2.11" "numpy<2"
```

Copy the cuDNN DLL so TensorFlow can find it:

```powershell
copy "C:\Users\<USERNAME>\miniconda3\envs\respan_gpu\Library\bin\cudnn64_8.dll" "C:\Users\<USERNAME>\miniconda3\envs\respan99\Library\bin\"
```

Verify GPU detection:

```powershell
python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

---

## Step 6 — Launch RESPAN GUI

```powershell
conda activate respan99
python C:\Users\<USERNAME>\Desktop\Respan\RESPAN\RESPAN\Scripts\RESPAN_GUI_DIST.py
```

Expected output in the GUI log:

```
Initializing RESPAN and checking environment...
RESPAN location: C:\...\RESPAN\Scripts
GPU available:
     NVIDIA RTX XXXX
     Total Memory: XXXX MB
     ...
RESPAN environment found at C:\...\miniconda3\envs\respan99
Initialization complete.
```

---

## Environment Summary

| Environment | Python | Purpose | Key Packages |
|---|---|---|---|
| `respan_gpu` | 3.9 | Image processing, TensorFlow CARE | TensorFlow, CuPy, scikit-image, numba |
| `respan_nnunet` | 3.9 | nnU-Net segmentation | PyTorch, nnU-Net v2.3.1 |
| `respan99` | 3.9 | GUI launcher | PyQt5, TensorFlow, pynvml |

---

## Troubleshooting

**`conda` not recognized in PowerShell**
Run `conda init powershell` from Anaconda Prompt, close and reopen PowerShell.

**TensorFlow not detecting GPU**
Ensure `cudnn64_8.dll` is present in `Library\bin` of the active environment. cuDNN 9.x is not compatible with TensorFlow 2.10 — install `cudnn=8.9.2.26` from `pkgs/main`.

**NumPy version conflict with TensorFlow**
TensorFlow 2.10 requires NumPy 1.x. Run: `pip install "numpy<2"`

**`python-gdcm` build failure**
This package requires CMake and C++ build tools. Either install Visual Studio Build Tools, or skip it using `pip install -e . --no-deps` and install dependencies manually.

**`mamba activate` not working in PowerShell**
Use `conda activate <env>` instead — mamba activate is not supported in PowerShell.

**GUI shows "TensorFlow not installed. Using CPU."**
TensorFlow must be installed in the `respan99` environment specifically, not just `respan_gpu`.
