# spec.md — Módulo: Configuración de Eventos (M3)

> **Módulo:** M3 — Configuración de Eventos
> **Responsable:** Equipo de Core
> **Versión:** 1.0 | **Estado:** Borrador 
>
> **Stack, convenciones y modelos compartidos:** ver `/project.md` — no se repiten acá.
> **Interfaces con otros módulos:** ver `/contracts.md §M3`.

## 1. Objetivo y Contexto

### ¿Qué resuelve este módulo?
Provee las herramientas administrativas necesarias para que los organizadores puedan crear y configurar eventos académicos desde cero. Se encarga de definir el marco del evento (fechas, tipo, cupos) y las reglas de inscripción asociadas.

### Lugar en el sistema
Es el pilar estructural de la plataforma. El ciclo de vida de cualquier actividad académica nace aquí. Este módulo alimenta el catálogo público (M2) y provee los límites temporales y de capacidad que el Motor de Inscripciones (M4) debe respetar a rajatabla.

### Fuera del alcance de este módulo
- El cobro o procesamiento de pagos para la inscripción.
- La gestión detallada de la agenda, bloques horarios específicos o asignación de salas (responsabilidad del M6 — Agenda y Contenidos).
- La gestión del check-in presencial (M5).

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-01 — Creación y configuración inicial del evento
**Como** Organizador,
**quiero** crear un nuevo evento definiendo su tipo, fechas de realización, cupos y ventana de inscripciones,
**para que** la plataforma tenga la base estructural necesaria para recibir participantes.

**Criterios de aceptación:**
- [ ] El sistema permite seleccionar el tipo de evento ("CURSO", "JORNADA", "CONGRESO", "CHARLA").
- [ ] Deben existir validaciones que aseguren que la `fechaFin` es mayor o igual a la `fechaInicio`.
- [ ] La fecha de límite de inscripción no puede ser posterior a la fecha de finalización del evento.
- [ ] El cupo máximo (si se define) debe ser obligatoriamente mayor o igual al cupo mínimo.
- [ ] Al crearse, el evento asume automáticamente el estado "BORRADOR".

### HU-02 — Publicación y finalización del evento
**Como** Organizador,
**quiero** cambiar el estado de mi evento a publicado o finalizado,
**para que** el evento aparezca en el catálogo o se disparen las acciones post-evento.

**Criterios de aceptación:**
- [ ] Un evento solo puede pasar a "PUBLICADO" si tiene definidos sus parámetros obligatorios (título, fechas, configuración de inscripción).
- [ ] Al marcar un evento como "FINALIZADO", el sistema debe notificar al módulo de Feedback (M8) siguiendo el contrato `EVT-02`.

---

## 3. Requisitos Funcionales y Reglas de Negocio

### 3.1 Validaciones específicas de este módulo

| Campo | Regla |
|-------|-------|
| Fechas de Evento | `fechaFin` >= `fechaInicio`. No se pueden crear eventos en el pasado. |
| Fechas de Inscripción | `fechaLimiteInscripcion` <= `fechaFin` del evento. `fechaInicioInscripcion` < `fechaLimiteInscripcion`. |
| Cupos | `cupoMaximo` >= `cupoMinimo` (y `cupoMinimo` >= 0). Si `cupoMaximo` es `null`, es ilimitado. |
| Roles | Solo un usuario con rol de `Organizador` asignado a este evento en particular puede modificar su configuración. |

### 3.2 Estados de un evento y transiciones válidas

```text
BORRADOR → PUBLICADO → FINALIZADO
         → CANCELADO
```
*   **BORRADOR:** Solo visible para los Organizadores asignados.
*   **PUBLICADO:** Visible en el Catálogo (M2). Las inscripciones (M4) abren según la configuración de fechas.
*   **FINALIZADO:** Dispara el evento `EVT-02` hacia el M8.
*   **CANCELADO:** Detiene todas las interacciones, oculta del catálogo.

---

## 4. Restricciones técnicas específicas de este módulo

> El stack completo está en `/project.md §2`. Acá solo van restricciones adicionales que aplican únicamente a este módulo.

- **Integración Sincrónica M8 (Feedback):** Al transicionar un evento al estado `FINALIZADO`, el servicio debe ejecutar un HTTP POST nativo (usando `fetch`) hacia `/api/feedback/eventos/habilitar-encuesta` (contrato `EVT-02`). La falla en este POST no debe revertir el cambio de estado del evento, pero debe loguearse como un error para reintentos manuales.
- **Validación de Fechas en Zod:** Las validaciones de relaciones temporales (ej: `fechaFin` > `fechaInicio`) deben resolverse usando el método `.refine()` de Zod a nivel de esquema global (body validation middleware).

---

## 5. Modelo de datos de este módulo

> Los modelos `Usuario` y `Evento` están en `/project.md §6`. No redefinirlos acá. Agregar solo los modelos nuevos al `/prisma/schema.prisma`.

