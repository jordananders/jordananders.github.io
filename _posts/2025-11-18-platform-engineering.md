---
layout: default
title:  "Platform Engineering: Building Internal Developer Platforms"
date:   2025-11-18 12:00:00
categories: DevOps PlatformEngineering IDP
---

Platform engineering builds self-service capabilities for developers. For ML teams, this means providing standardized model deployment, GPU allocation, and experiment tracking.

## Internal Developer Platform Components

```
┌─────────────────────────────────────────┐
│           Developer Portal              │
│    (Backstage / Port / Humanitec)       │
├─────────────┬─────────────┬─────────────┤
│  Templates  │  Service    │  Resource   │
│  & Docs     │  Catalog    │  Requests   │
├─────────────┴─────────────┴─────────────┤
│           Platform APIs                  │
├─────────────┬─────────────┬─────────────┤
│ Kubernetes  │  CI/CD      │  Observ-    │
│ Operators   │  Pipelines  │  ability    │
├─────────────┴─────────────┴─────────────┤
│        Infrastructure Layer              │
│   (Kubernetes, Cloud, Databases)         │
└─────────────────────────────────────────┘
```

## Backstage Developer Portal

### Catalog Setup

```yaml
# catalog-info.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ml-inference-service
  description: Real-time ML inference API
  tags:
    - python
    - ml
    - pytorch
  annotations:
    github.com/project-slug: myorg/ml-inference
    backstage.io/techdocs-ref: dir:.
    prometheus.io/alert: ml-inference
spec:
  type: service
  lifecycle: production
  owner: ml-platform-team
  system: recommendation-system
  dependsOn:
    - resource:feature-store
    - component:model-registry
  providesApis:
    - ml-inference-api

---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: ml-inference-api
  description: ML Inference REST API
spec:
  type: openapi
  lifecycle: production
  owner: ml-platform-team
  definition:
    $text: ./openapi.yaml
```

### Software Templates

```yaml
# templates/ml-service/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: ml-service-template
  title: ML Service Template
  description: Create a new ML inference service
  tags:
    - python
    - ml
    - kubernetes
spec:
  owner: ml-platform-team
  type: service

  parameters:
    - title: Service Configuration
      required:
        - name
        - model_type
      properties:
        name:
          title: Service Name
          type: string
          pattern: '^[a-z0-9-]+$'
        description:
          title: Description
          type: string
        model_type:
          title: Model Type
          type: string
          enum:
            - pytorch
            - tensorflow
            - onnx
            - transformers
        gpu_required:
          title: Requires GPU
          type: boolean
          default: false

    - title: Resource Configuration
      properties:
        replicas:
          title: Replicas
          type: integer
          default: 2
        cpu_limit:
          title: CPU Limit
          type: string
          default: "2"
        memory_limit:
          title: Memory Limit
          type: string
          default: "4Gi"
        gpu_count:
          title: GPU Count
          type: integer
          default: 1

    - title: Repository
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com

  steps:
    - id: fetch-base
      name: Fetch Base Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          model_type: ${{ parameters.model_type }}

    - id: fetch-kubernetes
      name: Fetch Kubernetes Manifests
      action: fetch:template
      input:
        url: ./kubernetes
        targetPath: ./kubernetes
        values:
          name: ${{ parameters.name }}
          replicas: ${{ parameters.replicas }}
          cpu_limit: ${{ parameters.cpu_limit }}
          memory_limit: ${{ parameters.memory_limit }}
          gpu_required: ${{ parameters.gpu_required }}
          gpu_count: ${{ parameters.gpu_count }}

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: ['github.com']
        repoUrl: ${{ parameters.repoUrl }}
        description: ${{ parameters.description }}

    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

    - id: create-argocd-app
      name: Create ArgoCD Application
      action: argocd:create-resources
      input:
        appName: ${{ parameters.name }}
        argoInstance: main
        namespace: ml-services
        repoUrl: ${{ steps.publish.output.remoteUrl }}
        path: kubernetes

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
```

## Crossplane for Infrastructure

### Composite Resource Definition

