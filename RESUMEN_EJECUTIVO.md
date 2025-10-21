# Resumen Ejecutivo - Pipeline ETL Hotmart para Agentes IA

## Situación Actual

El equipo de Soporte Academia actualmente enfrenta un proceso completamente manual para actualizar el conocimiento de los OpenAI Assistants con contenido educativo de Hotmart:

- **Tiempo de procesamiento**: 4 días completos
- **Proceso**: 100% manual, ejecutado localmente
- **Riesgos**:
  - Pérdida de trazabilidad (incidente de disco dañado)
  - Dependencia de un solo recurso
  - Sin sincronización automática con nuevo contenido
  - Scripts desconectados sin orquestación

## Solución Propuesta

### Automatización Completa End-to-End

Implementar un pipeline ETL automatizado que:

1. **Detecta** automáticamente nuevo contenido en Hotmart (cada 24h)
2. **Descarga** videos y materiales educativos
3. **Procesa** mediante:
   - Transcripción con Google Speech-to-Text
   - Análisis multimodal con Google Gemini
   - OCR de contenido visual
4. **Clasifica** contenido por escuela/tema/nivel
5. **Sincroniza** automáticamente con OpenAI Assistants
6. **Registra** trazabilidad completa del proceso

### Beneficios Clave

| Métrica | Antes | Después | Mejora |
|---------|-------|---------|--------|
| Tiempo de actualización | 4 días | < 24 horas | 75% reducción |
| Intervención manual | 100% | 0% | Automatización total |
| Trazabilidad | Nula | Completa | Risk mitigation |
| Escalabilidad | Limitada | Ilimitada | Cloud-native |
| Confiabilidad | Baja | Alta (99.5%) | SLA garantizado |

---

## Arquitectura Técnica

### Stack Tecnológico Recomendado

#### Orquestación
- **n8n.io**: Orquestación visual del pipeline con workflows

#### Backend
- **Python 3.11+** con FastAPI: Servicios y workers
- **Celery**: Task queue para procesamiento asíncrono
- **RabbitMQ / Google Pub/Sub**: Message broker

#### Datos
- **PostgreSQL 15+**: Base de datos principal con pgvector
- **Redis**: Cache de resultados
- **Google Cloud Storage / AWS S3**: Almacenamiento de archivos

#### APIs Externas
- **Google Cloud AI**:
  - Speech-to-Text v2 (transcripciones)
  - Gemini API (análisis multimodal)
  - Vision API (OCR)
- **OpenAI**:
  - Assistants API v2
  - Vector Stores API
  - Embeddings API

#### Infraestructura
- **Docker + Kubernetes**: Containerización y orquestación
- **GitHub Actions**: CI/CD
- **Prometheus + Grafana**: Monitoreo y métricas
- **ELK Stack**: Logs centralizados

### Flujo del Pipeline

```
HOTMART → Detector → Downloader → [Transcriber + Analyzer] → Classifier → OpenAI Sync
                                          ↓
                                    Cloud Storage
                                          ↓
                                     PostgreSQL
```

**Detalles del flujo**:

1. **Detección** (cron cada 24h): Consulta API de Hotmart y detecta nuevo contenido
2. **Descarga** (paralelo max 5): Descarga videos a Cloud Storage
3. **Procesamiento** (paralelo):
   - Worker 1: Transcripción con Google STT
   - Worker 2: Análisis multimodal con Gemini + OCR
4. **Enriquecimiento**: Clasificación automática y generación de metadatos
5. **Sincronización**: Upload a OpenAI Vector Stores y actualización de Assistants
6. **Notificación**: Envío de resumen a equipo vía Slack/Email

---

## Plan de Implementación

### Fases y Timeline

