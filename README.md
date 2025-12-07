# Telco Churn – End-to-End ML Project

## Purpose
Build and ship a full machine-learning solution for predicting customer churn in a telecom setting—from data preparation and modeling to an API with web UI deployed on Google Cloud Platform.

## Problem Solved & Benefits

**Faster decisions**: Predicts which customers are likely to churn so teams can act before they leave.

**Operationalized ML**: Model is accessible via a REST API and a simple UI; anyone can test it without notebooks.

**Repeatable delivery**: CI/CD with containers means every change can be rebuilt, tested, and redeployed in a consistent way.

**Traceable experiments**: MLflow tracks runs, metrics, and artifacts for reproducibility and auditing.

## What I Built

**Data & Modeling**: Feature engineering with XGBoost classifier; experiments logged to MLflow for tracking and comparison.

**Model tracking**: Runs, metrics, and the serialized model logged under a named MLflow experiment for versioning and auditability.

**Inference service**: FastAPI app exposing /predict (POST) for predictions and a root health check at / for monitoring.

**Web UI**: Gradio interface mounted at /ui for quick, shareable manual testing without code.

**Containerization**: Docker image with uvicorn entrypoint (src.app.main:app) listening on port 8000.

**CI/CD**: GitHub Actions builds the image and pushes to Docker Hub; optionally triggers a Cloud Run service update.

**Orchestration**: Google Cloud Run manages the container deployment with automatic scaling and serverless compute.

**Networking**: Cloud Run provides HTTPS endpoints automatically; custom domain mapping available via Cloud Load Balancing if needed.

**Security**: IAM policies control access; Cloud Run allows unauthenticated invocations for public APIs or requires authentication for internal services.

**Observability**: Cloud Logging captures container stdout/stderr and service-level events for debugging and monitoring.

## Deployment Flow (High-Level)

1. Push to main branch triggers GitHub Actions workflow
2. Workflow builds Docker image and pushes to Docker Hub with latest tag
3. Cloud Run service is updated (manually via gcloud CLI or automatically in workflow)
4. Cloud Run deploys new revision and performs health checks on / endpoint
5. Traffic is gradually shifted to new revision once healthy
6. Users call POST /predict or open the Gradio UI at /ui via the Cloud Run URL

## Architecture Overview

The system follows a microservices pattern with clear separation of concerns:

**Training pipeline**: Processes raw data, engineers features, trains XGBoost model, and logs everything to MLflow.

**Inference service**: Loads the trained model and exposes it through FastAPI endpoints for programmatic access.

**Web interface**: Gradio UI provides a no-code interface for business users to test predictions interactively.

**Container layer**: Docker packages the application with all dependencies for consistent execution across environments.

**Cloud infrastructure**: Google Cloud Run handles scaling, networking, and infrastructure management automatically.

## Roadblocks & How We Solved Them

### Unhealthy Service in Cloud Run

**Cause**: Application didn't respond at the health-check path; container port misconfiguration.

**Fixes**: Added GET / health endpoint returning 200 OK; configured Cloud Run container port to 8000; verified startup probe settings.

### Module Import Error in Container (ModuleNotFoundError: serving)

**Cause**: Python path in the image didn't include src/ directory.

**Fixes**: Set PYTHONPATH=/app/src in the Dockerfile; corrected uvicorn app path to src.app.main:app; verified module structure in container.

### Cloud Run URL Timing Out

**Cause**: Container not listening on correct port or taking too long to start.

**Fixes**: Confirmed PORT environment variable set to 8000; reduced container startup time by optimizing dependencies; increased Cloud Run timeout settings.

### Cloud Run Not Picking Up New Image

**Cause**: Service still running previous revision; image pull policy cached old version.

**Fixes**: Force new deployment with --no-traffic flag then migrate traffic; use specific image tags instead of latest; clear Cloud Run image cache.

### Gradio UI Error ("No runs found in experiment")

**Cause**: Inference/UI expected an MLflow-logged model but couldn't resolve a run in the tracking server.

**Fixes**: Standardized MLflow experiment name across training and inference; ensured model logging completes before inference; added fallback to local model path for development.

### Local Testing vs. Production Paths

**Cause**: MLflow artifact URIs differ between local development and containerized production environment.

**Fixes**: For local dev, load via direct ./mlruns/.../artifacts/model path; in production, container loads the packaged model path baked into the image at build time; environment variables control path resolution.

