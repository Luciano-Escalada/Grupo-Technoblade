
# Contracts

Este archivo define las interfaces públicas de cada módulo, incluyendo DTOs compartidos, endpoints expuestos y eventos sincrónicos entre módulos. Ningún cambio a una interfaz pública puede hacerse de forma unilateral.

## Sección 1 — DTOs compartidos

Un módulo nunca expone el objeto de la BD directamente en su API. Los siguientes DTOs estandarizan la información transversal:

**UsuarioResumen**
Usado cuando un módulo (ej. Certificación, M7) necesita los datos básicos del individuo.
```typescript
{
  id: number;
  email: string;
  nombreCompleto: string; // nombre + apellido
}
```

**EventoResumen**
Usado por Catálogo (M2) e Inscripciones (M4).
```typescript
{
  id: number;
  titulo: string;
  tipo: string;
  estado: string;
  fechas: {
    inicio: string; // ISO 8601
    fin: string;    // ISO 8601
  }
}
```

## Sección 2 — Catálogo de endpoints públicos por módulo

### M2 — Catálogo y Descubrimiento
`GET /api/catalogo/eventos?estado=PUBLICADO&tipo=CONGRESO`
Devuelve la lista de eventos para ser consumida por el frontend público y por el M4 (Inscripciones).
```json
{
  "data": [
    { "...": "EventoResumen", "cuposDisponibles": 50 }
  ],
  "error": null,
  "message": "OK"
}
```

### M4 — Motor de Inscripciones
`GET /api/inscripciones/evento/:eventoId/listado`
Utilizado por el M5 (Logística) para obtener la lista de personas que deben realizar el check-in.
```json
{
  "data": [
    {
      "inscripcionId": 105,
      "usuario": { "...": "UsuarioResumen" },
      "estadoInscripcion": "CONFIRMADA"
    }
  ],
  "error": null,
  "message": "OK"
}
```

### M5 — Logística y Acreditación
`GET /api/acreditacion/evento/:eventoId/asistentes`
Consultado por el M7 (Certificación) para saber a quiénes emitirles certificados de asistencia.
```json
{
  "data": [
    {
      "usuario": { "...": "UsuarioResumen" },
      "acreditadoEn": "2026-05-10T08:30:00Z"
    }
  ],
  "error": null,
  "message": "OK"
}
```

## Sección 3 — Eventos entre módulos

Los eventos describen qué notifica un módulo cuando algo importante ocurre. Se implementan como llamadas HTTP POST directas entre los backends correspondientes (simulando un Event Bus).

**EVT-01: Acreditación de participante registrada**
- **Origen:** M5 (Logística) al confirmar la asistencia de un participante.
- **Destino:** M7 (Certificación).
- **Acción:** M7 recibe la notificación y pre-genera el PDF del certificado de asistencia en background para que esté listo cuando el usuario lo solicite.
```json
// POST /api/certificacion/generar/asistencia
{
  "eventoId": 42,
  "usuarioId": 128,
  "tipoCertificado": "ASISTENCIA"
}
```

**EVT-02: Evento marcado como FINALIZADO**
- **Origen:** M3 (Configuración) o M6 (Agenda) al concluir el cronograma.
- **Destino:** M8 (Feedback y Reporting).
- **Acción:** M8 habilita automáticamente el envío o publicación del formulario de encuesta de satisfacción a los participantes acreditados.
```json
// POST /api/feedback/eventos/habilitar-encuesta
{
  "eventoId": 42,
  "fechaFinalizacion": "2026-05-12T18:00:00Z"
}
```