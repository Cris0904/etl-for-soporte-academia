# Arquitectura Técnica - Pipeline ETL Hotmart

## 1. ARQUITECTURA GENERAL

### 1.1 Diagrama de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────┐
│                         HOTMART API                              │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ├─── Webhook (opcional)
                 └─── Polling Schedule
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                    INGESTION LAYER                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Detector   │  │  Downloader  │  │  Storage Manager     │  │
│  │   Service    │──│   Service    │──│  (S3/GCS)            │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ Message Queue (RabbitMQ/SQS)
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                   PROCESSING LAYER                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Transcription│  │   Multimodal │  │   Classifier         │  │
│  │ Worker       │  │   Analyzer   │  │   Service            │  │
│  │ (Google STT) │  │   (Gemini)   │  │   (Embeddings)       │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 │ Message Queue
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                    ENRICHMENT LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Metadata    │  │   Chunking   │  │   Vector Store       │  │
│  │  Generator   │  │   Service    │  │   Manager            │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                  SYNCHRONIZATION LAYER                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  OpenAI      │  │  Version     │  │   Notification       │  │
│  │  Sync        │  │  Control     │  │   Service            │  │
│  │  Service     │  │              │  │   (Slack/Email)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                     DATA LAYER                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  PostgreSQL  │  │    Redis     │  │   Cloud Storage      │  │
│  │  (Metadata)  │  │   (Cache)    │  │   (Videos/Files)     │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  OBSERVABILITY LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Logging     │  │   Metrics    │  │   Tracing            │  │
│  │  (ELK/Cloud) │  │ (Prometheus) │  │   (Jaeger/OTEL)      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. STACK TECNOLÓGICO PROPUESTO

### 2.1 Opción A: Cloud-Native (Recomendado para escalabilidad)

#### Backend / Orquestación
- **n8n.io** (Auto-hosted): Orquestación visual del pipeline ETL
  - Workflows para cada etapa del proceso
  - Triggers programados y webhooks
  - Integración nativa con múltiples servicios
  - UI para monitoreo y debugging

#### Lenguajes y Frameworks
- **Python 3.11+**: Workers de procesamiento
  - FastAPI: APIs REST
  - Celery: Task queue para procesamiento asíncrono
  - SQLAlchemy: ORM para PostgreSQL
  - Pydantic: Validación de datos

#### Message Queue
- **RabbitMQ** o **Google Cloud Pub/Sub**
  - Desacoplamiento entre servicios
  - Retry logic automático
  - Dead letter queues

#### Base de Datos
- **PostgreSQL 15+**: Datos estructurados
  - Extensión pgvector para embeddings (opcional)
  - JSONB para metadata flexible

- **Redis**: Cache y sesiones
  - Cache de resultados de APIs
  - Rate limiting

#### Almacenamiento
- **Google Cloud Storage** o **AWS S3**
  - Videos temporales
  - Archivos procesados
  - Backups

#### APIs Externas
- **Google Cloud AI**:
  - Speech-to-Text API v2
  - Video Intelligence API
  - Vision API (OCR)
  - Gemini API (análisis multimodal)

- **OpenAI**:
  - Assistants API v2
  - Embeddings API (text-embedding-3-large)
  - Vector Stores API

- **Hotmart**:
  - REST API (documentación oficial)

#### Containerización
- **Docker** + **Docker Compose**: Desarrollo local
- **Kubernetes** (GKE/EKS): Producción
  - Horizontal Pod Autoscaler
  - StatefulSets para servicios stateful

#### CI/CD
- **GitHub Actions** o **GitLab CI**
  - Tests automatizados
  - Build de imágenes Docker
  - Deploy a staging/producción

#### Observabilidad
- **Prometheus + Grafana**: Métricas y dashboards
- **ELK Stack** (Elasticsearch, Logstash, Kibana): Logs centralizados
- **Jaeger** o **OpenTelemetry**: Distributed tracing
- **Sentry**: Error tracking