| Fase | Duración | Descripción | Entregables |
|------|----------|-------------|-------------|
| **Fase 1: Fundación** | 2 semanas | Setup de infraestructura base | DB, queues, storage, CI/CD, n8n |
| **Fase 2: Ingestion** | 2 semanas | Detección y descarga de contenido | Detector Service, Downloader Service |
| **Fase 3: Processing** | 3 semanas | Transcripción y análisis multimodal | Transcription Worker, Multimodal Analyzer |
| **Fase 4: Enrichment** | 2.5 semanas | Clasificación y metadatos | Classifier Service, Metadata Generator |
| **Fase 5: Synchronization** | 3 semanas | Sync con OpenAI y trazabilidad | OpenAI Sync Service, Logging System |
| **Fase 6: UX** | 2.5 semanas | Notificaciones y dashboard | Notification Service, Admin Dashboard |
| **Fase 7-9: Hardening** | 3 semanas | Performance, HA, seguridad | Optimizaciones, tests, security audit |
| **Fase 10-12: Quality** | 3.5 semanas | Tests, costos, documentación | Test suite, cost optimization, docs |
| **Fase 13-14: Launch** | 2 semanas | Testing y lanzamiento | Dry-run, producción live |

**Total: 16 semanas (4 meses)**

### Hitos Principales

- **Semana 4**: MVP funcional (detección + descarga + transcripción básica)
- **Semana 8**: Pipeline automático end-to-end funcionando
- **Semana 12**: Sistema optimizado y con dashboard
- **Semana 16**: Producción live con monitoreo completo

---

## Requerimientos del Sistema

### Funcionales (Top 5)

1. **RF-001**: Detección automática de nuevo contenido cada 24h
2. **RF-002**: Descarga automática de videos y materiales
3. **RF-003**: Transcripción con precisión > 90% (Google STT)
4. **RF-004**: Análisis multimodal (conceptos, diagramas, código)
5. **RF-007**: Sincronización automática con OpenAI Assistants

### No Funcionales (Top 5)

1. **RNF-001**: Procesamiento de 1h de video en < 30 min
2. **RNF-002**: Soporte concurrente para 5+ videos
3. **RNF-003**: Uptime del 99.5%
4. **RNF-004**: Seguridad: credenciales en secrets manager, HTTPS/TLS
5. **RNF-005**: Mantenibilidad: cobertura de tests > 80%

---

## Costos Estimados

### Procesamiento de 100 horas de video/mes

| Categoría | Detalle | Costo Mensual |
|-----------|---------|---------------|
| **Google Cloud** | Speech-to-Text, Gemini, Vision, Storage, SQL | $424 |
| **OpenAI** | Embeddings, Vector Stores | $23 |
| **Infraestructura** | Cloud Run/K8s, Pub/Sub | $155 |
| **n8n** | Pro plan (cloud) | $50 |
| **Monitoreo** | Cloud Logging | $25 |
| **TOTAL** | | **~$677/mes** |

**Nota**: Costos optimizables mediante:
- Cache agresivo (reduce llamadas a APIs)
- Auto-scaling (apaga servicios idle)
- Lifecycle policies (limpia archivos temporales)
- Batch processing

### ROI Estimado

**Ahorro de tiempo**:
- Tiempo actual: 4 días × 1 persona = 32 horas/actualización
- Tiempo futuro: 0 horas (automatizado)
- Frecuencia: 2 actualizaciones/mes (estimado)
- **Ahorro**: 64 horas/mes

**Ahorro monetario** (asumiendo $50/hora):
- Ahorro de horas: 64h × $50 = $3,200/mes
- Costo del sistema: $677/mes
- **ROI neto**: $2,523/mes ($30,276/año)
- **Payback period**: < 1 mes

**Beneficios intangibles**:
- Eliminación de riesgo de pérdida de datos
- Conocimiento siempre actualizado para clientes
- Escalabilidad sin límite
- Mejor calidad de soporte

---

## Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| Cambio en API de Hotmart | Media | Alto | Versionado de API, monitoreo de deprecations |
| Costos excesivos de APIs | Media | Medio | Límites configurables, alertas de presupuesto |
| Calidad baja de transcripciones | Baja | Alto | Revisión manual opcional, múltiples proveedores |
| Pérdida de datos procesados | Baja | Crítico | Backups automáticos, replicación |
| Sobrecarga de OpenAI Assistants | Media | Medio | Chunking inteligente, múltiples assistants |
| Retrasos en implementación | Media | Medio | Metodología ágil, entregas incrementales |

---

## Métricas de Éxito

### Criterios de Aceptación

1. **Reducción de tiempo**: De 4 días a < 24 horas para actualización completa
2. **Automatización**: 0% intervención manual en flujo normal
3. **Precisión**: > 90% accuracy en transcripciones
4. **Disponibilidad**: 99.5% uptime del sistema
5. **Trazabilidad**: 100% de procesamiento registrado
6. **Satisfacción**: Feedback positivo del equipo de soporte académico

### KPIs de Monitoreo Continuo

- Cursos procesados por mes
- Tiempo promedio de procesamiento por hora de video
- Tasa de errores por etapa del pipeline
- Costos mensuales de APIs
- Tiempo de detección de nuevo contenido
- Calidad de transcripciones (confidence score promedio)
- Uptime del sistema

---

## Próximos Pasos

### Inmediatos (Esta semana)

1. **Aprobación ejecutiva** del proyecto y presupuesto
2. **Asignación de recursos**: Equipo de desarrollo (2-3 ingenieros)
3. **Setup inicial**:
   - Crear cuentas en Google Cloud, OpenAI
   - Configurar accesos a Hotmart API
   - Provisionar infraestructura inicial

### Semana 1-2 (Fundación)

1. Setup de infraestructura base (PostgreSQL, Redis, RabbitMQ, Cloud Storage)
2. Configuración de n8n para orquestación
3. Setup de CI/CD pipeline
4. Configuración de observabilidad (Prometheus, Grafana, ELK)

### Mes 1 (MVP)

1. Implementar Detector Service
2. Implementar Downloader Service
3. Implementar Transcription Worker básico
4. Primera prueba end-to-end con 1 curso

---

## Conclusión

La implementación de este Pipeline ETL automatizado representa una transformación fundamental en cómo el equipo de Soporte Academia gestiona el conocimiento de los OpenAI Assistants.

**Beneficios principales**:
- **75% reducción** en tiempo de actualización (4 días → < 24h)
- **100% automatización** del proceso
- **Trazabilidad completa** eliminando riesgo de pérdida de datos
- **Escalabilidad ilimitada** para crecimiento futuro
- **ROI positivo** desde el primer mes ($2,523/mes de ahorro neto)

**Inversión requerida**:
- **Tiempo**: 16 semanas (4 meses) de desarrollo
- **Recursos**: 2-3 ingenieros de software
- **Costo operacional**: $677/mes (recuperado con creces en ahorro de tiempo)

La propuesta técnica es sólida, utiliza tecnologías maduras y probadas, y el plan de implementación es realista y por fases, permitiendo entregar valor incremental cada 2-3 semanas.

**Recomendación**: Proceder con la implementación según el plan propuesto.

---

## Apéndices

### A. Documentos Relacionados

- `REQUERIMIENTOS.md`: Requerimientos funcionales y no funcionales detallados
- `ARQUITECTURA_TECNICA.md`: Arquitectura técnica completa con código de ejemplo
- **Tareas en Vibe Kanban**: 20 tareas creadas en el proyecto "ETL For Soporte Academia"

### B. Contactos

- **Equipo de desarrollo**: [Asignar]
- **Product Owner**: [Asignar]
- **Stakeholders**: Equipo de Soporte Academia

### C. Referencias

- [Hotmart API Documentation](https://developers.hotmart.com/)
- [Google Cloud Speech-to-Text](https://cloud.google.com/speech-to-text)
- [Google Gemini API](https://ai.google.dev/gemini-api/docs)
- [OpenAI Assistants API](https://platform.openai.com/docs/assistants/overview)
- [n8n Documentation](https://docs.n8n.io/)
