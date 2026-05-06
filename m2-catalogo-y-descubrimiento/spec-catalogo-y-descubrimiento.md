# Módulo: Catálogo y Descubrimiento (M2)

> **Módulo:** M2 — Catálogo y Descubrimiento
> **Responsable:** —
> **Versión:** 1.0 | **Estado:** Borrador 
>
> **Stack, convenciones y modelos compartidos:** ver `project.md` — no se repiten acá.
> **Interfaces con otros módulos:** ver `contracts.md §M2`.

## 1. Objetivo y Contexto

### ¿Qué resuelve este módulo?

El módulo de Catálogo y Descubrimiento actúa como la vidriera pública de la plataforma. Su responsabilidad es permitir a los usuarios (tanto visitantes anónimos como participantes autenticados) explorar, buscar y filtrar la oferta de eventos académicos disponibles (cursos, jornadas, congresos, charlas) para encontrar aquellos de su interés.

### Lugar en el sistema

Es un módulo principalmente de **lectura** que consume los datos generados por el M3 (Configuración de Eventos). Es el punto de partida del flujo del usuario que culminará en el M4 (Motor de Inscripciones). Solo expone información pública y curada.

### Fuera del alcance de este módulo

- Creación, edición o eliminación de eventos (corresponde a M3).
- Proceso de inscripción o pago de los eventos (corresponde a M4).
- Gestión del cronograma interno o disertantes (corresponde a M6).

---

## 2. Historias de Usuario y Criterios de Aceptación

### CAT-HU-01 — Listado y Descubrimiento de Eventos

**Como** visitante de la plataforma,
**quiero** ver un listado de los próximos eventos académicos disponibles,
**para** conocer la oferta educativa actual.

**Criterios de aceptación:**
- [ ] El sistema muestra por defecto una grilla con los eventos cuyo `estado` sea `PUBLICADO`.
- [ ] Los eventos en estado `BORRADOR` o `CANCELADO` nunca deben ser visibles en el catálogo.
- [ ] Cada tarjeta de evento muestra: título, tipo, fechas de inicio/fin y un indicador visual si quedan cupos disponibles (derivado del `cupoMaximo` y las inscripciones activas).
- [ ] La lista debe estar paginada u ofrecer scroll infinito (20 ítems por página).

### CAT-HU-02 — Búsqueda y Filtrado

**Como** visitante o participante,
**quiero** buscar eventos por palabras clave y filtrarlos por tipo y fecha,
**para** encontrar rápidamente un evento específico que se adapte a mis intereses.

**Criterios de aceptación:**
- [ ] Puedo ingresar texto en un buscador que filtre coincidencias en el `titulo` o `descripcion` (búsqueda case-insensitive).
- [ ] Puedo filtrar por el `tipo` de evento (`CURSO`, `JORNADA`, `CONGRESO`, `CHARLA`).
- [ ] Puedo filtrar por estado temporal: "Próximos eventos" (fechaInicio > hoy) o "Eventos pasados" (fechaFin < hoy).
- [ ] Los filtros y la búsqueda de texto deben poder combinarse.

---

## 3. Requisitos Funcionales y Reglas de Negocio

### 3.1 Validaciones y Reglas Específicas

| Criterio | Regla de Negocio |
|----------|------------------|
| **Visibilidad** | Estrictamente limitados a eventos con `estado === "PUBLICADO"`.
| **Ordenamiento** | Por defecto, los eventos se ordenan por `fechaInicio` ascendente (los más próximos primero). Si se buscan "Eventos pasados", el orden es `fechaFin` descendente (los más recientes primero). |
| **Cálculo de Cupos** | El catálogo no bloquea la vista si el evento está lleno, pero debe mostrar una etiqueta visual de "Cupos Agotados" calculando de forma eficiente la disponibilidad, sin exponer la lista de inscriptos. |

### 3.2 Endpoint Público Mandatorio (Contrato)

Este módulo debe exponer obligatoriamente la interfaz definida en `contracts.md`:
`GET /api/catalogo/eventos?estado=PUBLICADO&tipo=CONGRESO`
Respuesta estandarizada devolviendo objetos tipo `EventoResumen`.

---

## 4. Restricciones técnicas específicas de este módulo

> El stack completo (React 18, Node 20, Prisma 5, etc.) y formato de respuesta HTTP estándar están en `project.md`. Aquí se listan las reglas exclusivas de M2.

- **Performance de lectura:** Al ser la vista pública con mayor tráfico potencial, las consultas a la base de datos deben estar optimizadas. Se requiere crear índices en la base de datos para los campos `estado`, `tipo` y `fechaInicio` de la tabla `eventos`.
- **Zod Middlewares:** Los query parameters (`?tipo=...&busqueda=...&pagina=...`) deben ser validados y parseados por un middleware de Zod específico para validación de queries (`req.query`), asegurando tipos correctos (ej. convertir `pagina` de string a number) antes de llegar al controller.
- **Frontend CSS:** El diseño de la grilla de eventos debe construirse exclusivamente con utilidades de Tailwind CSS (usando CSS Grid), garantizando total responsividad desde dispositivos móviles hasta pantallas ultrawide.

---

## 5. Modelo de datos de este módulo

