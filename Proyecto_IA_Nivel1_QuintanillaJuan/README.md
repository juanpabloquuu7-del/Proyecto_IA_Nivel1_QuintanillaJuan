# Proyecto_IA_Nivel1_QuintanillaJuan

## AgendaBot — Bot conversacional de gestión de citas, tareas, hábitos y recordatorios

**Estudiante:** Juan S Quintanilla  
**Correo:** jsrolon01@gmail.com  
**Asignatura:** Inteligencia Artificial — Nivel 1  
**Stack:** Telegram · n8n Community Edition · Google Sheets

---

## Descripción

AgendaBot es un bot conversacional en Telegram que permite agendar citas, gestionar tareas, crear hábitos y recordatorios, sin depender de ninguna API de pago. Toda la lógica corre en n8n Community Edition y los datos se almacenan en Google Sheets.

El usuario interactúa exclusivamente mediante números. El bot nunca asume intención: siempre muestra opciones y permite cancelar o volver en cualquier momento (opción 9).

---

## Estructura del repositorio

```
Proyecto_IA_Nivel1_QuintanillaJuan/
├── README.md
├── docs/
│   └── AgendaBot.md          ← Documentación técnica + 60 pruebas
├── workflows/
│   └── AgendaBot_Completo_Final.json   ← Workflow n8n importable
└── evidencias/
    └── (capturas de las pruebas)
```

---

## Stack tecnológico

| Componente | Herramienta |
|---|---|
| Interfaz conversacional | Telegram Bot API |
| Automatización y lógica | n8n Community Edition |
| Almacenamiento de datos | Google Sheets (AgendaBot_DB) |

✅ Sin APIs de pago · ✅ Sin tarjeta de crédito · ✅ Sin embeddings ni IA

---

## Cómo importar y configurar

### 1. Google Sheets — Hojas requeridas

Crear documento `AgendaBot_DB` con estas hojas:

| Hoja | Columnas principales |
|---|---|
| CITAS | id_cita, fecha, hora, nombre, motivo, canal, estado, creado_por, timestamp_creacion |
| TAREAS | id_tarea, titulo, prioridad, estado, fecha_objetivo, creado_por |
| HABITOS | id_habito, nombre, frecuencia, hora_recordatorio, estado, creado_por |
| RECORDATORIOS | ID, DESCRIPCION, FECHA_LIMIT, HORA, ESTADO, PRIORIDAD, MEDIO, NOTIFICADO |
| SESSIONS | telegram_user, pantalla_actual, paso_actual, datos_parciales, timestamp_ultima_interaccion |
| LOGS | timestamp, telegram_user, pantalla, opcion_elegida, resultado |
| USUARIOS | telegram_user, nombre, rol, permitido |

### 2. Importar el workflow en n8n

1. Ir a **Workflows → ⋯ → Import from file**
2. Seleccionar `workflows/AgendaBot_Completo_Final.json`
3. El workflow ya trae las credenciales configuradas
4. Activar el workflow con el toggle

### 3. Probar

Enviar `/start` al bot en Telegram.

---

## Funcionalidades implementadas

### Opción 1 — Agenda
- Agendar cita (wizard 6 pasos con validaciones)
- Consultar agenda por fecha
- Reprogramar cita por ID
- Cancelar cita por ID
- Marcar cita como completada

### Opción 2 — Tareas
- Crear tarea (título, prioridad, fecha objetivo)
- Ver tareas pendientes (consulta real desde Sheets)
- Cambiar estado de tarea (pendiente / en progreso / completada / cancelada)

### Opción 3 — Recordatorios
- Crear recordatorio (mensaje, fecha, hora)
- Ver recordatorios activos (consulta real desde Sheets)

### Opción 4 — Hábitos
- Crear hábito (nombre, frecuencia, hora de recordatorio)
- Ver hábitos registrados (consulta real desde Sheets)
- Activar / Desactivar hábito por ID

### Otras
- Menú de ayuda (opción 0)
- Panel de administrador (opción 8)
- Registro automático de logs en cada interacción
- Sesión persistente por usuario en Google Sheets
- Validaciones: formato de fecha YYYY-MM-DD, hora HH:MM, fecha no en el pasado, opciones inválidas

---

## Pruebas

Ver `docs/AgendaBot.md` — sección de pruebas con 60 casos documentados.

Las evidencias (capturas de pantalla) se encuentran en la carpeta `evidencias/`.
