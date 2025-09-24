# laboratorio4-fhir
# Laboratorio 4 — Operaciones extendidas en FHIR con API Gateway (Nginx)

Este laboratorio demuestra el uso de la operación **`$validate`** de FHIR en un servidor **HAPI FHIR** y la integración con un **API Gateway (Nginx)** para aplicar **rate limiting**.  
Se utilizó **Nginx** en lugar de Kong, ya que es más ligero y adecuado para entornos con recursos limitados (ej. VM con 6 GB RAM).

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

### 1. Crear recurso inválido (`observation-invalid.json`)
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