> Los modelos core `Usuario` y `Evento` ya existen en el `schema.prisma` compartido. **No se redefinen aquí.**
> Para potenciar el motor de descubrimiento, M2 introduce un sistema de categorización mediante etiquetas.

```prisma
// Nuevos modelos exclusivos de M2 agregados al schema.prisma compartido

model Etiqueta {
  id              Int              @id @default(autoincrement())
  nombre          String           @unique
  colorHex        String?          @default("#3B82F6") // Para badges en el Frontend (Tailwind default)
  eventos         EventoEtiqueta[]
  creadoEn        DateTime         @default(now())

  @@map("etiquetas")
}

// Tabla intermedia para relación N:M entre el Evento (existente) y Etiqueta
model EventoEtiqueta {
  eventoId        Int
  etiquetaId      Int
  
  // Relaciones (El modelo Evento ya existe en el project.md, solo se vincula)
  evento          Evento   @relation(fields: [eventoId], references: [id], onDelete: Cascade)
  etiqueta        Etiqueta @relation(fields: [etiquetaId], references: [id], onDelete: Cascade)

  @@id([eventoId, etiquetaId])
  @@map("evento_etiquetas")
}

// Nota técnica: Se asume que en el modelo Evento existente 
// se agregará la relación inversa (etiquetas EventoEtiqueta[]) 
// durante la migración para satisfacer a Prisma.
```

---

## 6. Plan de Tareas

> **Instrucción para Agentes de IA (leer antes de empezar):**
> Implementá las tareas de a una por vez, en el orden estricto indicado.
> Al finalizar cada tarea, completá el **Reporte de cierre** y escribí la frase de pausa exacta. No avances a la siguiente tarea hasta recibir confirmación explícita ("continuar", "ok").

### T1 — Modelo de datos e Índices

**Entregables:**
- Modelos `Etiqueta` y `EventoEtiqueta` agregados a `prisma/schema.prisma`.
- Añadir la relación `etiquetas EventoEtiqueta[]` al modelo `Evento` existente.
- Añadir índices en el modelo `Evento` existente: `@@index([estado, tipo, fechaInicio])`.
- Ejecutar: `prisma migrate dev --name add-catalogo-tags-indexes`.
- Crear un seed (`prisma/seed.ts` si no existe) que genere 10 eventos de prueba (5 publicados, 3 borradores, 2 pasados) y les asigne etiquetas.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados:
- Resultado de `prisma migrate dev`: 
- Resultado de la ejecución del seed:
- Cantidad de eventos `PUBLICADO` generados:
> ⏸ **STOP — T1 completa. Esperando confirmación para continuar con T2.**

### T2 — Capa de Repositorio y Servicio (Backend)

**Entregables:**
- `src/repositories/catalogo.repository.ts`: Funciones de Prisma para buscar eventos (con `where`, `orderBy`, paginación).
- `src/services/catalogo.service.ts`: Lógica de armado del DTO `EventoResumen` y cálculo de "cuposDisponibles" (mockeado si M4 no está listo, pero estructurado).

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados:
- Estructura principal del DTO implementado:
> ⏸ **STOP — T2 completa. Esperando confirmación para continuar con T3.**

### T3 — Controladores, Rutas y Validación (Backend)

**Entregables:**
- `src/middlewares/catalogo.validator.ts`: Schema Zod para los query params.
- `src/controllers/catalogo.controller.ts`: Orquestación y formato de respuesta HTTP estándar (`{ data, error, message }`).
- `src/routes/catalogo.routes.ts`: Definición de `GET /api/catalogo/eventos`.
- Pruebas Vitest básicas para la ruta (1 success, 2 errors).

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados:
- Resultado de la ejecución de Vitest:
> ⏸ **STOP — T3 completa. Esperando confirmación para continuar con T4.**

### T4 — Interfaz de Catálogo (Frontend)

**Entregables:**
- `src/services/catalogo.ts`: Función `fetch` nativa para consumir la API.
- `src/pages/CatalogoPage.tsx`: Vista principal.
- `src/components/EventoCard.tsx`: Componente de UI usando Tailwind CSS.
- Integración de buscador, filtros select y renderizado de la grilla.

**Reporte de cierre — completar antes de detenerse:**
- Archivos creados o modificados:
- Validaciones visuales implementadas (manejo de loading, empty states):
> ⏸ **STOP — T4 completa. Módulo M2 finalizado.**

---

## 7. Estrategia de Verificación

Se deben validar manualmente y a través de test automatizados los siguientes escenarios para garantizar el correcto funcionamiento de M2 y sus contratos:

- [ ] **Aislamiento de Borradores:** Ejecutar una llamada directa a `GET /api/catalogo/eventos` y verificar en la base de datos que ningún ID devuelto posea estado `BORRADOR`.
- [ ] **Contrato de Integración:** Verificar que el payload de respuesta coincide exactamente carácter por carácter con la estructura `EventoResumen` descrita en el archivo `contracts.md`.
- [ ] **Performance de Búsqueda:** Simular la existencia de 5,000 eventos en BD y verificar que el endpoint filtrando por texto libre y tipo responde en menos de 300ms gracias al indexado.
- [ ] **Validación Zod:** Enviar tipos de datos incorrectos en los query params (ej. `?pagina=abc`) y confirmar que se devuelve un status `400` con la estructura de error estándar JSON definida en el proyecto.