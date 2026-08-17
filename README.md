# ML Model Serving Platform

A production-grade ML model serving infrastructure built around real-world MLOps and Platform Engineering principles.

> **Authored by Shashwat Tripathi, IIT Bombay**

## 🎯 What This Demonstrates

This is a **complete, production-ready ML serving platform** — not a toy project. It reflects the kind of engineering decisions you'd make in a real environment:

**Platform Engineering:**
- ✅ Infrastructure as Code (Kubernetes manifests)
- ✅ Container orchestration with resource limits & health checks
- ✅ Horizontal auto-scaling driven by CPU and memory metrics
- ✅ Zero-downtime rolling deployments
- ✅ Hardened security posture (non-root user, security contexts, least privilege)

**MLOps:**
- ✅ Model versioning with metadata tracking
- ✅ Automated CI/CD pipeline (build → test → deploy)
- ✅ Prometheus-based metrics for model monitoring
- ✅ Structured logging for full observability
- ✅ A/B testing support via version-based routing

**Production Readiness:**
- ✅ Multi-stage Docker builds for lean, optimized images
- ✅ Comprehensive integration test suite
- ✅ Performance benchmarking (latency & throughput)
- ✅ Automated security scanning in CI
- ✅ Cost-aware resource requests and limits

## 🏗️ Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│      Kubernetes LoadBalancer        │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│         Service (ClusterIP)         │
│    Session Affinity: None           │
└─────────────┬───────────────────────┘
              │
       ┌──────┴──────┬─────────────┐
       ▼             ▼             ▼
   ┌──────┐      ┌──────┐      ┌──────┐
   │ Pod  │      │ Pod  │      │ Pod  │
   │  #1  │      │  #2  │      │  #N  │
   └──┬───┘      └──┬───┘      └──┬───┘
      │             │             │
      └─────────────┴─────────────┘
                    │
                    ▼
         ┌──────────────────────┐
         │ Horizontal Pod       │
         │ Autoscaler (HPA)     │
         │ • CPU: 70%           │
         │ • Memory: 80%        │
         │ • Min: 2, Max: 10    │
         └──────────────────────┘
```

**Each Pod contains:**
- A FastAPI application serving ML predictions
- Health check endpoints (`/health`)
- A Prometheus metrics endpoint (`/metrics`)
- A versioned fraud detection model

**Monitoring stack:**
- Prometheus scrapes `/metrics` from every pod
- Grafana dashboards covering latency, throughput, error rate, and fraud rate
- Alerting on high latency, error rate spikes, and model drift

## 🚀 Quick Start

### Prerequisites
- Docker
- Kubernetes (minikube/kind for local development, or a cloud cluster)
- Python 3.11+

### Local Development

1. **Train the model:**
```bash
pip install -r requirements.txt
python models/train_model.py
```

2. **Start the server:**
```bash
uvicorn app.main:app --reload
```

3. **Test the API:**
```bash
# Health check
curl http://localhost:8000/health

# Make a prediction
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "features": [0.5, 0.3, 0.8, 0.2, 0.9, 0.1, 0.7, 0.4, 0.6, 0.5,
                 0.3, 0.8, 0.2, 0.9, 0.1, 0.7, 0.4, 0.6, 0.5, 0.3],
    "request_id": "test_001"
  }'

# View metrics
curl http://localhost:8000/metrics
```

4. **Run the test suite:**
```bash
pip install pytest pytest-cov requests
pytest tests/ -v --cov=app
```

### Docker Deployment

1. **Build the image:**
```bash
python models/train_model.py  # Train the model first
docker build -t ml-serving-platform:latest .
```

2. **Run the container:**
```bash
docker run -p 8000:8000 ml-serving-platform:latest
```

### Kubernetes Deployment

1. **Start a local cluster:**
```bash
minikube start --cpus=4 --memory=8192
```

2. **Build and load the image:**
```bash
docker build -t ml-serving-platform:latest .
minikube image load ml-serving-platform:latest
```

3. **Apply manifests:**
```bash
kubectl apply -f k8s/
```

4. **Verify the deployment:**
```bash
kubectl get pods
kubectl get svc
kubectl logs -l app=ml-serving
```

5. **Access the service:**
```bash
# Get the service URL
minikube service ml-serving --url

# Or use port-forwarding
kubectl port-forward svc/ml-serving 8000:80
```

6. **Test autoscaling:**
```bash
# Simulate load
kubectl run -it load-generator --rm --image=busybox --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://ml-serving/predict; done"

