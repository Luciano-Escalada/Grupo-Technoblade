# Project


## 1. Descripción del sistema
El producto a desarrollar es un software de organización y gestión de eventos académicos. Permite que grupos de personas puedan organizar eventos de tipo académico (cursos, jornadas, congresos, charlas, etc). Se requiere contar con una interfaz accesible desde la web para facilitar su uso desde cualquier dispositivo.


Módulos del Sistema:
- M1 — Identidad y Acceso (IAM)
- M2 — Catálogo y Descubrimiento
- M3 — Configuración de Eventos
- M4 — Motor de Inscripciones
- M5 — Logística y Acreditación
- M6 — Agenda y Contenidos
- M7 — Certificación
- M8 — Feedback y Reporting


## 2. Stack tecnológico (no negociable)
Estas decisiones aplican a todos los módulos. Ninguna spec puede contradecirlas ni reemplazarlas.


**Frontend**
- Framework: React 18 con Vite
- Estilos: Tailwind CSS — sin librerías de componentes (no MUI, no Ant Design, no shadcn)
- Validaciones cliente: Zod
- HTTP: fetch nativo — no instalar axios ni similares
- Routing: React Router v6


**Backend**
- Runtime: Node.js 20 LTS
- Framework: Express 4
- Validaciones servidor: Zod (en middlewares, no en controllers)
- ORM: Prisma 5


**Base de datos**
- Motor: PostgreSQL 15
- Esquema compartido: Un solo schema.prisma en /prisma/schema.prisma
- Migraciones: prisma migrate dev — nunca modificar la BD directamente


**Testing**
- Framework: Vitest
- Cobertura mínima: flujo exitoso + al menos dos casos de error por endpoint


## 3. Estructura del repositorio

**Backend (estructura interna de cada módulo)**
```text
src/
  controllers/    ← Recibe requests HTTP, delega, devuelve respuestas
  services/       ← Lógica de negocio (nunca en controllers)
  repositories/   ← Acceso a la base de datos vía Prisma
  routes/         ← Definición de endpoints y asignación de middlewares
  middlewares/    ← Validación de esquemas con Zod
```

**Frontend (estructura compartida)**
```text
src/
  pages/          ← Una página por flujo principal
  components/     ← Componentes reutilizables (formularios, tablas, badges)
  services/       ← Funciones fetch para llamar a la API
  hooks/          ← Custom hooks por dominio
```


## 4. Convenciones de código
- Variables y funciones de JS: camelcase. Ejemplo: usuarioId, inscribirParticipante().
- Tablas en BD: snake_case. Ejemplo: eventos_academicos.


## 5. Formato estándar de respuestas HTTP
Todos los endpoints tienen que responder con esta estructura:

```json
{
  "data": { },
  "error": null,
  "message": "Operación exitosa"
}
```


En caso de error:
```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "fields": {
      "email": "Formato de email inválido"
    }
  },
  "message": "Error de validación"
}
```

 
Códigos HTTP estándar:
| Código | Cuándo usarlo |
|--------|---------------|
| 200 | Operación exitosa (GET, PATCH, PUT) |
| 201 | Recurso creado (POST) |
| 400 | Error de validación de formato o campos |
| 404 | Recurso no encontrado |
| 409 | Conflicto de unicidad |
| 422 | Error de lógica de negocio |
| 500 | Error interno del servidor |


## 6. Modelos de datos compartidos
Estos modelos tienen que ser usados por todos los módulos. Se definen en (el archivo de schema del ORM) y no se deben reimplementar.
```prisma
model Usuario {
  id              Int      @id @default(autoincrement())
  email           String   @unique
  nombre          String
  apellido        String
  creadoEn        DateTime @default(now())
  actualizadoEn   DateTime @updatedAt
}

model Evento {
  id              Int      @id @default(autoincrement())
  titulo          String
  descripcion     String?
  tipo            String   // "CURSO" | "JORNADA" | "CONGRESO" | "CHARLA"
  fechaInicio     DateTime
  fechaFin        DateTime
  cupoMinimo      Int?     @default(0)
  cupoMaximo      Int?
  estado          String   @default("BORRADOR")
  creadoEn        DateTime @default(now())
  actualizadoEn   DateTime @updatedAt
}
```


## 7. Roles del sistema
Los roles que existen en el sistema. Cada módulo define en su spec qué acciones puede realizar cada rol dentro de ese módulo.

| Rol | Descripción |
|-----|-------------|
| Organizador | Encargado de la gestión de eventos, configuración, reportes y certificados. |
| Participante | Los participantes pueden generar un usuario en la plataforma y hacer la inscripción a un evento. |
| Disertante | Usuario con rol asignado para exponer en el cronograma del evento. |