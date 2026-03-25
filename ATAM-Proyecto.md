# ATAM - Architectural Tradeoff Analysis Method  
## Plataforma de Aprendizaje Adaptativo y Colaborativo

**Stakeholders Participantes:**
- Product Manager / Dueño del Negocio
- Arquitecto de Software
- Tech Lead
- Representante de Operaciones
- Representante de Usuarios (club deportivo)

**Fecha de Evaluación:** [26/03/22026]  

# TABLA DE CONTENIDOS

1. INTRODUCCIÓN  
2. CONTEXTO Y DRIVERS DEL NEGOCIO  
3. PRESENTACIÓN DE LA ARQUITECTURA  
4. ENFOQUES ARQUITECTURALES  
5. ÁRBOL DE UTILIDAD  
6. PASO 6: ANÁLISIS DE ESCENARIOS  
7. PASO 7: BRAINSTORMING Y RE-ANÁLISIS  
8. PASO 9: RESULTADOS Y HALLAZGOS  
9. PLAN DE ACCIÓN  
10. CONCLUSIONES  
11. ANEXOS  

---

# 1. INTRODUCCIÓN

## 1.1 Propósito

Este documento presenta la evaluación arquitectónica del sistema Plataforma de Aprendizaje Adaptativo y Colaborativo utilizando el método ATAM.

El objetivo es analizar cómo la arquitectura propuesta satisface los atributos de calidad definidos en el SRS y el SAD, identificando:
---

## 1.2 Hallazgos Principales

### Fortalezas Identificadas

- Arquitectura basada en servicios que favorece la escalabilidad y mantenibilidad  
- Separación clara de responsabilidades (módulos: usuarios, cursos, evaluaciones, adaptación, colaboración)  
- Uso de caché (Redis) que mejora el rendimiento en operaciones frecuentes  
- Procesamiento asíncrono con RabbitMQ para reducir latencia percibida  
- Uso de estándares modernos (REST, JWT, OAuth 2.0) que favorecen interoperabilidad  
- Diseño orientado a eventos para el motor de adaptación  

---

### Riesgos Críticos Identificados

- **Base de datos compartida:** Puede generar alto acoplamiento entre servicios y dificultar la escalabilidad independiente. **Nivel:** Alto  
- **Motor de adaptación:** Puede convertirse en un cuello de botella bajo alta carga concurrente. **Nivel:** Alto  
- **Dependencia de servicios externos:** Fallos en OAuth o en el servicio de email pueden afectar la disponibilidad. **Nivel:** Medio  
- **Complejidad tecnológica:** El uso de múltiples tecnologías como Redis, RabbitMQ y ECS aumenta la dificultad operativa. **Nivel:** Medio  
- **Rendimiento bajo alta carga:** Puede haber degradación si no se gestionan correctamente los picos de usuarios. **Nivel:** Alto  

---

### Trade-offs Aceptados

- **Simplicidad vs Escalabilidad extrema:** Se adopta arquitectura basada en servicios en lugar de microservicios completos  
- **Consistencia fuerte vs rendimiento:** Uso de PostgreSQL ACID en lugar de bases NoSQL  
- **Complejidad vs rendimiento:** Se introduce caché y mensajería asíncrona  
- **Acoplamiento vs rapidez de desarrollo:** Base de datos compartida para acelerar el desarrollo del MVP  
- **Costo vs disponibilidad:** Uso de infraestructura cloud gestionada (AWS)  

---

# 2. CONTEXTO Y DRIVERS DEL NEGOCIO

## 2.1 Contexto del Negocio

La **Plataforma de Aprendizaje Adaptativo y Colaborativo** surge como respuesta a las limitaciones de los sistemas educativos tradicionales.

### Situación actual

- Las plataformas LMS tradicionales no personalizan el aprendizaje  
- Todos los estudiantes reciben el mismo contenido sin considerar su desempeño  
- Existe poca interacción colaborativa entre estudiantes  
- Los profesores carecen de herramientas analíticas avanzadas  

---

### Necesidades del negocio

- Personalización del aprendizaje según desempeño individual  
- Soporte para miles de usuarios concurrentes  
- Alta disponibilidad (24/7)  
- Herramientas de analítica para profesores  
- Fomento del aprendizaje colaborativo  

---

### Objetivo del sistema

Construir una plataforma que:

- Adapte dinámicamente el contenido educativo  
- Permita colaboración entre estudiantes  
- Provea analítica en tiempo real  
- Sea escalable, segura y mantenible  

---

## 2.2 Drivers Arquitecturales (ASRs)

### DR-01: Rendimiento

- **Descripción:** El sistema debe responder rápidamente a las interacciones del usuario  
- **Métrica:**  
  - Lecturas ≤ 1 segundo (P95)  
  - Escrituras ≤ 2 segundos (P95)  
- **Prioridad:** Alta  
- **Justificación:** Experiencia de usuario crítica en plataformas educativas  

---

### DR-02: Escalabilidad

- **Descripción:** El sistema debe soportar crecimiento en usuarios y carga  
- **Métrica:**  
  - Hasta 10,000 usuarios concurrentes  
  - Escalamiento horizontal  
- **Prioridad:** Alta  
- **Justificación:** El sistema debe soportar uso masivo institucional  

---

### DR-03: Disponibilidad

- **Descripción:** El sistema debe estar disponible de forma continua  
- **Métrica:**  
  - ≥ 99.5% uptime mensual  
  - Recuperación ≤ 30 segundos  
- **Prioridad:** Alta  
- **Justificación:** Plataforma educativa accesible en todo momento  

---

### DR-04: Seguridad

- **Descripción:** Protección de datos y control de acceso  
- **Métrica:**  
  - HTTPS obligatorio  
  - JWT + RBAC  
  - Auditoría de accesos  
