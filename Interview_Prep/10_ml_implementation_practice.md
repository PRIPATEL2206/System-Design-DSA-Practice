# ML Implementation Practice Problems

> For AI/ML Engineer roles — model serving, ML pipelines, feature engineering, and MLOps coding.

---

## Problem 1: Build a Model Serving API with FastAPI

> Deploy your Plant Disease Classification model as a REST API with batching, caching, and health monitoring.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import List, Optional
import numpy as np
from PIL import Image
import io
import time
import hashlib
from functools import lru_cache
from collections import deque
from datetime import datetime

app = FastAPI(title="Plant Disease Classifier API")

# --- Model Loading (singleton) ---
class ModelService:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._load_model()
        return cls._instance
    
    def _load_model(self):
        import tensorflow as tf
        self.model = tf.keras.models.load_model("models/plant_disease_v2.h5")
        self.class_names = [
            "Apple_Scab", "Apple_Black_Rot", "Apple_Cedar_Rust", "Apple_Healthy",
            "Corn_Gray_Leaf_Spot", "Corn_Common_Rust", "Corn_Healthy",
            "Grape_Black_Rot", "Grape_Healthy", "Tomato_Early_Blight",
            "Tomato_Late_Blight", "Tomato_Healthy"
        ]
        self.input_shape = (224, 224)
        # Warm up model
        dummy = np.zeros((1, 224, 224, 3))
        self.model.predict(dummy)
    
    def preprocess(self, image: Image.Image) -> np.ndarray:
        image = image.resize(self.input_shape)
        arr = np.array(image) / 255.0
        if arr.shape[-1] == 4:  # RGBA → RGB
            arr = arr[:, :, :3]
        return np.expand_dims(arr, axis=0)
    
    def predict(self, images: List[np.ndarray]) -> List[dict]:
        batch = np.concatenate(images, axis=0)
        predictions = self.model.predict(batch)
        
        results = []
        for pred in predictions:
            top_idx = np.argsort(pred)[-3:][::-1]  # Top 3
            results.append({
                "prediction": self.class_names[top_idx[0]],
                "confidence": float(pred[top_idx[0]]),
                "top_3": [
                    {"class": self.class_names[i], "confidence": float(pred[i])}
                    for i in top_idx
                ]
            })
        return results

# --- Prediction Cache ---
class PredictionCache:
    def __init__(self, max_size=1000, ttl_seconds=3600):
        self.cache = {}
        self.max_size = max_size
        self.ttl = ttl_seconds
    
    def _hash_image(self, image_bytes: bytes) -> str:
        return hashlib.md5(image_bytes).hexdigest()
    
    def get(self, image_bytes: bytes) -> Optional[dict]:
        key = self._hash_image(image_bytes)
        if key in self.cache:
            entry = self.cache[key]
            if time.time() - entry["timestamp"] < self.ttl:
                return entry["result"]
            del self.cache[key]
        return None
    
    def set(self, image_bytes: bytes, result: dict):
        if len(self.cache) >= self.max_size:
            oldest = min(self.cache, key=lambda k: self.cache[k]["timestamp"])
            del self.cache[oldest]
        key = self._hash_image(image_bytes)
        self.cache[key] = {"result": result, "timestamp": time.time()}

# --- Monitoring ---
class MetricsCollector:
    def __init__(self):
        self.predictions_total = 0
        self.latencies = deque(maxlen=1000)
        self.errors = 0
        self.class_distribution = {}
        self.start_time = datetime.utcnow()
    
    def record_prediction(self, latency: float, predicted_class: str):
        self.predictions_total += 1
        self.latencies.append(latency)
        self.class_distribution[predicted_class] = self.class_distribution.get(predicted_class, 0) + 1
    
    def record_error(self):
        self.errors += 1
    
    def get_stats(self) -> dict:
        latencies = list(self.latencies)
        return {
            "total_predictions": self.predictions_total,
            "errors": self.errors,
            "uptime_seconds": (datetime.utcnow() - self.start_time).total_seconds(),
            "latency_p50_ms": float(np.percentile(latencies, 50)) * 1000 if latencies else 0,
            "latency_p95_ms": float(np.percentile(latencies, 95)) * 1000 if latencies else 0,
            "latency_p99_ms": float(np.percentile(latencies, 99)) * 1000 if latencies else 0,
            "class_distribution": self.class_distribution,
            "cache_hit_rate": cache.cache.get("hits", 0)  # Simplified
        }

