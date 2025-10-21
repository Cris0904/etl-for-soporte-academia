# Requerimientos del Sistema - Pipeline ETL Hotmart

## 1. INFORMACIÓN GENERAL

### 1.1 Propósito del Sistema
Automatizar el proceso de extracción, transformación y carga (ETL) de contenido educativo desde Hotmart hacia OpenAI Assistants, reduciendo el tiempo de procesamiento de 4 días a ejecución automática continua.

### 1.2 Alcance
- **Módulos incluidos**: Detección de contenido, descarga, procesamiento multimedia, extracción de conocimiento, clasificación y sincronización con OpenAI Assistants
- **Usuarios objetivo**: Equipo de soporte académico, administradores de contenido
- **Límites del sistema**: No incluye modificación de contenido fuente en Hotmart, solo lectura

---

## 2. REQUERIMIENTOS FUNCIONALES

### RF-001: Detección Automática de Nuevo Contenido
- **Prioridad**: Alta
- **Descripción**: El sistema debe detectar automáticamente cuando se publica nuevo contenido en Hotmart
- **Criterios de aceptación**:
  - Verificación cada 24 horas de nuevos cursos/módulos/lecciones
  - Identificación de contenido modificado mediante hash/timestamp
  - Registro de detección en base de datos con metadata completa
  - Notificación de nuevo contenido detectado

### RF-002: Descarga de Contenido Multimedia
- **Prioridad**: Alta
- **Descripción**: Descargar videos educativos y recursos asociados desde Hotmart
- **Criterios de aceptación**:
  - Autenticación segura con API de Hotmart
  - Descarga de videos en calidad óptima para procesamiento
  - Descarga de materiales complementarios (PDFs, presentaciones)
  - Verificación de integridad de archivos descargados
  - Reintentos automáticos en caso de fallo
  - Almacenamiento temporal organizado por curso/módulo

### RF-003: Extracción de Transcripciones
- **Prioridad**: Alta
- **Descripción**: Convertir audio de videos a texto mediante Google Multimodal AI
- **Criterios de aceptación**:
  - Transcripción automática con timestamps
  - Identificación de hablantes (speaker diarization)
  - Detección de idioma automática
  - Precisión mínima del 90%
  - Formato estructurado (JSON/SRT)

### RF-004: Análisis Multimodal de Contenido
- **Prioridad**: Alta
- **Descripción**: Extraer información visual y contextual de los videos
- **Criterios de aceptación**:
  - Extracción de frames clave cada N segundos
  - Identificación de diagramas, código en pantalla, presentaciones
  - OCR de texto visible en pantalla
  - Descripción de contenido visual relevante
  - Sincronización con transcripción

### RF-005: Clasificación Automática de Contenido
- **Prioridad**: Media
- **Descripción**: Organizar contenido por escuela, tema y nivel de complejidad
- **Criterios de aceptación**:
  - Taxonomía predefinida (escuelas: programación, marketing, diseño, etc.)
  - Clasificación automática mediante embeddings semánticos
  - Extracción de palabras clave y conceptos principales
  - Asignación de nivel (básico, intermedio, avanzado)
  - Revisión manual opcional antes de publicar

### RF-006: Generación de Metadatos Estructurados
- **Prioridad**: Alta
- **Descripción**: Crear metadatos enriquecidos para cada pieza de contenido
- **Criterios de aceptación**:
  - Título, descripción, duración, instructor
  - Estructura de curso (módulos → lecciones)
  - Prerequisitos identificados
  - Objetivos de aprendizaje extraídos
  - Referencias y recursos mencionados
  - Formato JSON Schema validado

### RF-007: Sincronización con OpenAI Assistants
- **Prioridad**: Alta
- **Descripción**: Actualizar automáticamente los OpenAI Assistants con nuevo conocimiento
- **Criterios de aceptación**:
  - Creación/actualización de Vector Stores en OpenAI
  - Chunking inteligente de contenido (tamaño óptimo para embeddings)
  - Actualización incremental (solo nuevo contenido)
  - Versionado de conocimiento
  - Validación de sincronización exitosa
  - Rollback en caso de error

