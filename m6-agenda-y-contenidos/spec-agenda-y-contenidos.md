# Módulo: Agenda y Contenidos (M6)

> **Módulo:** M6 — Agenda y Contenidos
> **Responsable:** —
> **Versión:** 1.0 | **Estado:** Borrador 
>
> **Stack, convenciones y modelos compartidos:** ver `/project.md` — no se repiten acá.
> **Interfaces con otros módulos:** ver `/contracts.md`.

## 1. Objetivo y Contexto

### ¿Qué resuelve este módulo?
Proporciona las herramientas para la planificación del cronograma (agenda) de un evento académico y la gestión de la información de los disertantes (ponentes) y autores involucrados. Permite estructurar bloques de tiempo (charlas, talleres, recesos) y asignarles responsables.

### Lugar en el sistema
Depende directamente del **M3 (Configuración de Eventos)**, ya que no puede existir una agenda sin un `Evento` previamente creado.
Alimenta de información al **M2 (Catálogo y Descubrimiento)** para mostrar el cronograma al público, y es responsable de emitir el evento de sistema **EVT-02** hacia el **M8 (Feedback)** cuando concluye la última actividad del cronograma.

### Fuera del alcance de este módulo
- La recepción y evaluación de *papers* o resúmenes (Call for Papers). Solo gestiona actividades ya aprobadas y estructuradas.
- El control de asistencia por cada charla individual (eso corresponde al M5 — Logística).
- La creación del evento principal.

---

## 2. Historias de Usuario y Criterios de Aceptación

### AGD-HU-01 — Gestión del cronograma por el Organizador
**Como** Organizador,
**quiero** crear, editar y eliminar actividades (charlas, breaks, paneles) dentro de las fechas de un evento,
**para** armar la agenda oficial que verán los participantes.

**Criterios de aceptación:**
- [ ] Puedo definir título, tipo de actividad, hora de inicio, hora de fin y sala/ubicación.
- [ ] El sistema valida que la fecha/hora de la actividad esté estrictamente dentro de los límites de `fechaInicio` y `fechaFin` del `Evento`.
- [ ] El sistema muestra una advertencia (pero permite guardar) si hay solapamiento de horarios en la misma ubicación.

### AGD-HU-02 — Asignación de Disertantes
**Como** Organizador,
**quiero** asignar usuarios con el rol de Disertante a las actividades de la agenda,
**para** que el público sepa quién imparte cada sesión.

**Criterios de aceptación:**
- [ ] Puedo buscar usuarios registrados (por email o nombre) y agregarlos a una actividad.
- [ ] Puedo definir el rol específico del usuario en esa actividad ("TITULAR", "CO-AUTOR", "MODERADOR").
- [ ] Un disertante puede estar asignado a múltiples actividades en el mismo evento.

### AGD-HU-03 — Visualización pública de la Agenda
**Como** Participante o visitante anónimo,
**quiero** ver el cronograma completo de un evento ordenado cronológicamente,
**para** planificar mi asistencia.

**Criterios de aceptación:**
- [ ] La agenda se agrupa por día.
- [ ] Se muestra claramente el título, horario, ubicación y nombres de los disertantes de cada actividad.
- [ ] Los recesos/breaks se visualizan integrados en la línea de tiempo.

---

## 3. Requisitos Funcionales y Reglas de Negocio

### 3.1 Validaciones específicas
| Entidad | Campo | Regla de Negocio |
|---------|-------|------------------|
| Actividad | `inicio`, `fin` | `inicio` debe ser estrictamente menor a `fin`. |
| Actividad | Fechas | Deben estar comprendidas dentro de `Evento.fechaInicio` y `Evento.fechaFin`. |
| Disertante | `usuarioId` | El usuario asignado debe existir previamente en el sistema (M1). |

### 3.2 Integración Sincrónica (Contratos)
De acuerdo a `/contracts.md`, este módulo debe disparar la notificación de finalización:
- **Regla:** Cuando la fecha/hora actual supera el `fin` de la última actividad programada de un evento, el módulo (vía un job programado o hook) debe enviar un POST a `/api/feedback/eventos/habilitar-encuesta` (**EVT-02**).

---

## 4. Restricciones técnicas específicas de este módulo

- **Husos Horarios (Timezones):** Todas las horas de inicio y fin de las actividades se guardan en la base de datos en formato **UTC**. El frontend (React) es el responsable de formatear y mostrar estas fechas en la zona horaria del usuario utilizando la API nativa `Intl.DateTimeFormat` (no instalar librerías externas para esto).
- **Formatos de respuesta:** Todos los endpoints (GET, POST, PATCH, DELETE) deben envolver sus respuestas en el objeto estándar definido en `project.md` (`{ data, error, message }`).
- **Validación de payload:** Los requests para crear/editar actividades deben ser validados mediante middlewares de `Zod` antes de llegar a los controllers.
- **Rendimiento:** El endpoint que devuelve la agenda completa de un evento debe ejecutar una única consulta optimizada (con `include` en Prisma) para traer las actividades y sus disertantes, evitando el problema de consultas N+1.

---

## 5. Modelo de datos de este módulo

> Los modelos `Usuario` y `Evento` ya existen en `/prisma/schema.prisma`. Se referencian aquí pero **no se deben reescribir ni modificar**.