# Initialize
model_service = ModelService()
cache = PredictionCache()
metrics = MetricsCollector()

# --- Endpoints ---
class PredictionResponse(BaseModel):
    prediction: str
    confidence: float
    top_3: List[dict]
    cached: bool = False
    latency_ms: float

@app.post("/api/v1/predict", response_model=PredictionResponse)
async def predict_single(file: UploadFile = File(...)):
    if not file.content_type.startswith("image/"):
        raise HTTPException(400, "File must be an image")
    
    start = time.time()
    image_bytes = await file.read()
    
    # Check cache
    cached_result = cache.get(image_bytes)
    if cached_result:
        latency = time.time() - start
        return PredictionResponse(**cached_result, cached=True, latency_ms=latency*1000)
    
    try:
        image = Image.open(io.BytesIO(image_bytes))
        processed = model_service.preprocess(image)
        results = model_service.predict([processed])
        result = results[0]
        
        latency = time.time() - start
        cache.set(image_bytes, result)
        metrics.record_prediction(latency, result["prediction"])
        
        return PredictionResponse(**result, cached=False, latency_ms=latency*1000)
    except Exception as e:
        metrics.record_error()
        raise HTTPException(500, f"Prediction failed: {str(e)}")

@app.post("/api/v1/predict/batch")
async def predict_batch(files: List[UploadFile] = File(...)):
    if len(files) > 32:
        raise HTTPException(400, "Maximum 32 images per batch")
    
    start = time.time()
    processed_images = []
    
    for file in files:
        image_bytes = await file.read()
        image = Image.open(io.BytesIO(image_bytes))
        processed_images.append(model_service.preprocess(image))
    
    results = model_service.predict(processed_images)
    latency = time.time() - start
    
    return {
        "predictions": results,
        "batch_size": len(files),
        "total_latency_ms": latency * 1000,
        "per_image_ms": (latency * 1000) / len(files)
    }

@app.get("/api/v1/model/info")
async def model_info():
    return {
        "model_version": "v2.1",
        "classes": model_service.class_names,
        "input_shape": list(model_service.input_shape),
        "framework": "tensorflow"
    }

@app.get("/metrics")
async def get_metrics():
    return metrics.get_stats()

@app.get("/health")
async def health():
    try:
        dummy = np.zeros((1, 224, 224, 3))
        model_service.model.predict(dummy)
        return {"status": "healthy", "model_loaded": True}
    except Exception as e:
        return JSONResponse(status_code=503, content={"status": "unhealthy", "error": str(e)})