### RF-008: Sistema de Trazabilidad Completa
- **Prioridad**: Alta
- **Descripción**: Mantener registro detallado de todo el proceso ETL
- **Criterios de aceptación**:
  - Log de cada etapa del pipeline con timestamps
  - Registro de errores con stack traces
  - Métricas de procesamiento (tiempo, recursos, éxito/fallo)
  - Historial de versiones de contenido procesado
  - Capacidad de reproducir procesamiento de cualquier pieza de contenido
  - Dashboard de monitoreo en tiempo real

### RF-009: Notificaciones y Alertas
- **Prioridad**: Media
- **Descripción**: Informar al equipo sobre eventos importantes del pipeline
- **Criterios de aceptación**:
  - Notificación de nuevo contenido detectado
  - Alertas de errores críticos
  - Resumen diario de procesamiento
  - Notificación de sincronización completada
  - Integración con Slack/Email/Discord

### RF-010: Interfaz de Administración
- **Prioridad**: Baja
- **Descripción**: Panel web para gestionar el pipeline
- **Criterios de aceptación**:
  - Visualización de estado del pipeline
  - Trigger manual de procesamiento
  - Configuración de parámetros (frecuencia, calidad, etc.)
  - Consulta de logs y métricas
  - Gestión de taxonomía y clasificaciones

---

## 3. REQUERIMIENTOS NO FUNCIONALES

### RNF-001: Rendimiento
- Procesamiento de 1 hora de video en máximo 30 minutos
- Soporte concurrente para mínimo 5 videos simultáneos
- Detección de nuevo contenido en menos de 5 minutos desde publicación
- API de consulta con latencia < 200ms

### RNF-002: Escalabilidad
- Arquitectura que soporte crecimiento a 1000+ cursos
- Procesamiento distribuido mediante workers
- Almacenamiento escalable (cloud storage)
- Auto-scaling basado en carga

### RNF-003: Disponibilidad
- Uptime del 99.5% para detección de contenido
- Recuperación automática de fallos
- Procesamiento eventual garantizado (retry logic)
- Sin pérdida de datos en caso de fallo

### RNF-004: Seguridad
- Credenciales almacenadas en secrets manager
- Comunicación cifrada (HTTPS/TLS)
- Autenticación y autorización para API
- Cumplimiento con políticas de privacidad de contenido
- Logs sin información sensible

### RNF-005: Mantenibilidad
- Código modular y desacoplado
- Documentación técnica completa
- Tests unitarios con cobertura > 80%
- Tests de integración para flujos críticos
- CI/CD automatizado

### RNF-006: Observabilidad
- Logging estructurado (JSON)
- Métricas exportadas a Prometheus/CloudWatch
- Distributed tracing (Jaeger/OpenTelemetry)
- Alertas configurables
- Dashboards de monitoreo

### RNF-007: Costos
- Optimización de llamadas a APIs de terceros
- Uso eficiente de almacenamiento (limpieza automática)
- Auto-scaling para minimizar costos en períodos de baja demanda
- Presupuesto máximo mensual definido

---

## 4. REQUERIMIENTOS DE INTEGRACIÓN

### RI-001: Hotmart API
- Autenticación OAuth 2.0
- Endpoints: listado de cursos, descarga de contenido
- Rate limiting respetado
- Webhook para notificaciones (si disponible)

### RI-002: Google Multimodal AI
- Google Cloud Video Intelligence API
- Speech-to-Text API con modelos mejorados
- Vision API para OCR
- Gestión de cuotas y límites

### RI-003: OpenAI Platform
- Assistants API v2
- Vector Stores API
- File uploads API
- Manejo de límites de tamaño y tokens

### RI-004: Almacenamiento
- Cloud storage (S3/GCS/Azure Blob)
- Base de datos relacional (PostgreSQL)
- Cache distribuido (Redis) opcional
- CDN para recursos procesados

---

## 5. REQUERIMIENTOS DE DATOS

