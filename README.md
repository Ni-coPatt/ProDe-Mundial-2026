# Prode Mundial 2026 — API Backend 

API para gestionar el fixture y el sistema de pronósticos (ProDe) del Mundial 2026.

Desarrollada con Python, Flask y MySQL.

---

## Estructura del proyecto

```
prode-mundial/
├── run.py                  # Punto de entrada
├── requirements.txt
├── .env.example            # Variables de entorno (renombrar a .env)
│
├── config/
│   └── settings.py         # Configuración de la app (DB, secret key)
│
├── app/
│   ├── __init__.py         # create_app(): fábrica de la aplicación Flask
│   ├── extensions.py       # Instancia compartida de SQLAlchemy
│   │
│   ├── models/             # Modelos ORM (tablas de la base de datos)
│   │   ├── partido.py      # Tabla partidos
│   │   ├── resultado.py    # Tabla resultados (relación 1-a-1 con partido)
│   │   ├── usuario.py      # Tabla usuarios
│   │   └── prediccion.py   # Tabla predicciones (usuario + partido + marcador)
│   │
│   ├── routes/             # Blueprints: reciben requests y devuelven responses
│   │   ├── partidos.py     # GET/POST/PUT/PATCH/DELETE /partidos
│   │   ├── resultados.py   # PUT /partidos/<id>/resultado
│   │   ├── usuarios.py     # CRUD /usuarios
│   │   ├── predicciones.py # POST /partidos/<id>/prediccion
│   │   └── ranking.py      # GET /ranking
│   │
│   ├── services/           # Lógica de negocio y validaciones
│   │   ├── partido_service.py
│   │   ├── resultado_service.py
│   │   ├── usuario_service.py
│   │   ├── prediccion_service.py
│   │   └── ranking_service.py
│   │
│   └── utils/
│       └── pagination.py   # Helper HATEOAS: _limit, _offset, _first, _prev, _next, _last
│
├── migrations/
│   └── crear_tablas.sql    # Script SQL para crear la DB manualmente
│
└── tests/
    └── test_partidos.py    # Tests con pytest (usan SQLite en memoria)
```

---

## Cómo correr el proyecto

### 1. Clonar el repositorio

```bash
git clone https://github.com/TU_USUARIO/prode-mundial.git
cd prode-mundial
```

### 2. Crear entorno virtual e instalar dependencias

```bash
python -m venv venv
source venv/bin/activate        # En Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Configurar variables de entorno

```bash
cp .env.example .env
# Editar .env con tus credenciales de MySQL
```

### 4. Crear la base de datos

**Opción A** — con el script SQL:
```bash
mysql -u root -p < migrations/crear_tablas.sql
```

**Opción B** — automático al levantar la app:
```bash
python run.py   # db.create_all() crea las tablas si no existen
```

### 5. Levantar el servidor

```bash
python run.py
# Servidor corriendo en http://localhost:5000
```

---

## Ejemplos de uso

### Crear un partido

```bash
curl -X POST http://localhost:5000/partidos/ \
  -H "Content-Type: application/json" \
  -d '{
    "equipo_local": "Argentina",
    "equipo_visitante": "México",
    "estadio": "MetLife Stadium",
    "ciudad": "New York",
    "fecha": "2026-06-22T21:00:00",
    "fase": "Grupos"
  }'
```

### Listar partidos con filtros y paginación

```bash
# Filtrar por equipo
curl "http://localhost:5000/partidos/?equipo=Argentina"

# Filtrar por fase, con paginación
curl "http://localhost:5000/partidos/?fase=Grupos&_limit=5&_offset=0"
```

### Cargar resultado de un partido

```bash
curl -X PUT http://localhost:5000/partidos/1/resultado \
  -H "Content-Type: application/json" \
  -d '{"goles_local": 2, "goles_visitante": 0}'
```

### Crear usuario y registrar predicción

```bash
# 1. Crear usuario
curl -X POST http://localhost:5000/usuarios/ \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Juan Pérez", "email": "juan@example.com"}'

# 2. Predecir resultado de un partido (antes de que se juegue)
curl -X POST http://localhost:5000/partidos/1/prediccion \
  -H "Content-Type: application/json" \
  -d '{"id_usuario": 1, "local": 2, "visitante": 0}'
```

### Consultar el ranking

```bash
curl "http://localhost:5000/ranking/?_limit=10&_offset=0"
```

Respuesta de ejemplo:
```json
{
  "total": 3,
  "ranking": [
    {"id_usuario": 1, "puntos": 7},
    {"id_usuario": 3, "puntos": 4},
    {"id_usuario": 2, "puntos": 1}
  ],
  "_first": "/ranking?_limit=10&_offset=0",
  "_prev": null,
  "_next": null,
  "_last": "/ranking?_limit=10&_offset=0"
}
```

---

## Sistema de puntaje (ProDe)

| Situación | Puntos |
|---|---|
| Resultado exacto (mismo marcador) | 3 |
| Ganador/empate correcto, marcador distinto | 1 |
| Resultado incorrecto | 0 |

---

## Correr los tests

```bash
pip install pytest
pytest tests/ -v
```

---

## Supuestos / Hipótesis

- Un partido no puede recibir predicciones si ya tiene resultado cargado o si su fecha ya pasó.
- El email de usuario es único en el sistema.
- Los campos `estadio` y `ciudad` son opcionales al crear un partido.
- El ranking solo considera partidos con resultado ya cargado.
- Las fechas se manejan en formato ISO 8601 (`2026-06-22T21:00:00`).
