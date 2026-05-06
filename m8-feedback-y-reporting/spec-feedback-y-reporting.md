# Módulo: Feedback y Reporting (M8)

> **Módulo:** M8 — Feedback y Reporting
> **Responsable:** —
> **Versión:** 1.0 | **Estado:** Borrador | **Última revisión:** 2026-05
>
> **Stack, convenciones y modelos compartidos:** ver `/project.md` — no se repiten acá.
> **Interfaces con otros módulos:** ver `/contracts.md §M8`.

## 1. Objetivo y Contexto

### ¿Qué resuelve este módulo?

La recolección de opiniones y niveles de satisfacción de los participantes una vez concluido un evento académico, así como la generación de reportes estadísticos agregados para que los organizadores puedan evaluar el éxito de la convocatoria.

### Lugar en el sistema

Este módulo actúa en la etapa final del ciclo de vida de un evento. Se activa automáticamente a través de eventos sincrónicos (EVT-02) cuando el módulo de Configuración (M3) o Agenda (M6) marca un evento como `FINALIZADO`. 

### Fuera del alcance de este módulo

- Encuestas previas al evento o formularios de pre-inscripción.
- Votaciones en vivo o encuestas en tiempo real durante el transcurso del evento.
- Generación de reportes financieros o de cobros (esto pertenece a otro dominio).

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-01 — Habilitación automática de encuestas
**Como** sistema,
**quiero** habilitar el formulario de feedback cuando un evento finaliza,
**para que** los asistentes puedan evaluar su experiencia inmediatamente.

**Criterios de aceptación:**
- [ ] El módulo debe exponer el endpoint `POST /api/feedback/eventos/habilitar-encuesta` según el contrato EVT-02.
- [ ] Al recibir la notificación, el sistema registra que las encuestas para ese `eventoId` están abiertas.

### HU-02 — Envío de feedback por el participante
**Como** participante acreditado,
**quiero** completar una encuesta de satisfacción sobre un evento finalizado,
**para que** los organizadores conozcan mi opinión.

**Criterios de aceptación:**
- [ ] Solo puedo acceder a la encuesta si el evento está finalizado y la encuesta habilitada.
- [ ] El sistema debe verificar que participé del evento (llamando a la API del M5: `GET /api/acreditacion/evento/:eventoId/asistentes`).
- [ ] Puedo calificar el evento del 1 al 5 y dejar un comentario de texto libre.
- [ ] No puedo enviar más de una respuesta para el mismo evento.

### HU-03 — Visualización de reportes
**Como** organizador,
**quiero** ver las estadísticas de satisfacción de mi evento,
**para que** pueda medir la calidad de la propuesta académica.

**Criterios de aceptación:**
- [ ] Puedo ver el promedio general de calificación (estrellas).
- [ ] Puedo ver la lista de comentarios dejados por los participantes (de forma anónima o con nombre, dependiendo de la configuración, por defecto anónimo).
- [ ] Puedo ver la tasa de respuesta (cantidad de encuestas respondidas vs cantidad de asistentes acreditados).

---

## 3. Requisitos Funcionales y Reglas de Negocio

### 3.1 Validaciones específicas de este módulo

| Campo | Regla |
|-------|-------|
| Puntuación | Entero, mínimo 1, máximo 5. |
| Comentario | Opcional. Máximo 500 caracteres. |
| Unicidad | Un usuario (`usuarioId`) solo puede tener un registro de `RespuestaEncuesta` por `eventoId`. |

### 3.2 Reglas de Estado
- Las encuestas permanecen cerradas (`habilitada: false`) por defecto hasta recibir el evento EVT-02.
- Un participante que no figure en la lista de acreditados del M5 recibirá un error HTTP 403 (Forbidden) o 422 si intenta enviar una encuesta, validando contra las reglas de negocio estandarizadas.

---

## 4. Restricciones técnicas específicas de este módulo

> El stack completo está en `/project.md §2`. Acá solo van restricciones adicionales que aplican únicamente a este módulo.

- **Cálculo de promedios:** Para evitar sobrecargar la base de datos de la aplicación Node, los cálculos de promedios de calificación deben resolverse utilizando funciones de agregación del ORM (`prisma.respuestaEncuesta.aggregate({ _avg: { puntuacion: true } })`), no iterando arrays en memoria.
- **Validación Inter-módulo:** Para verificar si un usuario está acreditado, el backend del M8 debe hacer un `fetch` interno hacia `http://localhost:PORT/api/acreditacion/evento/:eventoId/asistentes` (o la URL base configurada para servicios internos) y buscar el `usuarioId` en la respuesta.
- No se instalarán librerías de gráficos complejas en el frontend en esta fase; las métricas se mostrarán mediante tarjetas de resumen (Cards) estándar con Tailwind CSS y listas simples de comentarios.

---

## 5. Modelo de datos de este módulo