```

**Interview tip:** Key production concerns: model warm-up (avoid cold start on first request), caching predictions (same image = same result), batch endpoint (amortize overhead), metrics for monitoring drift, health check verifies model is actually loaded.
</details>

---

## Problem 2: Implement a Feature Store (Online + Offline)

> Build a feature store that serves features for both model training (offline/batch) and real-time inference (online).

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from typing import Dict, List, Optional, Any
from datetime import datetime, timedelta
from dataclasses import dataclass, field
import pandas as pd
import json
import time

@dataclass
class FeatureDefinition:
    name: str
    dtype: str  # int, float, string, list
    description: str
    entity_key: str  # e.g., "hcp_id"
    ttl_seconds: int = 86400  # 24 hours default
    source: str = ""  # Where this feature comes from

@dataclass
class FeatureValue:
    value: Any
    timestamp: datetime
    source: str

class FeatureStore:
    def __init__(self):
        self.feature_definitions: Dict[str, FeatureDefinition] = {}
        self.online_store: Dict[str, Dict[str, FeatureValue]] = {}  # entity_id -> {feature_name -> value}
        self.offline_store: List[Dict] = []  # Historical records for training

    # --- Feature Registration ---
    def register_feature(self, feature: FeatureDefinition):
        self.feature_definitions[feature.name] = feature

    def register_feature_group(self, group_name: str, features: List[FeatureDefinition]):
        for f in features:
            f.source = group_name
            self.register_feature(f)

    # --- Online Store (Real-time Serving) ---
    def set_online(self, entity_id: str, features: Dict[str, Any], source: str = ""):
        if entity_id not in self.online_store:
            self.online_store[entity_id] = {}
        
        now = datetime.utcnow()
        for name, value in features.items():
            self.online_store[entity_id][name] = FeatureValue(
                value=value, timestamp=now, source=source
            )
        
        # Also write to offline store (for training data)
        self.offline_store.append({
            "entity_id": entity_id,
            "timestamp": now.isoformat(),
            **features
        })

    def get_online(self, entity_id: str, feature_names: List[str]) -> Dict[str, Any]:
        """Get features for real-time inference."""
        if entity_id not in self.online_store:
            return {name: None for name in feature_names}
        
        result = {}
        now = datetime.utcnow()
        
        for name in feature_names:
            fv = self.online_store[entity_id].get(name)
            if fv is None:
                result[name] = None
            elif name in self.feature_definitions:
                ttl = self.feature_definitions[name].ttl_seconds
                if (now - fv.timestamp).total_seconds() > ttl:
                    result[name] = None  # Expired
                else:
                    result[name] = fv.value
            else:
                result[name] = fv.value
        
        return result

    def get_online_batch(self, entity_ids: List[str], feature_names: List[str]) -> List[Dict]:
        """Get features for multiple entities (batch inference)."""
        return [self.get_online(eid, feature_names) for eid in entity_ids]

    # --- Offline Store (Training Data) ---
    def get_training_data(self, feature_names: List[str], 
                         start_date: datetime = None, 
                         end_date: datetime = None) -> pd.DataFrame:
        """Get historical feature data for model training."""
        df = pd.DataFrame(self.offline_store)
        
        if start_date:
            df = df[df["timestamp"] >= start_date.isoformat()]
        if end_date:
            df = df[df["timestamp"] <= end_date.isoformat()]
        
        columns = ["entity_id", "timestamp"] + [f for f in feature_names if f in df.columns]
        return df[columns]

    def get_point_in_time(self, entity_id: str, feature_names: List[str], 
                         as_of: datetime) -> Dict[str, Any]:
        """Get features as they were at a specific point in time (for backfilling)."""
        entity_records = [
            r for r in self.offline_store 
            if r["entity_id"] == entity_id and r["timestamp"] <= as_of.isoformat()
        ]
        
        if not entity_records:
            return {name: None for name in feature_names}
        
        # Get most recent record before as_of
        latest = max(entity_records, key=lambda r: r["timestamp"])
        return {name: latest.get(name) for name in feature_names}

    # --- Feature Statistics ---
    def get_statistics(self, feature_name: str) -> Dict:
        """Compute feature statistics for monitoring drift."""
        values = [
            self.online_store[eid][feature_name].value
            for eid in self.online_store
            if feature_name in self.online_store[eid]
            and isinstance(self.online_store[eid][feature_name].value, (int, float))
        ]
        
        if not values:
            return {}
        
        import numpy as np
        return {
            "count": len(values),
            "mean": float(np.mean(values)),
            "std": float(np.std(values)),
            "min": float(np.min(values)),
            "max": float(np.max(values)),
            "p25": float(np.percentile(values, 25)),
            "p50": float(np.percentile(values, 50)),
            "p75": float(np.percentile(values, 75)),
        }

# --- Usage ---
store = FeatureStore()

# Register features
store.register_feature_group("hcp_features", [
    FeatureDefinition("total_interactions", "int", "Total interactions count", "hcp_id"),
    FeatureDefinition("avg_prescription_value", "float", "Average prescription value", "hcp_id"),
    FeatureDefinition("days_since_last_visit", "int", "Days since last visit", "hcp_id", ttl_seconds=3600),
    FeatureDefinition("specialty_encoded", "int", "One-hot encoded specialty", "hcp_id"),
    FeatureDefinition("region_encoded", "int", "One-hot encoded region", "hcp_id"),
])

# Ingest features (would be done by ETL pipeline)
store.set_online("HCP001", {
    "total_interactions": 150,
    "avg_prescription_value": 450.5,
    "days_since_last_visit": 3,
    "specialty_encoded": 2,
    "region_encoded": 1
}, source="daily_etl")

# Real-time serving (for model inference)
features = store.get_online("HCP001", [
    "total_interactions", "avg_prescription_value", "days_since_last_visit"
])
# → {"total_interactions": 150, "avg_prescription_value": 450.5, "days_since_last_visit": 3}

# Training data retrieval
training_df = store.get_training_data(
    feature_names=["total_interactions", "avg_prescription_value"],
    start_date=datetime(2025, 1, 1)
)
```

