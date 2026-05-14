# Dental Clinic Directus Stack

## Overview
Este skill permite consultar y gestionar datos de clínicas dentales odontológicas en Directus. Sistema multi-tenant con gestión de pacientes, citas, tratamientos y dentistas.

## Directus Configuration
- **URL Base**: `https://directus.drawood.co`
- **Authentication**: Bearer Token (usar variable `DENTAL_DIRECTUS_TOKEN`)
- **Database**: PostgreSQL (directus_odontia)
- **Multi-tenant**: Todos los queries DEBEN incluir filtro por `clinic_id`

## Collections Schema

### clinics
Clínicas dentales registradas en el sistema.

**Campos principales:**
- `id` (integer): ID único de la clínica
- `name` (string): Nombre de la clínica
- `address` (text): Dirección física
- `phone` (string): Teléfono de contacto
- `email` (string): Email de contacto
- `plan_type` (string): basic, professional, enterprise
- `subscription_status` (string): trial, active, cancelled, suspended

### patients
Pacientes de las clínicas dentales.

**Campos principales:**
- `id` (integer): ID único del paciente
- `clinic_id` (integer): **FILTRO OBLIGATORIO** - ID de la clínica
- `first_name` (string): Nombre del paciente
- `last_name` (string): Apellido del paciente
- `email` (string): Email del paciente
- `phone` (string): Teléfono (importante para WhatsApp)
- `birth_date` (date): Fecha de nacimiento
- `document_type` (string): CC, CE, PA, TI
- `document_number` (string): Número de documento
- `address` (text): Dirección del paciente
- `medical_history` (json): Historial médico
- `notes` (text): Notas adicionales

**Relaciones:**
- `clinic_id` → clinics.id
- Referenciado por: appointments, treatments

### dentists
Profesionales odontólogos de las clínicas.

**Campos principales:**
- `id` (integer): ID único del dentista
- `directus_users_id` (uuid): **IMPORTANTE** - Relación con directus_users (NO user_id)
- `clinic_id` (integer): **FILTRO OBLIGATORIO** - ID de la clínica
- `specialty` (string): Especialidad (orthodontics, oral_surgery, endodontics, etc.)
- `license_number` (string): Número de licencia profesional
- `active` (boolean): Si está activo o no

**CRÍTICO:** Para obtener nombre del dentista:
fields: dentist_id.directus_users_id.first_name,dentist_id.directus_users_id.last_name

**Relaciones:**
- `directus_users_id` → directus_users.id
- `clinic_id` → clinics.id
- Referenciado por: appointments, treatments

### appointments
Citas programadas en las clínicas.

**Campos principales:**
- `id` (integer): ID único de la cita
- `clinic_id` (integer): **FILTRO OBLIGATORIO** - ID de la clínica
- `patient_id` (integer): ID del paciente
- `dentist_id` (integer): ID del dentista
- `date_time` (timestamp): Fecha y hora de la cita
- `duration_minutes` (integer): Duración en minutos (default: 30)
- `status` (string): pending, scheduled, confirmed, cancelled, completed
- `treatment_type` (string): Tipo de tratamiento programado
- `notes` (text): Notas sobre la cita
- `reminder_sent` (boolean): Si se envió recordatorio (para WhatsApp)

**Estados válidos:**
- `pending`: Recién creada, pendiente de confirmación
- `scheduled`: Programada
- `confirmed`: Confirmada por el paciente (importante para WhatsApp)
- `cancelled`: Cancelada
- `completed`: Completada

**Relaciones:**
- `clinic_id` → clinics.id
- `patient_id` → patients.id
- `dentist_id` → dentists.id

### treatments
Tratamientos realizados o en proceso.

**Campos principales:**
- `id` (integer): ID único del tratamiento
- `clinic_id` (integer): **FILTRO OBLIGATORIO** - ID de la clínica
- `appointment_id` (integer): ID de la cita relacionada (nullable)
- `patient_id` (integer): ID del paciente
- `dentist_id` (integer): ID del dentista
- `treatment_type` (string): Tipo de tratamiento (Ortodoncia, Limpieza dental, etc.)
- `description` (text): Descripción detallada del tratamiento
- `tooth_number` (string): Número de diente (sistema FDI: 11-48 adultos, 51-85 niños)
- `cost` (decimal): Costo en COP (pesos colombianos)
- `status` (string): pending, in_progress, completed, cancelled
- `images` (json): Imágenes del tratamiento

**Tipos de tratamiento comunes:**
- Consulta general
- Limpieza dental
- Ortodoncia
- Endodoncia
- Extracción
- Implante
- Corona
- Puente
- Blanqueamiento
- Cirugía
- Periodoncia
- Prótesis