```prisma
model ConfiguracionInscripcion {
  id                     Int      @id @default(autoincrement())
  eventoId               Int      @unique
  evento                 Evento   @relation(fields: [eventoId], references: [id])
  fechaInicioInscripcion DateTime
  fechaLimiteInscripcion DateTime
  requiereAprobacion     Boolean  @default(false)
  creadoEn               DateTime @default(now())
  actualizadoEn          DateTime @updatedAt
}

model EventoOrganizador {
  eventoId  Int
  evento    Evento  @relation(fields: [eventoId], references: [id])
  usuarioId Int
  usuario   Usuario @relation(fields: [usuarioId], references: [id])
  asignadoEn DateTime @default(now())

  @@id([eventoId, usuarioId])
  @@index([usuarioId])
}
```

---

## 6. Plan de Tareas

> **Instrucción para Claude Code (o OpenCode - leer antes de empezar):**
> Implementá las tareas de a una por vez, en el orden indicado.
> Al finalizar cada tarea, completá el **Reporte de cierre** que figura al final de ella y escribí la frase de pausa exacta. No avances a la siguiente tarea hasta recibir una confirmación explícita del usuario. Si el usuario escribe "continuar", "ok", "aprobado" o similar, recién entonces pasás a la tarea siguiente.

---

### T1 — Modelo de datos y Seed

**Entregables:**
- Modelos `ConfiguracionInscripcion` y `EventoOrganizador` agregados al `schema.prisma` compartido.
- Ajuste menor en el modelo `Evento` e `Usuario` para mapear las relaciones inversas (sin alterar sus campos existentes).
- Migración ejecutada: `prisma migrate dev --name add-m3-configuracion`.
- Seed con: 2 eventos "Borrador", 2 eventos "Publicado", sus respectivas configuraciones de inscripción y usuarios organizadores vinculados.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados: (listar)
- Resultado de `prisma migrate dev`: (pegar output)
- Resultado de `prisma db seed`: (pegar output)
- Qué NO se implementó en esta tarea que pertenece a tareas siguientes: (describir)
> ⏸ **STOP — T1 completa. Esperando confirmación para continuar con T2.**

---

### T2 — Endpoints y Servicios de Configuración

**Entregables:**
- `POST /api/eventos` y `PATCH /api/eventos/:id` (incluyendo la creación/actualización atómica de su `ConfiguracionInscripcion` mediante Prisma Nested Writes).
- Middlewares de validación Zod con `.refine()` para toda la lógica de validación de fechas y cupos.
- `GET /api/eventos/:id/configuracion` para que el frontend pueda cargar el formulario.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados: (listar)
- Cobertura de tests (Vitest) para casos exitosos y fallas de Zod: (describir)
> ⏸ **STOP — T2 completa. Esperando confirmación para continuar con T3.**

---

### T3 — Gestión de Estados y Contrato EVT-02

**Entregables:**
- `PATCH /api/eventos/:id/estado` con la lógica para rechazar la publicación si faltan datos obligatorios.
- Implementación de la notificación HTTP (`fetch`) al M8 cuando el estado pase a `FINALIZADO`. Manejo del timeout/catch sin romper la transacción de BD.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados: (listar)
- Demostración de cómo se simuló/testeó la llamada al M8: (describir)
> ⏸ **STOP — T3 completa. Esperando confirmación para continuar con T4.**

---

### T4 — Frontend: Páginas de Configuración

**Entregables:**
- Página principal `/admin/eventos/nuevo` y `/admin/eventos/:id/editar`.
- Formularios usando React, manejando el estado local y aplicando validaciones de Zod cliente equivalentes al backend.
- Hook custom `useEventoConfiguracion` para encapsular la lógica del fetch nativo.
- UI estructurada estrictamente con Tailwind CSS sin depender de librerías externas.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados: (listar)
- Resumen de la estructura de componentes reutilizables extraídos: (describir)
> ⏸ **STOP — T4 completa. Esperando confirmación.**

---

## 7. Estrategia de Verificación

Se deben garantizar las siguientes validaciones automatizadas y manuales:

- [ ] Intentar guardar un evento con fecha de fin anterior al inicio debe retornar `400 Bad Request` indicando claramente el campo `fechaFin`.
- [ ] Intentar publicar un evento sin configuración de inscripción asociada debe retornar `422 Unprocessable Entity` con un mensaje descriptivo.
- [ ] Ejecutar el cambio de estado a `FINALIZADO` y verificar vía logs y mocks (msw en Vitest) que el request `POST` hacia la URL del M8 se genera correctamente con el body requerido en `Contracts §EVT-02`.
- [ ] Verificar concurrencia básica: dos organizadores modificando el mismo evento no deben corromper los cupos (uso de bloqueos optimistas si es estrictamente necesario, aunque por ahora la última escritura gana).

### Orden de verificación

```text
T1 (BD + seed) → T2 (API CRUD Config) → T3 (Estados API + EventBus Mock) → T4 (Frontend Forms + Hooks)
```
