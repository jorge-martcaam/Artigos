# Monitorizar Power BI Pro

Aínda que a plantilla de Microsoft 365 Usage Analytics aporta información, non aporta unha ollada completa á hora de monitorizar o uso de Power BI. Específicamente, non axuda a responder as preguntas sobre o uso efectivo da plataforma. Ter licenzas de Power BI non significa que se usen. Unha organización pode pagar as súas licenzas e, ao mesmo tempo, non saber que informes abre cada persoa, que espazos de traballo reciben visitas ou que datos saen da plataforma.

O problema é de goberno, non de tecnoloxía. Sen datos de uso, renovar, reasignar ou dar de baixa unha licenza decídese a ollo.

## Que queremos saber

Tres preguntas guían a monitorización:

- **Usuarios:** quen entra, con que frecuencia e en que informes.
- **Contido:** que espazos de traballo e modelos existen, e cales non abriu ninguén.
- **Exportacións:** quen descarga datos desde un informe, desde cal e con que frecuencia.

## Tres fontes, tres preguntas

Ningunha fonte responde soa:

| Fonte | Pregunta que responde | Que achega |
|---|---|---|
| Activity Log (API ActivityEvents) | Que ocorreu? | Quen fixo que, cando e en que elemento |
| Scanner API (metadata scanning) | Que existe? | Espazos de traballo, informes, modelos, etiquetas e aval |
| Microsoft Graph | Quen ten licenza? | Usuarios e licenzas asignadas |

O Activity Log só rexistra accións. O que ninguén abre non aparece nel, polo que o inventario da Scanner API é imprescindible para detectar contido sen uso.

## Arquitectura en tres pasos

1. **Extrae.** Un script programado (PowerShell ou Python) chama á API de actividade e pagina co token de continuación ata obter todos os eventos.
2. **Garda.** Un ficheiro por día. Ao reintentar unha execución, deduplica por `Id` para non contar dúas veces un mesmo evento.
3. **Modela.** Un modelo en estrela en Power BI:
   - `DIM_Usuarios` (desde Microsoft Graph), relacionada con `FCT_Actividad[UserId]`.
   - `DIM_Calendario`, relacionada con `FCT_Actividad[Fecha]`.
   - `DIM_Informes` e `DIM_Workspaces`, relacionadas cos seus identificadores.

## Indicadores de uso

### Segmentación de usuarios

Coa última actividade de cada usuario fórmanse catro grupos:

| Grupo | Criterio orientativo |
|---|---|
| Frecuente | Actividade nos últimos 7 días |
| Ocasional | Actividade entre 8 e 30 días |
| Dormido | Sen actividade entre 31 e 90 días |
| Inactivo | Sen actividade durante máis de 90 días, ou nunca |

### Espazos de traballo sen uso

Espazos de traballo con cero visualizacións (`ViewReport`) nos últimos 60 días. É unha proposta de xestión, non un estándar de Microsoft.

### Fuga a Excel

Compara as exportacións coas visualizacións:

```dax
Ratio Exportación =
VAR var_Exportaciones =
    CALCULATE ( COUNTROWS ( FCT_Actividad ), FCT_Actividad[Operation] = "ExportArtifact" )
VAR var_Visualizaciones =
    CALCULATE ( COUNTROWS ( FCT_Actividad ), FCT_Actividad[Operation] = "ViewReport" )
RETURN
    DIVIDE ( var_Exportaciones, var_Visualizaciones, 0 )
```

Unha ratio alta indica que o informe non responde á pregunta de negocio e que o usuario acaba traballando os datos fóra da plataforma. É un sinal para revisar o informe, non para vixiar á persoa.

### Distribución gobernada

Visualizacións en Apps dividido entre visualizacións totais. É unha proposta de xestión: mide canto consumo pasa por Apps en lugar de por ligazóns soltas.

## Que non responde o monitor

- Non mide a calidade das decisións tomadas cos informes.
- Os límites dos grupos son orientativos; cada organización debe axustalos ao seu caso.
- O cruce co censo depende de que os eventos traian un identificador de usuario utilizable. Convén comprobalo antes de empezar.

Achego un carrusel con información técnica sobre como resolver estas dúbidas.

## Referencias

- [Track user activities in Power BI](https://learn.microsoft.com/fabric/enterprise/powerbi/service-admin-auditing#activity-log)
- [REST API: Admin - Get Activity Events](https://learn.microsoft.com/rest/api/power-bi/admin/get-activity-events)
- [Metadata scanning overview](https://learn.microsoft.com/fabric/governance/metadata-scanning-overview)
- [Power BI implementation planning: Tenant-level auditing](https://learn.microsoft.com/power-bi/guidance/powerbi-implementation-planning-auditing-monitoring-tenant-level-auditing)
- [Microsoft Fabric admin operation list](https://learn.microsoft.com/fabric/admin/operation-list)