- **Prioridad:** Crítica  
- **Justificación:** Manejo de datos personales y académicos  

---

### DR-05: Mantenibilidad

- **Descripción:** Facilidad para evolucionar el sistema  
- **Métrica:**  
  - Cambios en ≤ 2 módulos  
  - Cobertura ≥ 70%  
- **Prioridad:** Alta  
- **Justificación:** Proyecto en evolución constante  

---

### DR-06: Adaptabilidad

- **Descripción:** Capacidad de personalizar el aprendizaje  
- **Métrica:**  
  - Reglas aplicadas en < 2 segundos  
- **Prioridad:** Alta  
- **Justificación:** Es el núcleo del sistema (diferenciador principal)  
---

# 3. PRESENTACIÓN DE LA ARQUITECTURA

## 3.1 Decisión Arquitectural Principal

La arquitectura del sistema adopta un enfoque de **Service-Based Architecture (SBA)**.

Esta decisión implica:

- Separación del sistema en servicios por dominio  
- Comunicación mediante APIs REST  
- Uso de procesamiento asíncrono para tareas no críticas  
- Base de datos compartida para el MVP  

**Justificación:**

- Permite balancear simplicidad y escalabilidad  
- Reduce la complejidad frente a microservicios puros  
- Facilita el desarrollo por un equipo pequeño  
- Permite evolución progresiva hacia una arquitectura distribuida  

---

## 3.2 Servicios Identificados

### Servicio de Autenticación

- **Responsabilidad:** Gestionar autenticación y autorización de usuarios  
- **Funciones clave:**
  - Registro de usuarios  
  - Login (JWT y OAuth 2.0)  
  - Gestión de sesiones  
  - Recuperación de contraseña  
- **Tecnología:** Node.js + Express + TypeScript  
- **Base de datos:** PostgreSQL  

---

### Servicio de Usuarios

- **Responsabilidad:** Gestión de perfiles y roles  
- **Funciones clave:**
  - CRUD de usuarios  
  - Asignación de roles (RBAC)  
  - Gestión de permisos  
- **Tecnología:** Node.js + Express  
- **Base de datos:** PostgreSQL  

---

### Servicio de Cursos

- **Responsabilidad:** Gestión de cursos y contenidos educativos  
- **Funciones clave:**
  - Creación y edición de cursos  
  - Gestión de módulos y materiales  
  - Control de prerrequisitos  
- **Tecnología:** Node.js + Express  
- **Base de datos:** PostgreSQL  

---

### Servicio de Evaluaciones

- **Responsabilidad:** Gestión de evaluaciones y resultados  
- **Funciones clave:**
  - Creación de evaluaciones  
  - Calificación automática  
  - Almacenamiento de resultados  
- **Tecnología:** Node.js + Express  
- **Base de datos:** PostgreSQL  

---

### Servicio de Adaptación

- **Responsabilidad:** Personalizar el aprendizaje del estudiante  
- **Funciones clave:**
  - Análisis de desempeño  
  - Aplicación de reglas de adaptación  
  - Generación de recomendaciones  
- **Tecnología:** Node.js + lógica basada en reglas  
- **Base de datos:** PostgreSQL  

---

### Servicio de Colaboración

- **Responsabilidad:** Facilitar interacción entre estudiantes  
- **Funciones clave:**
  - Foros  
  - Grupos de estudio  
  - Tutorías entre pares  
- **Tecnología:** Node.js + Express  
- **Base de datos:** PostgreSQL  

---

### Servicio de Analítica

- **Responsabilidad:** Generar métricas e insights para profesores  
- **Funciones clave:**
  - Reportes de desempeño  
  - Visualización de métricas  
  - Exportación de datos  
- **Tecnología:** Python + FastAPI  
- **Base de datos:** PostgreSQL  

---

## 3.3 Componentes Adicionales

- **Frontend Web:**  
  Aplicación en React que permite la interacción con el sistema  

- **API Gateway (ALB):**  
  Punto de entrada único para las solicitudes  

- **Cache (Redis):**  
  Almacenamiento temporal para mejorar el rendimiento  

- **Message Broker (RabbitMQ):**  
  Procesamiento asíncrono de eventos (notificaciones, adaptación)  

- **Sistema de almacenamiento (S3):**  
  Archivos multimedia (videos, PDFs)  

- **Servicio de Email:**  
  Notificaciones a usuarios  

---

## 3.4 Infraestructura

| Componente | Servicio AWS | Configuración |
|-----------|-------------|--------------|
| Frontend | S3 + CloudFront | Hosting estático + CDN |
| Backend (Servicios) | ECS Fargate | Contenedores auto-escalables |
| Load Balancer | Application Load Balancer | Distribución de tráfico |
| Base de datos | RDS PostgreSQL | Multi-AZ, 100GB almacenamiento |
| Cache | ElastiCache Redis | Cluster en memoria |
| Message Broker | EC2 (RabbitMQ) | Instancia t3.small |
| Archivos | S3 | Almacenamiento escalable |
| Monitoreo | CloudWatch | Logs y métricas |
| DNS | Route 53 | Gestión de dominio |
| Secretos | Secrets Manager | Gestión de credenciales |

---

## 3.5 Decisiones Arquitecturales Clave (ADRs)

### ADR-001: Uso de Service-Based Architecture

- **Decisión:** Adoptar arquitectura basada en servicios  
- **Alternativas:**
  - Monolito  
  - Microservicios  
- **Trade-off:**
  - Menor complejidad que microservicios  
  - Menor independencia que microservicios  
- **Estado:** Aceptado  

---

### ADR-002: Base de datos PostgreSQL compartida

