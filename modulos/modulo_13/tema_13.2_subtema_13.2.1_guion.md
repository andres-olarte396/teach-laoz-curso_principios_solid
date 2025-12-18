# Guion de Video: Kubernetes Orchestration

**Duración**: 35 minutos  
**Objetivo**: Explicar Deployments, Services, ConfigMaps, Secrets y health checks con demos en vivo

---

## [00:00-01:30] Introducción

Bienvenidos al módulo de **Kubernetes Orchestration**. En este video veremos cómo Kubernetes gestiona aplicaciones containerizadas a escala empresarial.

**Problema que resuelve Kubernetes**:

Imagina que tienes una aplicación en Docker:

```bash
docker run -d -p 8080:8080 myapp:1.0.0
```

Esto funciona en tu laptop, pero en producción necesitas:

✅ Alta disponibilidad: 10 réplicas corriendo simultáneamente  
✅ Self-healing: Si un container falla, reiniciarlo automáticamente  
✅ Load balancing: Distribuir tráfico entre todas las réplicas  
✅ Rolling updates: Actualizar sin downtime  
✅ Configuración dinámica: Sin rebuild de images

**Kubernetes hace todo esto de forma declarativa**. Tú declaras el estado deseado, Kubernetes lo mantiene.

Hoy veremos:

1. Pods y Deployments
2. Services y load balancing
3. ConfigMaps y Secrets
4. Health checks
5. Demo completo: Sistema e-commerce

---

## [01:30-05:00] Pods: La Unidad Básica

Un **Pod** es el objeto más pequeño en Kubernetes. Puede contener 1 o más containers que comparten:

- Red (misma IP)
- Almacenamiento (volumes compartidos)
- Ciclo de vida (se crean/destruyen juntos)

**Demo: Pod simple**

```yaml
# nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
```

Aplicamos:

```bash
kubectl apply -f nginx-pod.yaml
kubectl get pods
```

Veremos:

```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          5s
```

**Problema**: Si eliminamos este pod manualmente, desaparece para siempre:

```bash
kubectl delete pod nginx-pod
kubectl get pods
# No pods found
```

**Solución**: Deployments.

---

## [05:00-09:00] Deployments: Gestión Declarativa

Un **Deployment** gestiona pods de forma declarativa. Proporciona:

✅ Réplicas automáticas  
✅ Rolling updates  
✅ Rollbacks  
✅ Self-healing

**Demo: Deployment con 3 réplicas**

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

Aplicamos:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -l app=nginx
```

Veremos 3 pods corriendo:

```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-7d8f5c9b7f-abc12  1/1     Running   0          10s
nginx-deployment-7d8f5c9b7f-def34  1/1     Running   0          10s
nginx-deployment-7d8f5c9b7f-ghi56  1/1     Running   0          10s
```

**Test de self-healing**: Eliminamos un pod manualmente:

```bash
kubectl delete pod nginx-deployment-7d8f5c9b7f-abc12
kubectl get pods -l app=nginx --watch
```

Veremos:

```
NAME                               READY   STATUS        RESTARTS   AGE
nginx-deployment-7d8f5c9b7f-abc12  1/1     Terminating   0          30s
nginx-deployment-7d8f5c9b7f-jkl78  0/1     ContainerCreating  0      1s
nginx-deployment-7d8f5c9b7f-jkl78  1/1     Running       0          5s
```

**Kubernetes detectó que faltan réplicas y creó una nueva automáticamente**. Siempre mantiene 3 réplicas corriendo.

**Escalado manual**:

```bash
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods -l app=nginx
```

Ahora hay 5 pods:

```
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-7d8f5c9b7f-def34  1/1     Running   0          2m
nginx-deployment-7d8f5c9b7f-ghi56  1/1     Running   0          2m
nginx-deployment-7d8f5c9b7f-jkl78  1/1     Running   0          1m
nginx-deployment-7d8f5c9b7f-mno90  1/1     Running   0          5s
nginx-deployment-7d8f5c9b7f-pqr12  1/1     Running   0          5s
```

---

## [09:00-13:00] Services: Networking y Load Balancing

**Problema**: Los pods son efímeros. Cada vez que se reinician, obtienen una IP nueva. ¿Cómo accederlos de forma estable?

**Solución**: Services.

Un **Service** proporciona:

✅ IP estable  
✅ DNS name  
✅ Load balancing automático entre pods

**Tipos de Services**:

| Tipo          | Uso                                    |
| ------------- | -------------------------------------- |
| ClusterIP     | Interno (comunicación entre servicios) |
| NodePort      | Externo (desarrollo/testing)           |
| LoadBalancer  | Externo (producción cloud)             |

**Demo: ClusterIP Service**

```yaml
# nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Aplicamos:

