# TaskFlow

Sistema de gestión de tareas construido con una arquitectura de microservicios en Python, contenedorizado con Docker, orquestado con Kubernetes y desplegado en Oracle Cloud (Always Free Tier).

Este proyecto no busca reinventar un task manager, sino servir como demostración práctica de un flujo completo cloud-native: microservicios independientes, comunicación asíncrona por eventos, despliegue declarativo en Kubernetes, CI/CD automatizado y observabilidad en producción.

🔗 **Demo en vivo:** `https://taskflow.<IP>.sslip.io` *(actualizar con la URL real una vez desplegado)*

---

## Tabla de contenidos

- [Arquitectura](#arquitectura)
- [Stack tecnológico](#stack-tecnológico)
- [Servicios](#servicios)
- [Decisiones técnicas](#decisiones-técnicas)
- [Cómo correrlo en local](#cómo-correrlo-en-local)
- [Cómo desplegarlo en Kubernetes](#cómo-desplegarlo-en-kubernetes)
- [CI/CD](#cicd)
- [Observabilidad](#observabilidad)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Capturas](#capturas)
- [Roadmap / Mejoras futuras](#roadmap--mejoras-futuras)

---

## Arquitectura

```
                              ┌─────────────────┐
                              │   Ingress (NGINX) │
                              └────────┬─────────┘
                                       │
                 ┌─────────────────────┼─────────────────────┐
                 │                     │                     │
         ┌───────▼───────┐    ┌────────▼────────┐   ┌────────▼─────────┐
         │  auth-service  │    │  tasks-service   │   │ notifications-svc│
         │   (FastAPI)    │    │   (FastAPI)      │   │  (FastAPI+Celery)│
         └───────┬────────┘    └────────┬─────────┘   └────────┬─────────┘
                 │                      │                       │
                 │             ┌────────▼─────────┐             │
                 └────────────►│    PostgreSQL     │             │
                               └───────────────────┘             │
                                        │                        │
                               ┌────────▼─────────┐              │
                               │       Redis        │◄────────────┘
                               │  (cola de eventos)  │
                               └────────────────────┘

         Observabilidad: Prometheus + Grafana (métricas de todos los pods)
```

**Flujo típico:**
1. El usuario se registra/loguea contra `auth-service`, que devuelve un JWT.
2. Con ese JWT, crea tareas contra `tasks-service`.
3. `tasks-service` guarda la tarea en Postgres y publica un evento en Redis.
4. `notifications-service` consume el evento de forma asíncrona y dispara la notificación, sin bloquear la respuesta al usuario.
5. Todos los servicios exponen métricas que Prometheus scrapea y Grafana visualiza.

---

## Stack tecnológico

| Categoría | Tecnología |
|---|---|
| Lenguaje | Python 3.11 |
| Framework API | FastAPI |
| ORM / Migraciones | SQLAlchemy + Alembic |
| Autenticación | JWT (python-jose) + bcrypt (passlib) |
| Base de datos | PostgreSQL |
| Cola de eventos | Redis |
| Workers asíncronos | Celery |
| Contenedores | Docker (multi-stage builds) |
| Orquestación | Kubernetes (k3s en producción, kind en desarrollo local) |
| Empaquetado K8s | Helm |
| CI/CD | GitHub Actions |
| Registro de imágenes | GitHub Container Registry (ghcr.io) |
| Observabilidad | Prometheus + Grafana |
| Cloud | Oracle Cloud Infrastructure (Always Free Tier) |
| TLS | cert-manager + Let's Encrypt |

---

## Servicios

### `auth-service`
Gestiona registro, login y emisión/validación de JWT.

| Método | Endpoint | Descripción |
|---|---|---|
| POST | `/register` | Crea un nuevo usuario |
| POST | `/login` | Devuelve un JWT válido |
| GET | `/me` | Devuelve los datos del usuario autenticado |

### `tasks-service`
CRUD de tareas, protegido por JWT.

| Método | Endpoint | Descripción |
|---|---|---|
| POST | `/tasks` | Crea una tarea |
| GET | `/tasks` | Lista las tareas del usuario |
| GET | `/tasks/{id}` | Detalle de una tarea |
| PUT | `/tasks/{id}` | Actualiza una tarea |
| DELETE | `/tasks/{id}` | Elimina una tarea |

### `notifications-service`
Worker asíncrono que escucha eventos en Redis y dispara notificaciones (simuladas o vía SMTP de prueba).

Cada servicio expone su documentación interactiva en `/docs` (Swagger UI) gracias a FastAPI.

---

## Decisiones técnicas

- **FastAPI en vez de Flask/Django**: tipado nativo con Pydantic, documentación automática, y rendimiento async, que se ajusta bien a un contexto de microservicios.
- **Redis en vez de RabbitMQ**: para el volumen de este proyecto, Redis cubre la necesidad de cola de eventos con mucha menos complejidad operativa. RabbitMQ sería la elección natural si el proyecto necesitara garantías de entrega más estrictas o colas con lógica de enrutamiento compleja.
- **k3s en vez de un clúster gestionado (EKS/AKS)**: los servicios gestionados de Kubernetes cobran por el control plane desde el primer minuto, incluso en free tier. k3s corriendo sobre una VM Always Free de Oracle da un clúster real de Kubernetes sin ese costo, ideal para un proyecto sostenido en el tiempo sin presupuesto.
- **Oracle Cloud en vez de AWS/Azure**: el free tier de Oracle no expira y ofrece significativamente más cómputo (4 OCPUs / 24 GB RAM) que las alternativas, que limitan sus recursos gratuitos a 12 meses.
- **JWT stateless en vez de sesiones**: al validar el token con la clave pública compartida, `tasks-service` no necesita llamar a `auth-service` en cada request, reduciendo acoplamiento y latencia entre servicios.

---

## Cómo correrlo en local

Requiere Docker y Docker Compose instalados.

```bash
git clone https://github.com/<tu-usuario>/taskflow.git
cd taskflow
cp .env.example .env
docker-compose up --build
```

Una vez levantado:
- `auth-service` → http://localhost:8001/docs
- `tasks-service` → http://localhost:8002/docs
- `notifications-service` (worker, sin UI)
- Gateway (Nginx) → http://localhost:8080

---

## Cómo desplegarlo en Kubernetes

### Localmente con `kind`

```bash
kind create cluster --name taskflow
kubectl apply -f k8s/base/
kubectl get pods -w
```

### En Oracle Cloud con `k3s`

1. Crear una VM Ampere A1 (ARM) en Oracle Cloud Free Tier.
2. Instalar k3s:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```
3. Copiar el kubeconfig y aplicar los manifiestos:
   ```bash
   kubectl apply -f k8s/base/
   kubectl apply -f k8s/ingress/
   ```
4. Configurar DNS gratuito con `sslip.io` y TLS con `cert-manager`.

Instrucciones detalladas en [`docs/deploy-oracle.md`](docs/deploy-oracle.md).

---

## CI/CD

El pipeline de GitHub Actions (`.github/workflows/ci-cd.yaml`) corre en cada push a `main`:

1. **Test** — linter (`ruff`) + tests unitarios (`pytest`) para cada servicio.
2. **Build** — construye las imágenes Docker y las publica en `ghcr.io`, tageadas con el SHA del commit.
3. **Deploy** — aplica los manifiestos actualizados contra el clúster.

![CI/CD Status](https://github.com/<tu-usuario>/taskflow/actions/workflows/ci-cd.yaml/badge.svg)

---

## Observabilidad

- **Prometheus** scrapea métricas expuestas por cada servicio FastAPI (`prometheus-fastapi-instrumentator`).
- **Grafana** visualiza: requests/segundo, latencia p95, tasa de errores, uso de CPU/memoria por pod.
- Instalación vía Helm:
  ```bash
  helm install monitoring prometheus-community/kube-prometheus-stack -f monitoring/prometheus-values.yaml
  ```

---

## Estructura del repositorio

```
taskflow/
├── services/
│   ├── auth-service/
│   ├── tasks-service/
│   └── notifications-service/
├── k8s/
│   ├── base/
│   └── helm-chart/
├── monitoring/
├── docs/
├── .github/workflows/
├── docker-compose.yaml
└── README.md
```

---

## Capturas

| Swagger UI | Dashboard Grafana | Pipeline CI/CD |
|---|---|---|
| *(captura)* | *(captura)* | *(captura)* |

---

## Roadmap / Mejoras futuras

- [ ] Tests de integración end-to-end
- [ ] Rate limiting en el gateway
- [ ] Trazas distribuidas (OpenTelemetry)
- [ ] Frontend simple para consumir la API (opcional)

---

## Autor

**[Lucas L. Suárez]**
[LinkedIn] · [Portfolio] · [Email]