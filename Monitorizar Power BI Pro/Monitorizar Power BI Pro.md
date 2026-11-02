# Monitorizar Power BI Pro

Aunque la plantilla de Microsoft 365 Usage Analytics aporta información, no aporta una perspectiva completa a la hora de monitorizar el uso de Power BI. Específicamente, no ayuda a responder las preguntas sobre el uso efectivo de la plataforma. Tener licencias de Power BI no significa que se usen. Una organización puede pagar sus licencias y, al mismo tiempo, no saber qué informes abre cada persona, qué espacios de trabajo reciben visitas o qué datos salen de la plataforma.

El problema es de gobierno, no de tecnología. Sin datos de uso, renovar, reasignar o dar de baja una licencia se decide a ojo.

## Qué queremos saber

Tres preguntas guían el monitor:

- **Usuarios:** quién entra, con qué frecuencia y en qué informes.
- **Contenido:** qué espacios de trabajo y modelos existen, y cuáles no ha abierto nadie.
- **Exportaciones:** quién descarga datos desde un informe, desde cuál y con qué frecuencia.

## Tres fuentes, tres preguntas

Ninguna fuente responde sola:

| Fuente | Pregunta que responde | Qué aporta |
|---|---|---|
| Activity Log (API ActivityEvents) | ¿Qué ha ocurrido? | Quién hizo qué, cuándo y en qué elemento |
| Scanner API (metadata scanning) | ¿Qué existe? | Espacios de trabajo, informes, modelos, etiquetas y aval |
| Microsoft Graph | ¿Quién tiene licencia? | Usuarios y licencias asignadas |

El Activity Log solo registra acciones. Lo que nadie abre no aparece en él, por eso el inventario de la Scanner API es imprescindible para detectar contenido sin uso.

## Arquitectura en tres pasos

1. **Extrae.** Un script programado (PowerShell o Python) llama a la API de actividad y pagina con el token de continuación hasta obtener todos los eventos.
2. **Guarda.** Un fichero por día. Al reintentar una ejecución, deduplica por `Id` para no contar dos veces un mismo evento.
3. **Modela.** Un modelo en estrella en Power BI:
   - `DIM_Usuarios` (desde Microsoft Graph), relacionada con `FCT_Actividad[UserId]`.
   - `DIM_Calendario`, relacionada con `FCT_Actividad[Fecha]`.
   - `DIM_Informes` y `DIM_Workspaces`, relacionadas con sus identificadores.

## Indicadores de uso

### Segmentación de usuarios

Con la última actividad de cada usuario se forman cuatro grupos:

| Grupo | Criterio orientativo |
|---|---|
| Frecuente | Actividad en los últimos 7 días |
| Ocasional | Actividad entre 8 y 30 días |
| Dormido | Sin actividad entre 31 y 90 días |
| Inactivo | Sin actividad durante más de 90 días, o nunca |

### Espacios de trabajo sin uso

Espacios de trabajo con cero visualizaciones (`ViewReport`) en los últimos 60 días. Es una propuesta de gestión, no un estándar de Microsoft.

### Fuga a Excel

Compara las exportaciones con las visualizaciones:

```dax
Ratio Exportación =
VAR var_Exportaciones =
    CALCULATE ( COUNTROWS ( FCT_Actividad ), FCT_Actividad[Operation] = "ExportArtifact" )
VAR var_Visualizaciones =
    CALCULATE ( COUNTROWS ( FCT_Actividad ), FCT_Actividad[Operation] = "ViewReport" )
RETURN
    DIVIDE ( var_Exportaciones, var_Visualizaciones, 0 )
```

Un ratio alto indica que el informe no responde a la pregunta de negocio y que el usuario acaba trabajando los datos fuera de la plataforma. Es una señal para revisar el informe, no para vigilar a la persona.

### Distribución gobernada

Vistas en Apps dividido entre vistas totales. Es una propuesta de gestión: mide cuánto consumo pasa por Apps en lugar de por enlaces sueltos.

## Qué no responde el monitor

- No mide la calidad de las decisiones tomadas con los informes.
- Los límites de los grupos son orientativos; cada organización debe ajustarlos a su caso.
- El cruce con el censo depende de que los eventos traigan un identificador de usuario utilizable. Conviene comprobarlo antes de empezar.

Adjunto un carrusel con información técnica sobre cómo resolver estas preguntas.

## Referencias

- [Track user activities in Power BI](https://learn.microsoft.com/fabric/enterprise/powerbi/service-admin-auditing#activity-log)
- [REST API: Admin - Get Activity Events](https://learn.microsoft.com/rest/api/power-bi/admin/get-activity-events)
- [Metadata scanning overview](https://learn.microsoft.com/fabric/governance/metadata-scanning-overview)
- [Power BI implementation planning: Tenant-level auditing](https://learn.microsoft.com/power-bi/guidance/powerbi-implementation-planning-auditing-monitoring-tenant-level-auditing)
- [Microsoft Fabric admin operation list](https://learn.microsoft.com/fabric/admin/operation-list)