```bash
kubectl apply -f nginx-service.yaml
kubectl get services
```

Veremos:

```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.123.45   <none>        80/TCP    10s
```

**Test desde otro pod**:

```bash
kubectl run test-pod --image=busybox -it --rm -- sh

# Dentro del pod
wget -qO- http://nginx-service:80
```

Veremos el HTML de nginx. El Service está balanceando requests entre los 5 pods automáticamente.

**Verificar load balancing**: Modificamos nginx para mostrar el hostname:

```bash
# En cada pod nginx
kubectl exec -it <pod-name> -- bash
echo "Pod: $(hostname)" > /usr/share/nginx/html/index.html
```

Repetimos el test:

```bash
for i in $(seq 1 10); do
  wget -qO- http://nginx-service:80
done
```

Veremos diferentes hostnames en cada response, confirmando load balancing.

---

## [13:00-17:00] ConfigMaps y Secrets

**Problema**: Hardcodear configuración en la imagen es mala práctica (Factor 3 de 12-Factor App).

**Solución**:

- **ConfigMap**: Configuración no sensible (DATABASE_HOST, LOG_LEVEL)
- **Secret**: Credenciales (PASSWORD, API_KEY)

**Demo: Aplicación con configuración dinámica**

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  APP_ENV: "production"
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM= # base64: "password123"
  API_KEY: c2stYWJjMTIzeHl6 # base64: "sk-abc123xyz"
```

**Crear Secret desde línea de comandos**:

```bash
kubectl create secret generic app-secrets \
  --from-literal=DATABASE_PASSWORD=password123 \
  --from-literal=API_KEY=sk-abc123xyz
```

**Deployment usando ConfigMap y Secret**:

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
```

Aplicamos:

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f app-deployment.yaml
```

**Verificar variables inyectadas**:

```bash
kubectl exec -it <pod-name> -- env | grep -E "DATABASE|LOG_LEVEL|API_KEY"
```

Veremos:

```
DATABASE_HOST=postgres-service
DATABASE_PORT=5432
DATABASE_PASSWORD=password123
LOG_LEVEL=info
API_KEY=sk-abc123xyz
```

**Ventaja**: Para cambiar configuración, solo modificamos el ConfigMap, no la imagen.

---

## [17:00-21:00] Health Checks: Liveness y Readiness Probes

**Problema**: ¿Cómo sabe Kubernetes si un container está realmente funcionando?

Un container con STATUS Running puede estar:

- ❌ Deadlocked (proceso vivo pero colgado)
- ❌ Sin acceso a DB (app "running" pero no puede procesar requests)

**Solución**: Health checks.

| Probe      | Propósito                      | Acción al fallar                  |
| ---------- | ------------------------------ | --------------------------------- |
| Liveness   | ¿El container está vivo?       | Reinicia el container             |
| Readiness  | ¿Listo para recibir tráfico?   | Remueve del Service temporalmente |
| Startup    | ¿App terminó de iniciar?       | Espera antes de liveness          |

**Demo: App con health checks**

```typescript
// app.ts - Endpoints de health
app.get("/health", (req, res) => {
  res.json({ status: "healthy" });
});