- **Decisión:** Usar una base de datos relacional única  
- **Alternativas:**
  - Base de datos por servicio  
  - NoSQL  
- **Trade-off:**
  - Mayor simplicidad  
  - Mayor acoplamiento  
- **Estado:** Aceptado  

---

### ADR-003: Uso de Redis como caché

- **Decisión:** Implementar caché en memoria  
- **Alternativas:**
  - Sin caché  
  - Caché en aplicación  
- **Trade-off:**
  - Mejora rendimiento  
  - Añade complejidad  
- **Estado:** Aceptado  

---

### ADR-004: Uso de RabbitMQ para procesamiento asíncrono

- **Decisión:** Implementar mensajería asíncrona  
- **Alternativas:**
  - Procesamiento síncrono  
  - AWS SQS  
- **Trade-off:**
  - Mejora rendimiento y resiliencia  
  - Incrementa complejidad operativa  
- **Estado:** Aceptado  

---

### ADR-005: Autenticación basada en JWT

- **Decisión:** Usar JWT para autenticación  
- **Alternativas:**
  - Sesiones tradicionales  
  - OAuth únicamente  
- **Trade-off:**
  - Escalabilidad (stateless)  
  - Mayor responsabilidad en manejo de seguridad  
- **Estado:** Aceptado  

 # 4. ENFOQUES ARQUITECTURALES

En esta sección se analizan los enfoques arquitecturales adoptados para satisfacer los principales drivers del sistema.  
Para cada atributo de calidad, se identifican decisiones arquitecturales y su contribución.

---

### 4.1 Enfoque para Performance (DR-01)

| Decisión Arquitectural | Cómo Contribuye | Atributo Mejorado |
|------------------------|-----------------|-------------------|
| **Redis Cache** | Reduce acceso a base de datos en consultas frecuentes (cursos, progreso) | Performance |
| **CDN (CloudFront)** | Disminuye latencia en carga de contenido estático (videos, PDFs, UI) | Performance |
| **Database Indexing** | Optimiza consultas sobre progreso, usuarios y cursos | Performance |
| **RabbitMQ (Async Processing)** | Desacopla tareas pesadas como notificaciones y análisis | Performance |
| **Connection Pooling** | Reduce overhead en conexiones a base de datos | Performance |

---

### 4.2 Enfoque para Escalabilidad (DR-02)

| Decisión Arquitectural | Cómo Contribuye | Atributo Mejorado |
|------------------------|-----------------|-------------------|
| **ECS Fargate Auto-scaling** | Escala automáticamente instancias según carga | Scalability |
| **Arquitectura basada en servicios** | Permite escalar servicios de forma independiente | Scalability |
| **Stateless Services (JWT)** | Facilita replicación horizontal sin dependencia de sesiones | Scalability |
| **Separación Frontend/Backend** | Permite escalar capas de forma independiente | Scalability |
| **Uso de CDN** | Reduce carga en backend distribuyendo contenido | Scalability, Performance |

---

### 4.3 Enfoque para Disponibilidad (DR-03)

| Decisión Arquitectural | Cómo Contribuye | Atributo Mejorado |
|------------------------|-----------------|-------------------|
| **RDS Multi-AZ** | Replica base de datos y permite failover automático | Availability |
| **Health Checks (ECS + ALB)** | Detecta fallos y reinicia instancias automáticamente | Availability |
| **Rolling Updates** | Permite despliegues sin downtime | Availability |
| **Retry Mechanisms** | Reintentos automáticos ante fallos temporales | Availability |
| **Desacoplamiento con RabbitMQ** | Evita fallos en cascada en servicios críticos | Availability |

---

### 4.4 Enfoque para Seguridad (DR-04)

| Decisión Arquitectural | Cómo Contribuye | Atributo Mejorado |
|------------------------|-----------------|-------------------|
| **JWT Authentication** | Permite autenticación segura y stateless | Security |
| **RBAC (Control por roles)** | Restringe acceso según tipo de usuario | Security |
| **HTTPS/TLS** | Protege datos en tránsito | Security |
| **Hashing (bcrypt)** | Protege contraseñas almacenadas | Security |
| **Rate Limiting** | Previene ataques de fuerza bruta | Security |
| **Auditoría de accesos** | Permite trazabilidad de acciones sensibles | Security |

---

### 4.5 Enfoque para Mantenibilidad (DR-05)

| Decisión Arquitectural | Cómo Contribuye | Atributo Mejorado |
|------------------------|-----------------|-------------------|
| **Separación por dominios (servicios)** | Reduce acoplamiento y facilita cambios | Maintainability |
| **TypeScript** | Mejora calidad del código y reduce errores | Maintainability |
| **Testing automatizado** | Detecta errores antes de producción | Maintainability |
| **CI/CD (GitHub Actions)** | Automatiza despliegues y reduce errores humanos | Maintainability |
| **Documentación API (OpenAPI)** | Facilita comprensión del sistema | Maintainability |


# 5. ÁRBOL DE UTILIDAD

├── PERFORMANCE (Alta)
│   ├── Consulta de cursos < 1s (P95) [H, M]
│   ├── Envío de evaluaciones < 2s (P95) [H, M]
│   └── Aplicación de reglas de adaptación < 2s [H, H]

├── AVAILABILITY (Alta)
│   ├── Uptime ≥ 99.5% mensual [H, H]
│   ├── Recuperación ante fallos ≤ 30s [H, H]
│   └── Despliegues sin downtime [M, M]

├── SCALABILITY (Alta)
│   ├── Soporte de 10,000 usuarios concurrentes [H, H]
│   ├── Escalado horizontal automático [H, M]
│   └── Degradación < 20% al duplicar carga [H, H]