### RD-001: Modelo de Datos Principal
```
Curso
├── id: UUID
├── hotmart_id: String
├── titulo: String
├── descripcion: Text
├── instructor: String
├── escuela: Enum
├── nivel: Enum
├── fecha_creacion: Timestamp
├── fecha_actualizacion: Timestamp
├── estado_procesamiento: Enum
└── modulos: [Modulo]

Modulo
├── id: UUID
├── curso_id: FK
├── numero_orden: Integer
├── titulo: String
├── descripcion: Text
└── lecciones: [Leccion]

Leccion
├── id: UUID
├── modulo_id: FK
├── numero_orden: Integer
├── titulo: String
├── duracion_segundos: Integer
├── url_original: String
├── estado_procesamiento: Enum
├── transcripcion: Text
├── analisis_visual: JSONB
├── metadata: JSONB
├── vector_store_id: String
└── fecha_procesamiento: Timestamp

ProcesamientoLog
├── id: UUID
├── leccion_id: FK
├── etapa: Enum (descarga, transcripción, análisis, sincronización)
├── estado: Enum (iniciado, exitoso, fallido)
├── mensaje: Text
├── duracion_ms: Integer
├── metadata: JSONB
└── timestamp: Timestamp
```

### RD-002: Retención de Datos
- Videos originales: 7 días post-procesamiento
- Transcripciones: Permanente
- Logs: 90 días
- Métricas agregadas: Permanente

---

## 6. CASOS DE USO PRINCIPALES

### CU-001: Procesamiento Automático de Nuevo Curso
**Actor**: Sistema (automatizado)
**Flujo**:
1. Sistema detecta nuevo curso en Hotmart
2. Descarga todos los videos del curso
3. Procesa cada video (transcripción + análisis)
4. Clasifica contenido automáticamente
5. Genera metadatos estructurados
6. Sincroniza con OpenAI Assistant correspondiente
7. Envía notificación de completitud
8. Limpia archivos temporales

### CU-002: Reprocesamiento de Contenido Existente
**Actor**: Administrador
**Flujo**:
1. Admin selecciona curso/lección para reprocesar
2. Sistema valida permisos
3. Elimina datos procesados anteriores
4. Re-ejecuta pipeline completo
5. Compara resultados con versión anterior
6. Actualiza Assistant con nueva versión
7. Registra cambios en historial

### CU-003: Consulta de Estado de Procesamiento
**Actor**: Usuario de soporte
**Flujo**:
1. Usuario consulta API/Dashboard
2. Sistema muestra estado actual de todos los cursos
3. Usuario puede filtrar por escuela/estado/fecha
4. Sistema muestra métricas y logs relevantes

---

## 7. RESTRICCIONES Y SUPOSICIONES

### Restricciones
- Acceso válido a API de Hotmart con permisos de lectura
- Presupuesto mensual para APIs de Google y OpenAI
- Contenido en español/inglés principalmente
- No se pueden modificar cursos en Hotmart

### Suposiciones
- Hotmart provee API estable para acceso a contenido
- Calidad de audio en videos es suficiente para transcripción
- OpenAI Assistants soporta el volumen de conocimiento requerido
- Equipo técnico disponible para mantenimiento

---

## 8. CRITERIOS DE ÉXITO

1. **Reducción de tiempo**: De 4 días a < 24 horas para actualización completa
2. **Automatización**: 0% intervención manual en flujo normal
3. **Precisión**: > 90% accuracy en transcripciones
4. **Disponibilidad**: 99.5% uptime del sistema
5. **Trazabilidad**: 100% de procesamiento registrado
6. **Satisfacción**: Feedback positivo del equipo de soporte académico

---

## 9. RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| Cambio en API de Hotmart | Media | Alto | Versionado de API, monitoreo de deprecations |
| Costos excesivos de APIs | Media | Medio | Límites configurables, alertas de presupuesto |
| Calidad baja de transcripciones | Baja | Alto | Revisión manual opcional, múltiples proveedores |
| Pérdida de datos procesados | Baja | Crítico | Backups automáticos, replicación |
| Sobrecarga de OpenAI Assistants | Media | Medio | Chunking inteligente, múltiples assistants |

---

## 10. FASES DE IMPLEMENTACIÓN

### Fase 1 - MVP (Semanas 1-4)
- Detección básica de contenido
- Descarga de videos
- Transcripción con Google Speech-to-Text
- Sincronización manual con OpenAI

### Fase 2 - Automatización (Semanas 5-8)
- Pipeline automático end-to-end
- Clasificación automática
- Sistema de trazabilidad
- Notificaciones básicas

### Fase 3 - Optimización (Semanas 9-12)
- Análisis multimodal completo
- Procesamiento concurrente
- Dashboard de administración
- Optimización de costos

### Fase 4 - Escalamiento (Semanas 13-16)
- Auto-scaling
- Alta disponibilidad
- Monitoreo avanzado
- Documentación completa
