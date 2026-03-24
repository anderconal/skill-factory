# Technical Stories: AI/ML

Stories técnicas específicas para proyectos de IA/ML. Complementan las stories de `_base.md`.

> **Nota:** Proyectos de IA suelen tener UI (web/mobile) para consumir el modelo. Combinar con templates de `web.md` o `mobile.md` según aplique.

## Stories

```yaml
stories:
  - ref: "ai-skeleton"
    title: "Walking skeleton - AI inference end-to-end"
    effort: 3
    area: "walking-skeleton"
    description: |
      As a development team,
      I want a minimal AI inference deployed,
      so that we validate the ML pipeline and serving infrastructure before complex models.
    acceptanceCriteria:
      - "Simple model deployed (can be placeholder/dummy)"
      - "API endpoint accepts input and returns prediction"
      - "Response time under 5 seconds for simple input"
      - "Works in deployed environment (not just notebook)"
    verifiableValue: "Call /predict endpoint → get model response"
    priority: "High"
    notes: "Model can be trivial (e.g., echo input). Goal is to validate infrastructure."

  - ref: "ai-api"
    title: "ML API design and implementation"
    effort: 3
    area: "infrastructure"
    description: |
      As a client application,
      I want a well-designed API to consume ML models,
      so that integration is straightforward and responses are predictable.
    acceptanceCriteria:
      - "REST or GraphQL endpoint for inference"
      - "Request/response schema documented (OpenAPI/GraphQL schema)"
      - "Proper error responses (400 for bad input, 500 for model errors)"
      - "Request validation before hitting model"
      - "Async endpoint option for long-running predictions (optional)"
    verifiableValue: "API documentation available and matches implementation"
    dependsOn: ["ai-skeleton"]

  - ref: "ai-infra"
    title: "ML compute infrastructure"
    effort: 3
    area: "infrastructure"
    description: |
      As a development team,
      I want proper compute resources for ML workloads,
      so that models train and serve efficiently.
    acceptanceCriteria:
      - "GPU instances available for training (if needed)"
      - "Inference runs on appropriate compute (GPU/CPU based on model)"
      - "Auto-scaling for inference (optional for MVP)"
      - "Cost monitoring in place"
    verifiableValue: "Check cloud console → ML resources provisioned and model running"
    notes: "Scope depends heavily on model requirements"

  - ref: "ai-monitoring"
    title: "ML model monitoring"
    effort: 2
    area: "observability"
    description: |
      As a development team,
      I want to monitor model performance in production,
      so that we detect degradation and users continue to get accurate predictions.
    acceptanceCriteria:
      - "Inference latency tracked (p50, p95, p99)"
      - "Prediction distribution monitored (detect drift)"
      - "Error rate tracked"
      - "Alerts for anomalies"
    verifiableValue: "Dashboard shows inference metrics and prediction distribution"
    notes: "Data drift detection may be phase 2"

  - ref: "ai-training-pipeline"
    title: "Model training pipeline"
    effort: 5
    area: "infrastructure"
    description: |
      As a data scientist,
      I want a reproducible training pipeline,
      so that we can retrain models consistently and track experiments.
    acceptanceCriteria:
      - "Training code in repository (not just notebooks)"
      - "Training data versioned or referenced by version"
      - "Hyperparameters configurable"
      - "Training runs logged (MLflow/W&B/TensorBoard)"
      - "Model artifacts stored with versioning"
    verifiableValue: "Trigger training run → model artifact produced and logged"
    optional: true
    notes: "May not be needed if using pre-trained/fine-tuned models"

  - ref: "ai-model-registry"
    title: "Model versioning and registry"
    effort: 2
    area: "infrastructure"
    description: |
      As a development team,
      I want models versioned and tracked,
      so that we can deploy specific versions and rollback if needed.
    acceptanceCriteria:
      - "Models stored in registry (MLflow/SageMaker/Vertex AI/custom)"
      - "Each model has version identifier"
      - "Metadata tracked (training date, metrics, data version)"
      - "Easy to deploy specific version"
    verifiableValue: "View registry → see model versions with metadata"
    dependsOn: ["ai-skeleton"]

  - ref: "ai-web-ui"
    title: "Web UI for AI consumption"
    effort: 3
    area: "ux"
    description: |
      As a user,
      I want a web interface to interact with the AI,
      so that I can use the model without technical knowledge.
    acceptanceCriteria:
      - "Web page with input form"
      - "Submit triggers API call to model"
      - "Results displayed clearly"
      - "Loading state while model processes"
      - "Error messages for failures"
    verifiableValue: "Open web UI → submit input → see AI response"
    notes: "If project needs full web app, combine with web.md template"
```

## Architecture Considerations

### Inference Serving
- **Real-time:** API endpoint, low latency required
- **Batch:** Scheduled jobs, process many inputs
- **Streaming:** WebSocket/SSE for progressive output (e.g., LLM)

### Model Deployment Options
- Containerized (Docker + Kubernetes)
- Serverless (Lambda, Cloud Functions) - watch cold start
- Managed services (SageMaker, Vertex AI, Azure ML)

### Common Frameworks
- **Training:** PyTorch, TensorFlow, scikit-learn
- **Serving:** TorchServe, TensorFlow Serving, FastAPI + custom
- **Experiment tracking:** MLflow, Weights & Biases, Neptune

## Typical Epic Structure

```
Epic: Technical Infrastructure (AI/ML)
├── Walking skeleton - AI inference end-to-end
├── [from _base] Infrastructure as Code setup
├── [from _base] Basic CI/CD pipeline
├── ML compute infrastructure
├── ML API design and implementation
├── [from _base] Basic telemetry and error tracking
├── ML model monitoring
├── Model versioning and registry
├── Model training pipeline (optional)
├── [from _base] User authentication (if UI)
└── Web UI for AI consumption (if needed)
```

## Notes for AI Projects

1. **Start simple:** Use a placeholder model for walking skeleton
2. **Separate concerns:** Training pipeline vs serving infrastructure
3. **Monitor everything:** AI systems degrade silently (data drift)
4. **Cost awareness:** GPU compute is expensive; monitor usage
5. **Consider hybrid:** Combine with web.md or mobile.md for full product