**Interview tip:** Feature store is the #1 MLOps concept. Key ideas: online vs offline serving, TTL for freshness, point-in-time lookup (prevents data leakage in training), statistics for drift detection.
</details>

---

## Problem 3: Implement a Data Drift Detector

> Build a system that detects when incoming data distribution changes compared to training data.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
import numpy as np
from scipy import stats
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass
from datetime import datetime
from collections import deque

@dataclass
class DriftResult:
    feature_name: str
    drift_detected: bool
    drift_score: float  # 0-1, higher = more drift
    test_used: str
    p_value: float
    threshold: float
    details: Dict = None

class DriftDetector:
    def __init__(self, significance_level: float = 0.05, window_size: int = 1000):
        self.significance_level = significance_level
        self.window_size = window_size
        self.reference_distributions: Dict[str, np.ndarray] = {}
        self.current_windows: Dict[str, deque] = {}
        self.drift_history: List[DriftResult] = []

    def set_reference(self, feature_name: str, data: np.ndarray):
        """Set reference distribution (from training data)."""
        self.reference_distributions[feature_name] = data
        self.current_windows[feature_name] = deque(maxlen=self.window_size)

    def add_observation(self, feature_name: str, value: float):
        """Add a new observation to the sliding window."""
        if feature_name not in self.current_windows:
            self.current_windows[feature_name] = deque(maxlen=self.window_size)
        self.current_windows[feature_name].append(value)

    def check_drift(self, feature_name: str) -> DriftResult:
        """Check if current window has drifted from reference."""
        if feature_name not in self.reference_distributions:
            return DriftResult(feature_name, False, 0, "none", 1.0, self.significance_level,
                             {"error": "No reference distribution"})
        
        reference = self.reference_distributions[feature_name]
        current = np.array(self.current_windows.get(feature_name, []))
        
        if len(current) < 30:
            return DriftResult(feature_name, False, 0, "insufficient_data", 1.0, 
                             self.significance_level, {"samples": len(current)})
        
        # KS Test (Kolmogorov-Smirnov)
        ks_stat, ks_p_value = stats.ks_2samp(reference, current)
        
        # PSI (Population Stability Index)
        psi = self._calculate_psi(reference, current)
        
        # Determine drift
        drift_detected = ks_p_value < self.significance_level or psi > 0.2
        drift_score = min(1.0, psi / 0.5)  # Normalize to 0-1
        
        result = DriftResult(
            feature_name=feature_name,
            drift_detected=drift_detected,
            drift_score=drift_score,
            test_used="KS + PSI",
            p_value=ks_p_value,
            threshold=self.significance_level,
            details={
                "ks_statistic": float(ks_stat),
                "ks_p_value": float(ks_p_value),
                "psi": float(psi),
                "reference_mean": float(np.mean(reference)),
                "current_mean": float(np.mean(current)),
                "reference_std": float(np.std(reference)),
                "current_std": float(np.std(current)),
                "sample_size": len(current)
            }
        )
        
        self.drift_history.append(result)
        return result

    def _calculate_psi(self, reference: np.ndarray, current: np.ndarray, bins: int = 10) -> float:
        """Population Stability Index — measures distribution shift."""
        # Create bins from reference
        breakpoints = np.percentile(reference, np.linspace(0, 100, bins + 1))
        breakpoints[0] = -np.inf
        breakpoints[-1] = np.inf
        
        # Calculate proportions in each bin
        ref_counts = np.histogram(reference, bins=breakpoints)[0]
        cur_counts = np.histogram(current, bins=breakpoints)[0]
        
        # Add small epsilon to avoid division by zero
        eps = 1e-6
        ref_pct = ref_counts / len(reference) + eps
        cur_pct = cur_counts / len(current) + eps
        
        # PSI formula
        psi = np.sum((cur_pct - ref_pct) * np.log(cur_pct / ref_pct))
        return float(psi)

    def check_all_features(self) -> Dict[str, DriftResult]:
        """Check drift for all monitored features."""
        results = {}
        for feature_name in self.reference_distributions:
            results[feature_name] = self.check_drift(feature_name)
        return results

    def get_alert_summary(self) -> Dict:
        """Get summary of drifting features."""
        results = self.check_all_features()
        drifting = {k: v for k, v in results.items() if v.drift_detected}
        
        return {
            "timestamp": datetime.utcnow().isoformat(),
            "total_features_monitored": len(results),
            "features_drifting": len(drifting),
            "alert_level": "critical" if len(drifting) > len(results) * 0.3 else 
                          "warning" if drifting else "healthy",
            "drifting_features": {
                k: {"drift_score": v.drift_score, "psi": v.details.get("psi", 0)}
                for k, v in drifting.items()
            }
        }