#### Frontend (Opcional)
- **Next.js 14+**: Dashboard administrativo
  - Server Components
  - API routes
  - Tailwind CSS

### 2.2 Opción B: Serverless (Recomendado para minimizar costos)

#### Orquestación
- **n8n Cloud** o **AWS Step Functions**
  - Orquestación sin servidor
  - Pay-per-use

#### Compute
- **Google Cloud Functions** / **AWS Lambda**
  - Detector: Cloud Function programada (cron)
  - Downloader: Cloud Function con timeout extendido
  - Processors: Cloud Functions con concurrencia configurada

#### Message Queue
- **Google Cloud Pub/Sub** / **AWS SQS**

#### Base de Datos
- **Cloud SQL (PostgreSQL)** / **AWS RDS**
- **Firestore** para datos semi-estructurados (alternativa)

#### Almacenamiento
- **Cloud Storage** / **S3**

---

## 3. ARQUITECTURA DE SERVICIOS

### 3.1 Detector Service
**Responsabilidad**: Detectar nuevo contenido en Hotmart

```python
# Pseudo-código
class DetectorService:
    def __init__(self, hotmart_client, db):
        self.hotmart = hotmart_client
        self.db = db

    async def detect_new_content(self):
        # 1. Obtener lista de cursos desde Hotmart
        courses = await self.hotmart.get_courses()

        # 2. Comparar con DB
        for course in courses:
            if not self.db.exists(course.id):
                # 3. Crear registro
                await self.db.create_course(course)

                # 4. Encolar para procesamiento
                await self.queue.publish(
                    'course.new',
                    {'course_id': course.id}
                )

        # 5. Detectar actualizaciones
        for course in courses:
            if self.has_updates(course):
                await self.queue.publish(
                    'course.updated',
                    {'course_id': course.id}
                )
```

**Tecnologías**:
- Python + FastAPI
- Scheduled job (cron/Cloud Scheduler)
- Hotmart SDK/API client

**Métricas**:
- Cursos nuevos detectados
- Tiempo de ejecución
- Errores de API

---

### 3.2 Downloader Service
**Responsabilidad**: Descargar videos y materiales

```python
class DownloaderService:
    async def download_course(self, course_id):
        # 1. Obtener estructura del curso
        modules = await self.hotmart.get_modules(course_id)

        for module in modules:
            lessons = await self.hotmart.get_lessons(module.id)

            for lesson in lessons:
                # 2. Descargar video
                video_path = await self.download_video(
                    lesson.video_url,
                    quality='1080p'
                )

                # 3. Validar integridad
                if not self.validate_checksum(video_path):
                    raise DownloadError(f"Corrupted: {lesson.id}")

                # 4. Subir a Cloud Storage
                gcs_path = await self.storage.upload(
                    video_path,
                    f"raw/{course_id}/{module.id}/{lesson.id}.mp4"
                )

                # 5. Actualizar DB
                await self.db.update_lesson(
                    lesson.id,
                    status='downloaded',
                    storage_path=gcs_path
                )

                # 6. Encolar para procesamiento
                await self.queue.publish(
                    'lesson.downloaded',
                    {'lesson_id': lesson.id, 'path': gcs_path}
                )

                # 7. Cleanup local
                os.remove(video_path)
```

**Tecnologías**:
- Python + aiohttp (async downloads)
- Cloud Storage SDK
- Retry logic con exponential backoff

**Consideraciones**:
- Descarga paralela con límite (ej: 5 simultáneos)
- Timeout extendido (videos grandes)
- Espacio en disco temporal suficiente

---

### 3.3 Transcription Worker
**Responsabilidad**: Convertir audio a texto

