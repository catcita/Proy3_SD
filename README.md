# Proyecto 3 - Sistemas Distribuidos
## TicketFlow - Sistema de Venta de Tickets con Tolerancia a Fallos

Universidad de Talca - Noviembre 2025

**Integrantes:**
- Catalina Herrera
- Camilo Fuentes
- Demian Maturana

---

## 📋 Descripción

Sistema distribuido de venta de tickets que desacopla la reserva de asientos (crítico) de la emisión de documentos (asíncrono), garantizando alta disponibilidad y tolerancia a fallos.

## 🏗️ Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                         USUARIOS                             │
└──────────────┬──────────────────────────────┬───────────────┘
               │                               │
               ▼                               ▼
     ┌─────────────────┐           ┌─────────────────┐
     │   App1 (Go)     │           │  App3 (Python)  │
     │  Load Balancer  │◄─────────►│  Portal Venta   │
     │   ├─ Replica1   │           │                 │
     │   └─ Replica2   │           └────────┬────────┘
     │  PostgreSQL     │                    │
     │  (Patroni)      │                    │
     └────────┬────────┘                    │
              │                             │
              │         ┌───────────────────┤
              │         │                   │
              │         ▼                   ▼
              │   ┌──────────┐      ┌──────────────┐
              └──►│Middleware│      │  App2 (Py)   │
                  │(Go+RMQ)  │─────►│ Facturación  │
                  └──────────┘      │   MariaDB    │
                                    └──────────────┘
```

## 🚀 Inicio Rápido

### Configuración Inicial

```bash
# Copiar archivo de configuración
cp .env.example .env
```

### Iniciar todo el sistema

```bash
# Desde el directorio raíz
docker-compose up -d --build
```

### Detener todo

```bash
docker-compose down
```

### Ver logs

```bash
# Todos los servicios
docker-compose logs -f

# Solo App1
docker-compose logs -f app1_replica1 app1_replica2 nginx

# Solo App2
docker-compose logs -f app2

# Solo PostgreSQL
docker-compose logs -f patroni_master patroni_slave haproxy
```

## 🌐 Acceso a Servicios

### App1 - Gestor de Reservas ✅
- **Frontend**: http://localhost:8083
- **Health Check**: http://localhost:8083/health
- **API**: http://localhost:8083/api

### App2 - Sistema de Tickets ✅
- **Web**: http://localhost:5002
- **Endpoints**: Ver sección API de App2

### App3 - Portal de Venta ✅
- **Frontend**: http://localhost:5003

### Middleware ✅
- **API**: http://localhost:8000

### Infraestructura
- **HAProxy Stats (PostgreSQL)**: http://localhost:7001
- **RabbitMQ Management**: http://localhost:15672 (guest/guest)
- **PostgreSQL**: localhost:5432
- **MariaDB**: localhost:3306

## 📦 Componentes del Sistema

### ✅ App1 - Gestor de Recintos y Reservas
- **Tecnología**: Go (Golang) + PostgreSQL
- **Funcionalidad**:
  - Gestión de eventos
  - Reserva de asientos
  - Prevención de double-booking
  - API RESTful
- **Alta Disponibilidad**:
  - Nginx Load Balancer
  - 2 réplicas de aplicación
  - PostgreSQL con Patroni (master/slave)
  - Failover automático
  - Health checks
- **Puertos**: 8083 (frontend), 5432 (PostgreSQL), 7001 (HAProxy stats)

### ✅ App2 - Motor de Facturación (Ticketera)

Sistema de gestión, venta y validación de entradas a eventos con integración asíncrona mediante RabbitMQ.

#### Características Principales
- **Recepción Asíncrona de Tickets**: Escucha activa de cola RabbitMQ
- **Gestión de Usuarios Híbrida**:
  - Registro completo con RUT, Nombre, Email y Contraseña
  - Usuarios Placeholder automáticos (solo RUT)
  - Vinculación automática al registrarse
- **Flujo de Compra**:
  - Estado inicial: `PENDING_PAYMENT`
  - Simulación de Pasarela de Pago
  - Transición a `PAID`
- **Uso de Tickets**: Validación y "quemado" de entradas (Check-in) → `USED`
- **Reembolsos**: Lógica para devoluciones (solo tickets pagados y no usados)

#### Estructura de Datos

**Usuario (`User`)**
| Campo | Tipo | Descripción |
| :--- | :--- | :--- |
| `rut` | INT (PK) | Identificador único nacional (sin DV) |
| `full_name` | VARCHAR | Nombre completo |
| `email` | VARCHAR | Único. Puede ser `NULL` (Placeholder) |
| `password_hash` | VARCHAR | Hash seguro. Puede ser `NULL` (Placeholder) |

**Ticket (`Ticket`)**
| Campo | Tipo | Descripción |
| :--- | :--- | :--- |
| `id` | INT (PK) | ID interno |
| `external_id` | VARCHAR | ID proveniente del Middleware |
| `status` | ENUM | `PENDING_PAYMENT`, `PAID`, `USED`, `REFUNDED` |
| `user_rut` | INT (FK) | Referencia al dueño del ticket |

#### Integración Middleware

El sistema consume mensajes de la cola `new_ticket` en RabbitMQ.

**Formato del Mensaje (JSON)**:
```json
{
  "id": "EXT-UUID-1234",    
  "rut": 12345678,       
  "price": 5000.00,      
  "event": "Nombre del Evento" 
}
```

### ✅ App3 - Portal de Venta
- **Tecnología**: Python (Flask)
- **Funcionalidad**: 
  - Interfaz web para usuarios finales
  - Visualización de eventos
  - Selección de asientos
  - Integración con App1 y Middleware

### ✅ Middleware - Orquestador
- **Tecnología**: Go + RabbitMQ
- **Funcionalidad**: Comunicación entre App1, App2 y App3

## 🧪 Pruebas de Tolerancia a Fallos

### 1. Caída de Réplica de App1

```bash
# Detener una réplica
docker stop app1_replica1