# --- Usage ---
detector = DriftDetector(significance_level=0.05)

# Set reference from training data
np.random.seed(42)
detector.set_reference("avg_prescription_value", np.random.normal(450, 100, 10000))
detector.set_reference("total_interactions", np.random.poisson(50, 10000).astype(float))

# Simulate production data (no drift)
for val in np.random.normal(450, 100, 500):
    detector.add_observation("avg_prescription_value", val)

result = detector.check_drift("avg_prescription_value")
print(f"Drift: {result.drift_detected}, Score: {result.drift_score:.3f}")
# Drift: False, Score: 0.02

# Simulate production data (WITH drift — mean shifted)
for val in np.random.normal(600, 150, 500):  # Mean shifted from 450 to 600
    detector.add_observation("avg_prescription_value", val)

result = detector.check_drift("avg_prescription_value")
print(f"Drift: {result.drift_detected}, Score: {result.drift_score:.3f}")
# Drift: True, Score: 0.85
```

**Interview tip:** PSI thresholds: < 0.1 (no drift), 0.1-0.2 (moderate), > 0.2 (significant). Mention: data drift vs concept drift (data changes vs relationship changes). When drift is detected → alert team → investigate → potentially retrain model.
</details>

---

## Problem 4: Implement an ML Experiment Tracker (Mini MLflow)

> Build a lightweight experiment tracking system for comparing model runs.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field
from datetime import datetime
import json
import os
import hashlib

@dataclass
class Run:
    run_id: str
    experiment_name: str
    status: str = "running"  # running, completed, failed
    start_time: datetime = None
    end_time: Optional[datetime] = None
    params: Dict[str, Any] = field(default_factory=dict)
    metrics: Dict[str, List[float]] = field(default_factory=dict)
    tags: Dict[str, str] = field(default_factory=dict)
    artifacts: List[str] = field(default_factory=list)

class ExperimentTracker:
    def __init__(self, tracking_dir: str = "./mlruns"):
        self.tracking_dir = tracking_dir
        self.experiments: Dict[str, List[Run]] = {}
        self.active_run: Optional[Run] = None
        os.makedirs(tracking_dir, exist_ok=True)

    def start_run(self, experiment_name: str, tags: Dict = None) -> Run:
        run_id = hashlib.md5(
            f"{experiment_name}_{datetime.utcnow().isoformat()}".encode()
        ).hexdigest()[:12]
        
        run = Run(
            run_id=run_id,
            experiment_name=experiment_name,
            start_time=datetime.utcnow(),
            tags=tags or {}
        )
        
        self.experiments.setdefault(experiment_name, []).append(run)
        self.active_run = run
        return run

    def log_param(self, key: str, value: Any):
        if not self.active_run:
            raise RuntimeError("No active run")
        self.active_run.params[key] = value

    def log_params(self, params: Dict[str, Any]):
        for k, v in params.items():
            self.log_param(k, v)

    def log_metric(self, key: str, value: float, step: int = None):
        if not self.active_run:
            raise RuntimeError("No active run")
        self.active_run.metrics.setdefault(key, []).append(value)

    def log_metrics(self, metrics: Dict[str, float]):
        for k, v in metrics.items():
            self.log_metric(k, v)

    def log_artifact(self, filepath: str):
        if not self.active_run:
            raise RuntimeError("No active run")
        self.active_run.artifacts.append(filepath)

    def end_run(self, status: str = "completed"):
        if not self.active_run:
            return
        self.active_run.status = status
        self.active_run.end_time = datetime.utcnow()
        self._save_run(self.active_run)
        self.active_run = None

    def _save_run(self, run: Run):
        run_dir = os.path.join(self.tracking_dir, run.experiment_name, run.run_id)
        os.makedirs(run_dir, exist_ok=True)
        
        with open(os.path.join(run_dir, "run.json"), "w") as f:
            json.dump({
                "run_id": run.run_id,
                "experiment": run.experiment_name,
                "status": run.status,
                "start_time": run.start_time.isoformat(),
                "end_time": run.end_time.isoformat() if run.end_time else None,
                "params": run.params,
                "metrics": {k: v[-1] for k, v in run.metrics.items()},  # Last value
                "metric_history": run.metrics,
                "tags": run.tags,
                "duration_seconds": (run.end_time - run.start_time).total_seconds() if run.end_time else None
            }, f, indent=2)

    def compare_runs(self, experiment_name: str, metric: str = "accuracy",
                    top_k: int = 5) -> List[Dict]:
        """Compare runs in an experiment, sorted by metric."""
        runs = self.experiments.get(experiment_name, [])
        completed_runs = [r for r in runs if r.status == "completed"]
        
        comparisons = []
        for run in completed_runs:
            last_metric = run.metrics.get(metric, [None])[-1]
            if last_metric is not None:
                comparisons.append({
                    "run_id": run.run_id,
                    "params": run.params,
                    f"{metric}": last_metric,
                    "duration": (run.end_time - run.start_time).total_seconds() if run.end_time else None,
                })
        
        comparisons.sort(key=lambda x: x.get(metric, 0), reverse=True)
        return comparisons[:top_k]

    def get_best_run(self, experiment_name: str, metric: str = "accuracy") -> Optional[Run]:
        """Get the best run by a specific metric."""
        runs = self.experiments.get(experiment_name, [])
        best_run = None
        best_value = -float("inf")
        
        for run in runs:
            if run.status == "completed" and metric in run.metrics:
                value = run.metrics[metric][-1]
                if value > best_value:
                    best_value = value
                    best_run = run
        
        return best_run

# --- Usage: Training loop with tracking ---
tracker = ExperimentTracker()

# Run 1: Baseline model
run = tracker.start_run("plant_disease_classification", tags={"author": "prince"})
tracker.log_params({
    "model": "MobileNetV2",
    "learning_rate": 0.001,
    "batch_size": 32,
    "epochs": 20,
    "augmentation": True,
    "dropout": 0.3
})

# Simulate training
for epoch in range(20):
    train_loss = 1.0 - epoch * 0.04 + np.random.normal(0, 0.01)
    val_accuracy = 0.5 + epoch * 0.025 + np.random.normal(0, 0.005)
    tracker.log_metric("train_loss", train_loss, step=epoch)
    tracker.log_metric("val_accuracy", val_accuracy, step=epoch)

tracker.log_metrics({"test_accuracy": 0.94, "test_f1": 0.93})
tracker.end_run()

# Run 2: With different hyperparameters
run = tracker.start_run("plant_disease_classification")
tracker.log_params({
    "model": "ResNet50",
    "learning_rate": 0.0005,
    "batch_size": 64,
    "epochs": 30,
    "augmentation": True,
    "dropout": 0.5
})
tracker.log_metrics({"test_accuracy": 0.96, "test_f1": 0.95})
tracker.end_run()

# Compare runs
print(tracker.compare_runs("plant_disease_classification", metric="test_accuracy"))
best = tracker.get_best_run("plant_disease_classification", "test_accuracy")
print(f"Best run: {best.run_id} with params: {best.params}")
```