```python
class TranscriptionWorker:
    def __init__(self, speech_client, storage, db):
        self.speech = speech_client
        self.storage = storage
        self.db = db

    async def transcribe_lesson(self, lesson_id, video_path):
        # 1. Extraer audio
        audio_path = await self.extract_audio(
            video_path,
            format='flac',  # Google STT prefiere FLAC
            sample_rate=16000
        )

        # 2. Subir audio a Cloud Storage (requerido por Google)
        audio_uri = await self.storage.upload(audio_path)

        # 3. Configurar reconocimiento
        config = {
            'language_code': 'es-ES',
            'enable_automatic_punctuation': True,
            'enable_word_time_offsets': True,
            'enable_speaker_diarization': True,
            'diarization_speaker_count': 2,
            'model': 'video',  # Modelo optimizado para videos
            'use_enhanced': True
        }

        # 4. Ejecutar transcripción (long-running operation)
        operation = await self.speech.long_running_recognize(
            audio=audio_uri,
            config=config
        )

        # 5. Esperar resultado (polling)
        result = await operation.result(timeout=3600)

        # 6. Estructurar transcripción
        transcription = self.format_transcription(result)

        # 7. Guardar en DB
        await self.db.update_lesson(
            lesson_id,
            transcription=transcription,
            status='transcribed'
        )

        # 8. Encolar siguiente etapa
        await self.queue.publish(
            'lesson.transcribed',
            {'lesson_id': lesson_id}
        )

        # 9. Cleanup
        await self.storage.delete(audio_uri)

    def format_transcription(self, result):
        """Convierte resultado de Google a formato estructurado"""
        return {
            'text': result.alternatives[0].transcript,
            'confidence': result.alternatives[0].confidence,
            'words': [
                {
                    'word': word.word,
                    'start_time': word.start_time.seconds,
                    'end_time': word.end_time.seconds,
                    'speaker': word.speaker_tag
                }
                for word in result.alternatives[0].words
            ]
        }
```

**Tecnologías**:
- Google Cloud Speech-to-Text API v2
- FFmpeg para extracción de audio
- Async processing con Celery

**Optimizaciones**:
- Batch processing cuando sea posible
- Cache de resultados por hash de audio
- Reintentos con modelo alternativo si falla

---

### 3.4 Multimodal Analyzer
**Responsabilidad**: Análisis visual del video

```python
class MultimodalAnalyzer:
    async def analyze_lesson(self, lesson_id, video_path):
        # 1. Extraer frames clave
        frames = await self.extract_keyframes(
            video_path,
            interval_seconds=10,
            quality='high'
        )

        # 2. Análisis con Gemini
        analysis = await self.gemini.analyze_video(
            video_path=video_path,
            prompt="""
            Analiza este video educativo y extrae:
            1. Conceptos principales mostrados visualmente
            2. Diagramas o esquemas (describe su contenido)
            3. Código en pantalla (transcríbelo)
            4. Texto importante en slides
            5. Demostraciones prácticas

            Responde en JSON estructurado.
            """
        )

        # 3. OCR en frames con texto
        text_frames = []
        for frame in frames:
            if self.has_text(frame):
                ocr_result = await self.vision.detect_text(frame)
                text_frames.append({
                    'timestamp': frame.timestamp,
                    'text': ocr_result.text,
                    'confidence': ocr_result.confidence
                })

        # 4. Estructurar resultado
        visual_analysis = {
            'gemini_analysis': analysis,
            'text_detected': text_frames,
            'keyframes_count': len(frames),
            'has_code': self.detect_code_patterns(analysis),
            'has_diagrams': self.detect_diagrams(analysis)
        }

        # 5. Guardar en DB
        await self.db.update_lesson(
            lesson_id,
            visual_analysis=visual_analysis,
            status='analyzed'
        )

        # 6. Encolar siguiente etapa
        await self.queue.publish(
            'lesson.analyzed',
            {'lesson_id': lesson_id}
        )
```

**Tecnologías**:
- Google Gemini API (multimodal)
- Google Vision API (OCR)
- OpenCV para extracción de frames

---

### 3.5 Classifier Service
**Responsabilidad**: Clasificar contenido automáticamente