**Relaciones:**
- `clinic_id` → clinics.id
- `appointment_id` → appointments.id
- `patient_id` → patients.id
- `dentist_id` → dentists.id

### ai_chats
Historial de conversaciones con el asistente IA.

**Campos principales:**
- `id` (integer): ID único del chat
- `clinic_id` (integer): **FILTRO OBLIGATORIO** - ID de la clínica
- `user_id` (uuid): ID del usuario que chatea
- `team` (string): Nombre del team de AgentCrew (default: 'dental-clinic')
- `messages` (json): Array de mensajes [{role, content, timestamp}]

## Python Code Examples

### Setup y Configuración

```python
import requests
import os
from datetime import datetime, timedelta
from typing import Dict, List, Optional

DIRECTUS_URL = "https://directus.drawood.co"
TOKEN = os.getenv("DENTAL_DIRECTUS_TOKEN")

if not TOKEN:
    raise ValueError("DENTAL_DIRECTUS_TOKEN environment variable is required")

HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

def directus_request(endpoint: str, method: str = "GET", params: Dict = None, data: Dict = None):
    """
    Realiza una petición a Directus
    """
    url = f"{DIRECTUS_URL}{endpoint}"
    
    if method == "GET":
        response = requests.get(url, params=params, headers=HEADERS)
    elif method == "POST":
        response = requests.post(url, json=data, headers=HEADERS)
    elif method == "PATCH":
        response = requests.patch(url, json=data, headers=HEADERS)
    elif method == "DELETE":
        response = requests.delete(url, headers=HEADERS)
    
    response.raise_for_status()
    return response.json()
```

### 1. Obtener Citas del Día

```python
def get_appointments_today(clinic_id: int) -> List[Dict]:
    """
    Obtiene todas las citas programadas para hoy de una clínica específica.
    
    Args:
        clinic_id: ID de la clínica
        
    Returns:
        Lista de citas con información del paciente y dentista
    """
    today = datetime.now().strftime("%Y-%m-%d")
    tomorrow = (datetime.now() + timedelta(days=1)).strftime("%Y-%m-%d")
    
    params = {
        "filter[clinic_id][_eq]": clinic_id,
        "filter[date_time][_gte]": today,
        "filter[date_time][_lt]": tomorrow,
        "fields": ",".join([
            "*",
            "patient_id.id",
            "patient_id.first_name",
            "patient_id.last_name",
            "patient_id.phone",
            "patient_id.email",
            "dentist_id.id",
            "dentist_id.directus_users_id.first_name",
            "dentist_id.directus_users_id.last_name",
            "dentist_id.specialty"
        ]),
        "sort": "date_time"
    }
    
    result = directus_request("/items/appointments", params=params)
    return result.get("data", [])
```

### 2. Obtener Citas por Fecha

```python
def get_appointments_by_date(clinic_id: int, date: str) -> List[Dict]:
    """
    Obtiene citas de una fecha específica.
    
    Args:
        clinic_id: ID de la clínica
        date: Fecha en formato YYYY-MM-DD
        
    Returns:
        Lista de citas
    """
    next_day = (datetime.strptime(date, "%Y-%m-%d") + timedelta(days=1)).strftime("%Y-%m-%d")
    
    params = {
        "filter[clinic_id][_eq]": clinic_id,
        "filter[date_time][_gte]": date,
        "filter[date_time][_lt]": next_day,
        "fields": ",".join([
            "*",
            "patient_id.first_name",
            "patient_id.last_name",
            "patient_id.phone",
            "dentist_id.directus_users_id.first_name",
            "dentist_id.directus_users_id.last_name"
        ]),
        "sort": "date_time"
    }
    
    result = directus_request("/items/appointments", params=params)
    return result.get("data", [])
```

### 3. Obtener Citas Pendientes de Confirmación (para WhatsApp)

```python
def get_unconfirmed_appointments(clinic_id: int, hours_ahead: int = 24) -> List[Dict]:
    """
    Obtiene citas que necesitan confirmación vía WhatsApp.
    
    Args:
        clinic_id: ID de la clínica
        hours_ahead: Horas de anticipación (default 24h)
        
    Returns:
        Lista de citas pendientes de confirmación
    """
    now = datetime.now()
    future = now + timedelta(hours=hours_ahead)
    
    params = {
        "filter[clinic_id][_eq]": clinic_id,
        "filter[date_time][_gte]": now.isoformat(),
        "filter[date_time][_lte]": future.isoformat(),
        "filter[status][_in]": "pending,scheduled",
        "filter[reminder_sent][_eq]": "false",
        "fields": ",".join([
            "*",
            "patient_id.first_name",
            "patient_id.last_name",
            "patient_id.phone",
            "dentist_id.directus_users_id.first_name",
            "dentist_id.directus_users_id.last_name"
        ]),
        "sort": "date_time"
    }
    
    result = directus_request("/items/appointments", params=params)
    return result.get("data", [])
```