### IAM Permission Issues

**Cause**: Service account lacked permissions to access Cloud Storage or other GCP resources.

**Fixes**: Created service account with appropriate roles (Storage Object Viewer, Cloud Run Invoker); attached service account to Cloud Run service; verified IAM bindings in GCP Console.

### Cold Start Latency

**Cause**: Cloud Run instances shut down after idle period; first request after idle incurs startup time.

**Fixes**: Configured minimum instances to keep at least one warm; optimized Docker image size to reduce pull time; implemented model caching strategy to avoid reloading on every request.

## Google Cloud Deployment Details

### Prerequisites

- Google Cloud Project with billing enabled
- Docker Hub account for image registry
- gcloud CLI installed and authenticated
- GitHub repository with Actions enabled

### Cloud Run Configuration

**Service name**: telco-churn-api

**Region**: us-central1 (or your preferred region)

**Container port**: 8000

**CPU allocation**: CPU is always allocated (for FastAPI)

**Memory**: 2Gi

**Concurrency**: 80 requests per instance

**Min instances**: 0 (or 1 to avoid cold starts)

**Max instances**: 10 (adjust based on expected load)

### Deployment Commands

Build and push image:
```bash
docker build -t your-dockerhub-username/telco-churn:latest .
docker push your-dockerhub-username/telco-churn:latest
```

Deploy to Cloud Run:
```bash
gcloud run deploy telco-churn-api \
  --image your-dockerhub-username/telco-churn:latest \
  --platform managed \
  --region us-central1 \
  --port 8000 \
  --memory 2Gi \
  --allow-unauthenticated
```

Update existing service:
```bash
gcloud run services update telco-churn-api \
  --image your-dockerhub-username/telco-churn:latest \
  --region us-central1
```

### CI/CD Integration

GitHub Actions workflow includes:

1. Checkout code
2. Build Docker image with commit SHA tag
3. Push to Docker Hub
4. Authenticate to Google Cloud using service account key
5. Deploy to Cloud Run with new image
6. Run smoke tests against deployed endpoint

### Monitoring and Logging

Access logs via Cloud Logging:
```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=telco-churn-api" --limit 50
```

View service metrics in Cloud Console under Cloud Run > telco-churn-api > Metrics.

## API Endpoints

### Health Check
```
GET /
Returns: {"status": "healthy"}
```

### Prediction
```
POST /predict
Content-Type: application/json

Request body:
{
  "gender": "Male",
  "SeniorCitizen": 0,
  "Partner": "Yes",
  "Dependents": "No",
  "tenure": 12,
  "PhoneService": "Yes",
  "MultipleLines": "No",
  "InternetService": "Fiber optic",
  "OnlineSecurity": "No",
  "OnlineBackup": "No",
  "DeviceProtection": "No",
  "TechSupport": "No",
  "StreamingTV": "Yes",
  "StreamingMovies": "Yes",
  "Contract": "Month-to-month",
  "PaperlessBilling": "Yes",
  "PaymentMethod": "Electronic check",
  "MonthlyCharges": 84.45,
  "TotalCharges": 1010.40
}

Response:
{
  "churn_prediction": 1,
  "churn_probability": 0.78
}
```

### Web UI
```
GET /ui
Interactive Gradio interface for manual predictions
```

## Local Development

Install dependencies:
```bash
pip install -r requirements.txt
```

Run training pipeline:
```bash
python src/models/train.py
```

Start API server:
```bash
uvicorn src.app.main:app --reload --port 8000
```

Access locally:
- API: http://localhost:8000
- Docs: http://localhost:8000/docs
- UI: http://localhost:8000/ui


## Technologies Used

- Python 3.9+
- XGBoost for modeling
- MLflow for experiment tracking
- FastAPI for API framework
- Gradio for web UI
- Docker for containerization
- GitHub Actions for CI/CD
- Google Cloud Run for deployment
- Google Cloud Logging for observability

## Future Enhancements

- Add A/B testing capability for model versions
- Implement model retraining pipeline triggered by data drift
- Add Prometheus metrics for detailed performance monitoring
- Integrate with Cloud Storage for scalable artifact storage
- Add authentication and rate limiting for production API
- Implement batch prediction endpoint for bulk processing
- Add model explainability features (SHAP values)
- Create dashboard for real-time monitoring and alerts

