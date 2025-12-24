# Evaluación: 12-Factor App y Containerización

## Instrucciones Generales

Deberás migrar una aplicación monolítica legacy a una arquitectura cloud-native siguiendo los principios de 12-Factor App y containerización con Docker.

**Duración estimada**: 6 horas

**Puntuación total**: 100 puntos

---

## Parte 1. Preguntas Teóricas (20 puntos)

### Pregunta 1 (5 puntos)

Explica por qué el Factor 6 (Processes - Stateless) es crítico para aplicaciones cloud-native. Proporciona un ejemplo específico de qué problema ocurriría en Kubernetes si una aplicación almacena sesiones en memoria.

**Criterios de evaluación**:

- Explicación clara del principio stateless (2 puntos)
- Ejemplo concreto con múltiples pods (2 puntos)
- Solución propuesta (Redis, etc.) (1 punto)

---

### Pregunta 2 (5 puntos)

Compara las siguientes dos estrategias de logging y explica cuál es 12-Factor compliant y por qué:

**Estrategia A**:

```typescript
fs.appendFileSync('/var/log/app.log', message);
```

**Estrategia B**:

```typescript
console.log(JSON.stringify({ timestamp: new Date(), level: 'info', message }));
```

**Criterios de evaluación**:

- Identificación correcta (Estrategia B) (1 punto)
- Explicación de problemas con Estrategia A (2 puntos)
- Beneficios de Estrategia B en Kubernetes (2 puntos)

---

### Pregunta 3 (5 puntos)

Un desarrollador dice: "No necesito Docker Compose en desarrollo. Puedo instalar PostgreSQL y Redis localmente en mi máquina". Explica por qué esta aproximación viola el Factor 10 (Dev/Prod Parity) y qué problemas puede causar.

**Criterios de evaluación**:

- Identificación de diferencias dev/prod (2 puntos)
- Problemas potenciales (versiones, configuración, etc.) (2 puntos)
- Beneficio de Docker Compose (1 punto)

---

### Pregunta 4 (5 puntos)

Describe el propósito de cada stage en un Dockerfile multi-stage y explica cómo esto mejora la seguridad y reduce el tamaño de la imagen final.

**Ejemplo**:

```dockerfile
FROM node:18 AS builder
# ... build steps ...

FROM node:18-alpine
# ... runtime only ...
```

**Criterios de evaluación**:

- Explicación de cada stage (2 puntos)
- Beneficios de seguridad (no incluir build tools) (2 puntos)
- Beneficio de tamaño reducido (1 punto)

---

## Parte 2: Análisis de Código (25 puntos)

Se te proporciona el siguiente código de una aplicación legacy:

```javascript
// server.js (Legacy App)
const express = require('express');
const session = require('express-session');
const fs = require('fs');
const app = express();

// Configuration
const config = {
    database: {
        host: 'localhost',
        port: 3306,
        user: 'root',
        password: 'admin123',  // ❌
        database: 'myapp'
    },
    port: 3000,
    uploadDir: './uploads'
};

// Session middleware (in-memory)
app.use(session({
    secret: 'my-secret-key',  // ❌
    resave: false,
    saveUninitialized: false
}));

// Database connection
const mysql = require('mysql');
const db = mysql.createConnection(config.database);

// Logging to file
function log(message) {
    const logMessage = `${new Date().toISOString()} - ${message}\n`;
    fs.appendFileSync('./app.log', logMessage);  // ❌
}

// Routes
app.post('/login', (req, res) => {
    const { username, password } = req.body;
    
    db.query(
        'SELECT * FROM users WHERE username = ? AND password = ?',
        [username, password],
        (err, results) => {
            if (err) {
                log(`Login error: ${err.message}`);
                return res.status(500).json({ error: 'Database error' });
            }
            
            if (results.length > 0) {
                req.session.userId = results[0].id;  // ❌ In-memory session
                log(`User ${username} logged in`);
                res.json({ success: true });
            } else {
                res.status(401).json({ error: 'Invalid credentials' });
            }
        }
    );
});

app.post('/upload', (req, res) => {
    const file = req.file;
    const filepath = `${config.uploadDir}/${file.originalname}`;  // ❌
    
    fs.writeFileSync(filepath, file.buffer);
    log(`File uploaded: ${filepath}`);
    
    res.json({ path: filepath });
});

app.get('/profile', (req, res) => {
    if (!req.session.userId) {  // ❌
        return res.status(401).json({ error: 'Not authenticated' });
    }
    
    db.query(
        'SELECT * FROM users WHERE id = ?',
        [req.session.userId],
        (err, results) => {
            if (err) {
                log(`Profile error: ${err.message}`);
                return res.status(500).json({ error: 'Database error' });
            }
            res.json(results[0]);
        }
    );
});

// Start server
app.listen(config.port, () => {
    log(`Server started on port ${config.port}`);
});

// No graceful shutdown handling
```