**Interview tip:** This is basically what MLflow does. Key concepts: reproducibility (log all params), comparison (which config worked best), persistence (save to disk/S3). In production, you'd add: model registry, model staging (staging → production), automatic retraining triggers.
</details>

---

## Problem 5: Implement an A/B Testing Framework for Models

> Build a router that splits traffic between model versions and tracks metrics.

### Try it yourself first!

<details>
<summary>Solution</summary>

```python
from typing import Dict, Optional, Callable
from dataclasses import dataclass, field
from datetime import datetime
import random
import hashlib
import numpy as np
from scipy import stats

@dataclass
class ModelVariant:
    name: str
    model: Callable  # prediction function
    traffic_percentage: float
    predictions: int = 0
    total_latency: float = 0.0
    outcomes: list = field(default_factory=list)  # (predicted, actual) pairs

@dataclass
class Experiment:
    name: str
    variants: Dict[str, ModelVariant]
    start_date: datetime
    min_samples: int = 1000
    significance_level: float = 0.05
    status: str = "running"  # running, concluded

class ABTestRouter:
    def __init__(self):
        self.experiments: Dict[str, Experiment] = {}
    
    def create_experiment(self, name: str, variants: Dict[str, dict], 
                         min_samples: int = 1000) -> Experiment:
        """
        variants = {
            "control": {"model": model_v1, "traffic": 0.5},
            "treatment": {"model": model_v2, "traffic": 0.5}
        }
        """
        model_variants = {}
        for variant_name, config in variants.items():
            model_variants[variant_name] = ModelVariant(
                name=variant_name,
                model=config["model"],
                traffic_percentage=config["traffic"]
            )
        
        experiment = Experiment(
            name=name,
            variants=model_variants,
            start_date=datetime.utcnow(),
            min_samples=min_samples
        )
        self.experiments[name] = experiment
        return experiment

    def route(self, experiment_name: str, user_id: str) -> str:
        """Deterministic routing based on user_id (consistent experience)."""
        experiment = self.experiments[experiment_name]
        
        # Hash user_id for deterministic bucket assignment
        hash_val = int(hashlib.md5(f"{experiment_name}:{user_id}".encode()).hexdigest(), 16)
        bucket = (hash_val % 10000) / 10000.0  # 0.0 - 1.0
        
        cumulative = 0.0
        for variant_name, variant in experiment.variants.items():
            cumulative += variant.traffic_percentage
            if bucket < cumulative:
                return variant_name
        
        return list(experiment.variants.keys())[-1]

    def predict(self, experiment_name: str, user_id: str, input_data: dict) -> dict:
        """Route user to variant and get prediction."""
        import time
        
        experiment = self.experiments[experiment_name]
        variant_name = self.route(experiment_name, user_id)
        variant = experiment.variants[variant_name]
        
        start = time.time()
        prediction = variant.model(input_data)
        latency = time.time() - start
        
        variant.predictions += 1
        variant.total_latency += latency
        
        return {
            "prediction": prediction,
            "variant": variant_name,
            "experiment": experiment_name,
            "latency_ms": latency * 1000
        }

    def record_outcome(self, experiment_name: str, user_id: str, 
                      predicted: float, actual: float):
        """Record ground truth for statistical analysis."""
        variant_name = self.route(experiment_name, user_id)
        variant = self.experiments[experiment_name].variants[variant_name]
        variant.outcomes.append((predicted, actual))

    def analyze(self, experiment_name: str) -> Dict:
        """Statistical analysis of experiment results."""
        experiment = self.experiments[experiment_name]
        
        results = {}
        for name, variant in experiment.variants.items():
            outcomes = variant.outcomes
            if not outcomes:
                results[name] = {"status": "insufficient_data"}
                continue
            
            predicted, actual = zip(*outcomes)
            accuracy = sum(1 for p, a in outcomes if p == a) / len(outcomes)
            
            results[name] = {
                "samples": len(outcomes),
                "accuracy": accuracy,
                "avg_latency_ms": (variant.total_latency / max(variant.predictions, 1)) * 1000,
                "predictions_served": variant.predictions
            }
        
        # Statistical significance test (if both variants have data)
        variant_names = list(experiment.variants.keys())
        if len(variant_names) == 2:
            v1 = experiment.variants[variant_names[0]]
            v2 = experiment.variants[variant_names[1]]
            
            if v1.outcomes and v2.outcomes:
                # Convert to binary success/failure for proportions test
                success_1 = sum(1 for p, a in v1.outcomes if p == a)
                success_2 = sum(1 for p, a in v2.outcomes if p == a)
                n1, n2 = len(v1.outcomes), len(v2.outcomes)
                
                # Two-proportion z-test
                p1 = success_1 / n1
                p2 = success_2 / n2
                p_pool = (success_1 + success_2) / (n1 + n2)
                se = np.sqrt(p_pool * (1 - p_pool) * (1/n1 + 1/n2))
                
                if se > 0:
                    z_stat = (p1 - p2) / se
                    p_value = 2 * (1 - stats.norm.cdf(abs(z_stat)))
                else:
                    z_stat, p_value = 0, 1.0
                
                results["statistical_test"] = {
                    "test": "two_proportion_z_test",
                    "z_statistic": float(z_stat),
                    "p_value": float(p_value),
                    "significant": p_value < experiment.significance_level,
                    "winner": variant_names[0] if p1 > p2 else variant_names[1],
                    "improvement": abs(p1 - p2) * 100,
                    "confidence": (1 - p_value) * 100,
                    "min_samples_reached": min(n1, n2) >= experiment.min_samples
                }
        
        return results

    def conclude_experiment(self, experiment_name: str, winner: str):
        """Conclude experiment and route all traffic to winner."""
        experiment = self.experiments[experiment_name]
        experiment.status = "concluded"
        
        # Set winner to 100% traffic
        for name, variant in experiment.variants.items():
            variant.traffic_percentage = 1.0 if name == winner else 0.0

# --- Usage ---
router = ABTestRouter()

# Define models
def model_v1(data): return 1 if data.get("score", 0) > 0.5 else 0
def model_v2(data): return 1 if data.get("score", 0) > 0.45 else 0  # Lower threshold

router.create_experiment("disease_classifier_threshold", {
    "control": {"model": model_v1, "traffic": 0.5},
    "treatment": {"model": model_v2, "traffic": 0.5}
})

# Simulate traffic
for i in range(2000):
    user_id = f"user_{i}"
    data = {"score": random.random()}
    result = router.predict("disease_classifier_threshold", user_id, data)
    
    # Simulate ground truth (treatment is slightly better)
    actual = 1 if data["score"] > 0.47 else 0
    router.record_outcome("disease_classifier_threshold", user_id, 
                         result["prediction"], actual)

# Analyze results
analysis = router.analyze("disease_classifier_threshold")
print(json.dumps(analysis, indent=2))
```