### 4. Actualizar Estado de Cita

```python
def update_appointment_status(appointment_id: int, new_status: str, send_reminder: bool = False) -> Dict:
    """
    Actualiza el estado de una cita.
    
    Args:
        appointment_id: ID de la cita
        new_status: Nuevo estado (confirmed, cancelled, completed, etc.)
        send_reminder: Marcar como recordatorio enviado
        
    Returns:
        Datos de la cita actualizada
    """
    valid_statuses = ["pending", "scheduled", "confirmed", "cancelled", "completed"]
    if new_status not in valid_statuses:
        raise ValueError(f"Estado inválido. Usar: {', '.join(valid_statuses)}")
    
    data = {"status": new_status}
    if send_reminder:
        data["reminder_sent"] = True
    
    result = directus_request(f"/items/appointments/{appointment_id}", method="PATCH", data=data)
    return result.get("data", {})
```

### 5. Buscar Paciente

```python
def search_patient(clinic_id: int, search_term: str) -> List[Dict]:
    """
    Busca pacientes por nombre, apellido o documento.
    
    Args:
        clinic_id: ID de la clínica
        search_term: Término de búsqueda
        
    Returns:
        Lista de pacientes que coinciden
    """
    params = {
        "filter[clinic_id][_eq]": clinic_id,
        "filter[_or][0][first_name][_contains]": search_term,
        "filter[_or][1][last_name][_contains]": search_term,
        "filter[_or][2][document_number][_contains]": search_term,
        "limit": 10
    }
    
    result = directus_request("/items/patients", params=params)
    return result.get("data", [])
```

### 6. Obtener Información Completa del Paciente

```python
def get_patient_info(clinic_id: int, patient_id: int) -> Dict:
    """
    Obtiene información detallada de un paciente incluyendo historial.
    
    Args:
        clinic_id: ID de la clínica
        patient_id: ID del paciente
        
    Returns:
        Datos completos del paciente
    """
    params = {
        "filter[clinic_id][_eq]": clinic_id,
        "filter[id][_eq]": patient_id
    }
    
    result = directus_request("/items/patients", params=params)
    patients = result.get("data", [])
    return patients[0] if patients else None
```

### 7. Obtener Tratamientos Pendientes

```python
def get_pending_treatments(clinic_id: int) -> List[Dict]:
    """
    Obtiene todos los tratamientos pendientes o en progreso.
    
    Args:
        clinic_id: ID de la clínica
        
    Returns:
        Lista de tratamientos
    """
    params = {
        "filter[clinic_id][_eq]": clinic_id,
        "filter[status][_in]": "pending,in_progress",
        "fields": ",".join([
            "*",
            "patient_id.first_name",
            "patient_id.last_name",
            "dentist_id.directus_users_id.first_name",
            "dentist_id.directus_users_id.last_name"
        ]),
        "sort": "-created_at"
    }
    
    result = directus_request("/items/treatments", params=params)
    return result.get("data", [])
```

### 8. Calcular Ingresos del Mes

```python
def get_monthly_revenue(clinic_id: int, year: int = None, month: int = None) -> Dict:
    """
    Calcula los ingresos del mes de tratamientos completados.
    
    Args:
        clinic_id: ID de la clínica
        year: Año (default: año actual)
        month: Mes (default: mes actual)
        
    Returns:
        Dict con total de ingresos y detalles
    """
    now = datetime.now()
    year = year or now.year
    month = month or now.month
    
    first_day = datetime(year, month, 1).strftime("%Y-%m-%d")
    if month == 12:
        last_day = datetime(year + 1, 1, 1).strftime("%Y-%m-%d")
    else:
        last_day = datetime(year, month + 1, 1).strftime("%Y-%m-%d")
    
    params = {
        "filter[clinic_id][_eq]": clinic_id,
        "filter[status][_eq]": "completed",
        "filter[created_at][_gte]": first_day,
        "filter[created_at][_lt]": last_day,
        "aggregate[sum]": "cost",
        "aggregate[count]": "id"
    }
    
    result = directus_request("/items/treatments", params=params)
    
    return {
        "total_revenue": result.get("data", [{}])[0].get("sum", {}).get("cost", 0),
        "total_treatments": result.get("data", [{}])[0].get("count", {}).get("id", 0),
        "period": f"{year}-{month:02d}"
    }
```

### 9. Crear Nueva Cita