```python
class ClassifierService:
    async def classify_lesson(self, lesson_id):
        # 1. Obtener contenido procesado
        lesson = await self.db.get_lesson(lesson_id)

        # 2. Crear embedding del contenido
        content = f"""
        Título: {lesson.title}
        Transcripción: {lesson.transcription.text}
        Análisis visual: {lesson.visual_analysis.summary}
        """

        embedding = await self.openai.create_embedding(
            model='text-embedding-3-large',
            input=content
        )

        # 3. Clasificar por similitud con taxonomía
        taxonomy = await self.db.get_taxonomy()

        similarities = []
        for category in taxonomy:
            similarity = cosine_similarity(
                embedding,
                category.embedding
            )
            similarities.append((category, similarity))

        # 4. Seleccionar categorías
        best_matches = sorted(
            similarities,
            key=lambda x: x[1],
            reverse=True
        )[:3]

        # 5. Extraer palabras clave
        keywords = await self.extract_keywords(content)

        # 6. Determinar nivel
        level = await self.classify_level(content)

        # 7. Guardar clasificación
        await self.db.update_lesson(
            lesson_id,
            categories=[m[0].id for m in best_matches],
            keywords=keywords,
            level=level,
            status='classified'
        )
```

**Tecnologías**:
- OpenAI Embeddings API
- spaCy o BERT para NLP
- Taxonomía predefinida en DB

---

### 3.6 OpenAI Sync Service
**Responsabilidad**: Sincronizar con OpenAI Assistants

```python
class OpenAISyncService:
    async def sync_lesson(self, lesson_id):
        # 1. Obtener lección completa
        lesson = await self.db.get_lesson_full(lesson_id)

        # 2. Generar documento estructurado
        document = self.generate_knowledge_document(lesson)

        # 3. Chunking inteligente
        chunks = self.chunk_document(
            document,
            max_tokens=8000,
            overlap=200
        )

        # 4. Crear archivo temporal
        file_path = f"/tmp/{lesson_id}.jsonl"
        with open(file_path, 'w') as f:
            for chunk in chunks:
                f.write(json.dumps(chunk) + '\n')

        # 5. Subir a OpenAI
        file = await self.openai.files.create(
            file=open(file_path, 'rb'),
            purpose='assistants'
        )

        # 6. Obtener/crear Vector Store para la escuela
        vector_store = await self.get_or_create_vector_store(
            lesson.course.school
        )

        # 7. Agregar archivo al Vector Store
        await self.openai.vector_stores.files.create(
            vector_store_id=vector_store.id,
            file_id=file.id
        )

        # 8. Esperar indexación
        await self.wait_for_indexing(vector_store.id, file.id)

        # 9. Actualizar Assistant
        assistant = await self.get_assistant(lesson.course.school)

        # Verificar que el vector store está asociado
        if vector_store.id not in assistant.tool_resources.file_search.vector_store_ids:
            await self.openai.assistants.update(
                assistant_id=assistant.id,
                tool_resources={
                    'file_search': {
                        'vector_store_ids': [
                            *assistant.tool_resources.file_search.vector_store_ids,
                            vector_store.id
                        ]
                    }
                }
            )

        # 10. Registrar sincronización
        await self.db.update_lesson(
            lesson_id,
            openai_file_id=file.id,
            vector_store_id=vector_store.id,
            status='synced',
            synced_at=datetime.utcnow()
        )

        # 11. Notificar
        await self.notifications.send(
            f"✅ Lección sincronizada: {lesson.title}"
        )

    def generate_knowledge_document(self, lesson):
        """Genera documento optimizado para retrieval"""
        return {
            'type': 'lesson',
            'id': lesson.id,
            'title': lesson.title,
            'course': lesson.course.title,
            'school': lesson.course.school,
            'level': lesson.level,
            'duration_minutes': lesson.duration_seconds / 60,
            'instructor': lesson.course.instructor,
            'content': {
                'transcription': lesson.transcription.text,
                'visual_elements': lesson.visual_analysis.summary,
                'key_concepts': lesson.keywords,
                'code_snippets': lesson.visual_analysis.code_detected,
            },
            'metadata': {
                'created_at': lesson.created_at.isoformat(),
                'hotmart_url': lesson.url_original,
                'prerequisites': lesson.prerequisites,
                'learning_objectives': lesson.objectives
            }
        }

    def chunk_document(self, doc, max_tokens, overlap):
        """Chunking semántico inteligente"""
        # Estrategia: dividir por secciones lógicas
        # manteniendo contexto con overlap
        # ...implementación...
        pass
```

