# 🛠️ Bitácora de Diagnóstico y Resolución de Fallas (Troubleshooting Log)

**Proyecto:** Despliegue Cloud, Seguridad y Troubleshooting  
**Curso:** Orquestadores y Automatización de Software  
**Sesión:** S12 | AP9  
**Namespace:** `practica-s12`  

---

## 📋 Resumen Ejecutivo de Fallas Detectadas

| Escenario | Categoría | Síntoma Principal | Causa Raíz | Estado |
| :--- | :--- | :--- | :--- | :--- |
| **Escenario A** | Configuración / Variables | Pods en `CrashLoopBackOff` / `CreateContainerConfigError` | Referencia a clave inexistente en el Secret (`secretKeyRef`) e inyección de variables nulas requeridas para el arranque | ✅ **Resuelto** |
| **Escenario B** | Red y Conectividad Interna | Backend no conecta a base de datos (`Connection refused` / `Timeout`) | Desajuste en el `targetPort` del Service `mongo-service` y selector de etiquetas no coincidente con el Pod de base de datos | ✅ **Resuelto** |
| **Escenario C** | Sondas de Salud (Probes) | Reinicios constantes del contenedor cada 30-40 segundos (`RestartCount > 0`) | Sondas `livenessProbe` y `readinessProbe` apuntando a ruta inexistente (`/invalid-health-probe`) generando HTTP 404 continuo | ✅ **Resuelto** |

---

## 🔍 Escenario A: Falla de Configuración (CrashLoopBackOff)

### 1. Síntoma Detectado
Al ejecutar el comando de estado en el clúster:
```bash
kubectl get pods -n practica-s12
```
Se observó que los Pods de la aplicación se quedaban en estado:
```text
NAME                              READY   STATUS                       RESTARTS   AGE
app-deployment-647d8f9979-abc12   0/1     CreateContainerConfigError   0          45s
app-deployment-647d8f9979-xyz34   0/1     CrashLoopBackOff             2          1m
```

### 2. Comandos de Diagnóstico Utilizados
```bash
kubectl describe pod app-deployment-647d8f9979-abc12 -n practica-s12
kubectl logs app-deployment-647d8f9979-xyz34 -n practica-s12
```

### 3. Evidencia del Log y Eventos del Clúster
En la sección `Events` de `kubectl describe pod`:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Warning  Failed     14s (x4 over 50s)  kubelet            Error: secret "db-secret" does not contain key "MONGO_INITDB_ROOT_PASS"
```

### 4. Causa Raíz
El archivo de despliegue `03-app-deployment.yaml` intentaba inyectar una variable sensible mediante:
```yaml
# ❌ CONFIGURACIÓN ERRÓNEA
env:
  - name: MONGO_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: MONGO_INITDB_ROOT_PASS   # <-- Clave inexistente, el secret define MONGO_INITDB_ROOT_PASSWORD
```
Al no existir la clave solicitada en el objeto `Secret`, el Kubelet no puede preparar el contenedor, disparando `CreateContainerConfigError` y posterior ciclo `CrashLoopBackOff`.

### 5. Corrección Aplicada
Se procedió a:
1. Estandarizar la inyección en bloque mediante la directiva `envFrom` tanto para `security-configmap` como para `db-secret`.
2. Asegurar que en `02-db-secret.yaml` las claves se encuentren correctamente nombradas (`MONGO_INITDB_ROOT_PASSWORD` y `DB_PASS`).
```yaml
# ✅ CONFIGURACIÓN CORREGIDA
envFrom:
  - configMapRef:
      name: security-configmap
  - secretRef:
      name: db-secret
```

---

## 🔍 Escenario B: Falla de Red y Conectividad Interna

### 1. Síntoma Detectado
El Pod de la aplicación arrancaba, pero al procesar transacciones o verificar el estado de conexión hacia la base de datos se generaba un error interno:
```text
MongooseServerSelectionError: connect ECONNREFUSED mongo-service:27017
    at Timeout._onTimeout (/app/server.js:154:19)
