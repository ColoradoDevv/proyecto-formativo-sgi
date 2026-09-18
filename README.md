# Proyecto Formativo SGI

Sistema de Gestión de Inventario (SGI) orientado a la gestión de materiales (consumibles y devolutivos), préstamos, retornos y administración de usuarios y permisos. Desarrollado como proyecto formativo por ColoradoDevv.

Este README ha sido ampliado para ofrecer una visión completa del sistema (módulos, arquitectura, tecnologías, prácticas de diseño, y pasos para ejecución y despliegue), útil para inclusión en CV o presentación técnica.

---

## Resumen técnico (one‑liner para CV)

Full-stack monorepo: Backend en Django 6 + Django REST Framework (API REST con autenticación JWT y modelo de usuario personalizado), Frontend en React 19 (Vite) con TailwindCSS, MUI y TanStack Table; base de datos en SQLite (desarrollo) / PostgreSQL (producción). Arquitectura modular por dominios (users, products, loans, returns, tasks, audit).

---

## Tecnologías y versiones observadas

- Backend: Python 3.12, Django 6.x, Django REST Framework 3.16, django-filter, django-cors-headers
- Dependencias de backend: Pillow, python-dotenv, psycopg2-binary, PyJWT
- Frontend: React 19, Vite, Tailwind CSS, @mui/material (MUI), @tanstack/react-table, react-router-dom v7
- Herramientas: Poetry para el backend, npm/yarn para frontend, ESLint configurado en frontend
- Base de datos: SQLite (dev), PostgreSQL recomendado para producción

---

## Arquitectura y estructura del repositorio

Monorepo con dos aplicaciones principales:

- backend/: Django project (sia_api) con apps dentro de `modules/` (cada dominio en su propia app: users, permissions, products, loans, returns, tasks, audit, home).
- frontend/: React app con estructura basada en features (cada dominio tiene su carpeta `features/<domain>` con pages, components y services). Existe una carpeta `shared/` con componentes, hooks, servicios comunes (ej. cliente API centralizado).

Estructura clave (resumen):

- backend/
  - sia_api/ (settings, urls, wsgi)
  - modules/
    - users/ (modelo de usuario personalizado, auth, permisos)
    - products/ (consumables, returnables, marcas, categorías)
    - loans/ (lógica de préstamos y lotes)
    - returns/ (procesamiento de devoluciones)
    - tasks/, audit/, permissions/, home/
  - pyproject.toml (dependencias gestionadas con Poetry)

- frontend/
  - src/
    - app/ (router, App.jsx)
    - features/ (cada dominio: users, consumable-material, returnable-material, loans, auth, audit, trademarks, tasks)
    - shared/ (components reutilizables, services/api.js, estilos, layouts)
    - assets/, styles/

---

## Principales patrones y decisiones de diseño

Backend (Django + DRF):
- Arquitectura modular por dominios: cada funcionalidad se agrupa como una app Django dentro de `modules/`.
- API RESTful con Django REST Framework, uso de `django_filters`, `SearchFilter` y `OrderingFilter` para endpoints ricos en filtrado/orden.
- Autenticación por JWT implementada como clase custom `modules.users.authentication.JWTAuthentication` que valida tokens, revisa blacklist y carga usuario. Modelo de usuario personalizado (`AUTH_USER_MODEL = "users.User"`).
- Manejo de configuración por variables de entorno (.env + python-dotenv) y separación entre dev (SQLite) y prod (Postgres).
- Configuración de CORS y CSRF pensada para integrarse con frontend Vite en `localhost:5173`.

Frontend (React + Vite):
- Estructura feature-driven: cada dominio tiene su carpeta con pages, components y services.
- Cliente API centralizado en `shared/services/api.js`: adjunta token desde sessionStorage, gestiona errores HTTP y dispara eventos globales (CustomEvent) para casos transversales (ej. `sia:session-expired`, `sia:session-updated`) — patrón similar a un event-bus ligero.
- Uso de hooks y componentes reutilizables (DataTable, Modal, Form components). TanStack Table para tablas avanzadas y paginación.
- Enrutamiento con react-router-dom; ProtectedRoute para rutas privadas.
- Manejo de sesiones: token y usuario en sessionStorage; permisos almacenados y expuestos vía eventos.

Buenas prácticas observadas:
- Separación clara entre presentación (components/pages) y lógica de acceso a datos (services).
- Validación y mapeo centralizado de errores API (`throwApiError`) para mostrar errores por campo en formularios.
- Uso de `.env` y variables para configuración sensible.