**Tecnologías**:
- OpenAI Python SDK
- tiktoken para conteo de tokens
- Semantic chunking

---

## 4. FLUJO DE DATOS COMPLETO

### 4.1 Pipeline End-to-End

```
1. DETECCIÓN (cada 24h)
   ├─ Consulta API Hotmart
   ├─ Compara con DB
   └─ Encola nuevos cursos

2. DESCARGA (por curso)
   ├─ Obtiene estructura (módulos/lecciones)
   ├─ Descarga videos en paralelo (max 5)
   ├─ Sube a Cloud Storage
   └─ Encola cada lección

3. PROCESAMIENTO (por lección)
   ├─ [Worker 1] Transcripción
   │  ├─ Extrae audio
   │  ├─ Llama Google STT
   │  └─ Guarda transcripción
   │
   ├─ [Worker 2] Análisis multimodal (paralelo)
   │  ├─ Extrae frames
   │  ├─ Llama Gemini
   │  ├─ OCR con Vision API
   │  └─ Guarda análisis
   │
   └─ Espera ambos workers

4. ENRIQUECIMIENTO
   ├─ Clasificación automática
   ├─ Extracción de keywords
   ├─ Determinación de nivel
   └─ Generación de metadata

5. SINCRONIZACIÓN
   ├─ Genera documento estructurado
   ├─ Chunking inteligente
   ├─ Sube a OpenAI
   ├─ Actualiza Vector Store
   └─ Verifica indexación

6. FINALIZACIÓN
   ├─ Marca como completado
   ├─ Envía notificación
   ├─ Limpia archivos temporales
   └─ Registra métricas
```

### 4.2 Manejo de Errores

```python
# Estrategia de retry por tipo de error

ERROR_STRATEGIES = {
    'NetworkError': {
        'max_retries': 5,
        'backoff': 'exponential',
        'max_delay': 300
    },
    'APIQuotaError': {
        'max_retries': 3,
        'backoff': 'linear',
        'delay': 3600  # 1 hora
    },
    'ValidationError': {
        'max_retries': 0,
        'action': 'log_and_skip'
    },
    'TranscriptionQualityError': {
        'max_retries': 1,
        'action': 'try_alternative_model'
    }
}
```

---

## 5. ESQUEMA DE BASE DE DATOS

