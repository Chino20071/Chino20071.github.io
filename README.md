# Sistema de Pre-aprobación Quirúrgica con IA

**Agente inteligente para validación automática de coberturas de seguros médicos**

> **Nota:** Este repositorio corresponde a una **demo funcional de alcance reducido**, desarrollada como presentación inicial ante el programa antes de comenzar la hackathon oficial. No representa la versión final del producto.

---

## Descripción del Proyecto

Sistema automatizado que **reduce de 48–72 horas a minutos** el tiempo de pre-aprobación de cirugías, mediante un agente de inteligencia artificial que:

1. **Recibe** el informe médico (hospital) y la póliza del paciente (aseguradora) *En texto, por temas de limitantes con los planes gratuitos que usamos*
2. **Valida** autenticidad, cobertura y períodos de carencia según el catálogo CIE-10
3. **Decide** en tiempo real: `Pre-aprobado` / `Documentos faltantes` / `Rechazado`
4. **Notifica** al paciente instantáneamente vía **Gmail**

### Problema que Resuelve

**Situación actual**
- Los liquidadores de seguros procesan manualmente cada solicitud
- Tiempo promedio de respuesta: 48–72 horas por caso
- Alto índice de errores humanos en la interpretación de pólizas
- Pacientes sin respuesta clara sobre cobertura antes del procedimiento

**Nuestra solución (demo):**
- Validación automática en **minutos**
- **5 validaciones cruzadas** de autenticidad documental
- **Justificación fundamentada** en cada decisión
- Trazabilidad completa en Notion
- Notificación directa al paciente vía **Gmail**
- **Costo operativo: $0 USD/mes** (plan gratuito Make + API gratuita de Gemini + Notion prueba gratuita)

---

## Arquitectura del Sistema (Demo)

### Diagrama de Flujo

```
┌─────────────────┐         ┌─────────────────┐
│   Hospital      │         │  Aseguradora    │
│ Informe médico  │         │  Póliza         │
└────────┬────────┘         └────────┬────────┘
         │                           │
         └───────────┬───────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │      Make.com         │
         │   Webhook Receptor    │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │  Notion Database      │
         │  Solicitudes          │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │  Agente IA            │
         │  Gemini (Google API)  │
         │  Análisis de Cobertura│
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────┐
         │  Decisión             │
         │  3 resultados posibles│
         └─┬──────────┬────────┬─┘
           │          │        │
    ┌──────▼──┐  ┌────▼────┐ ┌──▼──────┐
    │  Pre-   │  │Pendiente│ │Rechazado│
    │aprobado │  │         │ │         │
    └──────┬──┘  └────┬────┘ └──┬──────┘
           │          │         │
           └──────────┼─────────┘
                      │
            ┌─────────▼─────────┐
            │  Gmail            │
            │  Notificación     │
            │  al paciente      │
            └───────────────────┘
```
---

### Stack Tecnológico

| Componente | Tecnología | Función |
|------------|-----------|---------|
| **Base de Datos** | Notion | Almacenamiento de solicitudes, planes y catálogo CIE-10 |
| **Orquestación** | Make.com (plan gratuito) | Automatización sin código, 1.000 ops/mes |
| **Agente IA** | Gemini (Google AI API) | Validación de cobertura y toma de decisiones |

| **Notificaciones** | Gmail (Make módulo nativo) | Alerta por correo al paciente |

---

## Bases de Datos en Notion

### 1. Solicitudes de Pre-aprobación

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `Número de caso` | ID único | Auto-generado (ej: CASO-2025-001) |
| `Paciente` | Texto | Nombre completo |
| `Fecha solicitud` | Fecha | Timestamp de ingreso |
| `Estado` | Select | `Pendiente` · `En análisis` · `Pre-aprobado` · `Rechazado` |
| `Hospital` | Texto | Nombre del establecimiento |
| `Informe médico` | Texto | Detalle del informe médico |
| `Número de póliza` | Texto | Para búsqueda rápida |
| `Procedimiento` | Texto | Nombre del procedimiento |
| `Código CIE-10` | Texto | Código diagnóstico |
| `Cobertura detectada` | Select | `Cubierto` · `No cubierto` · `Parcial` |
| `Carencia cumplida` | Checkbox | ¿Cumple el período de carencia? |
| `Meses en póliza` | Número | Desde inicio de vigencia |
| `Datos faltantes` | Texto largo | Lista de requisitos faltantes |
| `Resultado IA` | Texto largo | Análisis completo del agente |
| `Justificación` | Texto largo | Fundamento contractual de la decisión |
| `Fecha resolución` | Fecha | Timestamp de cierre |

### 2. Planes de Póliza

| Campo | Tipo | Valores de ejemplo |
|-------|------|---------|
| `Nombre del plan` | Título | Básico · Estándar · Plus · Premium |
| `Código de plan` | Texto | BASIC · STD · PLUS · PREM |
| `Nivel (1-4)` | Número | Jerarquía para comparación por IA |
| `Prima mensual USD` | Número | 45 · 80 · 115 · 142 |
| `Suma asegurada USD` | Número | 8,000 · 18,000 · 30,000 · 50,000 |
| `Deducible USD` | Número | 500 · 400 · 300 · 200 |
| `Copago %` | Número | 20 · 15 · 10 · 5 |
| `Coberturas incluidas` | Relación | → BD Catálogo CIE-10 |
| `Activo` | Checkbox | Estado del plan |