```

### 2. Comandos de Diagnóstico Utilizados
```bash
# Inspección de endpoints asociados al servicio de base de datos
kubectl get endpoints mongo-service -n practica-s12
kubectl describe svc mongo-service -n practica-s12
```

### 3. Evidencia del Diagnóstico
Salida del comando `kubectl describe svc`:
```text
Name:              mongo-service
Namespace:         practica-s12
Selector:          app=database-service   <-- No coincide con las etiquetas del Pod (app=mongodb)
Port:              mongo-port  27017/TCP
TargetPort:        27018/TCP              <-- El contenedor MongoDB escucha en 27017, no en 27018
Endpoints:         <none>                 <-- Al no coincidir el selector, ningún Pod recibe tráfico
```

### 4. Causa Raíz
Existían dos desajustes de red:
1. El `selector` del servicio buscaba `app: database-service`, mientras que el Pod de MongoDB tenía la etiqueta `app: mongodb`. Por ello, `Endpoints` figuraba como `<none>`.
2. El `targetPort` estaba fijado en `27018` en lugar del puerto nativo de MongoDB (`27017`).

### 5. Corrección Aplicada en `03-app-deployment.yaml` (o Service)
```yaml
# ✅ CONFIGURACIÓN CORREGIDA
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
  namespace: practica-s12
spec:
  type: ClusterIP
  selector:
    app: mongodb             # <-- Corregido para emparejar con el Pod de Mongo
  ports:
    - name: mongo-port
      protocol: TCP
      port: 27017
      targetPort: 27017      # <-- Corregido al puerto real del motor de BD
```
Al aplicar el cambio, `kubectl get endpoints mongo-service -n practica-s12` mostró inmediatamente la IP interna del Pod de MongoDB asociada.

---

## 🔍 Escenario C: Falla de Sondas de Salud (Liveness/Readiness Probes)

### 1. Síntoma Detectado
El Pod se desplegaba, pero cada 30 segundos entraba en estado `NotReady` y Kubernetes lo terminaba forzosamente, incrementando el contador de reinicios (`RESTARTS: 5`):
```text
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7bb48c66ff-k9876   0/1     Running   6 (22s)    3m40s
```

### 2. Comandos de Diagnóstico Utilizados
```bash
kubectl describe pod -l app=secure-app -n practica-s12
```

### 3. Evidencia del Diagnóstico
En la sección `Events`:
```text
Warning  Unhealthy  12s (x3 over 32s)  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 404
Warning  Unhealthy  7s (x4 over 27s)   kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
Normal   Killing    7s                 kubelet  Container secure-app failed liveness probe, will be restarted
```

### 4. Causa Raíz
La configuración de los probes apuntaba a una ruta que el servidor web no tenía implementada:
```yaml
# ❌ CONFIGURACIÓN ERRÓNEA
livenessProbe:
  httpGet:
    path: /invalid-health-probe    # <-- Retorna HTTP 404 Not Found
    port: 3000
```
Como el Kubelet recibía un código `404` (diferente al rango aceptable 200-399), consideraba que el contenedor estaba muerto y ejecutaba la política de reinicio de forma ininterrumpida.

### 5. Corrección Aplicada
Se corrigió la ruta de las sondas hacia la ruta raíz `/` (o `/actuator/health` si se utiliza Spring Boot), y se ajustaron los tiempos de tolerancia (`initialDelaySeconds: 10` y `periodSeconds: 10`):
```yaml
# ✅ CONFIGURACIÓN CORREGIDA
livenessProbe:
  httpGet:
    path: /
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
```

---

## 🛡️ Hardening de Seguridad Aplicado (SecurityContext)

Para cumplir con las directivas de seguridad corporativa en AWS EKS y Kubernetes Cloud:
1. **Prevención de Root:** Se configuró `runAsNonRoot: true` tanto a nivel de Pod como a nivel de Contenedor para mitigar vulnerabilidades de escape de contenedor.
2. **UID Específico:** Se fijó `runAsUser: 10001` y `runAsGroup: 10001`.
3. **Escalado de Privilegios:** Se deshabilitó explícitamente mediante `allowPrivilegeEscalation: false`.
4. **Sistema de Archivos de Solo Lectura:** Se activó `readOnlyRootFilesystem: true`, montando volúmenes temporales volátiles tipo `emptyDir` en `/tmp` y directorios de sockets/locks para evitar escrituras no autorizadas en el sistema base sin afectar la operatividad del software.
5. **Drop de Capacidades del Kernel:** Se eliminaron todas las capacidades (`capabilities.drop: ["ALL"]`).