├── SECURITY (Crítica)
│   ├── Autenticación segura (JWT/OAuth) [H, M]
│   ├── Protección de datos (TLS + hashing) [H, M]
│   └── Control de acceso RBAC [H, M]

├── MAINTAINABILITY (Alta)
│   ├── Nuevas funcionalidades en ≤ 2 módulos [H, M]
│   ├── Cobertura de pruebas ≥ 70% [M, M]
│   └── Despliegue automatizado CI/CD [H, M]

## Leyenda

- **H (High):** Alto  
- **M (Medium):** Medio  
- **L (Low):** Bajo  

## 5.2 Escenarios Priorizados para Análisis

Basados en los atributos de calidad definidos en el SRS, se seleccionaron **8 escenarios críticos** para análisis profundo:

1. **[PERF-01]** Consulta de cursos y progreso < 1s con 100 usuarios concurrentes (H, M)  
2. **[PERF-02]** Procesamiento de evaluaciones y adaptación < 2s (P95) (H, H)  
3. **[AVAIL-01]** Disponibilidad ≥ 99.5% mensual (H, M)  
4. **[AVAIL-02]** Recuperación ante fallo ≤ 30 segundos (H, H)  
5. **[SCAL-01]** Soportar 10.000 usuarios concurrentes (H, H)  
6. **[SEC-01]** Protección de datos académicos (H, L)  
7. **[MOD-01]** Agregar funcionalidad sin afectar más de 2 módulos (M, M)  
8. **[ADAPT-01]** Adaptación del aprendizaje < 2s (H, H)  

---

## 6. ANÁLISIS DE ESCENARIOS

---

### 6.1 Escenario PERF-01: Consulta de cursos y progreso < 1s

#### Escenario Completo

> Cuando 100 estudiantes concurrentes consultan cursos y progreso, el sistema debe responder en menos de 1 segundo (P95).

---

#### Decisiones Arquitecturales Relevantes

| Decisión | Cómo Ayuda | Evidencia |
|----------|-----------|-----------|
| Redis Cache | Reduce consultas a BD | Latencia < 10ms |
| CDN (CloudFront) | Reduce carga de frontend | 20–50ms |
| Database Indexing | Optimiza queries | 500ms → 150ms |
| Load Balancer | Distribuye carga | Alta disponibilidad |

---

#### Análisis de Cumplimiento
Request path:

1. CDN: 50ms
2. ALB: 10ms
3. Backend: 20ms
4. Cache: 10ms

Cache HIT:
Total ≈ 90ms

Cache MISS:

DB: 150ms
Total ≈ 240ms

P95 ≈ 150ms


**Conclusión:** ✅ Cumple (< 1s)

---

#### Puntos de Sensibilidad

- PS-01: Redis Cache (ALTA)  
- PS-02: Índices en DB (MEDIA)  

---

#### Trade-offs

**TO-01: Cache vs Consistencia**
- Mejora: Performance  
- Empeora: Consistencia  
- Mitigación: invalidación  
- Estado: ACEPTADO  

---

### 6.2 Escenario PERF-02: Procesamiento de evaluaciones y adaptación < 2s

#### Escenario Completo

> Cuando un estudiante envía una evaluación, el sistema debe calificarla, almacenar el resultado y ejecutar el motor de adaptación en menos de 2 segundos (P95), actualizando el progreso y recomendando contenido.

---

#### Decisiones Arquitecturales Relevantes

| Decisión | Cómo Ayuda | Evidencia |
|----------|-----------|-----------|
| Motor de Adaptación dedicado | Aísla lógica compleja del sistema | Reduce acoplamiento |
| Procesamiento síncrono optimizado | Respuesta inmediata al usuario | Mejora UX |
| Redis Cache (reglas) | Evita recomputar reglas frecuentemente | Reducción de latencia |
| Indexing en evaluaciones | Acelera escritura y lectura | Queries eficientes |

---

#### Análisis de Cumplimiento

Flujo:

Envío evaluación: 100ms
Calificación automática: 500ms
Escritura en DB: 150ms
Motor de adaptación: 800ms
Actualización de progreso: 100ms

Total ≈ 1650ms
P95 ≈ 1.8s


**Conclusión:** ✅ CUMPLE (< 2s)

---

#### Puntos de Sensibilidad

**PS-03: Complejidad del Motor de Adaptación**
- Criticidad: ALTA  
- Más reglas → mayor latencia  

**PS-04: Escrituras concurrentes en DB**
- Criticidad: MEDIA  
- Puede generar cuello de botella  

---

#### Trade-offs Identificados

**TO-02: Procesamiento síncrono vs asincrónico**
- Mejora: Experiencia inmediata del usuario  
- Empeora: Uso de recursos en picos  
- Alternativa: procesamiento async con notificación  
- Estado: ✅ ACEPTADO  

---

#### Riesgos Identificados

**R-02: Sobrecarga del motor de adaptación**
- Probabilidad: Media  
- Impacto: Alto  
- Mitigación:
  - Optimización de reglas
  - Posible async en futuro

---

### 6.3 Escenario AVAIL-01: Disponibilidad ≥ 99.5%

#### Escenario Completo

> La plataforma debe estar disponible al menos el 99.5% del tiempo mensual, permitiendo acceso continuo a estudiantes y profesores.

---

#### Decisiones Arquitecturales Relevantes

| Decisión | Cómo Ayuda | Evidencia |
|----------|-----------|-----------|
| RDS Multi-AZ | Failover automático | SLA AWS |
| ECS con múltiples instancias | Redundancia | Alta disponibilidad |
| Load Balancer (ALB) | Distribución de tráfico | Evita caídas |
| Health Checks | Detección rápida de fallos | Recuperación automática |