# El sistema sigue funcionando (Nginx redirige a replica2)
curl http://localhost:8083/health

# Reiniciar
docker start app1_replica1
```

### 2. Caída de PostgreSQL Master

```bash
# Detener master
docker stop app1_patroni_master

# Patroni promueve automáticamente al slave como nuevo master
# Verificar en HAProxy stats: http://localhost:7001

# El sistema continúa funcionando
curl http://localhost:8083/api/events
```

### 3. Prevención de Double-Booking

```bash
# Ejecutar múltiples reservas simultáneas del mismo asiento
for i in {1..10}; do
  curl -X POST http://localhost:8083/api/reserve \
    -H "Content-Type: application/json" \
    -d '{"seat_id": 1, "user_id": '$i'}' &
done

# Solo UNA reserva debe ser exitosa
# El resto recibe error 409 (Conflict)
```

### 4. Prueba de Middleware con RabbitMQ

```bash
# Simular envío de tickets desde Middleware
pip install pika
python3 simulate_middleware.py

# Verificar en App2 que los tickets fueron recibidos
curl http://localhost:5002/api/tickets
```

## 📊 API de App1

### Obtener todos los eventos
```bash
GET /api/events

Response:
{
  "events": [
    {
      "id": 1,
      "name": "Concierto Rock Nacional",
      "venue": "Estadio Nacional",
      "date": "2025-12-15 20:00",
      "total_seats": 50,
      "available_seats": 45
    }
  ],
  "instance": "app1_replica1"
}
```

### Obtener asientos de un evento
```bash
GET /api/events/1/seats

Response:
{
  "seats": [
    {
      "id": 1,
      "event_id": 1,
      "seat_number": "A-1",
      "status": "available"
    }
  ]
}
```

### Reservar un asiento
```bash
POST /api/reserve
Content-Type: application/json

{
  "seat_id": 1,
  "user_id": 123
}

Response:
{
  "message": "Asiento reservado exitosamente",
  "seat_id": 1,
  "user_id": 123,
  "reserved_at": "2025-11-29T10:30:00Z",
  "instance": "app1_replica1"
}
```

## 📊 SLA/SLO

### App1 - Reserva de Asientos
| Métrica | Objetivo | Estrategia |
|---------|----------|------------|
| Disponibilidad | 99.9% | Réplicas + Patroni |
| Latencia (p99) | < 300ms | Optimización Go |
| RTO | < 1 min | Failover automático |
| Tasa de Error | < 0.01% | Validaciones + Transacciones |

### App2 - Facturación
| Métrica | Objetivo | Estrategia |
|---------|----------|------------|
| Disponibilidad | 99.0% | Cola de mensajes |
| Tiempo Procesamiento | < 5 min | Asíncrono |
| RTO | < 1 hora | Buffer en RabbitMQ |

## 🛠️ Comandos Útiles

### Docker Compose

```bash
# Iniciar todos los servicios
docker-compose up -d

# Iniciar con rebuild
docker-compose up -d --build

# Ver estado
docker-compose ps

# Ver logs
docker-compose logs -f

# Detener todo
docker-compose down

# Detener y eliminar volúmenes (⚠️ BORRA DATOS)
docker-compose down -v

# Reiniciar un servicio específico
docker-compose restart app1_replica1
```

### PostgreSQL

```bash
# Conectarse vía HAProxy (recomendado)
psql -h localhost -p 5432 -U postgres -d ticketflow

# Password: postgres123

# Conectarse directamente al master
docker exec -it app1_patroni_master psql -U postgres -d ticketflow

