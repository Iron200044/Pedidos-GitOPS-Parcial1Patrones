# Sistema de Gestión de Pedidos
### Parcial I — Patrones Arquitectónicos Avanzados

**Integrantes:** Samuel Acero García • Deivid Nicolás Urrea Lara • Samuel Esteban López Huertas  
**Repositorio:** https://github.com/Iron200044/Pedidos-GitOPS-Parcial1Patrones

---

## 1. Descripción General

Este proyecto implementa el despliegue completo de una aplicación de gestión de pedidos compuesta por tres capas: base de datos PostgreSQL, backend en Java Spring Boot y frontend en React. El empaquetado se realiza con Helm y la entrega continua es gestionada por ArgoCD sobre un clúster Kubernetes (Minikube).

| Componente | Imagen Docker | Puerto |
|------------|---------------|--------|
| PostgreSQL | bitnami/postgresql (Helm oficial) | 5432 |
| Backend | nicou21/pedido-backend:dev | 8080 |
| Frontend | nicou21/pedido-frontend:dev | 80 |

---

## 2. Estructura del Repositorio
```
charts/
  pedido-app/
    Chart.yaml            # Metadatos y dependencia de PostgreSQL (Bitnami)
    values-dev.yaml       # Valores para el entorno de desarrollo
    values-prod.yaml      # Valores para el entorno de producción
    charts/               # Subcharts generados por helm dependency update
    templates/            # Recursos Kubernetes parametrizados
      backend-deployment.yaml
      backend-service.yaml
      backend-secret.yaml
      backend-hpa.yaml
      frontend-deployment.yaml
      frontend-service.yaml
      ingress.yaml
environments/
  dev/application.yaml    # ArgoCD Application para pedido-dev
  prod/application.yaml   # ArgoCD Application para pedido-prod
```

---

## 3. Instalación Manual con Helm

### 3.1 Requisitos previos
- Minikube corriendo: `minikube start`
- Ingress habilitado: `minikube addons enable ingress`
- Helm instalado (v3+)
- kubectl configurado apuntando al cluster

### 3.2 Agregar el repositorio de Bitnami
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 3.3 Descargar dependencias del chart
```bash
cd charts/pedido-app
helm dependency update
```

### 3.4 Crear los namespaces
```bash
kubectl create namespace pedido-dev
kubectl create namespace pedido-prod
```

### 3.5 Desplegar en desarrollo
```bash
helm install pedido-app-dev . -f values-dev.yaml -n pedido-dev
```

### 3.6 Desplegar en producción
```bash
helm install pedido-app-prod . -f values-prod.yaml -n pedido-prod
```

### 3.7 Verificar el despliegue
```bash
kubectl get all -n pedido-dev
kubectl get all -n pedido-prod
```

### 3.8 Actualizar un despliegue existente
```bash
helm upgrade pedido-app-dev . -f values-dev.yaml -n pedido-dev
```

### 3.9 Eliminar un despliegue
```bash
helm uninstall pedido-app-dev -n pedido-dev
```

---

## 4. Configuración de ArgoCD

### 4.1 Instalación de ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4.2 Acceder a la UI de ArgoCD
```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```
Abrir en el navegador: `https://localhost:8081`  
Usuario: `admin`  
Contraseña:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

### 4.3 Aplicar las definiciones de Application
```bash
kubectl apply -f environments/dev/application.yaml
kubectl apply -f environments/prod/application.yaml
```

### 4.4 Cómo funciona la sincronización automática

Cada Application de ArgoCD apunta al repositorio Git y monitorea continuamente la rama `main`. Cuando se detecta un cambio en los archivos del chart o en los values, ArgoCD sincroniza automáticamente el estado del cluster sin necesidad de comandos manuales. La configuración incluye:

- `automated.prune: true` — elimina recursos que ya no están en Git
- `automated.selfHeal: true` — corrige desviaciones del estado deseado

| Application | Namespace | Values File | Sync Policy |
|-------------|-----------|-------------|-------------|
| pedido-app-dev | pedido-dev | values-dev.yaml | Automated |
| pedido-app-prod | pedido-prod | values-prod.yaml | Automated |

### 4.5 Verificar el estado de sincronización
```bash
kubectl get applications -n argocd
```
Estado esperado: `SYNC STATUS = Synced` y `HEALTH STATUS = Healthy` para ambas apps.

---

## 5. Endpoints de Acceso

### 5.1 Configurar el archivo hosts (Windows)
Ejecutar PowerShell como administrador:
```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 pedido-dev.local"
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 pedido-prod.local"
```

### 5.2 URLs de acceso

| Entorno | Frontend | Backend API | Ejemplo endpoint |
|---------|----------|-------------|------------------|
| Dev | http://pedido-dev.local | http://pedido-dev.local/api | http://pedido-dev.local/api/productos |
| Prod | http://pedido-prod.local | http://pedido-prod.local/api | http://pedido-prod.local/api/productos |

### 5.3 Reglas del Ingress
- `/` → redirige al frontend (puerto 80)
- `/api/productos` → redirige al backend (puerto 8080)
---

## 5.3 Endpoints del Backend

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/productos` | Obtener todos los productos |
| GET | `/api/productos/activos` | Obtener solo productos activos |
| GET | `/api/productos/{id}` | Obtener producto por ID |
| POST | `/api/productos` | Crear nuevo producto |
| PUT | `/api/productos/{id}` | Actualizar producto existente |
| DELETE | `/api/productos/{id}` | Eliminar producto |
| GET | `/api/productos/existe/{nombre}` | Verificar si existe producto por nombre |

## 6. Horizontal Pod Autoscaler (HPA)

El backend escala automáticamente según el uso de CPU:

| Parámetro | Dev | Prod |
|-----------|-----|------|
| Réplicas mínimas | 1 | 2 |
| Réplicas máximas | 5 | 8 |
| CPU objetivo | 70% | 70% |
```bash
kubectl get hpa -n pedido-dev
kubectl get hpa -n pedido-prod
```

---

## 7. Persistencia de Datos

PostgreSQL usa un PersistentVolumeClaim para mantener los datos aunque el pod se reinicie:

- Dev: PVC de 1Gi
- Prod: PVC de 2Gi
```bash
kubectl get pvc -n pedido-dev
kubectl get pvc -n pedido-prod
```

---

## 8. Buenas Prácticas Implementadas

- `limits` y `requests` de CPU/memoria definidos en todos los componentes
- Separación clara de valores por ambiente (`values-dev.yaml` / `values-prod.yaml`)
- Reutilización del chart oficial de PostgreSQL de Bitnami como dependencia
- Labels consistentes en todos los recursos Kubernetes
- Sincronización automática con `prune` y `selfHeal` habilitados en ArgoCD