### 3. Catálogo de Coberturas CIE-10

21 procedimientos quirúrgicos, cada uno con:
- Código CIE-10 (K37, K80.2, I25.1, etc.)
- Plan mínimo requerido
- Período de carencia en meses
- Límite de cobertura en USD
- Costo promedio de referencia
- Especialidad médica

---

## Agente de IA — Gemini

### Datos de entrada

El agente recibe un JSON consolidado con los datos extraídos de ambos documentos:

```json
{
  "numero_poliza": "SS-2024-00847",
  "nombre_paciente": "Carlos Mejía",
  "fecha_inicio_poliza": "2023-02-01",
  "plan_contratado": "PREM",
  "nivel_plan": 4,
  "fecha_informe": "2025-07-07",
  "diagnostico": "Colelitiasis",
  "cie10": "K80.2",
  "procedimiento": "Colecistectomía laparoscópica",
}
```

### Proceso de análisis (6 pasos)

1. **Extraer código CIE-10** del informe médico
2. **Buscar en el catálogo** → obtener `plan_mínimo` y `carencia_requerida`
3. **Comparar niveles**: `nivel_plan_paciente ≥ nivel_plan_mínimo`
4. **Calcular meses de vigencia** vs `carencia_requerida`
5. **Verificar datos** completos según tipo de procedimiento
6. **Emitir decisión** en JSON estructurado

### Validaciones de autenticidad (4 comprobaciones)

- Número de póliza del formulario valida
- Nombre del asegurado igual en póliza e informe
- Fecha de vencimiento posterior a la fecha del informe
- Sin contradicciones internas entre datos

### Respuesta JSON del agente

```json
{
  "decision": "PRE_APROBADO | PENDIENTE | RECHAZADO",
  "cobertura_ok": true,
  "carencia_ok": true,
  "meses_vigencia": 29,
  "carencia_requerida": 6,
  "limite_cobertura_usd": 9000,
  "justificacion": "Párrafo explicando la decisión con base contractual...",
  "proximo_paso": "Texto con la acción recomendada para el paciente..."
}
```

---

## Instalación y Configuración

### Requisitos previos