```prisma
model Actividad {
  id              Int                   @id @default(autoincrement())
  eventoId        Int
  evento          Evento                @relation(fields: [eventoId], references: [id], onDelete: Cascade)
  titulo          String
  descripcion     String?
  tipo            TipoActividad         @default(CHARLA)
  inicio          DateTime              // UTC
  fin             DateTime              // UTC
  ubicacion       String?               // Nombre de sala, aula, o link
  estado          EstadoActividad       @default(CONFIRMADA)
  
  disertantes     ActividadDisertante[]

  creadoEn        DateTime              @default(now())
  actualizadoEn   DateTime              @updatedAt

  @@index([eventoId, inicio])
}

model ActividadDisertante {
  actividadId     Int
  usuarioId       Int
  rol             RolDisertante         @default(TITULAR)
  
  actividad       Actividad             @relation(fields: [actividadId], references: [id], onDelete: Cascade)
  usuario         Usuario               @relation(fields: [usuarioId], references: [id], onDelete: Cascade)

  @@id([actividadId, usuarioId])
}

enum TipoActividad {
  CHARLA
  TALLER
  PANEL
  MESA_REDONDA
  BREAK
  ACREDITACION
}

enum EstadoActividad {
  BORRADOR
  CONFIRMADA
  CANCELADA
}

enum RolDisertante {
  TITULAR
  CO_AUTOR
  MODERADOR
}
```

---

## 6. Plan de Tareas

> **Instrucción para Agentes IA (Claude Code / OpenCode):**
> Ejecutá las tareas de a una por vez, en el estricto orden indicado. Al finalizar cada tarea, completá el **Reporte de cierre** y escribí la frase de pausa. No pases a la siguiente tarea sin la confirmación ("continuar", "ok") del usuario.

### T1 — Modelo de datos e Infraestructura
**Entregables:**
- Agregar los modelos `Actividad`, `ActividadDisertante` y los `enums` al `schema.prisma` global.
- Crear migración: `prisma migrate dev --name init-m6-agenda`.
- Crear el seeder en `prisma/seed.js` que inyecte 1 Evento (si no existe), 2 Usuarios, y al menos 4 Actividades simulando un día de congreso (con disertantes asignados).

**Reporte de cierre — completar antes de detenerse:**
- Migración creada: (nombre del archivo generado)
- Resultado de `prisma db seed`: (output del comando)
- Cantidad de actividades inyectadas: (número)
> ⏸ **STOP — T1 completa. Esperando confirmación para continuar con T2.**

### T2 — API REST: Gestión de Agenda
**Entregables:**
- Crear `ActividadesRouter`, `ActividadesController`, `ActividadesService` y `ActividadesRepository`.
- Endpoints CRUD bajo `/api/eventos/:eventoId/actividades`:
  - `GET /` (Lista agenda completa con disertantes, ordenada por fecha de inicio).
  - `POST /` (Crea actividad, valida fechas contra el evento padre).
  - `PATCH /:actividadId` (Actualiza datos).
  - `DELETE /:actividadId` (Elimina).
- Middleware de validación con Zod para el POST y PATCH.

**Reporte de cierre — completar antes de detenerse:**
- Rutas registradas en Express: (listar)
- Esquemas de Zod implementados: (listar campos validados)
- Tests básicos en Vitest ejecutados: (output de tests passing)
> ⏸ **STOP — T2 completa. Esperando confirmación para continuar con T3.**

### T3 — API REST: Gestión de Disertantes e Integración
**Entregables:**
- Endpoints bajo `/api/actividades/:actividadId/disertantes`:
  - `POST /` (Asigna un usuario a la actividad).
  - `DELETE /:usuarioId` (Remueve al usuario de la actividad).
- Implementar la lógica para disparar el webhook al M8 (`POST /api/feedback/eventos/habilitar-encuesta`) **EVT-02** si el evento finaliza (puede ser un endpoint utilitario `/api/agenda/check-finalizacion` por ahora).

**Reporte de cierre — completar antes de detenerse:**
- Rutas registradas: (listar)
- Implementación de EVT-02: (describir dónde se aloja la lógica)
> ⏸ **STOP — T3 completa. Esperando confirmación para continuar con T4.**

### T4 — Frontend: Vista Pública de Agenda
**Entregables:**
- Crear ruta `/eventos/:eventoId/agenda` en React Router.
- Crear `AgendaPage.jsx` para consumo de participantes.
- Componente `LineaDeTiempo.jsx` que agrupe las actividades obtenidas de la API por día y renderice tarjetas simples usando Tailwind CSS.
- Integrar la conversión de UTC a Local Time mediante `Intl.DateTimeFormat`.

**Reporte de cierre — completar antes de detenerse:**
- Componentes creados: (listar)
- Hook de conexión a API: (nombre)
> ⏸ **STOP — T4 completa. Esperando confirmación para finalizar M6.**

---

## 7. Estrategia de Verificación

Se deben realizar las siguientes pruebas (manuales o automatizadas en Vitest) para dar por válido el módulo:

1. **Test de Límites de Evento:** Intentar crear mediante la API un bloque de `Actividad` cuya fecha `inicio` caiga un día antes de la `fechaInicio` del `Evento`. La API debe rechazarlo con un código `400` y el formato de error estándar.
2. **Test de Solapamiento (Warning):** Crear dos actividades en la misma ubicación física (`ubicacion: "Sala A"`) en el mismo bloque horario. La API debe permitirlo, pero el frontend en el módulo Organizador debe mostrar un badge visual de advertencia.
3. **Test de Timezones:** Al cargar un evento a las 14:00 UTC en la base de datos, un cliente en el huso horario de Argentina (UTC-3) debe visualizar la actividad a las 11:00 AM en el navegador.
4. **Test de Estructura de Respuesta:** Verificar que el listado público de la agenda devuelve estrictamente un objeto `{"data": [...], "error": null, "message": "..."}` sin fallas de sintaxis JSON.
5. **Test de Integración (EVT-02):** Simular la conclusión de la última actividad del evento y verificar que el payload generado para el M8 coincide exactamente con el contrato estipulado en `contracts.md`.