```yaml
# xrd.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xmlplatforms.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XMLPlatform
    plural: xmlplatforms
  claimNames:
    kind: MLPlatform
    plural: mlplatforms
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                environment:
                  type: string
                  enum: [dev, staging, prod]
                gpu:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                    count:
                      type: integer
                    type:
                      type: string
                storage:
                  type: object
                  properties:
                    modelRegistry:
                      type: string
                    featureStore:
                      type: string
              required:
                - environment

---
# composition.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: mlplatform-aws
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XMLPlatform
  resources:
    # EKS Cluster with GPU nodes
    - name: eks-cluster
      base:
        apiVersion: eks.aws.crossplane.io/v1beta1
        kind: Cluster
        spec:
          forProvider:
            region: us-east-1
            version: "1.28"
      patches:
        - fromFieldPath: spec.environment
          toFieldPath: metadata.labels.environment

    # GPU Node Group
    - name: gpu-nodegroup
      base:
        apiVersion: eks.aws.crossplane.io/v1beta1
        kind: NodeGroup
        spec:
          forProvider:
            instanceTypes:
              - p3.2xlarge
            scalingConfig:
              minSize: 0
              maxSize: 10
      patches:
        - fromFieldPath: spec.gpu.count
          toFieldPath: spec.forProvider.scalingConfig.desiredSize
        - type: FromCompositeFieldPath
          fromFieldPath: spec.gpu.enabled
          toFieldPath: spec.forProvider.scalingConfig.minSize
          transforms:
            - type: convert
              convert:
                toType: int

    # S3 Bucket for Models
    - name: model-bucket
      base:
        apiVersion: s3.aws.crossplane.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-1
            acl: private
      patches:
        - fromFieldPath: spec.storage.modelRegistry
          toFieldPath: metadata.name
```

### Resource Claim

```yaml
# mlplatform-claim.yaml
apiVersion: platform.example.com/v1alpha1
kind: MLPlatform
metadata:
  name: recommendation-platform
  namespace: ml-team
spec:
  environment: prod
  gpu:
    enabled: true
    count: 4
    type: nvidia-a100
  storage:
    modelRegistry: recommendation-models
    featureStore: recommendation-features
```

## Kubernetes Operators

### Custom ML Operator

```go
// controllers/mlmodel_controller.go
package controllers

import (
    "context"
    mlv1 "github.com/example/ml-operator/api/v1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    ctrl "sigs.k8s.io/controller-runtime"
)

type MLModelReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *MLModelReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // Fetch MLModel resource
    var mlModel mlv1.MLModel
    if err := r.Get(ctx, req.NamespacedName, &mlModel); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Create serving deployment
    deployment := r.buildDeployment(&mlModel)
    if err := ctrl.SetControllerReference(&mlModel, deployment, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }

    // Create or update deployment
    if err := r.Create(ctx, deployment); err != nil {
        if errors.IsAlreadyExists(err) {
            if err := r.Update(ctx, deployment); err != nil {
                return ctrl.Result{}, err
            }
        } else {
            return ctrl.Result{}, err
        }
    }

    // Update status
    mlModel.Status.Phase = "Running"
    mlModel.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local",
        mlModel.Name, mlModel.Namespace)

    if err := r.Status().Update(ctx, &mlModel); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

func (r *MLModelReconciler) buildDeployment(model *mlv1.MLModel) *appsv1.Deployment {
    replicas := int32(model.Spec.Replicas)

    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      model.Name,
            Namespace: model.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{"app": model.Name},
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{"app": model.Name},
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "model-server",
                            Image: model.Spec.Image,
                            Ports: []corev1.ContainerPort{
                                {ContainerPort: 8080},
                            },
                            Resources: model.Spec.Resources,
                            Env: []corev1.EnvVar{
                                {
                                    Name:  "MODEL_NAME",
                                    Value: model.Spec.ModelName,
                                },
                                {
                                    Name:  "MODEL_VERSION",
                                    Value: model.Spec.ModelVersion,
                                },
                            },
                        },
                    },
                },
            },
        },
    }

    // Add GPU resources if specified
    if model.Spec.GPU.Enabled {
        deployment.Spec.Template.Spec.Containers[0].Resources.Limits[
            "nvidia.com/gpu"] = resource.MustParse(
            fmt.Sprintf("%d", model.Spec.GPU.Count))
    }

    return deployment
}
```

### Custom Resource

```yaml
# api/v1/mlmodel_types.go -> Generated CRD
apiVersion: ml.example.com/v1
kind: MLModel
metadata:
  name: recommendation-model
  namespace: ml-production
spec:
  modelName: recommendation
  modelVersion: v2.1.0
  image: myregistry/ml-server:v2.1.0
  replicas: 3
  resources:
    requests:
      memory: "4Gi"
      cpu: "2"
    limits:
      memory: "8Gi"
      cpu: "4"
  gpu:
    enabled: true
    count: 1
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPU: 70
```