```sql
-- Tabla principal de cursos
CREATE TABLE courses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hotmart_id VARCHAR(255) UNIQUE NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    instructor VARCHAR(255),
    school VARCHAR(100),  -- programación, marketing, diseño, etc.
    level VARCHAR(50),    -- básico, intermedio, avanzado
    thumbnail_url TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    processing_status VARCHAR(50),
    metadata JSONB,
    INDEX idx_hotmart_id (hotmart_id),
    INDEX idx_school (school),
    INDEX idx_status (processing_status)
);

-- Módulos de cada curso
CREATE TABLE modules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id UUID REFERENCES courses(id) ON DELETE CASCADE,
    hotmart_module_id VARCHAR(255),
    order_number INTEGER NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB,
    INDEX idx_course_id (course_id),
    UNIQUE(course_id, order_number)
);

-- Lecciones
CREATE TABLE lessons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    module_id UUID REFERENCES modules(id) ON DELETE CASCADE,
    hotmart_lesson_id VARCHAR(255),
    order_number INTEGER NOT NULL,
    title VARCHAR(500) NOT NULL,
    description TEXT,
    duration_seconds INTEGER,
    original_url TEXT,
    storage_path TEXT,  -- Cloud Storage path
    processing_status VARCHAR(50),
    transcription JSONB,  -- {text, confidence, words:[...]}
    visual_analysis JSONB,  -- {concepts, diagrams, code, ...}
    classification JSONB,  -- {categories, keywords, level}
    openai_file_id VARCHAR(255),
    vector_store_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    processed_at TIMESTAMP,
    synced_at TIMESTAMP,
    metadata JSONB,
    INDEX idx_module_id (module_id),
    INDEX idx_status (processing_status),
    INDEX idx_openai_file (openai_file_id),
    UNIQUE(module_id, order_number)
);

-- Logs de procesamiento
CREATE TABLE processing_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    stage VARCHAR(50) NOT NULL,  -- download, transcription, analysis, sync
    status VARCHAR(50) NOT NULL,  -- started, success, failed, retrying
    message TEXT,
    error_details JSONB,
    duration_ms INTEGER,
    timestamp TIMESTAMP DEFAULT NOW(),
    metadata JSONB,
    INDEX idx_lesson_id (lesson_id),
    INDEX idx_stage (stage),
    INDEX idx_status (status),
    INDEX idx_timestamp (timestamp)
);

-- Taxonomía (escuelas y categorías)
CREATE TABLE taxonomy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(50) NOT NULL,  -- school, category, topic
    name VARCHAR(255) NOT NULL,
    description TEXT,
    parent_id UUID REFERENCES taxonomy(id),
    embedding VECTOR(1536),  -- pgvector extension
    created_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB,
    INDEX idx_type (type),
    INDEX idx_parent (parent_id)
);

-- Vector Stores de OpenAI
CREATE TABLE vector_stores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    openai_vector_store_id VARCHAR(255) UNIQUE NOT NULL,
    school VARCHAR(100),
    name VARCHAR(255),
    file_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB,
    INDEX idx_school (school)
);

-- Assistants de OpenAI
CREATE TABLE assistants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    openai_assistant_id VARCHAR(255) UNIQUE NOT NULL,
    school VARCHAR(100) UNIQUE,
    name VARCHAR(255),
    instructions TEXT,
    model VARCHAR(100),
    vector_store_ids JSONB,  -- array de IDs
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    metadata JSONB,
    INDEX idx_school (school)
);

-- Métricas agregadas
CREATE TABLE metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    metric_type VARCHAR(100) NOT NULL,
    metric_name VARCHAR(255) NOT NULL,
    value NUMERIC,
    labels JSONB,
    timestamp TIMESTAMP DEFAULT NOW(),
    INDEX idx_type_name (metric_type, metric_name),
    INDEX idx_timestamp (timestamp)
);
```

---

## 6. CONFIGURACIÓN Y SECRETS

### 6.1 Variables de Entorno

```bash
# Hotmart
HOTMART_API_KEY=xxx
HOTMART_API_SECRET=xxx
HOTMART_WEBHOOK_SECRET=xxx

# Google Cloud
GOOGLE_CLOUD_PROJECT=xxx
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
GCS_BUCKET_NAME=hotmart-etl-storage

# OpenAI
OPENAI_API_KEY=sk-xxx
OPENAI_ORGANIZATION=org-xxx

# Database
DATABASE_URL=postgresql://user:pass@host:5432/dbname
REDIS_URL=redis://host:6379/0

# Message Queue
RABBITMQ_URL=amqp://user:pass@host:5672/
# o
PUBSUB_TOPIC=hotmart-etl-events

# Observability
SENTRY_DSN=https://xxx@sentry.io/xxx
PROMETHEUS_PORT=9090

# Notifications
SLACK_WEBHOOK_URL=https://hooks.slack.com/xxx
EMAIL_API_KEY=xxx

# Feature Flags
ENABLE_MULTIMODAL_ANALYSIS=true
ENABLE_AUTO_SYNC=true
MAX_CONCURRENT_DOWNLOADS=5
MAX_CONCURRENT_TRANSCRIPTIONS=3
```

