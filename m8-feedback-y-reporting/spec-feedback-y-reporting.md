# Módulo: Feedback y Reporting (M8)

> **Módulo:** M8 — Feedback y Reporting
> **Responsable:** —
> **Versión:** 1.0 | **Estado:** Borrador | **Última revisión:** 2026-05
>
> **Stack, convenciones y modelos compartidos:** ver `/project.md` — no se repiten acá.
> **Interfaces con otros módulos:** ver `/contracts.md §M8`.

---

## 1. Objetivo y Contexto

### ¿Qué resuelve este módulo?
La recolección de percepciones de los participantes una vez finalizado un evento académico y la consolidación de métricas clave para los organizadores. Permite evaluar la calidad de los disertantes, contenidos y logística.

### Lugar en el sistema
Este módulo reacciona a la finalización de eventos (notificado por M3/M6). Consume datos de participantes acreditados desde M5 para habilitar las encuestas y proporciona datos analíticos al dashboard del Organizador.

### Fuera del alcance de este módulo
- Encuestas durante el evento (en vivo).
- Envío de emails/notificaciones (se registra la intención, el envío depende de un servicio externo).
- Reportes financieros o de ventas (corresponden a M4).

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-01 — Completar encuesta de satisfacción
**Como** Participante acreditado,
**quiero** acceder a un formulario de feedback una vez finalizado el evento,
**para** calificar mi experiencia y ayudar a mejorar futuras ediciones.

**Criterios de aceptación:**
- Solo accesible si el evento tiene estado `FINALIZADO` y el usuario está en la lista de acreditados de M5.
- El formulario debe validar que el usuario no haya respondido previamente ese evento.
- Incluye valoración numérica (1-5) y campos de texto opcionales.

### HU-02 — Visualización de métricas de evento
**Como** Organizador,
**quiero** ver un resumen estadístico de las respuestas recibidas,
**para** tomar decisiones basadas en datos para el próximo evento.

**Criterios de aceptación:**
- Cálculo automático del promedio de satisfacción general (NPS simplificado).
- Listado de comentarios abiertos anonimizados (opcionalmente).
- Gráfico simple de distribución de puntajes.

---

## 3. Requisitos Funcionales y Reglas de Negocio

### 3.1 Validaciones específicas

| Campo | Regla |
|-------|-------|
| Puntaje | Entero entre 1 y 5. |
| Comentario | Opcional, máximo 500 caracteres. |
| Disponibilidad | La encuesta se habilita solo tras el evento `EVT-02`. |
| Unicidad | `(usuarioId, eventoId)` debe ser único en la tabla de respuestas. |

### 3.2 Reglas de Negocio
- **Anonimato:** Las respuestas se guardan vinculadas al `usuarioId` para control de integridad, pero el Organizador solo ve datos agregados o comentarios sin nombre.
- **Habilitación:** El módulo expone el endpoint `/api/feedback/eventos/habilitar-encuesta` definido en `contracts.md`.

---

## 4. Restricciones técnicas específicas

- **Gráficos:** No usar librerías pesadas (ej. Chart.js). Si se requiere visualización, usar barras de progreso con Tailwind CSS puro.
- **Performance:** El cálculo de promedios para el reporte no debe tardar más de 300ms. Usar agregaciones de Prisma (`_avg`, `_count`).
- **Privacidad:** Nunca exponer el `email` del participante en los endpoints de reporting.

---

## 5. Modelo de datos

> Los modelos `Usuario` y `Evento` están en `/project.md §6`.

```prisma
model EncuestaSatisfaccion {
  id              Int      @id @default(autoincrement())
  eventoId        Int
  evento          Evento   @relation(fields: [eventoId], references: [id])
  usuarioId       Int
  usuario         Usuario  @relation(fields: [usuarioId], references: [id])
  
  // Calificaciones (1-5)
  puntajeGeneral  Int
  puntajeContenido Int
  puntajeLogistica Int
  
  comentario      String?  @db.Text
  creadoEn        DateTime @default(now())

  @@unique([eventoId, usuarioId])
  @@index([eventoId])
}

model ConfiguracionFeedback {
  id              Int      @id @default(autoincrement())
  eventoId        Int      @unique
  evento          Evento   @relation(fields: [eventoId], references: [id])
  encuestaActiva  Boolean  @default(false)
  fechaHabilitacion DateTime?
}
```

---

## 6. Plan de Tareas

### T1 — Infraestructura de Datos
**Entregables:**
- Modelos `EncuestaSatisfaccion` y `ConfiguracionFeedback` en `schema.prisma`.
- Migración `prisma migrate dev --name feat-feedback-models`.
- Seed con 10 respuestas de prueba para un evento finalizado.
> ⏸ **STOP — T1 completa. Esperando confirmación.**

### T2 — API de Recepción y Lógica (Backend)
**Entregables:**
- Endpoint `POST /api/feedback/eventos/habilitar-encuesta` (Implementación de `EVT-02`).
- Endpoint `POST /api/feedback/responder` con validación Zod y check de acreditación (simulado o vía service).
- Tests en Vitest para: responder doble (error 409), puntaje fuera de rango (error 400).
> ⏸ **STOP — T2 completa. Esperando confirmación.**

### T3 — API de Reporting y Dashboard (Backend)
**Entregables:**
- Endpoint `GET /api/feedback/reporte/:eventoId` que devuelva promedios y lista de comentarios.
- Lógica de agregación usando Prisma `_avg`.
> ⏸ **STOP — T3 completa. Esperando confirmación.**

### T4 — Interfaz de Usuario (Frontend)
**Entregables:**
- Página `/eventos/:id/feedback`: Formulario de 5 estrellas con Tailwind.
- Componente `ResumenFeedback` para el organizador con barras de distribución.
- Integración con `fetch` nativo siguiendo el formato estándar de respuesta.
> ⏸ **STOP — T4 completa. Esperando confirmación.**

---

## 7. Estrategia de Verificación

- [ ] **Test de Integración:** Llamar a `EVT-02` y verificar que `ConfiguracionFeedback.encuestaActiva` pase a `true`.
- [ ] **Validación de Regla:** Intentar responder una encuesta con un `usuarioId` que no está acreditado (debe fallar según la lógica de negocio).
- [ ] **Carga:** Verificar que el reporte de un evento con 100 respuestas cargue en < 200ms.
- [ ] **UX:** El formulario debe mostrar un mensaje de "Gracias por tu participación" y bloquear re-envíos.
