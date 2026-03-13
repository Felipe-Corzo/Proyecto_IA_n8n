# 🤖 HelpDeskBot — Bot de soporte por Telegram con n8n

HelpDeskBot es un asistente de soporte técnico construido sobre **n8n** que permite a los usuarios crear, consultar y gestionar tickets de soporte directamente desde **Telegram**, usando **Google Sheets** como base de datos.

---

## 📋 Tabla de contenidos

- [Descripción general](#descripción-general)
- [Arquitectura del flujo](#arquitectura-del-flujo)
- [Funcionalidades](#funcionalidades)
- [Estructura de la base de datos](#estructura-de-la-base-de-datos)
- [Requisitos previos](#requisitos-previos)
- [Instalación y configuración](#instalación-y-configuración)
- [Cómo usar el bot](#cómo-usar-el-bot)
- [Estructura de nodos n8n](#estructura-de-nodos-n8n)
- [Consideraciones técnicas](#consideraciones-técnicas)

---

## Descripción general

HelpDeskBot gestiona conversaciones de múltiples pasos con cada usuario usando un sistema de **estado persistente en Google Sheets**. Cada mensaje que llega al bot es evaluado contra el último estado registrado del usuario para decidir qué acción ejecutar a continuación.

```
Usuario Telegram → n8n Webhook → Validación → Lógica de estado → Respuesta
```

---

## Arquitectura del flujo

El flujo principal sigue esta secuencia en cada mensaje recibido:

```
Telegram Trigger
    └── Check_Usuario          (verifica si el usuario existe en la hoja USUARIOS)
        └── Validar_Activo     (confirma que el usuario está activo)
            └── Cargar_Estado  (carga el historial de LOGS del usuario)
                └── Sort + Limit (obtiene el último estado registrado)
                    └── Switch_Principal (enruta según el estado actual)
                        ├── pantalla vacía / MENU_PRINCIPAL → menú de opciones
                        ├── ESPERANDO_TIPO     → selección de tipo de solicitud
                        ├── ESPERANDO_PRIORIDAD → selección de prioridad
                        ├── ESPERANDO_DESCRIPCION → captura de descripción
                        ├── ESPERANDO_TICKET   → búsqueda de ticket específico
                        └── fallback           → reset y mensaje de error
```

---

## Funcionalidades

### 1. Crear solicitud
Flujo conversacional de 3 pasos para registrar un nuevo ticket:
- Selección del tipo (Soporte técnico / Solicitud administrativa / Consulta general)
- Selección de prioridad (Baja / Media / Alta)
- Descripción libre del problema

Al finalizar se genera un ID de ticket en formato `TIC-XXXX` y se confirma al usuario.

### 2. Consultar estado de solicitud
Permite al usuario ingresar un número de ticket (`TIC-XXXX`) para consultar su estado actual, tipo y descripción registrada.

### 3. Mis solicitudes
Lista todas las solicitudes creadas por el usuario con su estado actual, tipo, descripción y fecha de creación.

### 4. Reportes *(en desarrollo)*
Sección reservada para futuras funcionalidades de reportes.

### 5. Configuración *(en desarrollo)*
Sección reservada para configuración de preferencias del usuario.

### Fallback automático
Si el bot pierde el contexto de la conversación, reinicia automáticamente la sesión del usuario y lo devuelve al menú principal.

---

## Estructura de la base de datos

El proyecto usa un único **Google Spreadsheet** (`HelpDeskBot_DB`) con 3 hojas:

### Hoja `USUARIOS`
Controla quién puede usar el bot.

| Campo | Tipo | Descripción |
|---|---|---|
| `telegram_user` | número | ID numérico de Telegram del usuario |
| `nombre` | texto | Nombre del usuario |
| `activo` | boolean | `TRUE` para permitir acceso |

### Hoja `LOGS`
Persiste el estado de la conversación de cada usuario. El bot lee siempre el registro más reciente para saber en qué pantalla está el usuario.

| Campo | Tipo | Descripción |
|---|---|---|
| `timestamp` | datetime | Fecha y hora del registro |
| `telegram_user` | número | ID del usuario |
| `pantalla` | texto | Estado actual (`MENU_PRINCIPAL`, `ESPERANDO_TIPO`, etc.) |
| `opcion` | texto | Opción seleccionada por el usuario |
| `resultado` | texto | Resultado de la acción |

**Estados posibles de `pantalla`:**

| Estado | Descripción |
|---|---|
| `MENU_PRINCIPAL` | Usuario en el menú, esperando opción 1-5 |
| `ESPERANDO_TIPO` | Esperando selección del tipo de solicitud |
| `ESPERANDO_PRIORIDAD` | Esperando selección de prioridad |
| `ESPERANDO_DESCRIPCION` | Esperando descripción libre del problema |
| `ESPERANDO_TICKET` | Esperando que el usuario ingrese un ID de ticket |

### Hoja `SOLICITUDES`
Almacena todos los tickets creados.

| Campo | Tipo | Descripción |
|---|---|---|
| `id_ticket` | texto | Identificador único (`TIC-XXXX`) |
| `tipo` | texto | Tipo de solicitud |
| `prioridad` | texto | Prioridad asignada |
| `descripcion` | texto | Descripción ingresada por el usuario |
| `estado` | texto | Estado del ticket (`ABIERTO`, etc.) |
| `creado_por` | texto | Nombre del usuario (first_name de Telegram) |
| `fecha_creacion` | datetime | Fecha de creación |

---

## Requisitos previos

- **n8n** versión compatible con `typeVersion 4.7` de Google Sheets y `typeVersion 1.2` de Telegram
- Cuenta de **Telegram** y un bot creado via [@BotFather](https://t.me/BotFather)
- Cuenta de **Google** con acceso a Google Sheets
- Credenciales configuradas en n8n:
  - `Telegram API` — token del bot
  - `Google Sheets OAuth2` — cuenta de Google con acceso al spreadsheet

---

## Instalación y configuración

### 1. Crear el bot de Telegram

1. Abre [@BotFather](https://t.me/BotFather) en Telegram
2. Ejecuta `/newbot` y sigue las instrucciones
3. Guarda el **token** que te entrega BotFather

### 2. Preparar Google Sheets

1. Crea un nuevo spreadsheet en Google Drive
2. Crea las 3 hojas con los nombres exactos: `USUARIOS`, `LOGS`, `SOLICITUDES`
3. Agrega los encabezados de cada hoja según la [estructura descrita arriba](#estructura-de-la-base-de-datos)
4. En la hoja `USUARIOS`, agrega tu propio ID de Telegram con `activo = TRUE`

> Para conocer tu ID de Telegram puedes usar el bot [@userinfobot](https://t.me/userinfobot)

### 3. Configurar credenciales en n8n

1. En n8n ve a **Settings → Credentials**
2. Crea una credencial de tipo **Telegram API** con el token de tu bot
3. Crea una credencial de tipo **Google Sheets OAuth2** y autoriza tu cuenta de Google

### 4. Importar el flujo

1. En n8n ve a **Workflows → Import from file**
2. Selecciona el archivo `Proyecto_Final.json`
3. Actualiza las credenciales en todos los nodos (Telegram y Google Sheets)
4. Actualiza el `documentId` del spreadsheet en todos los nodos de Google Sheets con el ID de tu spreadsheet

> El ID del spreadsheet está en la URL: `https://docs.google.com/spreadsheets/d/`**`ESTE_ES_EL_ID`**`/edit`

### 5. Activar el flujo

1. Haz clic en el toggle **Active** en la esquina superior derecha del workflow
2. Envía un mensaje a tu bot en Telegram para probar

---

## Cómo usar el bot

Una vez activo, envía cualquier mensaje al bot para ver el menú principal:

```
Menú principal:
0. Ayuda
1. Crear solicitud
2. Consultar estado de solicitud
3. Mis solicitudes
4. Reportes
5. Configuración
```

### Crear una solicitud (opción 1)

```
Usuario: 1
Bot: Selecciona el tipo:
     1. Soporte técnico
     2. Solicitud administrativa
     3. Consulta general
     9. Cancelar

Usuario: 1
Bot: Selecciona la prioridad:
     1. Baja 🟢  2. Media 🟡  3. Alta 🔴

Usuario: 2
Bot: Por favor, describe brevemente tu solicitud:

Usuario: No puedo acceder al sistema de facturación
Bot: ✅ Solicitud creada con éxito!
     🎫 Ticket: TIC-4821
     📝 Descripción: No puedo acceder al sistema de facturación
     🚦 Estado: ABIERTO
```

### Consultar un ticket (opción 2)

```
Usuario: 2
Bot: Por favor, escribe el número de ticket (ejemplo: TIC-1234):

Usuario: TIC-4821
Bot: ✅ Ticket encontrado:
     🎫 ID: TIC-4821
     🚦 Estado: ABIERTO
     🛠️ Tipo: Soporte técnico
     📝 Descripción: No puedo acceder al sistema de facturación
```

---

## Estructura de nodos n8n

El flujo contiene **41 nodos** organizados en las siguientes categorías:

| Categoría | Nodos | Función |
|---|---|---|
| **Trigger** | `Telegram Trigger` | Recibe todos los mensajes entrantes |
| **Validación** | `Check_Usuario`, `Validar_Activo` | Controla acceso al bot |
| **Estado** | `Cargar_Estado`, `Sort1`, `Limit1` | Lee el último estado del usuario |
| **Enrutamiento** | `Switch_Principal`, `Switch_Opciones_Menu`, `Switch_Tipos`, `Switch_Prioridad` | Dirige el flujo según el estado y la opción elegida |
| **Mensajes** | 12 nodos `Msg_*` | Envían respuestas al usuario |
| **Datos** | 10 nodos Google Sheets | Leen y escriben en la base de datos |
| **Lógica** | `nuevo_estado`, `Code in JavaScript` | Procesan datos con código JavaScript |
| **Logs** | 7 nodos `Log_*` | Registran el estado actualizado en LOGS |

---

## Consideraciones técnicas

**Manejo de estado entre ejecuciones:** n8n no tiene memoria entre ejecuciones. El estado de la conversación se persiste completamente en la hoja `LOGS`. Cada mensaje del usuario dispara una ejecución nueva que lee el último registro de LOGS para saber qué hacer.

**Identificación de usuarios:** El bot usa el **ID numérico de Telegram** (`message.from.id`) para identificar usuarios en las hojas `USUARIOS` y `LOGS`. La hoja `SOLICITUDES` usa `first_name` en el campo `creado_por` — se recomienda migrar a ID numérico para mayor robustez.

**Generación de IDs de ticket:** Los IDs se generan con `Math.random()` (`TIC-XXXX`). No garantizan unicidad absoluta bajo carga alta. Para producción se recomienda usar un contador incremental en Sheets.

**Fallback de sesión:** Si el bot recibe un mensaje que no coincide con ningún estado conocido, reinicia automáticamente la sesión del usuario escribiendo `MENU_PRINCIPAL` en LOGS y muestra el menú desde cero.

---

## Tecnologías utilizadas

- [n8n](https://n8n.io/) — plataforma de automatización de flujos
- [Telegram Bot API](https://core.telegram.org/bots/api) — interfaz de mensajería
- [Google Sheets API](https://developers.google.com/sheets/api) — base de datos

---

*Proyecto desarrollado como solución de helpdesk conversacional sin servidor de backend dedicado.*
