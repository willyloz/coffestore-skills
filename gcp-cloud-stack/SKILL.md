---
name: gcp-cloud-stack
description: >
  Expertise en Google Cloud Platform: GKE/Kubernetes, BigQuery, Cloud Run,
  Cloud Functions, IAM/seguridad, Terraform para IaC y arquitectura cloud.
  Usar cuando se diseñen o implementen soluciones en GCP, se escriban manifests
  de Kubernetes, módulos Terraform, queries BigQuery, configuraciones de IAM,
  pipelines Cloud Build o se evalúen costos de infraestructura cloud.
  Trigger: GCP, Google Cloud, GKE, Kubernetes, BigQuery, Cloud Run, Terraform,
  IAM, bucket, Cloud Storage, Pub/Sub, Dataflow, Cloud SQL, VPC, firewall,
  service account, deployment, pod, helm, kubectl, gcloud.
---

# Stack GCP — Google Cloud Platform

## GKE / Kubernetes — Manifests y operaciones

### Deployment estándar con best practices
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
  namespace: production
  labels:
    app: mi-app
    version: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      serviceAccountName: mi-app-sa   # Workload Identity
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
      - name: mi-app
        image: us-central1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

### HPA — Autoscaling
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mi-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mi-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Comandos kubectl frecuentes
```bash
# Ver estado de pods
kubectl get pods -n production -o wide
kubectl describe pod mi-app-xxx -n production
kubectl logs mi-app-xxx -n production --tail=100 -f

# Rollout
kubectl rollout status deployment/mi-app -n production
kubectl rollout undo deployment/mi-app -n production   # rollback

# Ejecutar comando en pod
kubectl exec -it mi-app-xxx -n production -- bash

# Port forward para debugging
kubectl port-forward svc/mi-app 8080:80 -n production
```

---

## BigQuery — Queries y optimización

### Estructura de datasets recomendada
```
proyecto/
  raw/          ← datos sin procesar (particionados por fecha ingesta)
  staging/      ← transformaciones intermedias
  analytics/    ← tablas finales para reportes
  temp/         ← tablas temporales (expiración 24h)
```

### Tabla particionada y clustered (siempre usar)
```sql
CREATE TABLE analytics.ventas_diarias
PARTITION BY DATE(fecha)
CLUSTER BY region, producto_id
OPTIONS (
  partition_expiration_days = 365,
  require_partition_filter = true
)
AS SELECT * FROM staging.ventas WHERE FALSE;
```

### Estimación de costo antes de ejecutar
```sql
-- En BigQuery Console: usar "Dry Run" o:
-- bq query --dry_run --use_legacy_sql=false 'SELECT ...'
-- Regla: $5 por TB procesado (on-demand)
-- Siempre agregar filtro de partición: WHERE DATE(fecha) = '2026-01-01'
```

### Queries analíticas frecuentes
```sql
-- Ventas por región con comparación período anterior
WITH actual AS (
  SELECT region, SUM(total) as ventas
  FROM analytics.ventas_diarias
  WHERE DATE(fecha) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) AND CURRENT_DATE()
  GROUP BY region
),
anterior AS (
  SELECT region, SUM(total) as ventas
  FROM analytics.ventas_diarias
  WHERE DATE(fecha) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
                        AND DATE_SUB(CURRENT_DATE(), INTERVAL 31 DAY)
  GROUP BY region
)
SELECT
  a.region,
  a.ventas AS ventas_actual,
  ant.ventas AS ventas_anterior,
  ROUND((a.ventas - ant.ventas) / ant.ventas * 100, 2) AS variacion_pct
FROM actual a
LEFT JOIN anterior ant USING(region)
ORDER BY variacion_pct DESC;
```

---

## Cloud Run — Despliegue serverless

### Deploy rápido con gcloud
```bash
# Build y push a Artifact Registry
gcloud builds submit --tag us-central1-docker.pkg.dev/MI-PROYECTO/mi-repo/mi-app:latest

# Deploy en Cloud Run
gcloud run deploy mi-servicio \
  --image us-central1-docker.pkg.dev/MI-PROYECTO/mi-repo/mi-app:latest \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --min-instances 1 \          # evitar cold start en producción
  --max-instances 10 \
  --memory 512Mi \
  --cpu 1 \
  --set-env-vars "ENV=production" \
  --set-secrets "DB_PASSWORD=db-password:latest"
```

