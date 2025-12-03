# NeuroVision AI — Brain Tumor Analysis

NeuroVision AI is a lightweight Flask web application for brain MRI analysis. It provides tumor classification, scan-plane detection, segmentation overlays and Grad-CAM explainability (heatmap) for supported images (PNG/JPG/JPEG, DICOM `.dcm`, and MATLAB `.mat` formats). The app is intended for research and prototyping — not for clinical use.

Key features
- Tumor classification
- Scan plane (axial/coronal/sagittal) detection
- Segmentation per plane
- Grad-CAM heatmap generation for explainability
- Frontend web UI to upload images, preview overlays and download segmentation
- Supports DICOM (.dcm) and .mat (v7.x/.v7.3) inputs with automatic conversion to PNG

Repository structure (important files)
- app.py — Flask application and core inference pipeline
- templates/ — several frontend HTML templates (index.html, old_index.html, etc.)
- models/ — (not included) place checkpoint files here (see below)
- requirements.txt — Python dependencies
- .gitignore
- LICENSE (Apache-2.0)

Quick start (local)
1. Clone the repository
   ```bash
   git clone https://github.com/Aftar-Ahmad-Sami/API.git
   cd API
   ```

2. Create & activate a Python virtual environment (recommended)
   ```bash
   python -m venv .venv
   source .venv/bin/activate   # Linux / macOS
   .venv\Scripts\activate      # Windows (PowerShell)
   ```

3. Install dependencies
   ```bash
   pip install -r requirements.txt
   ```

4. Download / place model checkpoints
   Create a `models/` directory and place your model files there using these names (as expected by `app.Config`):
   - `models/efficientnet_b0_best.pth` — tumor classifier checkpoint
   - `models/plane_classifier.pth` — plane detector checkpoint
   - `models/ax_best_model.pth` — axial segmentation model
   - `models/co_best_model.pth` — coronal segmentation model
   - `models/sa_best_model.pth` — sagittal segmentation model

   Note: The loader in `app.py` accepts checkpoints packaged either as a raw `state_dict` or wrapped inside `{'model_state_dict': ...}`. Models are loaded onto GPU if available.

5. Run the app
   ```bash
   python app.py
   # or (if you prefer flask CLI)
   export FLASK_APP=app.py
   flask run --host=0.0.0.0 --port=5000
   ```

6. Open the frontend
   - Visit: http://localhost:5000/ to use the web UI.

API endpoints
- POST /predict
  - Accepts multipart/form-data with:
    - `file` — uploaded image (PNG/JPG/JPEG), or `.dcm`, or `.mat`
    - optional `rotation` — integer degrees (e.g. -90) to rotate before processing
  - Returns JSON (examples below) with base64-encoded images and prediction metadata.

Example curl (upload a PNG/JPG)
```bash
curl -X POST "http://127.0.0.1:5000/predict" \
  -F "file=@/path/to/scan.png" \
  -F "rotation=0"
```

Example successful response (abridged)
```json
{
  "original_image": "<base64 PNG bytes>",
  "segmentation_image": "<base64 PNG RGBA overlay>",      // optional
  "heatmap_image": "<base64 PNG RGBA heatmap>",           // optional
  "tumor_class": "Glioma",
  "tumor_conf": 0.8932,
  "tumor_probs": {
    "Glioma": 0.8932,
    "Meningioma": 0.0456,
    "No Tumor": 0.034,
    "Pituitary": 0.0272
  },
  "planar_class": "axial"
}
```

Notes on features and behavior
- Grad-CAM heatmap is generated only when the classifier predicts a tumor (not "No Tumor"). The implementation uses hooks into the EfficientNet model and produces a colored RGBA PNG overlay.
- Segmentation only runs when a tumor is detected and a segmentation model exists for the detected plane.
- The returned segmentation image is an RGBA PNG where the alpha channel uses the predicted mask (so it can be overlayed directly).
- The app supports rotating inputs via `rotation` POST field; useful for scans oriented differently.

Supported input formats
- Standard images: PNG, JPG, JPEG
- DICOM (.dcm) — converted using pydicom
- MATLAB .mat (v7.x and v7.3) — parsed via scipy.io or h5py fallback

Model training / checkpointing
- This repository does not include pretrained weights. You must supply your own checkpoints in `models/`.
- `app.py` expects the classifier to be a timm `efficientnet_b0` with 4 output classes. The segmentation models are expected to match the `DeepLabV3Plus` architecture defined in the code.
- The loader handles checkpoints saved as raw `state_dict` or wrapped `{'model_state_dict': ...}`.

Development / debugging
- Logs print to console when models are loaded.
- The application runs with `debug=True` in `app.py` by default. Modify `app.run(...)` for production usage (use Gunicorn / uWSGI and disable debug).
- `Config.DEVICE` picks GPU if available.

Troubleshooting
- "No file uploaded" — ensure the multipart key is `file`.
- Large files rejected — `MAX_CONTENT_LENGTH` is set to 16MB in `app.py`.
- If models do not load: ensure checkpoint file names and formats match the expected keys. Check console output for loader messages and exceptions.
- For memory/CUDA issues: either use CPU (set torch device) or reduce batch sizes / image sizes.

Security, privacy, and disclaimer
- This software is for research and educational purposes only. It is NOT a medical device and must not be used for clinical decision-making.
- Uploaded images may be logged in debugging contexts. Be careful with protected health information (PHI).
- Do not deploy to production without implementing proper authentication, HTTPS, input validation and data handling practices.

Contributing
- Contributions welcome. Open an issue describing the change or create a pull request.
- Please follow the existing code style and include tests where applicable.

Acknowledgements
- Based on open-source ML libraries: PyTorch, torchvision, timm, pydicom, scipy, h5py, Pillow, matplotlib.
- Frontend templates inspired by modern interfaces for medical imaging tools.

License
- This project is licensed under the Apache License 2.0 — see the LICENSE file for details.

Contact
- Repository: https://github.com/Aftar-Ahmad-Sami/API
- Author: Aftar-Ahmad-Sami

If you want, I can also add a Dockerfile, CI workflow, or example notebook to demonstrate sending requests and decoding the returned base64 images. Just tell me which you'd like.