### Tarea 2.1 (10 puntos)

Identifica **todos** los factores de 12-Factor App que se están violando en el código anterior. Para cada violación:

- Indica qué factor se viola
- Explica por qué es una violación
- Describe qué problema causaría en producción (Kubernetes)

**Criterios de evaluación**:

- Factor 3 (Config): Identificado y explicado (2 puntos)
- Factor 6 (Processes): Sesiones y uploads identificados (2 puntos)
- Factor 9 (Disposability): No graceful shutdown (2 puntos)
- Factor 11 (Logs): Log a archivo identificado (2 puntos)
- Problemas en producción bien descritos (2 puntos)

---

### Tarea 2.2 (15 puntos)

Refactoriza el código anterior para que cumpla con **todos** los factores de 12-Factor App. Tu solución debe incluir:

1. **Config management** (Factor 3)
2. **Stateless processes** con Redis para sessions (Factor 6)
3. **S3 storage** para uploads (Factor 6)
4. **Structured logging** a stdout (Factor 11)
5. **Graceful shutdown** (Factor 9)
6. **Port binding** desde environment (Factor 7)

**Criterios de evaluación**:

- Config desde environment variables (3 puntos)
- Redis para sessions (3 puntos)
- S3 para file storage (3 puntos)
- Structured logging a stdout (2 puntos)
- Graceful shutdown implementado (2 puntos)
- Port binding configurable (2 puntos)

---

## Parte 3: Implementación Práctica (55 puntos)

### Contexto

Tu empresa tiene una aplicación de gestión de tareas (TODO app) que actualmente corre en un servidor físico. Debes migrarla a Kubernetes siguiendo principios cloud-native.

**Aplicación actual** (legacy):

- Backend: Python Flask
- Database: MySQL local
- File storage: Filesystem local
- Logs: Archivo `/var/log/todoapp.log`
- Config: Hardcoded en `config.py`

### Tarea 3.1. Dockerfile Multi-Stage (15 puntos)

Crea un Dockerfile multi-stage optimizado para la aplicación Python Flask.

**Requisitos**:

1. Stage 1 (builder):
   - Instalar dependencias
   - Copiar código
   - Build assets si es necesario

2. Stage 2 (runtime):
   - Imagen base mínima (python:3.11-slim)
   - Non-root user
   - Solo archivos necesarios para runtime
   - Health check

3. Optimizaciones:
   - Layer caching eficiente
   - Tamaño final < 200MB
   - Security best practices

**Entregable**: `Dockerfile`

**Criterios de evaluación**:

- Multi-stage build correctamente implementado (4 puntos)
- Non-root user configurado (2 puntos)
- Layer caching optimizado (dependencies antes de código) (3 puntos)
- Health check incluido (2 puntos)
- Security best practices (slim base, updates, etc.) (2 puntos)
- Tamaño final razonable (2 puntos)

---

### Tarea 3.2: Configuración Cloud-Native (15 puntos)

Refactoriza la aplicación para que sea 12-Factor compliant.

**app.py original**:

```python
from flask import Flask, request, jsonify, session
import mysql.connector
import os

app = Flask(__name__)
app.secret_key = 'hardcoded-secret'  # ❌

# Hardcoded config
db_config = {
    'host': 'localhost',
    'user': 'root',
    'password': 'password123',  # ❌
    'database': 'todoapp'
}

db = mysql.connector.connect(**db_config)

@app.route('/tasks', methods=['GET'])
def get_tasks():
    if 'user_id' not in session:  # ❌ In-memory session
        return jsonify({'error': 'Unauthorized'}), 401
    
    cursor = db.cursor()
    cursor.execute('SELECT * FROM tasks WHERE user_id = %s', (session['user_id'],))
    tasks = cursor.fetchall()
    
    # Log to file
    with open('/var/log/todoapp.log', 'a') as f:  # ❌
        f.write(f'{datetime.now()} - Tasks fetched for user {session["user_id"]}\n')
    
    return jsonify(tasks)

@app.route('/tasks', methods=['POST'])
def create_task():
    task = request.json
    cursor = db.cursor()
    cursor.execute(
        'INSERT INTO tasks (user_id, title, description) VALUES (%s, %s, %s)',
        (session['user_id'], task['title'], task['description'])
    )
    db.commit()
    
    with open('/var/log/todoapp.log', 'a') as f:  # ❌
        f.write(f'{datetime.now()} - Task created: {task["title"]}\n')
    
    return jsonify({'id': cursor.lastrowid}), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)  # No graceful shutdown
```

**Requisitos de refactorización**:

1. **Config** (Factor 3):
   - Todas las configuraciones desde environment variables
   - `DATABASE_URL`, `REDIS_URL`, `SECRET_KEY`, etc.
   - Validación de variables requeridas

2. **Stateless** (Factor 6):
   - Sessions en Redis
   - No estado en memoria

3. **Logging** (Factor 11):
   - Structured logging (JSON)
   - A stdout/stderr
   - Include context (user_id, task_id, etc.)

4. **Graceful Shutdown** (Factor 9):
   - Handler para SIGTERM
   - Cerrar conexiones DB/Redis
   - Completar requests en proceso

**Entregable**: `app.py` refactorizado

**Criterios de evaluación**:

- Config desde environment (4 puntos)
- Redis sessions implementado (4 puntos)
- Structured logging a stdout (3 puntos)
- Graceful shutdown handler (2 puntos)
- Código limpio y bien estructurado (2 puntos)

---

### Tarea 3.3: Docker Compose para Development (10 puntos)

Crea un `docker-compose.yml` que proporcione Dev/Prod parity (Factor 10).

**Requisitos**:

- Todo el stack en Docker (app, MySQL, Redis)
- Health checks para backing services
- Volumes para persistencia
- Hot reload para desarrollo (bind mount)
- Environment variables bien organizadas

**Entregable**: `docker-compose.yml`

**Criterios de evaluación**:

- Todos los servicios incluidos (app, DB, Redis) (3 puntos)
- Health checks configurados (2 puntos)
- Volumes para persistencia (2 puntos)
- Hot reload habilitado (1 punto)
- Environment variables bien organizadas (2 puntos)

---

### Tarea 3.4: Kubernetes Deployment (15 puntos)

Crea manifiestos de Kubernetes para deploy en producción.

**Requisitos**:

1. **ConfigMap**: Configuración no sensible
2. **Secret**: Credenciales (DATABASE_PASSWORD, etc.)
3. **Deployment**:
   - 3 replicas
   - Health probes (liveness, readiness)
   - Resource limits
   - Graceful termination
4. **Service**: Expose la app internamente
5. **HorizontalPodAutoscaler**: Autoscaling basado en CPU

**Entregables**:

- `configmap.yaml`
- `secret.yaml`
- `deployment.yaml`
- `service.yaml`
- `hpa.yaml`

**Criterios de evaluación**:

- ConfigMap y Secret correctamente separados (3 puntos)
- Deployment con health probes (4 puntos)
- Resource requests/limits definidos (2 puntos)
- Graceful termination configurado (2 puntos)
- Service correctamente configurado (2 puntos)
- HPA con métricas apropiadas (2 puntos)

---

## Parte 4: Bonus (10 puntos extra)

### Bonus 1. Observabilidad (5 puntos)

Implementa métricas Prometheus en la aplicación Flask. Incluye:

- Request count
- Request duration
- Error rate
- Custom business metrics (e.g., tasks_created_total)

### Bonus 2: CI/CD Pipeline (5 puntos)