## Self-Service Portal

### Resource Request API

```python
# platform_api.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import kubernetes

app = FastAPI(title="ML Platform API")

class GPURequest(BaseModel):
    team: str
    project: str
    gpu_type: str
    count: int
    duration_hours: int
    justification: str

class EnvironmentRequest(BaseModel):
    name: str
    team: str
    template: str
    config: dict

@app.post("/api/v1/gpu/request")
async def request_gpu(request: GPURequest):
    """Request GPU resources for ML training"""
    # Validate quota
    quota = await get_team_quota(request.team)
    if request.count > quota.available_gpus:
        raise HTTPException(
            status_code=400,
            detail=f"Requested {request.count} GPUs but only "
                   f"{quota.available_gpus} available"
        )

    # Create GPU allocation
    allocation = await create_gpu_allocation(
        team=request.team,
        project=request.project,
        gpu_type=request.gpu_type,
        count=request.count,
        duration=request.duration_hours
    )

    # Notify team
    await notify_team(request.team, f"GPU allocation {allocation.id} created")

    return {
        "allocation_id": allocation.id,
        "namespace": allocation.namespace,
        "expires_at": allocation.expires_at
    }

@app.post("/api/v1/environments")
async def create_environment(request: EnvironmentRequest):
    """Create a new development environment"""
    # Validate template exists
    template = await get_template(request.template)
    if not template:
        raise HTTPException(status_code=404, detail="Template not found")

    # Apply Crossplane claim
    k8s = kubernetes.client.CustomObjectsApi()
    claim = build_claim(request, template)

    result = k8s.create_namespaced_custom_object(
        group="platform.example.com",
        version="v1alpha1",
        namespace=request.team,
        plural="mlplatforms",
        body=claim
    )

    return {
        "environment_id": result['metadata']['name'],
        "status": "provisioning"
    }

@app.get("/api/v1/catalog/services")
async def list_services(team: str = None):
    """List available services from catalog"""
    services = await get_catalog_services(team)
    return {"services": services}
```

## Golden Paths

### ML Project Template Structure

```
ml-project-template/
├── .github/
│   └── workflows/
│       ├── train.yml
│       ├── test.yml
│       └── deploy.yml
├── src/
│   ├── data/
│   ├── features/
│   ├── models/
│   └── serving/
├── tests/
│   ├── unit/
│   └── integration/
├── kubernetes/
│   ├── base/
│   └── overlays/
├── notebooks/
├── configs/
│   ├── training.yaml
│   └── serving.yaml
├── Dockerfile
├── requirements.txt
├── pyproject.toml
└── catalog-info.yaml
```

## Metrics and SLOs

### Platform Metrics

```yaml
# platform-slos.yaml
apiVersion: sloth.slok.dev/v1
kind: PrometheusServiceLevel
metadata:
  name: ml-platform-slos
spec:
  service: "ml-platform"
  labels:
    team: platform
  slos:
    - name: "environment-provisioning"
      objective: 99
      description: "Environment provisioning success rate"
      sli:
        events:
          errorQuery: >
            sum(rate(platform_environment_provision_errors_total[5m]))
          totalQuery: >
            sum(rate(platform_environment_provision_total[5m]))
      alerting:
        pageAlert:
          disable: false
        ticketAlert:
          disable: false

    - name: "gpu-availability"
      objective: 95
      description: "Requested GPUs available within 5 minutes"
      sli:
        events:
          errorQuery: >
            sum(rate(platform_gpu_request_timeout_total[5m]))
          totalQuery: >
            sum(rate(platform_gpu_request_total[5m]))
```

## Best Practices

1. **Self-service first**: Reduce tickets, enable developers
2. **Golden paths**: Opinionated but flexible templates
3. **Documentation**: Keep docs close to code
4. **Automation**: Eliminate manual toil
5. **Observability**: Track platform health
6. **Feedback loops**: Iterate based on usage
7. **Security by default**: Bake in best practices
8. **Progressive complexity**: Simple start, advanced options

## Resources

- [Backstage Documentation](https://backstage.io/docs/)
- [Crossplane Guides](https://crossplane.io/docs/)
- [Platform Engineering Maturity Model](https://tag-app-delivery.cncf.io/whitepapers/platforms/)
- [Team Topologies](https://teamtopologies.com/)

---

*Questions about platform engineering? [Let me know](mailto:jordan@jordananderson.us).*