---

#### Análisis de Cumplimiento

Estimación de downtime anual:

DB failover: 5 min
Fallos servicios: 20 min
Deploys: 0 min (rolling updates)

Total ≈ 25 min/año
Uptime ≈ 99.995%


**Conclusión:** ✅ CUMPLE (>> 99.5%)

---

#### Puntos de Sensibilidad

**PS-05: Base de datos**
- Criticidad: CRÍTICA  
- Fallo → caída total  

**PS-06: Número de instancias**
- Criticidad: ALTA  
- 1 instancia → SPOF  

---

#### Trade-offs Identificados

**TO-03: Alta disponibilidad vs costo**
- Mejora: Confiabilidad  
- Empeora: Costos (infraestructura duplicada)  
- Estado: ✅ ACEPTADO  

---

#### Riesgos Identificados

**R-03: Falla catastrófica de base de datos**
- Probabilidad: Baja  
- Impacto: Muy alto  
- Mitigación:
  - Backups automáticos
  - Recovery plan

---

## 7. BRAINSTORMING Y RE-ANÁLISIS

### 7.1 Escenarios Propuestos por Stakeholders

Durante la sesión de brainstorming, se identificaron escenarios adicionales que no fueron considerados inicialmente pero que pueden impactar significativamente la arquitectura del sistema.

---

#### Escenario S-01: "Pico Masivo de Usuarios en Evaluaciones"
**Propuesto por:** Profesor / Área Académica  

> "¿Qué pasa si 500 estudiantes presentan una evaluación al mismo tiempo (por ejemplo, un parcial global)?"

**Priorización votada:** (H, H)

---

**Análisis:**

- Sistema diseñado inicialmente para ~50–100 usuarios concurrentes  
- 500 usuarios concurrentes = **5x–10x carga esperada**

**Evaluación por componente:**

- **ECS Auto-scaling:**
  - Configuración actual: 2–4 instancias
  - Capacidad estimada: ~100 req/sec
  - 500 usuarios concurrentes → posible saturación inicial

- **Base de Datos (PostgreSQL):**
  - Conexiones simultáneas limitadas (~200)
  - Alto volumen de escrituras (envío de evaluaciones)
  - Riesgo de contention en tablas de evaluaciones

- **RabbitMQ:**
  - Permite desacoplar procesamiento
  - Puede absorber picos temporalmente
  - Riesgo de backlog si el consumo es más lento que la producción

- **Redis Cache:**
  - Reduce carga en lecturas
  - No ayuda directamente en escrituras (evaluaciones)

---

**Conclusión:**
- ⚠️ Sistema puede escalar parcialmente  
- ❌ Riesgo alto de saturación en DB  
- ⚠️ Latencia puede superar 2s en picos  

---

**Mitigación recomendada:**

1. Aumentar límite de auto-scaling (hasta 6–8 instancias)
2. Implementar **connection pooling (pgBouncer)**
3. Uso más agresivo de **procesamiento asíncrono**
4. Particionar tabla de evaluaciones (futuro)

---

**Estado:** RIESGO IDENTIFICADO – requiere ajustes antes de despliegue real

---

#### Escenario S-02: "Cambio en Reglas de Adaptación"
**Propuesto por:** Product Owner  

> "¿Qué pasa si queremos cambiar completamente la lógica del motor de adaptación (por ejemplo, nuevas reglas o algoritmos)?"

**Priorización votada:** (M, M)

---

**Análisis:**

- El sistema usa un **motor de adaptación desacoplado**
- Reglas configurables por profesores (según SRS)

**Impacto del cambio:**

- Si las reglas están hardcodeadas → alto impacto
- Si están parametrizadas → bajo impacto

**Evaluación actual:**

- Arquitectura permite:
  - Separación del módulo de adaptación
  - Evolución independiente

---

**Conclusión:**
- ✅ Arquitectura SOPORTA cambios en reglas  
- ⚠️ Dependencia del diseño interno del motor  

---

**Mitigación recomendada:**

1. Externalizar reglas (configuración o base de datos)
2. Diseñar motor basado en reglas (rule engine)
3. Evitar lógica hardcodeada

---

**Estado:** ACEPTADO – sin riesgo crítico inmediato

---

#### Escenario S-03: "Fallo de Base de Datos en Horario Crítico"
**Propuesto por:** DevOps / Operaciones  

> "¿Qué pasa si la base de datos falla durante un parcial o entrega importante?"

**Priorización votada:** (H, H)

---

**Análisis:**

- Base de datos es **single point of failure lógico**
- Multi-AZ mejora disponibilidad, pero:

**Evaluación real:**

- Failover: 60–120 segundos  
- Durante ese tiempo:
  - No se pueden enviar evaluaciones  
  - Se pierde continuidad del sistema  

---

**BRECHA IDENTIFICADA:**
- El sistema **NO cumple recuperación < 30s (AVAIL-02)**

---

**Conclusión:**
- ❌ Riesgo crítico para el negocio  
- ❌ Impacto directo en experiencia académica  

---

**Mitigación recomendada:**

1. Implementar **cache temporal de respuestas (buffer)**
2. Reintentos automáticos en frontend
3. Evaluar **read replicas + failover optimizado**
4. Diseñar **modo degradado (solo lectura o cola temporal)**

---

**Estado:** RIESGO CRÍTICO IDENTIFICADO – prioridad alta

---

### 7.2 Re-priorización del Árbol de Utilidad 

