# InnovaTech — Orquestación en AWS EKS con CI/CD (EP3)

Sistema de despachos de InnovaTech desplegado en **Amazon EKS**, con pipeline
**CI/CD en GitHub Actions** (build → push a ECR → deploy) y **autoscaling** por HPA.


## 1. Arquitectura final

- **EKS** (`innovatech-eks`, us-east-1, 2 nodos `t3.medium`, managed node group).
- **Frontend** React servido por **nginx**, expuesto con un Service `LoadBalancer` (ELB público).
  nginx hace de *reverse proxy*: `/api/v1/ventas` → `ventas-svc`, `/api/v1/despachos` → `despachos-svc`.
- **Backends** Spring Boot `ventas` (:8080) y `despachos` (:8081), como `ClusterIP` (internos).
- **MySQL 8** interno (`ClusterIP` headless), 2 esquemas: `ventasdb`, `despachosdb`.
- Comunicación Front → Back por **DNS interno** del clúster.

(URL pública del frontend: `http://a8b5441d2fce246e7b3e3de9b387ef3d-474298067.us-east-1.elb.amazonaws.com/`)


## 2. Roles, redes, autoscaling y balanceadores

- **IAM:** se reutiliza `LabRole` como rol del clúster y de los nodos (restricción de AWS Academy).
- **Redes:** VPC creada por eksctl, subredes públicas, Security Groups gestionados por EKS.
- **Balanceador:** ELB público vía Service `LoadBalancer` del frontend (único componente expuesto).
- **Autoscaling:** **HPA** sobre CPU al **50%** para `ventas` y `despachos` (min 1, max 4).
  Justificación del umbral: margen ante picos sin saturar y sin *flapping*. Requiere `metrics-server`.
- **Secrets:** credenciales de BD en un `Secret` de Kubernetes (`db-secret`), inyectadas por `secretKeyRef`.

## 3. Pipeline CI/CD

- Workflow: `.github/workflows/deploy.yml`. Dispara en `push` a `main`.
- Pasos: credenciales AWS → login ECR → build & push de 3 imágenes (tag = SHA del commit) →
  `aws eks update-kubeconfig` → `kubectl set image` + `rollout status`.
- Secretos del repo: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`.

### Métricas del pipeline (IE6)
| Métrica | Valor |
|---|---|
| Duración total del job | TODO |
| Build (3 imágenes) | TODO |
| Deploy (rollouts) | TODO |
| Fallos / reintentos | TODO |

## 4. Evidencias

- Frontend público funcionando

- Front → Back operativo (flujo venta→despacho→cierre)
- `kubectl get pods,svc,hpa -n innovatech`
- Autoscaling bajo carga (HPA antes/después)
- Self-healing (pod borrado se recrea)
- Logs (`kubectl logs`)
- Pipeline verde en Actions

## 5. Problemas encontrados y cómo se resolvieron

1. **Frontend con IPs LAN hardcodeadas** (`192.168.30`, `192.168.320`): no conectaba a ningún backend.
   → URLs cambiadas a relativas + reverse proxy nginx.
2. **Bug `entregado` vs `despachado`**: el front leía un campo inexistente. → unificado a `despachado`.
3. **Tests de contexto fallaban sin MySQL**: la imagen se construye con `-DskipTests`.
4. **Learner Lab no permite crear roles IAM**: se reutiliza `LabRole`; OIDC desactivado.
5. **Credenciales temporales en CI/CD**: se documentó el refresco de secretos cada sesión

## 6. Cómo reproducir
```bash
bash scripts/01-setup-wsl-tools.sh
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/<ACCOUNT_ID>/${ACCOUNT_ID}/g" infra/eks-cluster.yaml
eksctl create cluster -f infra/eks-cluster.yaml
bash scripts/02-ecr-create.sh
bash scripts/03-build-push.sh latest
bash scripts/04-deploy.sh
```

## 7. Integrantes
- Jorge Farias
- Sebastian Garrido
