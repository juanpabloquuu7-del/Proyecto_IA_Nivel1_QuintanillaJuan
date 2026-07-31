# AgendaBot — Documentación Técnica

**Proyecto:** Proyecto_IA_Nivel1_QuintanillaJuan  
**Versión:** 1.0 — Funcional  

---

## 1. Descripción General

AgendaBot es un bot conversacional en Telegram diseñado para gestionar citas, tareas, hábitos y recordatorios sin necesidad de APIs de pago. Toda la lógica corre en n8n Community Edition y los datos persisten en Google Sheets.

**Principios (Artículo 3):**
- El usuario siempre elige escribiendo un número
- El bot siempre explica las opciones disponibles
- El bot nunca asume intención
- El bot siempre ofrece salida (opción 9 = volver / cancelar)

---

## 2. Arquitectura del Workflow

```
Telegram Trigger
       ↓
Extraer entrada          (chatId, telegramUser, text)
       ↓
Leer sesión (SESSIONS)   (lee toda la hoja, filtrado en código)
       ↓
Armar objeto sesión      (filtra por telegram_user, detecta sesión existente)
       ↓
Router (lógica principal) ← TODO el switch/case de pantallas aquí
       ↓ (5 salidas paralelas)
       ├── Guardar sesión (SESSIONS)   appendOrUpdate por telegram_user
       ├── Registrar log (LOGS)        append siempre
       ├── Enviar respuesta Telegram   texto de respuesta al usuario
       └── Switch Acción              enruta según accion=GUARDAR_*/VER_*
                ├── GUARDAR_CITA        → Preparar cita        → Guardar en CITAS
                ├── GUARDAR_TAREA       → Preparar tarea       → Guardar en TAREAS
                ├── GUARDAR_HABITO      → Preparar hábito      → Guardar en HABITOS
                ├── GUARDAR_RECORDATORIO→ Preparar recordatorio→ Guardar en RECORDATORIOS
                ├── VER_TAREAS          → Leer TAREAS          → Formato tareas → Enviar consulta
                ├── VER_RECORDATORIOS   → Leer RECORDATORIOS   → Formato recordatorios → Enviar consulta
                └── VER_HABITOS         → Leer HABITOS         → Formato hábitos → Enviar consulta
```

---

## 3. Modelo de Datos

### CITAS
| Campo | Descripción |
|---|---|
| id_cita | ID único (CITA-XXXXXX) |
| fecha | Fecha en formato YYYY-MM-DD |
| hora | Hora en formato HH:MM (24h) |
| nombre | Nombre del cliente |
| motivo | Motivo de la cita |
| canal | Presencial / Virtual / Llamada |
| estado | programada / completada / cancelada |
| creado_por | telegram_user |
| timestamp_creacion | ISO 8601 |

### TAREAS
| Campo | Descripción |
|---|---|
| id_tarea | ID único (TAREA-XXXXXX) |
| titulo | Título de la tarea |
| prioridad | Alta / Media / Baja |
| estado | pendiente / en_progreso / completada / cancelada |
| fecha_objetivo | YYYY-MM-DD o SIN |
| creado_por | telegram_user |

### HABITOS
| Campo | Descripción |
|---|---|
| id_habito | ID único (HABITO-XXXXXX) |
| nombre | Nombre del hábito |
| frecuencia | Diario / Semanal / Mensual |
| hora_recordatorio | HH:MM |
| estado | activo / inactivo |
| creado_por | telegram_user |

### RECORDATORIOS
| Campo | Descripción |
|---|---|
| ID | ID único (REC-XXXXXX) |
| DESCRIPCION | Mensaje del recordatorio |
| FECHA_LIMIT | YYYY-MM-DD |
| HORA | HH:MM |
| ESTADO | pendiente / completado |
| PRIORIDAD | Alta / Media / Baja |
| MEDIO | Telegram |
| NOTIFICADO | SI / NO |

### SESSIONS
| Campo | Descripción |
|---|---|
| telegram_user | ID numérico de Telegram |
| pantalla_actual | Estado actual (ej: CITA_WIZARD) |
| paso_actual | Paso del wizard (ej: PASO3_NOMBRE) |
| datos_parciales | JSON con datos en construcción |
| timestamp_ultima_interaccion | ISO 8601 |

### LOGS
| Campo | Descripción |
|---|---|
| timestamp | ISO 8601 |
| telegram_user | ID del usuario |
| pantalla | Pantalla activa |
| opcion_elegida | Texto que envió el usuario |
| resultado | Acción ejecutada |

---

## 4. Flujos de Navegación