```python
def create_appointment(clinic_id: int, patient_id: int, dentist_id: int, 
                      date_time: str, treatment_type: str = None, 
                      notes: str = None) -> Dict:
    """
    Crea una nueva cita.
    
    Args:
        clinic_id: ID de la clínica
        patient_id: ID del paciente
        dentist_id: ID del dentista
        date_time: Fecha/hora en formato ISO (YYYY-MM-DDTHH:MM:SS)
        treatment_type: Tipo de tratamiento
        notes: Notas adicionales
        
    Returns:
        Datos de la cita creada
    """
    data = {
        "clinic_id": clinic_id,
        "patient_id": patient_id,
        "dentist_id": dentist_id,
        "date_time": date_time,
        "status": "scheduled",
        "reminder_sent": False
    }
    
    if treatment_type:
        data["treatment_type"] = treatment_type
    if notes:
        data["notes"] = notes
    
    result = directus_request("/items/appointments", method="POST", data=data)
    return result.get("data", {})
```

### 10. Generar Mensaje de WhatsApp para Confirmación

```python
def generate_whatsapp_confirmation_message(appointment: Dict) -> str:
    """
    Genera un mensaje de WhatsApp para confirmar cita.
    
    Args:
        appointment: Datos de la cita
        
    Returns:
        Mensaje formateado para WhatsApp
    """
    patient_name = f"{appointment['patient_id']['first_name']} {appointment['patient_id']['last_name']}"
    dentist_name = f"Dr(a). {appointment['dentist_id']['directus_users_id']['first_name']} {appointment['dentist_id']['directus_users_id']['last_name']}"
    
    date_obj = datetime.fromisoformat(appointment['date_time'].replace('Z', '+00:00'))
    date_str = date_obj.strftime("%d de %B, %Y")
    time_str = date_obj.strftime("%I:%M %p")
    
    treatment = appointment.get('treatment_type', 'Consulta')
    
    message = f"""Hola {patient_name} 👋

Te recordamos tu cita:

📅 {date_str}
🕐 {time_str}
🦷 {treatment}
👨‍⚕️ {dentist_name}

Por favor responde:
✅ CONFIRMAR
❌ CANCELAR
📅 REPROGRAMAR

¡Gracias!"""
    
    return message
```

## Usage Instructions

### Cuando el usuario pregunta sobre:

**Citas:**
- "¿Cuántas citas tengo hoy?" → `get_appointments_today(clinic_id)`
- "Citas del 15 de mayo" → `get_appointments_by_date(clinic_id, "2026-05-15")`
- "Citas pendientes de confirmar" → `get_unconfirmed_appointments(clinic_id)`

**Pacientes:**
- "Buscar paciente Juan" → `search_patient(clinic_id, "Juan")`
- "Información de paciente ID 5" → `get_patient_info(clinic_id, 5)`

**Tratamientos:**
- "Tratamientos pendientes" → `get_pending_treatments(clinic_id)`
- "Ingresos del mes" → `get_monthly_revenue(clinic_id)`

**Confirmaciones (WhatsApp):**
- "Confirmar cita 3" → `update_appointment_status(3, "confirmed")`
- "Cancelar cita 5" → `update_appointment_status(5, "cancelled")`
- "Generar mensaje WhatsApp" → `generate_whatsapp_confirmation_message(appointment)`

### Reglas Importantes:

1. **SIEMPRE filtrar por `clinic_id`** - Multi-tenant obligatorio
2. **Para dentistas usar `directus_users_id`** NO `user_id`
3. **Citas próximas requieren confirmación** - Enviar WhatsApp 24h antes
4. **Estados válidos:** pending, scheduled, confirmed, cancelled, completed
5. **Formato de fecha:** YYYY-MM-DD para filtros, ISO para date_time
6. **Teléfono para WhatsApp:** Validar que patient.phone exista

## Error Handling

```python
try:
    appointments = get_appointments_today(clinic_id=1)
except requests.exceptions.HTTPError as e:
    if e.response.status_code == 401:
        print("Token inválido o expirado")
    elif e.response.status_code == 403:
        print("Sin permisos para acceder a esta clínica")
    else:
        print(f"Error: {e}")
```

## Testing

```python
# Test básico
clinic_id = 1  # Clínica Dental Sonrisas

# 1. Ver citas de hoy
print("=== Citas de hoy ===")
appointments = get_appointments_today(clinic_id)
for apt in appointments:
    print(f"- {apt['date_time']}: {apt['patient_id']['first_name']} con {apt['dentist_id']['directus_users_id']['first_name']}")

# 2. Ver tratamientos pendientes
print("\n=== Tratamientos pendientes ===")
treatments = get_pending_treatments(clinic_id)
print(f"Total: {len(treatments)}")

# 3. Ingresos del mes
print("\n=== Ingresos ===")
revenue = get_monthly_revenue(clinic_id)
print(f"Total: ${revenue['total_revenue']:,.0f} COP")
print(f"Tratamientos completados: {revenue['total_treatments']}")
```