| Atributo de Calidad | Sub-Atributo        | Escenario / Métrica                                         | Prioridad | Riesgo | Importancia |
|---------------------|--------------------|-------------------------------------------------------------|----------|--------|-------------|
| **PERFORMANCE**     | Latencia           | Respuesta < 2s en operaciones principales (P95)             | Alta     | Media  | ⭐ |
| **PERFORMANCE**     | Throughput         | Soportar múltiples evaluaciones concurrentes sin degradación | Alta     | Media  | ⭐ |
| **AVAILABILITY**    | Uptime             | Disponibilidad ≥ 99.5% mensual                              | Alta     | Media  | ⭐ |
| **AVAILABILITY**    | Recuperación       | Recuperación ante fallos < 30 segundos                      | Alta     | Alta   | ⭐ |
| **AVAILABILITY**    | Disaster Recovery  | Recuperación desde backup < 1 hora                          | Alta     | Alta   | ⭐ NUEVO |
| **AVAILABILITY**    | Backup Integrity   | Validación de backups semanal exitosa                       | Alta     | Media  | ⭐ NUEVO |
| **SECURITY**        | Protección de datos| Datos protegidos con TLS + hashing seguro                   | Alta     | Baja   | ⭐ |
| **SECURITY**        | Control de acceso  | RBAC aplicado correctamente en todos los módulos            | Alta     | Media  | ⭐ |
| **SCALABILITY**     | Crecimiento        | Soportar 10.000 usuarios concurrentes                       | Alta     | Alta   | ⭐ |
| **SCALABILITY**     | Escalado dinámico  | Escalar automáticamente sin cambios de código               | Media    | Media  | ⭐ NUEVO |
| **SCALABILITY**     | Carga pico         | Escalar a 500 usuarios concurrentes sin intervención manual | Media    | Media  | ⭐ NUEVO |
| **MODIFIABILITY**   | Evolución          | Agregar nuevas funcionalidades sin afectar el sistema       | Alta     | Media  | ⭐ |
| **USABILITY**       | Facilidad de uso   | Flujo de aprendizaje en ≤ 5 interacciones                   | Alta     | Media  | ⭐ |
| **INTEROPERABILITY**| Integración        | Integración con sistemas externos vía API REST / OAuth      | Media    | Media  | ⭐ |

## 8. RESULTADOS Y HALLAZGOS

### 8.1 Puntos de Sensibilidad Identificados

| ID | Decisión Arquitectural | Atributo Afectado | Impacto si Falla | Prioridad |
|----|------------------------|-------------------|------------------|-----------|
| **PS-01** | Redis Cache (TTL 30s) | Performance | Latencia aumenta de ~200ms a ~800ms; mayor carga en PostgreSQL | ALTA |
| **PS-02** | Indexación en PostgreSQL | Performance | Consultas de progreso y evaluaciones se vuelven 3-5x más lentas | ALTA |
| **PS-03** | Motor de Adaptación síncrono | Performance | Reglas tardan >2s → incumple RNF-03 | CRÍTICA |
| **PS-04** | RDS Multi-AZ | Availability | Uptime baja de 99.5% a ~98% | CRÍTICA |
| **PS-05** | Health Checks en ECS | Availability | Fallos no detectados → downtime prolongado | ALTA |
| **PS-06** | JWT como autenticación stateless | Scalability | Si falla validación → bloqueo de acceso total | ALTA |
| **PS-07** | Base de datos compartida | Modifiability | Cambios en un módulo afectan otros | ALTA |
| **PS-08** | API REST centralizada | Interoperability | Integraciones externas fallan si API cambia | MEDIA |
| **PS-09** | Frontend React (SPA) | Usability | Mala optimización → tiempos de carga altos | MEDIA |

---

### 8.2 Trade-offs Documentados

| ID | Trade-off | Gana | Pierde | Aceptable? |
|----|-----------|------|--------|------------|
| **TO-01** | Cache (Redis TTL 30s) | Performance (↓ latencia hasta 70%) | Consistency (datos ligeramente desactualizados) | ✅ SÍ |
| **TO-02** | Service-Based Architecture | Simplicidad, mantenibilidad | Escalabilidad granular menor vs microservicios | ✅ SÍ |
| **TO-03** | Base de datos compartida | Simplicidad y rapidez de desarrollo | Alto acoplamiento entre servicios | ⚠️ RIESGO aceptado |
| **TO-04** | Adaptación síncrona | Respuesta inmediata al usuario | Mayor carga y latencia en backend | ⚠️ PARCIAL |
| **TO-05** | JWT Stateless | Escalabilidad (sin sesiones) | Dificultad para invalidar tokens | ✅ SÍ |
| **TO-06** | Multi-AZ Deployment | Alta disponibilidad | Incremento de costos (~+40%) | ✅ SÍ |
| **TO-07** | APIs REST vs eventos | Simplicidad | Menor resiliencia ante picos | ⚠️ PARCIAL |

---

### 8.3 Riesgos Arquitecturales

#### Riesgos CRÍTICOS (requieren mitigación inmediata)

| ID | Riesgo | Probabilidad | Impacto | Mitigación |
|----|--------|--------------|---------|------------|
| **R-01** | Motor de adaptación no escala con alta concurrencia | Media | MUY ALTO | Desacoplar con procesamiento asíncrono (colas + workers) |
| **R-02** | Base de datos como cuello de botella | Alta | MUY ALTO | Read replicas + cache agresivo + optimización queries |
| **R-03** | Fallo en autenticación (JWT/Auth Service) | Baja | MUY ALTO | Implementar fallback + redundancia del servicio |

---

#### Riesgos ALTOS (mitigación en corto plazo)