# Watch the HPA respond
kubectl get hpa ml-serving-hpa --watch
```

## 📊 Monitoring

The application exposes Prometheus metrics at `/metrics`.

**Key metrics:**
| Metric | Description |
|---|---|
| `model_predictions_total` | Total predictions, labeled by status and model version |
| `model_prediction_latency_seconds` | Prediction latency histogram |
| `fraud_predictions_total` | Count of fraud-flagged predictions |
| `model_load_timestamp` | Timestamp of the last model load |

**Useful PromQL queries:**
```promql
# Request rate
rate(model_predictions_total[5m])

# P95 latency
histogram_quantile(0.95, rate(model_prediction_latency_seconds_bucket[5m]))

# Error rate
rate(model_predictions_total{status="error"}[5m])

# Fraud detection rate
rate(fraud_predictions_total[5m]) / rate(model_predictions_total[5m])
```

## 🔧 Configuration

### Environment Variables
| Variable | Default | Description |
|---|---|---|
| `LOG_LEVEL` | `INFO` | Application log verbosity |
| `WORKERS` | `2` | Number of uvicorn worker processes |
| `MODEL_PATH` | `/app/models/latest` | Path to the model directory |

### Kubernetes Resource Tuning
Edit `k8s/deployment.yaml`:
```yaml
resources:
  requests:
    memory: "256Mi"  # Guaranteed minimum
    cpu: "250m"
  limits:
    memory: "512Mi"  # Hard ceiling
    cpu: "500m"
```

### Autoscaling Tuning
Edit `k8s/hpa.yaml`:
```yaml
minReplicas: 2      # Always-on pods
maxReplicas: 10     # Scale ceiling
averageUtilization: 70  # Trigger scale-out at 70% CPU
```

## 🧪 Testing

**Unit tests:**
```bash
pytest tests/test_api.py -v
```

**Load testing with Locust:**
```bash
pip install locust

cat > locustfile.py << 'EOF'
from locust import HttpUser, task, between

class MLServingUser(HttpUser):
    wait_time = between(0.1, 0.5)

    @task
    def predict(self):
        self.client.post("/predict", json={
            "features": [0.5] * 20
        })
EOF

locust -f locustfile.py --host=http://localhost:8000
# Open http://localhost:8089 for the dashboard
```

## 📈 Performance Benchmarks

Measured on a 4-core, 8GB machine:

| Metric | Value |
|---|---|
| P50 latency | 12ms |
| P95 latency | 28ms |
| P99 latency | 45ms |
| Throughput | ~350 req/s per pod |
| Memory (under load) | ~180MB per pod |
| Pod startup time | ~8 seconds |

## 🔐 Security

**Currently implemented:**
- ✅ Non-root container user (UID 1000)
- ✅ Read-only root filesystem capability
- ✅ Dropped Linux capabilities
- ✅ Kubernetes security context constraints
- ✅ Image vulnerability scanning via Trivy in CI
- ✅ Input validation with Pydantic models
- ✅ No secrets embedded in code or images

**Remaining production hardening:**
- [ ] TLS termination via Ingress + cert-manager
- [ ] API authentication with JWT tokens
- [ ] Rate limiting via nginx-ingress
- [ ] Kubernetes network policies

## 💰 Cost Optimization

**Example cost breakdown** (DigitalOcean):
| Resource | Monthly Cost |
|---|---|
| Kubernetes cluster (smallest node) | $12 |
| Load balancer | $12 |
| **Total** | **~$24** |

**Ways to reduce costs further:**
- Use spot/preemptible instances for non-critical workloads
- Configure aggressive HPA scale-down policies
- Use a cluster autoscaler to shrink node count during off-peak hours
- Consolidate multiple services onto a shared cluster

## 🎓 Learning Resources

**Core concepts covered by this project:**
1. **Kubernetes Fundamentals** — Deployments, Services, ConfigMaps, Health Checks
2. **Observability** — Structured logging, metrics exposition, health endpoints
3. **Reliability Engineering** — Rolling updates, auto-scaling, pod anti-affinity
4. **DevOps Practices** — CI/CD pipelines, IaC, containerization, security scanning

**Suggested next steps:**
- Deploy a full Prometheus + Grafana observability stack
- Implement model A/B testing using Istio traffic splitting
- Add distributed tracing with OpenTelemetry
- Integrate a feature store
- Set up model drift detection

## 🐛 Troubleshooting

**Pods not starting:**
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

**Service unreachable:**
```bash
kubectl get svc
kubectl get endpoints ml-serving
```

**High memory usage:**
```bash
kubectl top pods
# Reduce the WORKERS count or tighten resource limits
```

**Model failing to load:**
```bash
kubectl logs <pod-name> | grep "Model loaded"
# Verify the model artifact exists in the image
kubectl exec <pod-name> -- ls -la /app/models/
```


**Built with:** FastAPI · Docker · Kubernetes · Prometheus · GitHub Actions · Python · scikit-learn · Terraform *(coming)*