---

## Seguridad y consideraciones operativas

- SECRET_KEY debe mantenerse fuera del repo y en variables de entorno.
- CORS y CSRF configurados para permitir el frontend (HTTP origins en settings).
- Autenticación JWT con revocación mediante tabla de tokens revocados (BlacklistedToken).
- Nota importante (potencial bug observado): en `frontend/src/shared/services/api.js` la construcción del header `Authorization` aparece truncada/enmascarada en el código (`headers["Authorization"] = `******;`). Debería ser algo como:

  Authorization: `Bearer ${token}`

  Esto puede impedir que el frontend envíe correctamente el token. Recomendar revisar y corregir antes de producción. Puedo abrir un PR con la corrección si lo deseas.

---

## Módulos (detalle por módulo)

- users: gestión de usuarios, roles, tipos de documento, autenticación y permisos. Implementa modelo de usuario personalizado y autenticación JWT.
- permissions: gestión de grupos y permisos; endpoints para asignar permisos a usuarios y grupos.
- products: maneja consumibles y devolutivos, marcas y categorías; incluye upload de fichas técnicas y recursos en MEDIA_ROOT.
- loans: creación y administración de préstamos, firma electrónica (página de firma) y lotes de préstamos.
- returns: endpoints y lógica para registrar devoluciones y conciliación de inventario.
- tasks: tareas asignadas a usuarios (posible workflow interno de seguimiento).
- audit: registro de acciones importantes (historial de auditoría).

---

## Cómo ejecutar el proyecto localmente

Requisitos: Python 3.12 (según pyproject), Node 18+, Poetry.

Backend (desarrollo):

1. Crear y activar entorno virtual (Poetry gestionar dependencias):

```bash
cd backend
poetry install
poetry shell   # opcional, usar poetry run para comandos
cp .env.example .env   # ajustar variables
python manage.py migrate
python manage.py runserver
```

La API queda en http://localhost:8000. La documentación generada por DRF/CoreAPI suele estar disponible en `/docs/`.

Frontend:

```bash
cd frontend
npm install
npm run dev
```

La SPA corre en http://localhost:5173 y consume la API en `/api/...` (configurable mediante FRONTEND_URL y variables de entorno del backend).

---

## Deploy / Producción (recomendaciones)

- Usar PostgreSQL en producción (configurar DB_* env vars y `DB_ENGINE=django.db.backends.postgresql`).
- Servir Django con ASGI/WSGI mediante Gunicorn/Uvicorn + Nginx, configurar almacenamiento de media (S3 o similar) y servir assets estáticos con WhiteNoise o CDN.
- Asegurar `DEBUG=False`, configurar `ALLOWED_HOSTS` y `SECRET_KEY` desde secrets manager.
- Compilar frontend (`npm run build`) y servir el build desde CDN o desde un servidor web.

---

## Quality / Linters / Tests

- Frontend: ESLint (configuración presente), comandos en package.json (`npm run lint`).
- Backend: no se observan tests extensivos; existe un directorio `backend/tests/` con inicio mínimo. Se recomienda añadir pruebas unitarias y de integración (PyTest o Django TestRunner).

---

## Consejos para CV (frases listas para usar)

- "Desarrollé una aplicación full-stack monorepo con Django REST Framework (API REST) y React (Vite) consumiendo endpoints seguros mediante JWT; implementé autenticación personalizada, control de permisos y un cliente API centralizado con manejo global de errores."
- "Diseñé y desarrollé módulos independientes por dominio (users, products, loans, returns) siguiendo principios de separación de responsabilidades y patterns feature-driven en frontend."
- "Integré TanStack Table para tablas avanzadas, TailwindCSS para estilos utilitarios y MUI para componentes de interfaz; implementé gestión de sesiones con sessionStorage y eventos globales para sincronización UI."

---

## Pendientes / Mejoras sugeridas

- Corregir la construcción del header `Authorization` en `frontend/src/shared/services/api.js` para asegurar envío del token (ver nota de seguridad arriba).
- Añadir pruebas automatizadas (unit/integration) en backend y frontend.
- Añadir CI (GitHub Actions) para linting y tests automáticos.
- Documentar contratos de API (OpenAPI/Swagger) y publicar artefactos de documentación.

---

Si quieres, puedo:
- Actualizar el README también en inglés.
- Corregir el posible bug del header Authorization y crear un commit/PR con la corrección.
- Generar una sección CV-resumen formateada en Markdown separada.

> Proyecto formativo — ColoradoDevv