# Ver estado del cluster
docker exec -it app1_patroni_master patronictl -c /etc/patroni.yml list

# Ver replicación
docker exec -it app1_patroni_master psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

### RabbitMQ

```bash
# Ver colas
docker exec ticketflow_rabbitmq rabbitmqctl list_queues

# Ver conexiones
docker exec ticketflow_rabbitmq rabbitmqctl list_connections
```

## 🔐 Credenciales

### PostgreSQL (App1)
- **Usuario**: postgres
- **Password**: postgres123
- **Database**: ticketflow
- **Puerto**: 5432

### MariaDB (App2)
- **Usuario**: Ver archivo `.env`
- **Password**: Ver archivo `.env`
- **Database**: Ver archivo `.env`
- **Puerto**: 3306

### RabbitMQ
- **Usuario**: guest
- **Password**: guest
- **Management**: http://localhost:15672

## 📁 Estructura del Proyecto

```
Proy3_SD/
├── app1/                       # App1 - Gestor de Reservas (Go)
│   ├── backend/
│   │   ├── main.go            # Código principal Go
│   │   ├── go.mod             # Dependencias
│   │   └── go.sum             # Checksums
│   ├── frontend/
│   │   └── index.html         # Frontend web
│   ├── Dockerfile             # Build de App1
│   ├── nginx.conf             # Config Load Balancer
│   ├── haproxy.cfg            # Config PostgreSQL LB
│   └── docker-compose.yml     # Compose específico App1
├── app2/                       # App2 - Facturación (Python)
│   ├── app/
│   │   ├── models.py          # Modelos de Datos
│   │   ├── services.py        # Lógica de Negocio
│   │   ├── routes.py          # Controladores Web
│   │   └── socket_listener.py # Consumidor RabbitMQ
│   ├── run.py                 # Entrypoint Web
│   ├── run_listener.py        # Entrypoint Listener
│   └── Dockerfile
├── app3/                       # App3 - Portal de Venta
│   ├── app.py
│   ├── templates/
│   │   ├── index.html
│   │   └── seats.html
│   └── Dockerfile
├── middleware/                 # Middleware - Orquestador (Go)
│   ├── main.go
│   ├── go.mod
│   └── Dockerfile
├── mariadb/                    # Configuración MariaDB
│   ├── Dockerfile
│   └── init.sql
├── diagramas/                  # Diagramas del sistema
│   └── app2/                   # Diagramas específicos App2
├── docker-compose.yml          # Orquestación completa
├── .env.example                # Ejemplo de variables de entorno
├── README.md                   # Esta documentación
├── Proyecto3.pdf              # Enunciado
└── Informe de contexto.pdf    # Contexto del proyecto
```

## 🐛 Troubleshooting

### Error: "Port already in use"

```bash
# Ver qué proceso usa el puerto
lsof -i :8083

# Matar proceso
kill -9 <PID>

# O cambiar puerto en docker-compose.yml
```

### PostgreSQL no se conecta

```bash
# Ver logs de Patroni
docker-compose logs patroni_master patroni_slave

# Ver estado de HAProxy
open http://localhost:7001

# Reiniciar cluster
docker-compose restart patroni_master patroni_slave haproxy
```

### Nginx no balancea

```bash
# Verificar configuración
docker exec app1_nginx_lb nginx -t

# Ver logs
docker-compose logs nginx

# Reiniciar
docker-compose restart nginx
```

### Contenedores no inician

```bash
# Ver logs detallados
docker-compose logs

# Reconstruir desde cero
docker-compose down -v
docker-compose build --no-cache
docker-compose up -d
```

### RabbitMQ no recibe mensajes

```bash
# Ver estado de colas
docker exec ticketflow_rabbitmq rabbitmqctl list_queues

# Ver logs
docker-compose logs rabbitmq

# Verificar conexiones
docker exec ticketflow_rabbitmq rabbitmqctl list_connections
```

## 📚 Documentación Adicional

- [App1 - Documentación Completa](app1/README.md)
- [Proyecto 3 - Enunciado](Proyecto3.pdf)
- [Informe de Contexto](Informe%20de%20contexto.pdf)

## 📊 Diagramas

Los diagramas del sistema se encuentran en el directorio `diagramas/`:

### App2 - Diagramas Disponibles
- Diagrama de Casos de Uso
- Diagrama de Clases
- Diagrama de Base de Datos (ER)
- Diagramas de Secuencia:
  - Recepción de Ticket (Middleware)
  - Registro de Usuario
  - Login
  - Pago de Ticket
  - Uso de Ticket
  - Devolución de Ticket

## 🎓 Proyecto Académico

Universidad de Talca - Sistemas Distribuidos 2025

**Equipo:**
- Catalina Herrera
- Camilo Fuentes
- Demian Maturana

**Profesor:** rpavez@utalca.cl
