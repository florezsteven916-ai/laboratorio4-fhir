# laboratorio4-fhir
# Laboratorio 4 — Operaciones extendidas en FHIR con API Gateway (Nginx)

Este laboratorio demuestra el uso de la operación **`$validate`** de FHIR en un servidor **HAPI FHIR** y la integración con un **API Gateway (Nginx)** para aplicar **rate limiting**.  

---

## 🔧 Herramientas utilizadas
- **HAPI FHIR Server** (ya levantado en `http://127.0.0.1:8081/fhir/`)  
- **Nginx** (como API Gateway con rate limiting)  
- **curl** (para ejecutar peticiones HTTP)  
- **Linux Mint** (máquina virtual)  

---

## 📂 Archivos creados
- `observation-invalid.json` → Recurso Observation inválido.  
- `/etc/nginx/nginx.conf` → Configuración global de Nginx (incluye `limit_req_zone`).  
- `/etc/nginx/sites-available/fhir.conf` → Configuración del sitio para proxy + rate limiting.  

---

## 📝 Pasos realizados

### 🔹 Paso 1: Crear recurso inválido
```json
{
  "resourceType": "Observation",
  "status": "final",
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "8480-6",
        "display": "Systolic BP"
      }
    ]
  },
  "valueQuantity": {
    "value": "ABC"
  }
}
```
El campo `"ABC"` es inválido porque debería ser numérico.

---

### 🔹 Paso 2: Probar `$validate` directamente en HAPI
```bash
curl -i -X POST "http://127.0.0.1:8081/fhir/Observation/$validate"      -H "Content-Type: application/fhir+json"      -d @observation-invalid.json
```

📌 **Resultado esperado:**  
`HTTP 400` con un recurso **OperationOutcome** indicando que `"ABC"` no es válido.

---

### 🔹 Paso 3: Configurar Nginx como API Gateway
```nginx
# En nginx.conf (dentro de http { ... })
limit_req_zone $binary_remote_addr zone=fhir:10m rate=3r/m;
```

```nginx
# En /etc/nginx/sites-available/fhir.conf
server {
    listen 8090;

    location /fhir/ {
        proxy_pass http://127.0.0.1:8081/fhir/;
        proxy_set_header Host $host;

        limit_req zone=fhir burst=1 nodelay;
    }
}
```

```bash
# Activar sitio y recargar Nginx
sudo ln -s /etc/nginx/sites-available/fhir.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

### 🔹 Paso 4: Probar `$validate` a través de Nginx
```bash
curl -i -X POST "http://127.0.0.1:8090/fhir/Observation/$validate"      -H "Content-Type: application/fhir+json"      -d @observation-invalid.json
```

📌 **Resultado esperado:**  
- Las primeras solicitudes → `HTTP 400` con **OperationOutcome** de error.  

---

### 🔹 Paso 5: Verificar **rate limiting**
```bash
for i in {1..6}; do
  echo "Intento $i:"
  curl -s -o /dev/null -w "HTTP %{http_code}\n"     -X POST "http://127.0.0.1:8090/fhir/Observation/$validate"     -H "Content-Type: application/fhir+json"     -d @observation-invalid.json
done
```

📌 **Resultado esperado:**  
- Intento 1 → `HTTP 400`  
- Intento 2 → `HTTP 400`  
- Intento 3 → `HTTP 400`  
- Intento 4 → `HTTP 503`  
- Intento 5 → `HTTP 503`  
- Intento 6 → `HTTP 503`  

---

## 📊 Resultados observados
- HAPI rechazó el recurso inválido con OperationOutcome.  
- Nginx limitó el acceso a **3 solicitudes por minuto**, devolviendo `503` en las siguientes.  
- Los logs confirmaron tanto la validación en HAPI como el rate limiting en Nginx.  

---

## 🎯 Conclusiones
- La operación **`$validate`** garantiza la **integridad de los datos** en FHIR antes de que sean aceptados por el servidor.  
- El uso de un **API Gateway (Nginx en este caso)** permite aplicar políticas como **rate limiting**, añadiendo una capa de **seguridad y control**.  
- Combinando ambas capas (aplicación + infraestructura) se obtiene un sistema más **robusto y confiable** para entornos de salud basados en FHIR.  

---

## 📌 Evidencias

### 🔹 Ejemplo de salida `$validate` (HAPI – recurso inválido)
```http
HTTP/1.1 400 
Server: nginx/1.18.0 (Ubuntu)
Content-Type: application/fhir+json;charset=UTF-8
X-Powered-By: HAPI FHIR 6.8.0 REST Server (FHIR Server; FHIR 4.0.1/R4)

{
  "resourceType": "OperationOutcome",
  "issue": [ {
    "severity": "error",
    "code": "processing",
    "diagnostics": "HAPI-0450: Failed to parse request body as JSON resource. Error was: HAPI-1821: [element=\"value\"] Invalid attribute value \"ABC\": Character A is neither a decimal digit number, decimal point, nor \"e\" notation exponential mark."
  } ]
}
```

### 🔹 Ejemplo de rate limiting (Nginx)
```text
Intento 1:
HTTP 400
Intento 2:
HTTP 400
Intento 3:
HTTP 400
Intento 4:
HTTP 503
Intento 5:
HTTP 503
Intento 6:
HTTP 503
```
