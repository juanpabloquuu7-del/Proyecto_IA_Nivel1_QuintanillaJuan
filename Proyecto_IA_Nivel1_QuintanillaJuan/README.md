[README.md](https://github.com/user-attachments/files/30759534/README.md)
# 🤖 AgendaBot

Bot de Telegram construido en **n8n** para gestión personal de citas, tareas, hábitos y recordatorios, con persistencia en **Google Sheets** y un módulo de **reportes de productividad**.

---

## 📋 Tabla de Contenidos

- [Descripción General](#-descripción-general)
- [Arquitectura](#-arquitectura)
- [Estructura de Datos (Google Sheets)](#-estructura-de-datos-google-sheets)
- [Menú del Bot](#-menú-del-bot)
- [Módulo de Reportes de Productividad](#-módulo-de-reportes-de-productividad)
- [Nodos del Workflow](#-nodos-del-workflow)
- [Instalación](#-instalación)
- [Variables y Credenciales](#-variables-y-credenciales)
- [Testing](#-testing)
- [Troubleshooting](#-troubleshooting)
- [Roadmap](#-roadmap)

---

## 🎯 Descripción General

AgendaBot es un asistente conversacional de Telegram que permite a cada usuario:

- 📅 Agendar, consultar, reprogramar, cancelar y completar **citas**
- ✅ Crear, consultar y actualizar el estado de **tareas**
- 🔁 Crear y gestionar **hábitos** recurrentes
- 🔔 Crear y consultar **recordatorios**
- 📊 Generar **reportes de productividad** por usuario
- 👤 Acceder a un **panel de administrador** (usuarios con permisos especiales)

Toda la lógica de navegación es manejada por un único nodo `Router (lógica principal)` que actúa como máquina de estados (state machine) basada en dos variables de sesión: `pantalla_actual` y `paso_actual`, persistidas en la hoja `SESSIONS`.

---

## 🏗️ Arquitectura

```
Telegram Trigger
      │
      ▼
Extraer entrada (Code)
      │
      ▼
Leer sesión (SESSIONS) ── Google Sheets
      │
      ▼
Armar objeto sesión (Code)
      │
      ▼
Router (lógica principal) (Code) ◄── Máquina de estados central
      │
      ├──► Guardar sesión (SESSIONS)
      ├──► Registrar log (LOGS)
      ├──► Enviar respuesta Telegram (mensaje inmediato)
      │
      ▼
Switch Acción (Switch) ── Enruta según $json.accion
      │
      ├─ GUARDAR_CITA          → Preparar cita → Guardar en CITAS
      ├─ GUARDAR_TAREA         → Preparar tarea → Guardar en TAREAS
      ├─ GUARDAR_HABITO        → Preparar hábito → Guardar en HABITOS
      ├─ GUARDAR_RECORDATORIO  → Preparar recordatorio → Guardar en RECORDATORIOS
      ├─ VER_TAREAS            → Leer TAREAS → Formato tareas
      ├─ VER_RECORDATORIOS     → Leer RECORDATORIOS → Formato recordatorios
      ├─ VER_HABITOS           → Leer HABITOS → Formato hábitos
      ├─ VER_USUARIOS          → Leer USUARIOS → Formato usuarios
      ├─ VER_LOGS              → Leer LOGS → Formato logs
      └─ VER_PRODUCTIVIDAD_USUARIO → (ver sección de Reportes) ⭐ NUEVO
                                            │
                                            ▼
                                   Enviar consulta (Telegram) ◄── Punto de salida común
```

### Principio de diseño clave

El `Router` **no ejecuta acciones de escritura/lectura directamente**: solo decide *qué* hacer y devuelve un objeto `{ respuesta, nuevaPantalla, nuevoPaso, nuevosDatos, accion }`. El `Switch Acción` es quien traduce ese campo `accion` en una rama de ejecución real. Esto mantiene toda la lógica conversacional centralizada y fácil de mantener.

---

## 🗂️ Estructura de Datos (Google Sheets)

Documento: **`AgendaBot_DB`**

| Hoja | Columnas | Notas |
|---|---|---|
| **CITAS** | `id_cita, fecha, hora, nombre, motivo, canal, estado, creado_por, timestamp_creacion` | El usuario que creó el registro está en `creado_por` (⚠️ no en `telegram_user`) |
| **TAREAS** | `id_tarea, titulo, prioridad, estado, fecha_objetivo, creado_por` | El usuario está en `creado_por` (⚠️ no en `telegram_user`) |
| **HABITOS** | `id_habito, nombre, frecuencia, hora, estado, creado_por` | — |
| **RECORDATORIOS** | `id_recordatorio, mensaje, fecha, hora, creado_por` | — |
| **LISTAS** | *(en desarrollo)* | Sección aún no implementada |
| **ITEMS_LISTA** | *(en desarrollo)* | Sección aún no implementada |
| **USUARIOS** | `telegram_user, ...` | — |
| **SESSIONS** | `telegram_user, pantalla_actual, paso_actual, datos_parciales` | Persiste el estado conversacional entre mensajes |
| **LOGS** | `timestamp, telegram_user, pantalla, opcion_elegida, resultado` | El usuario está en `telegram_user` ✅ |

> ⚠️ **Importante:** las hojas `CITAS` y `TAREAS` identifican al usuario con la columna `creado_por`, mientras que `LOGS` usa `telegram_user`. El módulo de reportes normaliza esta diferencia internamente (ver más abajo).

---

## 📱 Menú del Bot

### Menú Principal

```
¿En qué te puedo ayudar hoy?

Menú principal:
0. Ayuda
1. Agenda (citas)
2. Tareas
3. Recordatorios
4. Hábitos
5. Listas
6. Reportes
7. Configuración
8. Administrador
```

### Menú Reportes (opción 6)

```
Reportes

1. Reporte diario                                    🚧 Próximamente
2. Reporte semanal                                   🚧 Próximamente
3. Productividad (tareas)                            🚧 Próximamente
4. Hábitos (cumplimiento)                            🚧 Próximamente
5. Agenda (estado de citas)                          🚧 Próximamente
6. Productividad por usuario (reporte completo)      ✅ FUNCIONAL
9. Volver al menú principal
```

> Las opciones 1-5 están reservadas para futuras iteraciones y actualmente responden con un mensaje de "Próximamente" sin romper la navegación.

---

## 📊 Módulo de Reportes de Productividad

### Flujo de uso

1. Usuario navega: `Menú Principal → 6 (Reportes) → 6 (Productividad por usuario)`
2. Bot responde de inmediato:
   ```
   Perfecto. Voy a generarte el reporte de productividad.

   En un momento ya te lo muestro.
   ```
3. En paralelo (mismo flujo, sin acción del usuario), el bot:
   - Lee las hojas `CITAS`, `TAREAS` y `LOGS`
   - Agrupa y calcula métricas por usuario
   - Envía el reporte completo
4. El reporte termina con una pregunta de navegación:
   ```
   ¿Qué deseas hacer ahora?
   1. Volver al menú Reportes
   2. Volver al menú principal
   ```

### Ejemplo de salida real

```
Reporte de productividad (AgendaBot)

Resumen general
- Usuario más activo: @7975888104
- Total citas registradas: 11
- Total tareas registradas: 1
- Total interacciones con el bot: 30

Detalle por usuario

1) @7975888104
   - Citas: 11 (Completadas: 0 | Canceladas: 0)
   - Tareas: 1 (Hechas: 0 | Pendientes: 1)
   - Interacciones: 30

¿Qué deseas hacer ahora?
1. Volver al menú Reportes
2. Volver al menú principal
```

### Métricas calculadas

**Por usuario:**
| Campo | Descripción |
|---|---|
| `total_citas` | Total de citas creadas por el usuario |
| `citas_completadas` | Citas con `estado = "completada"` |
| `citas_canceladas` | Citas con `estado = "cancelada"` |
| `total_tareas` | Total de tareas creadas por el usuario |
| `tareas_completadas` | Tareas con `estado = "completada"` |
| `tareas_pendientes` | Tareas con cualquier otro estado |
| `interacciones` | Número de registros del usuario en `LOGS` |

**Resumen general:**
| Campo | Descripción |
|---|---|
| `total_citas_registradas` | Suma de `total_citas` de todos los usuarios |
| `total_tareas_registradas` | Suma de `total_tareas` de todos los usuarios |
| `total_interacciones_bot` | Suma de `interacciones` de todos los usuarios |
| `usuario_mas_activo` | Usuario con más `interacciones` |

### Reglas de negocio implementadas

- ✅ **Usuarios sin citas o tareas** aparecen igualmente en el listado, con esos contadores en `0`
- ✅ **Registros incompletos se ignoran**: un registro es válido solo si su campo de usuario (`creado_por` en CITAS/TAREAS, `telegram_user` en LOGS) existe y no está vacío tras `trim()`
- ✅ **Orden alfabético**: los usuarios se listan ordenados por su identificador, case-insensitive
- ✅ **Mapeo de columnas por hoja**: el código no asume un único nombre de campo para todas las hojas; normaliza `creado_por` (CITAS/TAREAS) y `telegram_user` (LOGS) a un identificador interno común

---

## 🔧 Nodos del Workflow

### Nodos originales (sin modificar)

| Nodo | Tipo | Función |
|---|---|---|
| Telegram Trigger | `telegramTrigger` | Punto de entrada, escucha mensajes |
| Extraer entrada | Code | Normaliza `chatId`, `telegramUser`, `text` |
| Leer sesión (SESSIONS) | Google Sheets | Recupera estado conversacional previo |
| Armar objeto sesión | Code | Combina entrada + sesión |
| Router (lógica principal) | Code | Máquina de estados central |
| Guardar sesión (SESSIONS) | Google Sheets | Persiste el nuevo estado |
| Registrar log (LOGS) | Google Sheets | Registra cada interacción |
| Enviar respuesta Telegram | Telegram | Envía la respuesta inmediata del Router |
| Switch Acción | Switch | Enruta por `$json.accion` |
| Preparar cita / tarea / hábito / recordatorio | Code | Arman el payload a guardar |
| Guardar en CITAS / TAREAS / HABITOS / RECORDATORIOS | Google Sheets | Persisten el nuevo registro |
| Leer TAREAS / RECORDATORIOS / HABITOS / USUARIOS / LOGS | Google Sheets | Lecturas para submenús de consulta |
| Formato tareas / recordatorios / hábitos / usuarios / logs | Code | Formatean la respuesta de cada consulta |
| Enviar consulta | Telegram | Punto de salida común para todas las consultas |

### Nodos nuevos (módulo de Reportes) ⭐

| Nodo | Tipo | Función |
|---|---|---|
| **Leer CITAS reportes** | Google Sheets | Lee la hoja `CITAS` completa |
| **Leer TAREAS reportes** | Google Sheets | Lee la hoja `TAREAS` completa |
| **Leer LOGS reportes** | Google Sheets | Lee la hoja `LOGS` completa |
| **Procesar datos reportes** | Code | Agrupa por usuario y calcula todas las métricas |
| **Formatear reporte productividad** | Code | Genera el mensaje final en Markdown |

> Los nodos de lectura para reportes están **separados** de los nodos `Leer TAREAS`/`Leer LOGS` originales (usados por otros submenús) para no interferir con esos flujos existentes.

---

## ⚙️ Instalación

### Requisitos previos

- Cuenta de n8n (Cloud o self-hosted, Community Edition)
- Bot de Telegram creado vía [@BotFather](https://t.me/BotFather)
- Documento de Google Sheets llamado `AgendaBot_DB` con las hojas descritas arriba
- Cuenta de Google con acceso OAuth2 a Sheets

### Pasos

1. Importar el archivo `test-juan.json` en n8n (`⋯` → *Import from File*)
2. Verificar que las credenciales se enlacen correctamente:
   - `Telegram account` (Telegram API)
   - `Google Sheets account` (Google Sheets OAuth2)
3. Confirmar que el `documentId` en los nodos de Google Sheets apunte a tu copia de `AgendaBot_DB`
4. Verificar que las hojas `CITAS`, `TAREAS`, `HABITOS`, `RECORDATORIOS`, `USUARIOS`, `SESSIONS`, `LOGS` existan con los nombres exactos (case-sensitive)
5. Activar el workflow (`Active: ON`)
6. Probar enviando `/start` al bot

---

## 🔑 Variables y Credenciales

| Variable | Valor / Ubicación |
|---|---|
| `documentId` (AgendaBot_DB) | `1MeMLjXf0LGrKQrp8LXB3QOsBR-PRrq5kL4s9XQ8x96Q` |
| Credencial Telegram | `Telegram account` (ID: `0CpG2ndOzGUboSzA`) |
| Credencial Google Sheets | `Google Sheets account` (ID: `FDMOCnFkG8fO19ZU`) |
| Admin IDs (hardcoded en Router) | `["7975888104"]` — controla acceso a `MENU_ADMIN` |

> Los IDs de administrador están definidos directamente en el código del nodo `Router (lógica principal)` (constante `ADMIN_IDS`). Para agregar más administradores, edita esa lista.

---

## 🧪 Testing

### Cobertura de pruebas ya validada (fuera de n8n, con Node.js)

| Caso | Resultado |
|---|---|
| Navegación completa: Principal → Reportes → Productividad → vuelta | ✅ |
| Mensaje de confirmación exacto al elegir opción 6 | ✅ |
| Usuario sin citas o sin tareas aparece con valores en `0` | ✅ |
| Registros sin campo de usuario (`creado_por` / `telegram_user`) se ignoran | ✅ |
| Orden alfabético de usuarios en el listado | ✅ |
| Cálculo correcto de `usuario_mas_activo` | ✅ |
| Cálculo correcto de completadas / canceladas / pendientes | ✅ |
| Opción inválida en Menú Reportes muestra mensaje de error sin romper el flujo | ✅ |
| JSON del workflow sin nodos huérfanos, IDs duplicados o conexiones rotas | ✅ |
| Verificado contra datos reales de `AgendaBot_DB` (11 citas, 1 tarea, 1 usuario) | ✅ |

### Cómo probar manualmente

1. Ve a **Executions** en n8n tras cada prueba para revisar tiempos e inputs/outputs de cada nodo
2. Envía `/start` → `6` → `6` desde Telegram y verifica los 3 mensajes esperados (confirmación, reporte, navegación)
3. Para forzar variedad en las métricas, edita manualmente algunas filas de `CITAS`/`TAREAS` cambiando `estado` a `completada` o `cancelada`, y agrega registros con distintos `creado_por` para validar orden alfabético y usuario más activo

---

## 🐛 Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| El reporte muestra 0 citas/tareas aunque hay datos | Nombre de columna de usuario no coincide (`creado_por` vs `telegram_user`) | Verificar el mapeo real de columnas en tu hoja y ajustar `Procesar datos reportes` |
| `Leer CITAS/TAREAS/LOGS reportes` devuelve vacío | `Always Output Data` desactivado, o `sheetName` incorrecto | Activar la opción y confirmar el nombre exacto de la hoja |
| El bot no responde nada | Credencial de Telegram inválida o workflow inactivo | Revisar credenciales y que `Active: ON` |
| Markdown no se ve bien en Telegram | `parseMode` mal configurado | Usar `Markdown`, no `HTML`, en el nodo de envío |
| Usuario aparece duplicado con distinto casing (ej. `Ana` y `ana`) | Falta de normalización de mayúsculas en origen | El código ya ordena case-insensitive, pero considera normalizar a minúsculas al guardar en Sheets si esto ocurre |

---

## 🚀 Roadmap

- [ ] Implementar opciones 1-5 del Menú Reportes (diario, semanal, tareas, hábitos, agenda)
- [ ] Exportar reporte de productividad a CSV/PDF
- [ ] Filtrar reporte por rango de fechas
- [ ] Comparativa semana vs. semana
- [ ] Implementar secciones `LISTAS` e `ITEMS_LISTA`
- [ ] Notificaciones automáticas programadas (recordatorios, hábitos diarios)
- [ ] Panel de administrador extendido (ver hoja `SESSIONS`, borrar usuarios, etc.)

---

## 📄 Licencia y Autoría

Proyecto desarrollado como parte de un ejercicio académico/personal de automatización con n8n.

**Última actualización:** Agosto 2026
**Versión:** 2.0 (incluye módulo de Reportes de Productividad)