### Menú Principal
```
/start → Bienvenida + Menú
0 → Ayuda
1 → Agenda
2 → Tareas
3 → Recordatorios
4 → Hábitos
8 → Admin
```

### Wizard Agenda — Agendar cita (6 pasos)
```
Paso 1: Fecha (YYYY-MM-DD) — valida formato y no pasado
Paso 2: Hora (HH:MM 24h)
Paso 3: Nombre del cliente (mín. 2 chars)
Paso 4: Motivo (mín. 2 chars)
Paso 5: Canal → 1=Presencial, 2=Virtual, 3=Llamada
Paso 6: Confirmación → 1=Guardar, 2=Editar, 3=Cancelar
→ Éxito: guarda en CITAS, muestra ID
```

### Wizard Tareas — Crear tarea (3 pasos)
```
Paso 1: Título
Paso 2: Prioridad → 1=Alta, 2=Media, 3=Baja
Paso 3: Fecha objetivo (YYYY-MM-DD o SIN)
→ Éxito: guarda en TAREAS, muestra ID
```

### Wizard Recordatorios — Crear (3 pasos)
```
Paso 1: Mensaje del recordatorio
Paso 2: Fecha (YYYY-MM-DD)
Paso 3: Hora (HH:MM)
→ Éxito: guarda en RECORDATORIOS, muestra ID
```

### Wizard Hábitos — Crear (3 pasos)
```
Paso 1: Nombre del hábito
Paso 2: Frecuencia → 1=Diario, 2=Semanal, 3=Mensual
Paso 3: Hora de recordatorio (HH:MM)
→ Éxito: guarda en HABITOS, muestra ID
```

---

## 5. Validaciones Implementadas

| Validación | Implementación |
|---|---|
| Opción válida según menú | Regex por pantalla, mensaje Artículo 7 si falla |
| Formato fecha YYYY-MM-DD | Regex + new Date() |
| Fecha no en el pasado | Comparación con fecha actual |
| Formato hora HH:MM | Regex 24h |
| Confirmación antes de guardar | Paso 6 del wizard de citas |
| Nombre/motivo mínimo 2 chars | length check |
| Cancelar en cualquier paso | Opción 9 siempre disponible |

---

## 6. Pruebas Requeridas (Artículo 13)

### 30 Pruebas de Navegación

| # | Acción | Entrada | Resultado esperado |
|---|---|---|---|
| N-01 | Primer uso | `/start` | Bienvenida + menú completo |
| N-02 | Menú → Agenda | `1` | Menú Agenda con 5 opciones |
| N-03 | Menú → Tareas | `2` | Menú Tareas con 3 opciones |
| N-04 | Menú → Recordatorios | `3` | Menú Recordatorios |
| N-05 | Menú → Hábitos | `4` | Menú Hábitos con 3 opciones |
| N-06 | Menú → Listas | `5` | Mensaje "próximamente" |
| N-07 | Menú → Reportes | `6` | Mensaje "próximamente" |
| N-08 | Menú → Config | `7` | Mensaje "próximamente" |
| N-09 | Menú → Admin | `8` | Panel administrador |
| N-10 | Menú → Ayuda | `0` | Texto de ayuda |
| N-11 | Agenda → Agendar | `1` (en Agenda) | Paso 1 de 6 |
| N-12 | Agenda → Consultar | `2` (en Agenda) | Pide fecha |
| N-13 | Agenda → Reprogramar | `3` (en Agenda) | Pide ID cita |
| N-14 | Agenda → Cancelar | `4` (en Agenda) | Pide ID cita |
| N-15 | Agenda → Completar | `5` (en Agenda) | Pide ID cita |
| N-16 | Volver desde Agenda | `9` (en Agenda) | Menú principal |
| N-17 | Tareas → Crear | `1` (en Tareas) | Paso 1 de 3 |
| N-18 | Tareas → Ver | `2` (en Tareas) | Lista de tareas |
| N-19 | Tareas → Estado | `3` (en Tareas) | Pide ID tarea |
| N-20 | Volver desde Tareas | `9` (en Tareas) | Menú principal |
| N-21 | Recordatorios → Crear | `1` (en Rec.) | Paso 1 de 3 |
| N-22 | Recordatorios → Ver | `2` (en Rec.) | Lista recordatorios |
| N-23 | Volver desde Recordatorios | `9` (en Rec.) | Menú principal |
| N-24 | Hábitos → Crear | `1` (en Hábitos) | Paso 1 de 3 |
| N-25 | Hábitos → Ver | `2` (en Hábitos) | Lista hábitos |
| N-26 | Hábitos → Toggle | `3` (en Hábitos) | Pide ID hábito |
| N-27 | Volver desde Hábitos | `9` (en Hábitos) | Menú principal |
| N-28 | Cancelar en wizard paso 1 | `9` (en PASO1) | Vuelve al menú anterior |
| N-29 | Cancelar en wizard paso 3 | `9` (en PASO3) | Vuelve al menú anterior |
| N-30 | /start con sesión activa | `/start` | Menú principal limpio |

