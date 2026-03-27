+++
comments = true
date = "2026-03-27T00:00:00-00:00"
draft = false
slug = "mlops-guide"
tags = ["mlops", "kubernetes", "machine-learning", "platform-engineering", "devops", "sre"]
title = "From Data Science to Production: MLOps CI/CD Pipeline with Real-time Accuracy Monitoring"
description = "Production machine learning requires more than training good models. This guide covers building a complete MLOps system with Kubernetes, Prometheus monitoring, and automated accuracy checks to catch model drift and prevent silent failures in production."
image = "img/mlops-header.jpg"
summary = "Production machine learning requires more than training good models. This guide covers building a complete MLOps system with Kubernetes, Prometheus monitoring, and automated accuracy checks to catch model drift and prevent silent failures in production."
+++

## The Problem

Your data science team trained an amazing ML model. Accuracy on test data: 94%.

You deploy it to Kubernetes. It runs great for 2 weeks.

Then accuracy silently drops to 72%.

Nobody notices for 10 days. By the time you catch it, the model has made 50,000 bad predictions.

This is the MLOps problem in a nutshell: **models decay, and without monitoring, you won't know until it's too late.**

---

## The Reality: Why Models Fail in Production

### It's Not the Training Code That Breaks

Your training pipeline works fine. You can rerun it on the same data, get the same model. The code is solid.

The problem is **data drift**. In production, your input data changes:
- User behavior shifts seasonally
- Feature distributions morph over time
- Edge cases you never saw in training appear in production
- The world changes, but your model doesn't

### The Traditional Approach (Manual and Fragile)

Most teams handle ML deployment like this:

1. Data scientist trains a model locally
2. "It's ready" - they export a pickle file or ONNX model
3. Engineer manually packages it into a Docker image
4. It gets deployed to Kubernetes (maybe)
5. It runs for a while
6. Something breaks (nobody's monitoring)
7. Someone manually checks logs
8. Maybe they retrain, maybe they downgrade the model
9. Repeat 6-8 three months later

**The problem:** Every step is manual. No automation. No feedback loop. No recovery.

### Why This Breaks at Scale

- **No versioning:** Which model version is running? What data trained it? Unknown.
- **No testing:** Did you test the model against a held-out dataset? Just hope it works.
- **No monitoring:** Is accuracy degrading? You won't know until someone complains.
- **No retraining:** When drift happens, you manually trigger retraining (if you remember).
- **No governance:** Audit trail? Compliance? Good luck explaining to regulators.

---

## The Solution: MLOps with Kubernetes + Prometheus + Automated Monitoring

**MLOps is the engineering discipline that brings DevOps practices to machine learning.**

Instead of manually managing model deployment, you:

1. **Automate the entire pipeline** - data validation → training → testing → deployment
2. **Monitor model performance** - accuracy, precision, recall, and data drift in real-time
3. **Trigger retraining automatically** - when drift is detected or on schedule
4. **Test before deployment** - validate model accuracy matches thresholds
5. **Deploy safely** - staged rollouts, canary deployments, instant rollback

---

## Our Approach: Kubernetes-Native MLOps

We built a production ML system on Kubernetes with these components:

### 1. **Automated Training Pipeline** (Orchestration)

Using Kubernetes CronJobs + custom training containers:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ml-retraining-pipeline
spec:
  schedule: "0 2 * * *"  # Runs daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: trainer
            image: myrepo/ml-trainer:latest
            env:
            - name: TRAINING_DATASET
              value: "s3://ml-data/training/"
            - name: MODEL_REGISTRY
              value: "http://mlflow:5000"
```

**What it does:**
- Fetches fresh data from S3 (or your data warehouse)
- Validates data quality (schema, statistical properties, anomalies)
- Trains the model from scratch
- Compares new model accuracy against baseline
- If accuracy passes threshold: register in MLflow Model Registry
- If accuracy fails: alert, don't deploy

### 2. **Model Registry** (Version Control for Models)

Using MLflow Model Registry:
- Every trained model is registered with metadata
- Tracks training dataset version, hyperparameters, metrics
- Maintains full lineage (which code, which data, which environment)
- Enables rollback to previous model versions

```
Model: fraud-detector
├── Version 1 (staging) - acc: 91.2%
├── Version 2 (staging) - acc: 93.1%
└── Version 3 (production) - acc: 93.1% (deployed 2 days ago)
```

### 3. **Staged Deployment** (Testing Before Production)

```
Data Flow: Training Data → Validation → Staging → Production

Staging Environment:
- Receives 10% of live traffic (shadow mode)
- Model makes predictions but doesn't affect users
- Prometheus compares staging accuracy vs baseline

If staging accuracy meets threshold (93%+) → Approve for production
```

### 4. **Real-time Accuracy Monitoring** (Prometheus)

This is the critical piece. You need to track model accuracy in production:

```yaml
# Prometheus config for ML metrics
global:
  scrape_interval: 15s

scrape_configs:
- job_name: 'ml-inference'
  static_configs:
  - targets: ['ml-service:8000']
```

Your ML service exposes metrics:

```python
from prometheus_client import Counter, Gauge, Histogram

# Counters
predictions_total = Counter('ml_predictions_total', 'Total predictions', ['model', 'outcome'])
correct_predictions = Counter('ml_correct_predictions_total', 'Correct predictions', ['model'])

# Gauges
model_accuracy = Gauge('ml_model_accuracy', 'Current model accuracy', ['model'])
data_drift_score = Gauge('ml_data_drift_score', 'Feature distribution drift', ['model'])

# In your inference code:
prediction = model.predict(features)
actual_label = get_ground_truth(request_id)  # From async feedback loop

if prediction == actual_label:
    correct_predictions.labels(model=model_name).inc()

predictions_total.labels(model=model_name, outcome='correct' if prediction == actual_label else 'incorrect').inc()
```

### 5. **Automated Alerts on Drift**

Prometheus alerting rules:

```yaml
groups:
- name: ml_alerts
  interval: 30s
  rules:
  - alert: ModelAccuracyDegraded
    expr: ml_model_accuracy < 0.90  # Alert if accuracy drops below 90%
    for: 5m
    annotations:
      summary: "Model accuracy dropped to {{ $value }}"
      action: "Check data drift. Consider retraining."

  - alert: DataDriftDetected
    expr: ml_data_drift_score > 0.15
    for: 10m
    annotations:
      summary: "Feature distribution shifted significantly"
      action: "Trigger emergency retraining"
```

When accuracy drops below 90%, you:
1. **Get notified immediately**
2. **Automatically trigger retraining**
3. **Compare new model vs baseline**
4. **If better: deploy to staging**
5. **If worse: rollback to previous version**

---

## Practical Example: Credit Risk Model

### The Setup

You have a credit risk model that predicts loan default probability.

- Training: 100K historical loans, 2% default rate
- Production: Running on Kubernetes, serving 500 requests/day
- Accuracy in test set: 94%

### Week 1: Everything Works
- Model accuracy: 93.8% ✓
- Data drift: 0.05 (normal)
- All systems green

### Week 3: Silent Failure
- Economy shifts: unemployment rises
- Default patterns change
- Model still predicts based on old economic patterns
- Model accuracy: 68% (unnoticed for 3 days)
- 1,200 bad predictions made

**Without MLOps:** You find out when the business notices the loan portfolio is worse than expected.

**With MLOps:**
- Day 1, hour 3: Prometheus alert fires
- Alert triggers automated retraining
- New model trained on last week's data (with new economic patterns)
- New model accuracy: 89% (not perfect, but better)
- Model deployed to staging automatically
- Accuracy verified in shadow mode
- Model promoted to production
- Total downtime: 2 hours instead of 72 hours

---

## The MLOps Maturity Progression

### Level 0: Manual Everything (Typical Today)
- Train model locally
- Export pickle/ONNX manually
- Docker image built manually
- Deployed manually to Kubernetes
- Zero monitoring
- Zero retraining automation

### Level 1: Automated Training (Easy Starting Point)
- Training pipeline triggers on schedule (CronJob)
- Data validation in pipeline
- Model automatically registered in MLflow
- Still manual deployment decision

### Level 2: Full CI/CD (Our Approach)
- Automated training on schedule + drift detection
- Automated testing (accuracy threshold checks)
- Automated deployment to staging (shadow mode)
- Real-time accuracy monitoring (Prometheus)
- Automated promotion to production if staging passes
- Automated retraining on drift detection
- Instant rollback if accuracy drops

---

## Implementation Checklist

To build this yourself on Kubernetes:

### 1. **Monitoring & Metrics**
- [ ] Prometheus deployed on cluster
- [ ] ML service exposes Prometheus metrics (predictions, accuracy, drift)
- [ ] Grafana dashboards for visualization
- [ ] Alert rules for accuracy degradation and drift

### 2. **Model Training Pipeline**
- [ ] CronJob for regular retraining
- [ ] Data validation step (schema, distributions)
- [ ] Model training containerized
- [ ] Accuracy testing before registration

### 3. **Model Registry**
- [ ] MLflow deployed
- [ ] Training pipeline registers models automatically
- [ ] Metadata tracked (dataset version, hyperparameters, metrics)

### 4. **Staged Deployment**
- [ ] Two ML service deployments (staging + production)
- [ ] Traffic mirroring or shadow mode for staging
- [ ] Automated promotion based on accuracy threshold

### 5. **Automated Retraining**
- [ ] Prometheus alerts trigger Kubernetes Job
- [ ] Or: CronJob checks drift score, triggers if needed
- [ ] New model compared against baseline
- [ ] Auto-rollback if model degrades

---

## Key Lessons

1. **Monitor accuracy, not just infrastructure** - CPU and memory are fine. Your model might be terrible.

2. **Data drift is silent** - Models don't throw errors. They just gradually get worse.

3. **Staging is critical** - Test model accuracy on 10% of real traffic before going 100%

4. **Automate the loop** - Without automation, you'll miss drift because nobody's monitoring 24/7

5. **Reproducibility matters** - Keep full lineage: training data version, code version, hyperparameters, metrics

6. **Speed matters** - From alert to retraining to deployment should be minutes, not days

---

## What We're Not Covering (Yet)

- Feature stores (Tecton, Feast) - manage features consistently across training/serving
- Advanced drift detection (Evidently AI, WhyLabs) - detect distribution shift automatically
- Online learning - retraining on single examples as feedback arrives
- Causal inference - understanding why accuracy changed
- Multi-armed bandits - A/B testing different models in production

These are Level 3 optimizations. Start with the basics above.

---

## Next Steps

1. **Deploy Prometheus** on your Kubernetes cluster
2. **Instrument your ML service** with prediction accuracy metrics
3. **Set up alerts** for accuracy drops
4. **Automate training** with CronJobs
5. **Add staging environment** for model testing
6. **Monitor** for the first week to catch issues

After one month, you'll have caught issues that would have been silent for weeks in the old system.

---

## References & Tools

- **Kubernetes CronJob:** https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
- **MLflow Model Registry:** https://mlflow.org/docs/latest/registry.html
- **Prometheus:** https://prometheus.io/
- **ML Monitoring:** Evidently AI, WhyLabs, Arize (enterprise options)
- **Feature Stores:** Tecton, Feast, Hopsworks