app.get("/ready", async (req, res) => {
  try {
    await database.ping();
    await redis.ping();
    res.json({ status: "ready" });
  } catch (error) {
    res.status(503).json({ status: "not ready", error: error.message });
  }
});
```

**Deployment con probes**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 2
```

Aplicamos y simulamos un fallo:

```bash
kubectl apply -f app-deployment.yaml

# Simular que DB se cae (readiness falla)
kubectl exec -it <postgres-pod> -- pg_ctl stop

# Ver estado de los pods
kubectl get pods
```

Veremos:

```
NAME                     READY   STATUS    RESTARTS   AGE
myapp-7d8f5c9b7f-abc12   0/1     Running   0          2m
myapp-7d8f5c9b7f-def34   0/1     Running   0          2m
```

READY es 0/1 porque readiness probe falla (DB no accesible).

**Verificar que los pods fueron removidos del Service**:

```bash
kubectl describe service myapp-service
```

En Endpoints veremos que los pods no están listados.

**Restaurar DB**:

```bash
kubectl exec -it <postgres-pod> -- pg_ctl start

# Readiness probe pasa de nuevo
kubectl get pods
```

```
NAME                     READY   STATUS    RESTARTS   AGE
myapp-7d8f5c9b7f-abc12   1/1     Running   0          3m
myapp-7d8f5c9b7f-def34   1/1     Running   0          3m
```

---

## [21:00-27:00] Rolling Updates sin Downtime

**Demo: Actualizar de v1.0.0 a v2.0.0**

```yaml
# deployment-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:1.0.0
```

Desplegamos v1.0.0:

```bash
kubectl apply -f deployment-v1.yaml
kubectl get pods
```

```
NAME                    READY   STATUS    RESTARTS   AGE
myapp-7d8f5c9b7f-abc12  1/1     Running   0          30s
myapp-7d8f5c9b7f-def34  1/1     Running   0          30s
myapp-7d8f5c9b7f-ghi56  1/1     Running   0          30s
```

**Actualizar a v2.0.0**:

```bash
kubectl set image deployment/myapp app=myapp:2.0.0
kubectl rollout status deployment/myapp --watch
```

Veremos en tiempo real:

```
Waiting for deployment "myapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "myapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp" rollout to finish: 2 out of 3 new replicas have been updated...
deployment "myapp" successfully rolled out
```

**En otra terminal, monitorear pods**:

```bash
kubectl get pods --watch
```

Veremos:

```
myapp-7d8f5c9b7f-abc12   1/1   Running       0     1m
myapp-7d8f5c9b7f-def34   1/1   Running       0     1m
myapp-7d8f5c9b7f-ghi56   1/1   Running       0     1m
myapp-6f9d8c7b8f-jkl78   0/1   ContainerCreating  0  1s   ← Nuevo pod v2.0.0
myapp-6f9d8c7b8f-jkl78   1/1   Running       0     5s
myapp-7d8f5c9b7f-abc12   1/1   Terminating   0     1m    ← Pod v1.0.0 terminándose
myapp-6f9d8c7b8f-mno90   0/1   ContainerCreating  0  1s
... continúa hasta que todos son v2.0.0
```

**Zero downtime**: Siempre hay al menos 2 pods running.

**Test durante rolling update**: En otra terminal, hacer requests continuos:

```bash
while true; do
  curl -s http://myapp-service/version | jq -r '.version'
  sleep 0.5
done
```

Veremos gradualmente cambiar de v1.0.0 a v2.0.0 sin errores.

**Rollback**: Si v2.0.0 tiene un bug:

```bash
kubectl rollout undo deployment/myapp
kubectl rollout status deployment/myapp
```

Volverá a v1.0.0 de forma segura.

---

## [27:00-32:00] Demo Completo: Sistema E-Commerce

Vamos a desplegar un sistema completo:

```
LoadBalancer
    ↓
Backend API (3 réplicas)
    ↓
┌─────────────┬─────────────┐
│             │             │
PostgreSQL   Redis
```