**Interview tip:** Key concepts: deterministic routing (same user always sees same model), statistical significance (don't declare winner too early), minimum sample size, practical significance vs statistical significance. Mention that you'd track business metrics too (not just accuracy).
</details>

---

## Summary — ML Problems

| # | Problem | Key Concepts | Difficulty |
|---|---------|-------------|-----------|
| 1 | Model Serving API | Batching, caching, health checks, latency | Medium |
| 2 | Feature Store | Online/offline, TTL, point-in-time | Hard |
| 3 | Drift Detection | KS test, PSI, statistical monitoring | Hard |
| 4 | Experiment Tracker | Reproducibility, comparison, persistence | Medium |
| 5 | A/B Testing | Deterministic routing, significance testing | Hard |

**Practice order:** 1 → 4 → 3 → 2 → 5

---

## Complete Practice Order (All Files)

**Week 1 Priority:**
1. File 07: Problems 8, 1, 3 (Retry, Two Sum, Producer-Consumer)
2. File 08: SQL-1, SQL-2, PySpark-1 (Windows, Consecutive, Dedup)
3. File 09: Problem 1, 4 (REST API, Background Tasks)
4. File 10: Problem 1 (Model Serving)

**Week 2 Deep Dive:**
1. File 07: Problems 6, 9, 10 (Consistent Hash, State Machine, DAG)
2. File 08: PySpark-4, PySpark-5 (Skew, Features)
3. File 09: Problem 6, 7 (Webhooks, CQRS)
4. File 10: Problem 2, 3 (Feature Store, Drift)