---

## 7. DEPLOYMENT

### 7.1 Docker Compose (Desarrollo Local)

```yaml
version: '3.8'

services:
  postgres:
    image: pgvector/pgvector:pg15
    environment:
      POSTGRES_DB: hotmart_etl
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: dev_password

  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=dev_password
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=hotmart_etl_n8n
      - DB_POSTGRESDB_USER=admin
      - DB_POSTGRESDB_PASSWORD=dev_password
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  detector:
    build:
      context: ./services/detector
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://admin:dev_password@postgres:5432/hotmart_etl
      - RABBITMQ_URL=amqp://admin:dev_password@rabbitmq:5672/
    depends_on:
      - postgres
      - rabbitmq

  downloader:
    build:
      context: ./services/downloader
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://admin:dev_password@postgres:5432/hotmart_etl
      - RABBITMQ_URL=amqp://admin:dev_password@rabbitmq:5672/
    volumes:
      - /tmp/downloads:/tmp/downloads
    depends_on:
      - postgres
      - rabbitmq

  transcription-worker:
    build:
      context: ./services/transcription
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://admin:dev_password@postgres:5432/hotmart_etl
      - RABBITMQ_URL=amqp://admin:dev_password@rabbitmq:5672/
    deploy:
      replicas: 2
    depends_on:
      - postgres
      - rabbitmq

  analyzer-worker:
    build:
      context: ./services/analyzer
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://admin:dev_password@postgres:5432/hotmart_etl
      - RABBITMQ_URL=amqp://admin:dev_password@rabbitmq:5672/
    depends_on:
      - postgres
      - rabbitmq

  classifier:
    build:
      context: ./services/classifier
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://admin:dev_password@postgres:5432/hotmart_etl
      - RABBITMQ_URL=amqp://admin:dev_password@rabbitmq:5672/
    depends_on:
      - postgres
      - rabbitmq

  sync-service:
    build:
      context: ./services/sync
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://admin:dev_password@postgres:5432/hotmart_etl
      - RABBITMQ_URL=amqp://admin:dev_password@rabbitmq:5672/
    depends_on:
      - postgres
      - rabbitmq

  api:
    build:
      context: ./services/api
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://admin:dev_password@postgres:5432/hotmart_etl
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
  n8n_data:
```

### 7.2 Kubernetes (Producción)

```yaml
# deployment.yaml (ejemplo para un servicio)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: transcription-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: transcription-worker
  template:
    metadata:
      labels:
        app: transcription-worker
    spec:
      containers:
      - name: worker
        image: gcr.io/project/transcription-worker:latest
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /secrets/gcp/service-account.json
        volumeMounts:
        - name: gcp-credentials
          mountPath: /secrets/gcp
          readOnly: true
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
      volumes:
      - name: gcp-credentials
        secret:
          secretName: gcp-service-account

---
# HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: transcription-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: transcription-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 8. MONITOREO Y ALERTAS

### 8.1 Métricas Clave

```python
# Prometheus metrics
from prometheus_client import Counter, Histogram, Gauge

# Contadores
courses_detected = Counter(
    'courses_detected_total',
    'Total de cursos detectados',
    ['school']
)

videos_downloaded = Counter(
    'videos_downloaded_total',
    'Total de videos descargados',
    ['status']  # success, failed
)

transcriptions_completed = Counter(
    'transcriptions_completed_total',
    'Total de transcripciones completadas',
    ['language']
)

# Histogramas (latencia)
transcription_duration = Histogram(
    'transcription_duration_seconds',
    'Duración de transcripciones',
    buckets=[30, 60, 120, 300, 600, 1800, 3600]
)

download_duration = Histogram(
    'download_duration_seconds',
    'Duración de descargas',
    buckets=[10, 30, 60, 120, 300, 600]
)

# Gauges (estado actual)
active_downloads = Gauge(
    'active_downloads',
    'Número de descargas activas'
)