### Cloud Run con VPC Connector (para Cloud SQL privado)
```bash
gcloud run deploy mi-servicio \
  --vpc-connector mi-vpc-connector \
  --vpc-egress all-traffic \
  --add-cloudsql-instances MI-PROYECTO:us-central1:mi-instancia-sql
```

### Cloud Scheduler → Cloud Run (cron jobs)
```bash
gcloud scheduler jobs create http mi-job \
  --schedule "0 */6 * * *" \
  --uri "https://mi-servicio-xxx.run.app/api/proceso" \
  --http-method POST \
  --oidc-service-account-email mi-sa@MI-PROYECTO.iam.gserviceaccount.com
```

---

## Terraform — IaC para GCP

### Estructura de módulos recomendada
```
terraform/
  environments/
    dev/
      main.tf          ← llama a módulos
      variables.tf
      terraform.tfvars
    prod/
      main.tf
  modules/
    gke-cluster/
      main.tf
      variables.tf
      outputs.tf
    cloud-run-service/
      main.tf
      variables.tf
      outputs.tf
```

### Módulo Cloud Run reutilizable
```hcl
# modules/cloud-run-service/main.tf
resource "google_cloud_run_v2_service" "service" {
  name     = var.name
  location = var.region
  project  = var.project_id

  template {
    service_account = var.service_account_email

    scaling {
      min_instance_count = var.min_instances
      max_instance_count = var.max_instances
    }

    containers {
      image = var.image

      resources {
        limits = {
          cpu    = var.cpu
          memory = var.memory
        }
      }

      dynamic "env" {
        for_each = var.env_vars
        content {
          name  = env.key
          value = env.value
        }
      }
    }
  }
}

resource "google_cloud_run_v2_service_iam_member" "public" {
  count    = var.allow_unauthenticated ? 1 : 0
  location = google_cloud_run_v2_service.service.location
  name     = google_cloud_run_v2_service.service.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}
```

### Backend en GCS con locking
```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = { source = "hashicorp/google", version = "~> 5.0" }
  }
  backend "gcs" {
    bucket = "mi-proyecto-terraform-state"
    prefix = "environments/production"
  }
}
```

---

## IAM — Principio de mínimo privilegio

### Service Account para Cloud Run
```bash
# Crear SA con nombre descriptivo
gcloud iam service-accounts create mi-app-sa \
  --display-name "SA para mi-app en Cloud Run"

# Asignar solo los roles necesarios
gcloud projects add-iam-policy-binding MI-PROYECTO \
  --member "serviceAccount:mi-app-sa@MI-PROYECTO.iam.gserviceaccount.com" \
  --role "roles/cloudsql.client"

gcloud projects add-iam-policy-binding MI-PROYECTO \
  --member "serviceAccount:mi-app-sa@MI-PROYECTO.iam.gserviceaccount.com" \
  --role "roles/secretmanager.secretAccessor"
```

### Workload Identity para GKE
```bash
# Vincular KSA (Kubernetes SA) con GSA (Google SA)
gcloud iam service-accounts add-iam-policy-binding mi-app-sa@MI-PROYECTO.iam.gserviceaccount.com \
  --role "roles/iam.workloadIdentityUser" \
  --member "serviceAccount:MI-PROYECTO.svc.id.goog[namespace/ksa-name]"
```

---

## Costos — Estimaciones rápidas

| Servicio | Costo aproximado |
|---|---|
| GKE Autopilot | $0.10/vCPU-hora + $0.01/GB-hora RAM |
| Cloud Run | $0.00002400/vCPU-segundo (primer 180K CPU-seg/mes gratis) |
| BigQuery | $5/TB procesado (on-demand) ó slots reservados desde $2,000/mes |
| Cloud SQL (db-f1-micro) | ~$15/mes |
| GCS Standard | $0.020/GB/mes + $0.12/GB egress |
| Secret Manager | $0.06/versión activa/mes |

**Regla de oro**: Siempre usar `--dry-run` en Terraform y revisar el plan antes de apply en producción.