### 10 Agendamientos Completos

| # | Datos | Resultado |
|---|---|---|
| A-01 | 2026-09-01, 10:00, Pedro Gómez, Consulta, Presencial | CITA-XXXXXX guardada |
| A-02 | 2026-09-02, 14:30, Ana López, Asesoría, Virtual | CITA-XXXXXX guardada |
| A-03 | 2026-09-03, 09:00, Carlos Ruiz, Seguimiento, Llamada | CITA-XXXXXX guardada |
| A-04 | 2026-09-04, 11:00, María Torres, Revisión, Presencial | CITA-XXXXXX guardada |
| A-05 | 2026-09-05, 16:00, Luis Mora, Demo, Virtual | CITA-XXXXXX guardada |
| A-06 | Cancelar en paso 3 | Vuelve a Agenda sin guardar |
| A-07 | Cancelar en confirmación (opción 3) | Vuelve a Agenda sin guardar |
| A-08 | Editar en confirmación (opción 2) | Reinicia wizard desde paso 1 |
| A-09 | Confirmar y volver a Agenda | Guarda + regresa a menú Agenda |
| A-10 | Confirmar e ir al menú principal | Guarda + regresa a menú principal |

### 10 Errores Controlados

| # | Entrada | Pantalla | Resultado esperado |
|---|---|---|---|
| E-01 | `hola` | Menú principal | "Ups, esa opción no existe..." |
| E-02 | `99` | Menú principal | "Ups, esa opción no existe..." |
| E-03 | `20/09/2026` | PASO1_FECHA | "Fecha inválida — usa YYYY-MM-DD" |
| E-04 | `2020-01-01` | PASO1_FECHA | "Fecha en el pasado" |
| E-05 | `2pm` | PASO2_HORA | "Hora inválida — usa HH:MM" |
| E-06 | `25:00` | PASO2_HORA | "Hora inválida" |
| E-07 | `A` | PASO3_NOMBRE | "Mínimo 2 caracteres" |
| E-08 | `5` | PASO5_CANAL | "Elige 1, 2 o 3" |
| E-09 | `abc` | PASO2_PRIORIDAD tarea | "Elige 1, 2 o 3" |
| E-10 | `xyz` | TAREA_ESTADO paso ID | "ID inválido — ejemplo: TAREA-123456" |

### 10 Pruebas de Recordatorios

| # | Acción | Resultado |
|---|---|---|
| R-01 | Crear recordatorio completo | REC-XXXXXX en RECORDATORIOS |
| R-02 | Ver recordatorios vacío | "No tienes recordatorios" |
| R-03 | Ver recordatorios con datos | Lista con ID, descripción, fecha, hora |
| R-04 | Fecha pasada en recordatorio | "Fecha inválida o en el pasado" |
| R-05 | Hora inválida en recordatorio | "Hora inválida" |
| R-06 | Cancelar en paso 1 | Vuelve a menú Recordatorios |
| R-07 | Cancelar en paso 2 | Vuelve a menú Recordatorios |
| R-08 | Cancelar en paso 3 | Vuelve a menú Recordatorios |
| R-09 | Crear segundo recordatorio | Segundo REC guardado |
| R-10 | Ver lista con 2 recordatorios | Muestra ambos numerados |

### 10 Pruebas de Permisos

| # | Situación | Resultado |
|---|---|---|
| P-01 | Usuario con sesión en SESSIONS | Acceso normal al menú |
| P-02 | Usuario sin sesión (primer uso) | Bienvenida + menú |
| P-03 | /start reinicia pantalla | Menú principal sin importar pantalla activa |
| P-04 | Opción 9 en cualquier wizard | Cancela y vuelve al menú anterior |
| P-05 | Opción 9 en menú de módulo | Vuelve al menú principal |
| P-06 | Sesión persiste entre mensajes | Wizard continúa desde donde quedó |
| P-07 | Crear cita → sesión queda en MENU_AGENDA | Siguiente mensaje ve menú Agenda |
| P-08 | Crear tarea → sesión queda en MENU_TAREAS | Siguiente mensaje ve menú Tareas |
| P-09 | Log registrado por cada interacción | Fila nueva en LOGS por cada mensaje |
| P-10 | Sesión actualizada en cada paso | pantalla_actual y paso_actual correctos en SESSIONS |