| ID | Riesgo | Probabilidad | Impacto | Mitigación |
|----|--------|--------------|---------|------------|
| **R-04** | Alta carga en evaluaciones simultáneas | Media | Alto | Escalado horizontal + colas para procesamiento |
| **R-05** | Fallo de Redis (cache) | Media | Alto | Fallback a DB + monitoreo de cache hit rate |
| **R-06** | Cascading failure entre servicios | Media | Alto | Circuit breakers + timeouts |
| **R-07** | Falta de validación de backups | Media | ALTO | Testing automático de backups + DR plan |

---

#### Riesgos MEDIOS (monitorear)

| ID | Riesgo | Probabilidad | Impacto | Mitigación |
|----|--------|--------------|---------|------------|
| **R-08** | UI lenta en dispositivos móviles | Media | Medio | Optimización frontend + lazy loading |
| **R-09** | Crecimiento no previsto de usuarios | Media | Medio | Auto-scaling + pruebas de carga periódicas |
| **R-10** | Cambios en APIs externas (OAuth) | Baja | Medio | Abstraction layer + versionado de API |
| **R-11** | Dependencia en servicios externos (email) | Baja | Medio | Retry + fallback provider |

---

### 8.4 No-Riesgos (Decisiones Validadas)

Las siguientes decisiones fueron evaluadas y **NO representan riesgos significativos**:

| Decisión | Validación |
|----------|-----------|
| **React + SPA para frontend** | Alta adopción, buen performance con buenas prácticas |
| **Node.js + Express** | Adecuado para I/O intensivo y APIs REST |
| **PostgreSQL** | Soporta transacciones, integridad y consultas complejas |
| **RBAC (Control de roles)** | Cumple requisitos de seguridad del SRS |
| **OAuth 2.0 integración** | Estándar ampliamente soportado |
| **Uso de Redis** | Tecnología madura, alto rendimiento |
| **Despliegue en AWS (ECS + RDS)** | Infraestructura confiable y escalable |

---

### 8.5 Hallazgos Positivos

La evaluación ATAM identificó múltiples **fortalezas clave** en la arquitectura:

1. ✅ **Alineación con el SRS:**  
   Los atributos de calidad (performance, disponibilidad, seguridad, escalabilidad) están correctamente reflejados en decisiones arquitecturales.

2. ✅ **Arquitectura modular bien definida:**  
   Separación clara por dominios (usuarios, cursos, evaluaciones, adaptación, colaboración).

3. ✅ **Motor de adaptación como diferenciador:**  
   La inclusión de lógica adaptativa cumple directamente con el objetivo principal del sistema.

4. ✅ **Uso adecuado de tácticas arquitecturales:**  
   Cache, auto-scaling, Multi-AZ y RBAC están correctamente aplicados.

5. ✅ **Preparación para crecimiento:**  
   La arquitectura soporta escalabilidad horizontal sin rediseño.

6. ✅ **Seguridad integrada desde el diseño:**  
   Uso de HTTPS, JWT y RBAC cumple requisitos regulatorios (Ley 1581).

7. ✅ **Capacidad de evolución:**  
   La arquitectura permite agregar nuevas funcionalidades sin afectar módulos existentes (RNF-05).

---

## 9. PLAN DE ACCIÓN

### 9.1 Acciones Inmediatas (Sprint 1)

| ID | Acción | Responsable | Fecha Límite | Prioridad |
|----|--------|-------------|--------------|-----------|
| **A-01** | Implementar testing automatizado de backups (restore en ambiente de prueba) | DevOps | Semana 2 | CRÍTICA |
| **A-02** | Documentar Disaster Recovery Runbook (RTO/RPO claros) | Tech Lead | Semana 2 | CRÍTICA |
| **A-03** | Implementar monitoreo y alertas de fallos en base de datos (RDS) | DevOps | Semana 1 | CRÍTICA |
| **A-04** | Optimizar queries críticas e índices en PostgreSQL (progreso, evaluaciones) | Backend Dev | Semana 2 | ALTA |
| **A-05** | Implementar circuit breakers en comunicación entre servicios | Backend Dev | Semana 3 | ALTA |
| **A-06** | Configurar health checks robustos en ECS (timeouts, retries) | DevOps | Semana 1 | ALTA |
| **A-07** | Instrumentar métricas de performance (P95 latencia) | DevOps | Semana 2 | ALTA |

---

### 9.2 Acciones Corto Plazo (Sprint 2-3)

| ID | Acción | Responsable | Fecha Límite | Prioridad |
|----|--------|-------------|--------------|-----------|
| **A-08** | Implementar Redis cache en consultas frecuentes (cursos, progreso) | Backend Dev | Sprint 2 | ALTA |
| **A-09** | Implementar estrategia de pre-warm de cache al iniciar servicios | Backend Dev | Sprint 2 | MEDIA |
| **A-10** | Agregar jitter al TTL del cache para evitar cache stampede | Backend Dev | Sprint 3 | MEDIA |
| **A-11** | Implementar read replicas para PostgreSQL (lecturas intensivas) | DevOps | Sprint 2 | ALTA |
| **A-12** | Desacoplar motor de adaptación usando procesamiento asíncrono (cola) | Backend Dev | Sprint 3 | CRÍTICA |
| **A-13** | Implementar rate limiting en APIs críticas | Backend Dev | Sprint 2 | MEDIA |
| **A-14** | Validación de entrada estricta en APIs (seguridad) | Backend Dev | Sprint 3 | MEDIA |

---

### 9.3 Acciones Largo Plazo (Post-MVP)