queue_depth = Gauge(
    'queue_depth',
    'Profundidad de cola',
    ['queue_name']
)
```

### 8.2 Alertas

```yaml
# Prometheus alerts
groups:
- name: hotmart_etl
  interval: 30s
  rules:

  # Alta tasa de errores
  - alert: HighErrorRate
    expr: rate(videos_downloaded_total{status="failed"}[5m]) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Alta tasa de errores en descargas"
      description: "{{ $value }} descargas fallando por minuto"

  # Cola creciendo
  - alert: QueueBacklog
    expr: queue_depth > 100
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Cola de procesamiento creciendo"
      description: "{{ $labels.queue_name }} tiene {{ $value }} mensajes"

  # Servicio caído
  - alert: ServiceDown
    expr: up{job="transcription-worker"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Servicio de transcripción caído"

  # Procesamiento lento
  - alert: SlowProcessing
    expr: |
      histogram_quantile(0.95,
        rate(transcription_duration_seconds_bucket[10m])
      ) > 1800
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Procesamiento de transcripciones muy lento"
```

---

## 9. COSTOS ESTIMADOS (Mensual)

### Procesamiento de 100 horas de video/mes

| Servicio | Uso | Costo Mensual |
|----------|-----|---------------|
| **Google Cloud** | | |
| Speech-to-Text | 100 horas | $144 (a $1.44/hora) |
| Video Intelligence | 100 horas | $120 |
| Gemini API | 1000 requests | $50 estimado |
| Cloud Storage | 500GB | $10 |
| Cloud SQL (PostgreSQL) | db-n1-standard-2 | $100 |
| **OpenAI** | | |
| Embeddings API | 10M tokens | $13 |
| Vector Stores | 10GB storage | $10 |
| **Infraestructura** | | |
| Cloud Run / K8s | 5 servicios | $150 |
| Pub/Sub | 10M mensajes | $5 |
| **n8n** | | |
| Cloud (Pro) | 1 instancia | $50 |
| **Monitoreo** | | |
| Cloud Logging | 50GB | $25 |
| **TOTAL** | | **~$677/mes** |

### Optimizaciones para reducir costos:
1. Usar modelos standard en vez de enhanced cuando sea posible
2. Cache agresivo de transcripciones repetidas
3. Batch processing para aprovechar descuentos
4. Auto-scaling para apagar servicios idle
5. Lifecycle policies para archivos temporales

---

## 10. ROADMAP DE IMPLEMENTACIÓN

### Semana 1-2: Fundación
- [ ] Setup de infraestructura base (DB, queues, storage)
- [ ] Configuración de n8n
- [ ] Setup de observabilidad (logs, metrics)
- [ ] CI/CD pipeline

### Semana 3-4: Ingestion
- [ ] Detector Service
- [ ] Downloader Service
- [ ] Integración con Hotmart API
- [ ] Tests de descarga

### Semana 5-6: Processing
- [ ] Transcription Worker
- [ ] Multimodal Analyzer
- [ ] Procesamiento paralelo
- [ ] Tests de calidad

### Semana 7-8: Enrichment
- [ ] Classifier Service
- [ ] Metadata Generator
- [ ] Taxonomía inicial
- [ ] Tests de clasificación

### Semana 9-10: Synchronization
- [ ] OpenAI Sync Service
- [ ] Chunking inteligente
- [ ] Vector Store management
- [ ] Tests end-to-end

### Semana 11-12: Polish
- [ ] Dashboard administrativo
- [ ] Notificaciones
- [ ] Documentación
- [ ] Performance tuning

### Semana 13-14: Testing & Launch
- [ ] Load testing
- [ ] Security audit
- [ ] Dry-run con datos reales
- [ ] Launch to production

### Semana 15-16: Monitoreo & Optimización
- [ ] Monitoreo de producción
- [ ] Ajustes de performance
- [ ] Optimización de costos
- [ ] Documentación final