Crea un GitHub Actions workflow que:

1. Run tests
2. Build Docker image
3. Push to registry
4. Deploy to Kubernetes (staging)

---

## Rúbrica de Evaluación

| Sección | Puntos | Descripción |
|---------|--------|-------------|
| **Parte 1. Teoría** | 20 | Comprensión de conceptos 12-Factor |
| Pregunta 1 | 5 | Factor 6 (Processes) |
| Pregunta 2 | 5 | Factor 11 (Logs) |
| Pregunta 3 | 5 | Factor 10 (Dev/Prod Parity) |
| Pregunta 4 | 5 | Multi-stage Dockerfile |
| **Parte 2: Análisis** | 25 | Identificación y refactorización |
| Tarea 2.1 | 10 | Identificar violaciones |
| Tarea 2.2 | 15 | Refactorizar código |
| **Parte 3: Práctica** | 55 | Implementación completa |
| Tarea 3.1 | 15 | Dockerfile optimizado |
| Tarea 3.2 | 15 | App cloud-native |
| Tarea 3.3 | 10 | Docker Compose |
| Tarea 3.4 | 15 | Kubernetes manifests |
| **Parte 4: Bonus** | 10 | Features avanzados |
| **TOTAL** | **110** | (100 base + 10 bonus) |

---

## Criterios de Aprobación

- **Excelente (90-110 puntos)**: Dominio completo de 12-Factor App y containerización
- **Bueno (75-89 puntos)**: Comprensión sólida con implementación funcional
- **Suficiente (60-74 puntos)**: Conceptos básicos aplicados correctamente
- **Insuficiente (<60 puntos)**: Revisión necesaria

---

## Solución de Referencia

### Tarea 3.2: app.py Refactorizado (Ejemplo)