> Los modelos `Usuario` y `Evento` están en `/project.md §6`. No redefinirlos acá. Agregar solo los modelos nuevos al `/prisma/schema.prisma`.

```prisma
model ConfiguracionEncuesta {
  eventoId      Int      @id // PK compartida con el ID del evento
  evento        Evento   @relation(fields: [eventoId], references: [id])
  habilitada    Boolean  @default(false)
  habilitadaEn  DateTime?
  creadoEn      DateTime @default(now())
  actualizadoEn DateTime @updatedAt
}

model RespuestaEncuesta {
  id            Int      @id @default(autoincrement())
  eventoId      Int
  evento        Evento   @relation(fields: [eventoId], references: [id])
  usuarioId     Int
  usuario       Usuario  @relation(fields: [usuarioId], references: [id])
  puntuacion    Int      // 1 a 5
  comentario    String?  @db.VarChar(500)
  creadoEn      DateTime @default(now())

  @@unique([eventoId, usuarioId]) // Garantiza una respuesta por participante por evento
  @@index([eventoId])
}
```

---

## 6. Plan de Tareas

> **Instrucción para Claude Code / OpenCode (leer antes de empezar):**
> Implementá las tareas de a una por vez, en el orden indicado.
> Al finalizar cada tarea, completá el **Reporte de cierre** que figura al final de ella y escribí la frase de pausa exacta. No avances a la siguiente tarea hasta recibir una confirmación explícita del usuario. Si el usuario escribe "continuar", "ok", "aprobado" o similar, recién entonces pasás a la tarea siguiente.

### T1 — Modelo de datos y seed

**Entregables:**
- Modelos `ConfiguracionEncuesta` y `RespuestaEncuesta` agregados al `schema.prisma`.
- Migración ejecutada: `prisma migrate dev --name add-feedback-reporting`.
- Seed con: 1 evento finalizado en `ConfiguracionEncuesta` (habilitado) y al menos 5 respuestas simuladas asociadas a ese evento y distintos usuarios existentes.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados: (listar)
- Resultado de `prisma migrate dev`: (pegar output)
- Resultado de `prisma db seed`: (pegar output)
> ⏸ **STOP — T1 completa. Esperando confirmación para continuar con T2.**

### T2 — API: Endpoint de integración EVT-02

**Entregables:**
- Controlador, ruta y esquema Zod para `POST /api/feedback/eventos/habilitar-encuesta`.
- Debe crear o actualizar la fila en `ConfiguracionEncuesta` marcando `habilitada: true`.
- Respuestas HTTP usando el formato estándar `{ data, error, message }`.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados/modificados: (listar)
- Ejemplo de request probada exitosamente: (pegar JSON)
> ⏸ **STOP — T2 completa. Esperando confirmación para continuar con T3.**

### T3 — API: Endpoints de Encuestas y Reportes

**Entregables:**
- `POST /api/feedback/eventos/:eventoId/respuestas`: Endpoint para guardar la encuesta. Debe validar payload con Zod (puntuación 1-5). Simular validación de asistencia asumiendo éxito por ahora (o integrando mock fetch al M5). Manejar error 409 si ya existe respuesta.
- `GET /api/feedback/eventos/:eventoId/reporte`: Endpoint que devuelva el promedio de puntuación, total de respuestas y lista de comentarios usando `prisma.aggregate`.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados/modificados: (listar)
- Formato de respuesta del GET de reporte: (pegar JSON)
> ⏸ **STOP — T3 completa. Esperando confirmación para continuar con T4.**

### T4 — Frontend: Vista de Participante y Organizador

**Entregables:**
- Componente/Página `FormularioEncuesta.jsx` que permita enviar estrellas (1-5) y comentario usando `fetch` nativo.
- Componente/Página `DashboardReporte.jsx` para el Organizador que consuma el endpoint de reportes y muestre las tarjetas resumen (Promedio, Total) usando Tailwind CSS.

**Reporte de cierre — completar antes de detenerse:**
- Componentes creados: (listar)
- Rutas agregadas a React Router: (listar)
> ⏸ **STOP — T4 completa. Módulo finalizado.**

---

## 7. Estrategia de Verificación

- [ ] Simular mediante una petición POST a `/api/feedback/eventos/habilitar-encuesta` el EVT-02 y verificar en BD que el evento queda habilitado para feedback.
- [ ] Comprobar que intentar hacer un POST de una respuesta para un evento NO habilitado retorna HTTP 422.
- [ ] Verificar la restricción de unicidad: Intentar enviar dos encuestas con el mismo `usuarioId` y `eventoId` debe devolver un error claro basado en la estructura de error JSON estándar.
- [ ] Ejecutar el endpoint de reportes y verificar de forma manual que el promedio devuelto en el JSON coincide matemáticamente con la suma de los valores en el seed dividida por la cantidad de respuestas.