**Estructura de archivos**:

```bash
ecommerce/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml
├── postgres-deployment.yaml
├── postgres-service.yaml
├── redis-deployment.yaml
├── redis-service.yaml
├── backend-deployment.yaml
└── backend-service.yaml
```

**Aplicamos todo**:

```bash
kubectl apply -f ecommerce/

# Verificar todo
kubectl get all -n ecommerce
```

Veremos:

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/postgres-...              1/1     Running   0          30s
pod/redis-...                 1/1     Running   0          30s
pod/backend-api-...-abc12     1/1     Running   0          20s
pod/backend-api-...-def34     1/1     Running   0          20s
pod/backend-api-...-ghi56     1/1     Running   0          20s

NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
service/postgres-service      ClusterIP      10.96.123.45    <none>          5432/TCP
service/redis-service         ClusterIP      10.96.234.56    <none>          6379/TCP
service/backend-api-service   LoadBalancer   10.96.345.67    35.123.45.67    80:31234/TCP

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/postgres     1/1     1            1           30s
deployment.apps/redis        1/1     1            1           30s
deployment.apps/backend-api  3/3     3            3           20s
```

**Inicializar base de datos**:

```bash
kubectl exec -n ecommerce -it <postgres-pod> -- psql -U postgres -d ecommerce_db

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  price DECIMAL(10, 2)
);

INSERT INTO products (name, price) VALUES
  ('Laptop', 999.99),
  ('Mouse', 29.99);
```

**Test API**:

```bash
EXTERNAL_IP=$(kubectl get service backend-api-service -n ecommerce -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Health check
curl http://$EXTERNAL_IP/health
# {"status":"healthy","database":"ok","redis":"ok"}

# Get products
curl http://$EXTERNAL_IP/api/products
# [{"id":1,"name":"Laptop","price":"999.99"},{"id":2,"name":"Mouse","price":"29.99"}]
```

**Verificar cache en Redis**:

```bash
kubectl exec -n ecommerce -it <redis-pod> -- redis-cli KEYS '*'
# 1) "products"
```

**Test self-healing**:

```bash
# Eliminar un pod backend
kubectl delete pod -n ecommerce <backend-api-pod>

# Inmediatamente hacer request
curl http://$EXTERNAL_IP/health
# {"status":"healthy"}  ← Sigue funcionando (load balancer usa otros pods)
```

---

## [32:00-35:00] Resumen y Próximos Pasos

**Conceptos cubiertos**:

✅ **Pods**: Unidad básica de ejecución  
✅ **Deployments**: Gestión de réplicas, rolling updates, self-healing  
✅ **Services**: Networking estable, load balancing (ClusterIP, LoadBalancer)  
✅ **ConfigMaps y Secrets**: Configuración dinámica  
✅ **Health Checks**: Liveness y readiness probes  
✅ **Zero-downtime updates**: Rolling updates con rollback

**Workflow de Kubernetes**:

```
1. Declaras estado deseado (YAML manifest)
2. kubectl apply -f manifest.yaml
3. Kubernetes reconcilia continuamente:
   - Crea pods faltantes (self-healing)
   - Balancea carga (Service)
   - Mantiene réplicas (Deployment)
   - Reinicia containers fallidos (liveness)
   - Remueve pods no listos del tráfico (readiness)
```

**Próximo tema**: Veremos conceptos avanzados:

- **Ingress**: HTTP routing y TLS termination
- **StatefulSets**: Para bases de datos (pods con identidad estable)
- **PersistentVolumes**: Almacenamiento persistente
- **HorizontalPodAutoscaler**: Escalado automático basado en métricas
- **Network Policies**: Seguridad de red

**Recursos**:

- Documentación oficial: kubernetes.io/docs
- Práctica: Minikube o Kind para cluster local
- Certificación: Certified Kubernetes Administrator (CKA)

**Gracias por ver este video. En el siguiente módulo exploraremos Serverless y Service Mesh.**