```python
# app.py - 12-Factor Compliant
import os
import sys
import signal
import json
from datetime import datetime
from flask import Flask, request, jsonify
import mysql.connector
import redis
from werkzeug.security import check_password_hash
import logging

# ===== Factor 3: Config from environment =====
class Config:
    def __init__(self):
        self.port = int(os.getenv('PORT', '5000'))
        self.database_url = self.require_env('DATABASE_URL')
        self.redis_url = self.require_env('REDIS_URL')
        self.secret_key = self.require_env('SECRET_KEY')
        self.log_level = os.getenv('LOG_LEVEL', 'INFO')
    
    @staticmethod
    def require_env(name: str) -> str:
        value = os.getenv(name)
        if not value:
            raise EnvironmentError(f'Environment variable {name} is required')
        return value

config = Config()

# ===== Factor 11. Structured logging to stdout =====
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'message': record.getMessage(),
            'logger': record.name,
        }
        
        # Add context if available
        if hasattr(record, 'user_id'):
            log_data['user_id'] = record.user_id
        if hasattr(record, 'task_id'):
            log_data['task_id'] = record.task_id
        
        return json.dumps(log_data)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JSONFormatter())
logger = logging.getLogger(__name__)
logger.addHandler(handler)
logger.setLevel(config.log_level)

# ===== Flask App =====
app = Flask(__name__)
app.secret_key = config.secret_key

# ===== Factor 4: Backing services as attached resources =====
# Database connection
def get_db_connection():
    # Parse DATABASE_URL (format: mysql://user:pass@host:port/dbname)
    from urllib.parse import urlparse
    url = urlparse(config.database_url)
    
    return mysql.connector.connect(
        host=url.hostname,
        port=url.port or 3306,
        user=url.username,
        password=url.password,
        database=url.path[1:]  # Remove leading '/'
    )

db = get_db_connection()

# Redis connection (for sessions)
redis_client = redis.from_url(config.redis_url)

# ===== Factor 6: Stateless - Sessions in Redis =====
def get_session_data(session_id: str):
    data = redis_client.get(f'session:{session_id}')
    return json.loads(data) if data else None

def set_session_data(session_id: str, data: dict, ttl: int = 3600):
    redis_client.setex(f'session:{session_id}', ttl, json.dumps(data))

def delete_session(session_id: str):
    redis_client.delete(f'session:{session_id}')

# ===== Routes =====
@app.route('/health', methods=['GET'])
def health():
    """Health check endpoint"""
    try:
        # Check DB connection
        cursor = db.cursor()
        cursor.execute('SELECT 1')
        cursor.fetchone()
        
        # Check Redis connection
        redis_client.ping()
        
        return jsonify({'status': 'healthy'}), 200
    except Exception as e:
        logger.error('Health check failed', extra={'error': str(e)})
        return jsonify({'status': 'unhealthy', 'error': str(e)}), 503

@app.route('/login', methods=['POST'])
def login():
    """User login"""
    data = request.json
    username = data.get('username')
    password = data.get('password')
    
    cursor = db.cursor(dictionary=True)
    cursor.execute('SELECT * FROM users WHERE username = %s', (username,))
    user = cursor.fetchone()
    
    if not user or not check_password_hash(user['password'], password):
        logger.warning('Login failed', extra={'username': username})
        return jsonify({'error': 'Invalid credentials'}), 401
    
    # Create session in Redis
    import secrets
    session_id = secrets.token_hex(32)
    set_session_data(session_id, {'user_id': user['id']})
    
    logger.info('User logged in', extra={'user_id': user['id']})
    
    return jsonify({'session_id': session_id}), 200

@app.route('/tasks', methods=['GET'])
def get_tasks():
    """Get user tasks"""
    session_id = request.headers.get('Authorization')
    if not session_id:
        return jsonify({'error': 'Unauthorized'}), 401
    
    session_data = get_session_data(session_id)
    if not session_data:
        return jsonify({'error': 'Invalid session'}), 401
    
    user_id = session_data['user_id']
    
    cursor = db.cursor(dictionary=True)
    cursor.execute('SELECT * FROM tasks WHERE user_id = %s', (user_id,))
    tasks = cursor.fetchall()
    
    logger.info('Tasks fetched', extra={'user_id': user_id, 'count': len(tasks)})
    
    return jsonify(tasks), 200

@app.route('/tasks', methods=['POST'])
def create_task():
    """Create new task"""
    session_id = request.headers.get('Authorization')
    if not session_id:
        return jsonify({'error': 'Unauthorized'}), 401
    
    session_data = get_session_data(session_id)
    if not session_data:
        return jsonify({'error': 'Invalid session'}), 401
    
    user_id = session_data['user_id']
    task = request.json
    
    cursor = db.cursor()
    cursor.execute(
        'INSERT INTO tasks (user_id, title, description) VALUES (%s, %s, %s)',
        (user_id, task['title'], task.get('description', ''))
    )
    db.commit()
    task_id = cursor.lastrowid
    
    logger.info('Task created', extra={
        'user_id': user_id,
        'task_id': task_id,
        'title': task['title']
    })
    
    return jsonify({'id': task_id}), 201

# ===== Factor 9: Graceful Shutdown =====
def shutdown(signum, frame):
    """Handle shutdown signals"""
    logger.info(f'Received signal {signum}, shutting down gracefully...')
    
    # Close database connection
    if db.is_connected():
        db.close()
        logger.info('Database connection closed')
    
    # Close Redis connection
    redis_client.close()
    logger.info('Redis connection closed')
    
    logger.info('Shutdown complete')
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown)
signal.signal(signal.SIGINT, shutdown)

# ===== Factor 7: Port Binding =====
if __name__ == '__main__':
    logger.info(f'Starting TODO app on port {config.port}')
    app.run(host='0.0.0.0', port=config.port)
```

### Tarea 3.4: deployment.yaml (Ejemplo)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todoapp
  labels:
    app: todoapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: todoapp
  template:
    metadata:
      labels:
        app: todoapp
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      
      containers:
      - name: todoapp
        image: myregistry/todoapp:1.0.0
        ports:
        - containerPort: 5000
          name: http
        
        env:
        - name: PORT
          value: "5000"
        
        envFrom:
        - configMapRef:
            name: todoapp-config
        - secretRef:
            name: todoapp-secrets
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      
      terminationGracePeriodSeconds: 30
```

**¡Éxito!** Esta evaluación valida tu comprensión completa de 12-Factor App y containerización.