| ID | Acción | Responsable | Timeline | Prioridad |
|----|--------|-------------|----------|-----------|
| **A-15** | Evaluar migración a arquitectura basada en eventos (event-driven) | Arquitecto | 6-12 meses | MEDIA |
| **A-16** | Evaluar separación de base de datos por servicio (eliminar acoplamiento) | Arquitecto | 12 meses | MEDIA |
| **A-17** | Implementar sistema de analítica avanzada (Big Data / BI) | Data Engineer | 12-18 meses | BAJA |
| **A-18** | Evaluar multi-region deployment para alta disponibilidad global | Arquitecto | 18 meses | BAJA |
| **A-19** | Migrar a microservicios si el equipo crece significativamente | Arquitecto | 18-24 meses | BAJA |
| **A-20** | Incorporar mecanismos avanzados de personalización (IA) | Equipo ML | 12-24 meses | BAJA |

---

### 9.4 Monitoreo Continuo

**Métricas a monitorear post-lanzamiento:**

| Métrica | Threshold de Alerta | Acción si Excede |
|---------|--------------------|------------------|
| **Latencia P95 (lectura)** | > 2s | Revisar cache, índices y carga en DB |
| **Latencia P95 (escritura)** | > 3s | Analizar queries, colas y throughput |
| **Uptime semanal** | < 99.5% | Análisis de incidentes + mejoras en HA |
| **Cache hit rate** | < 60% | Ajustar TTL, warming y claves |
| **Uso CPU ECS** | > 80% sostenido | Auto-scaling o optimización de código |
| **Conexiones DB** | > 80% del límite | Ajustar pooling o escalar DB |
| **Errores 5xx** | > 2% de requests | Debug de servicios y logs |
| **Tiempo adaptación aprendizaje** | > 2s | Optimizar motor o desacoplar |
| **Costos mensuales** | > presupuesto estimado | Right-sizing de infraestructura |

---
## 10. CONCLUSIONES

### 10.1 Recomendación Final

1. ✅ **Implementar mitigaciones críticas** (A-01 a A-07) antes del despliegue en producción, especialmente:
   - Testing automatizado de backups
   - Monitoreo de base de datos
   - Health checks y circuit breakers
2. ✅ **Validar el comportamiento del motor de adaptación** bajo carga real (riesgo crítico identificado)
3. ✅ **Monitorear métricas clave** durante las primeras 4 semanas post-lanzamiento
4. ✅ **Ejecutar pruebas de carga** para validar supuestos de concurrencia (≥ 100 usuarios concurrentes)
5. ✅ **Re-evaluar la arquitectura en 6 meses** para validar escalabilidad y evolución del sistema

---

### 10.2 Nivel de Confianza

La arquitectura propuesta tiene un **ALTO nivel de confianza (8.5/10)** para cumplir con los objetivos del sistema.

**Confianza alta en:**
- **Performance:** Uso de cache, indexación y escalado horizontal permite cumplir tiempos < 2s
- **Scalability:** Arquitectura stateless con auto-scaling permite crecimiento sin rediseño
- **Security:** Implementación de RBAC, JWT y HTTPS cumple requisitos del SRS
- **Modifiability:** Arquitectura basada en servicios facilita evolución del sistema

**Confianza media en:**
- **Availability:** Depende de correcta implementación de Multi-AZ, monitoreo y DR
- **Motor de adaptación:** Puede convertirse en cuello de botella bajo alta concurrencia
- **Base de datos compartida:** Riesgo de acoplamiento a largo plazo

---

### 10.3 Supuestos Críticos a Validar

Los siguientes supuestos fueron utilizados durante el análisis y deben validarse tras el despliegue:

| Supuesto | Cómo Validar |
|----------|--------------|
| Latencia promedio < 2s en operaciones clave | Monitoreo de métricas P95 en producción |
| Cache hit rate ≥ 60% | Métricas de Redis durante primeras semanas |
| Máximo de 100 usuarios concurrentes iniciales | Análisis de tráfico real (logs / analytics) |
| Motor de adaptación responde en < 2s | Pruebas de carga específicas al módulo |
| Uso correcto del sistema por parte de usuarios | Analítica de uso y comportamiento (UX metrics) |
| Base de datos soporta carga sin degradación | Monitoreo de conexiones, CPU y queries |

---

## ANEXOS

### ANEXO A: Participantes de la Evaluación

| Nombre | Rol | Organización                 | Participación |
|--------|-----|------------------------------|---------------|
| [Nombre] | Arquitecto Evaluador | Externa                      | Líder de evaluación |
| [Nombre] | Product Manager | Club Deportivo Universitario | Drivers de negocio |
| [Nombre] | Arquitecto de Software | Equipo CourtBooker           | Presentación de arquitectura |
| [Nombre] | Tech Lead | Equipo CourtBooker           | Análisis técnico |
| [Nombre] | DevOps Engineer | Equipo CourtBooker           | Infraestructura |
| [Nombre] | Representante de Socios | Club Deportivo Universitario | Perspectiva de usuario |

### ANEXO B: Referencias

- Software Architecture in Practice (3rd Edition) - Bass, Clements, Kazman
- ATAM Method Definition - SEI Technical Report CMU/SEI-2000-TR-004
- CourtBooker SRS v1.0
- CourtBooker ADR-001 a ADR-004
- CourtBooker SAD v1.0

### ANEXO C: Glosario

| Término | Definición |
|---------|------------|
| **ASR** | Architecturally Significant Requirement - requisito que impacta significativamente la arquitectura |
| **P95** | Percentil 95 - 95% de requests deben cumplir la métrica |
| **TTL** | Time To Live - tiempo de vida de entrada en cache |
| **Multi-AZ** | Multi Availability Zone - deployment en múltiples zonas de AWS |
| **PCI-DSS** | Payment Card Industry Data Security Standard |

---

**FIN DEL REPORTE DE EVALUACIÓN ATAM**

**Preparado por:** Equipo de Evaluación ATAM  
**Fecha:** [DD/MM/YYYY]  
**Versión:** 1.0  
**Confidencialidad:** Interno