- Cuenta en [Notion](https://notion.so) (plan gratuito)
- Cuenta en [Make.com](https://make.com) (plan gratuito: 1.000 ops/mes)
- API Key de [Google AI Studio](https://aistudio.google.com) (Gemini)
- Cuenta Gmail conectada en Make para envío de notificaciones

---

### Paso 1: Configurar Notion

1. Crear un workspace en Notion
2. Crear las 3 bases de datos usando las estructuras documentadas arriba:
   - `Solicitudes de Pre-aprobación`
   - `Planes de Póliza`
   - `Catálogo de Coberturas CIE-10`
3. Ir a [notion.so/my-integrations](https://notion.so/my-integrations)
4. Crear una nueva integración → copiar el **Integration Token**
5. Compartir las 3 bases de datos con esa integración

---

### Paso 2: Configurar Make.com

1. Crear cuenta en Make.com
2. Nuevo escenario: `Sistema Pre-aprobación Seguros - Demo`
3. Agregar módulos en este orden:

**Módulo 1 — Webhook receptor**
```
Webhooks > Custom webhook
```
Vincular el webhooks custom webhook principal al HTML

**Módulo 2 — Crear registro en Notion**
```
Notion > Create a database item
Database: Solicitudes de Pre-aprobación
Notion crear database en tipo base de datos crear valores y añadir propiedad específicas
```

**Módulo 3 — Agente de análisis (Gemini)**
```
HTTP > Make a request
Modelo: Gemini 2.5 Flash
URL: https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent
Method: POST
Body: JSON con datos consolidados del paciente + póliza + catálogo CIE-10
```


**Módulo 4 — Notificación por Gmail**
```
Gmail > Send an email
To: {{email_paciente}}
Subject: Resultado de su pre-aprobación quirúrgica
Body: (ver plantillas abajo)
```

---

### Paso 3: Variables de entorno en Make

En Make.com, configurar como **Variables del escenario**:

| Variable | Descripción |
|----------|-------------|
| `GEMINI_API_KEY` | API Key de Google AI Studio |
| `NOTION_TOKEN` | Token de integración de Notion |
| `NOTION_DB_SOLICITUDES` | ID de la base de datos de solicitudes |
| `NOTION_DB_PLANES` | ID de la base de datos de planes |
| `NOTION_DB_CIE10` | ID del catálogo CIE-10 |
| `GMAIL_SENDER` | Dirección de correo remitente |

---

## Plantillas de Notificación Gmail

### Pre-aprobado

```
Asunto: Su cirugía ha sido PRE-APROBADA — Caso {{numero_caso}}

Estimado/a {{nombre_paciente}},

Su solicitud de pre-aprobación quirúrgica ha sido procesada satisfactoriamente.

Procedimiento: {{procedimiento}}
Póliza: {{numero_poliza}} — Plan {{plan_contratado}}
Cobertura: USD {{limite_cobertura_usd}}
Deducible: USD {{deducible}}
Copago: {{copago}}%

La carta de garantía será enviada al hospital en las próximas 2 horas.
Número de autorización: {{numero_autorizacion}}

Saludos,
Sistema de Pre-aprobación — SaludSegura Ecuador
```

### Pendiente

```
Asunto: Datos adicionales requeridos — Caso {{numero_caso}}

Estimado/a {{nombre_paciente}},

Su solicitud requiere los siguientes datos para continuar:

{{lista_atos_faltantes}}

Envíelos a: autorizaciones@saludsegura.ec
Una vez recibidos, la resolución se emitirá en menos de 60 minutos.

Saludos,
Sistema de Pre-aprobación — SaludSegura Ecuador
```

### No Cubierto

```
Asunto: Procedimiento no cubierto por su plan actual — Caso {{numero_caso}}

Estimado/a {{nombre_paciente}},

El procedimiento solicitado no está cubierto por su plan de póliza actual.

Procedimiento: {{procedimiento}}
Plan actual: {{plan_contratado}} (nivel {{nivel_plan}})
Plan requerido: {{plan_requerido}}

Para solicitar un upgrade de plan, comuníquese al (02) 390-4500.
La cobertura aplicaría 12 meses después de realizado el upgrade.

Saludos,
Sistema de Pre-aprobación — SaludSegura Ecuador
```

---

## Métricas de Rendimiento

| Métrica | Sistema Actual | Con IA (demo) |
|---------|---------------|----------------|
| **Tiempo de respuesta** | 48–72 horas | < 10 minutos |
| **Validaciones de autenticidad** | 1–2 (visual) | 4 (automatizadas) |
| **Tasa de error humano** | ~15% | < 2% |
| **Costo por solicitud** | USD 8–12 | USD 0 |
| **Capacidad mensual** | ~200 casos/analista | Escalable |
| **Disponibilidad** | Horario laboral | 24/7 |

---

## Casos de Uso en la Demo

### Caso 1 — Pre-aprobación exitosa
**Paciente:** Carlos Mejía, Plan Premium (29 meses de vigencia)
**Procedimiento:** Colecistectomía laparoscópica (K80.2)
**Plan mínimo requerido:** Estándar · **Carencia requerida:** 6 meses
**Resultado:** PRE-APROBADO
**Justificación:** Plan superior al requerido y período de carencia cumplido.

### Caso 2 — Rechazo por plan insuficiente
**Paciente:** Plan Básico
**Procedimiento:** Cirugía oncológica (C80.1) — requiere Plan Premium
**Resultado:** RECHAZADO
**Justificación:** El procedimiento oncológico no está cubierto por el Plan Básico.

---

## Seguridad y Cumplimiento

- **Token secreto** en el header del webhook (previene accesos no autorizados)
- **4 comprobaciones cruzadas** de autenticidad documental
- **Logs de auditoría** trazables en Notion
- **Cifrado en tránsito** (HTTPS/TLS en todos los endpoints)
- Diagnósticos almacenados exclusivamente en Notion (workspace privado)
- API Keys gestionadas como variables de entorno en Make (nunca expuestas en código)
- Sistema de **asistencia a la decisión**, no reemplazo de la resolución legal humana

---

## Solución de Problemas Frecuentes


**El agente Gemini responde con error 401 / 403**
- Verificar que la API Key de Google AI Studio sea válida y esté activa
- Confirmar que el header `x-goog-api-key` esté correctamente configurado en Make

**Notion no actualiza el estado del caso**
- Verificar que la integración tenga permisos sobre la base de datos
- Revisar que el `Database ID` esté correctamente copiado
- Confirmar que los campos mapeados existan con exactamente el mismo nombre

**El sistema se acerca al límite de 1.000 operaciones gratuitas**
- Consolidar varias llamadas HTTP en un solo módulo donde sea posible
- Considerar el upgrade a plan Core de Make (USD 10/mes, 10k ops)

---

## Alcance de esta Demo vs. Versión Final

Esta demo cubre los flujos principales del sistema de forma simplificada. La versión presentada en la hackathon oficial incluirá:

| Característica | Demo actual | Versión hackathon |
|---|---|---|
| Notificaciones | Gmail | Gmail + WhatsApp Business |
| Catálogo CIE-10 | 21 procedimientos | 200+ procedimientos |
| Validación de pólizas | Lógica interna | API directa con aseguradoras |
| Dashboard | No incluido | Métricas en Notion |
| Anti-fraude | Validación básica | Firma digital / QR en pólizas |

---

## Equipo

**Demo desarrollada para presentación inicial al programa hackIAthon**

| Integrantes|
|------|
| Erick Gómez |
| Anthony Yepez |
| Ileana Alcaide |


---


<div align="center">

Construido con esfuerzo para la **hackAIthon** | Ecuador

</div>
