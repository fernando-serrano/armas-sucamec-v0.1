# ARMAS-SUCAMEC

Automatizacion asistida del flujo de programacion de citas en SEL (SUCAMEC), enfocada en:

- EXAMEN PARA POLIGONO DE TIRO

El proceso esta implementado de extremo a extremo sobre el wizard de SEL, incluyendo generacion de cita con reintentos rapidos cuando aplica.

## 1. Objetivo

Reducir trabajo manual repetitivo en la programacion de citas, manteniendo trazabilidad operativa por Excel y logs claros para diagnostico.

## 2. Estado actual (real del codigo)

El script actual en pipeline-armas.py:

- procesa registros Pendiente desde Excel,
- ordena y agrupa por reglas operativas (RUC/grupo, prioridad, doc, solicitud, fecha_programacion),
- ejecuta login por grupo de credenciales,
- navega y completa Paso 1, Paso 2, Fase 2 y Fase 3,
- ejecuta generar cita con reintentos rapidos,
- actualiza Excel en estado/observaciones,
- deja navegador abierto al final para uso manual,
- registra interrupciones y cierre de ventana con mensajes especificos.

## 3. Stack tecnico

- Python 3
- Playwright sync_api
- pandas + openpyxl
- python-dotenv
- Pillow + pytesseract (opcional OCR)

## 4. Configuracion (.env)

### 4.1 Variables de credenciales (grupo base)

- TIPO_DOC (default: RUC)
- NUMERO_DOCUMENTO
- USUARIO_SEL
- CLAVE_SEL

### 4.2 Variables de credenciales (grupo SELVA)

- SELVA_TIPO_DOC (default: RUC)
- SELVA_NUMERO_DOCUMENTO
- SELVA_USUARIO_SEL
- SELVA_CLAVE_SEL

### 4.3 Variables operativas

- EXCEL_PATH (default: data/programaciones-armas.xlsx)
- VALIDAR_FECHA_PROGRAMACION_HOY (default: 1)

Notas:

- si VALIDAR_FECHA_PROGRAMACION_HOY=1, solo procesa pendientes cuya fecha_programacion (o fecha) sea hoy,
- si VALIDAR_FECHA_PROGRAMACION_HOY=0, permite reprocesos fuera de fecha.

## 5. Excel: columnas consideradas

### 5.1 Requeridas para el flujo

- sede
- fecha
- hora_rango
- tipo_operacion
- nro_solicitud
- tipo_arma
- arma
- estado

### 5.2 Requeridas por logica operativa (se crean vacias si faltan)

- doc_vigilante (fallback: dni)
- dni
- ruc
- prioridad

### 5.3 Opcionales

- id_registro
- observacion / observaciones
- fecha_programacion (si no existe, se usa fecha)

## 6. Reglas de seleccion y agrupacion

### 6.1 Filtro base

Se toma estado que contenga PENDIENTE (case-insensitive).

### 6.2 Filtro por fecha del dia

Aplicado por defecto con VALIDAR_FECHA_PROGRAMACION_HOY=1.

### 6.3 Orden operativo

1. Grupo RUC: SELVA -> JV -> OTRO
2. Prioridad: ALTA antes de NORMAL
3. Orden original del Excel

### 6.4 Deduplicacion de trabajos

Se deduplica por:

- doc_vigilante/dni normalizado
- nro_solicitud
- fecha_programacion (o fecha) normalizada
- grupo RUC

Cada trabajo representa una cita/programacion a procesar.

### 6.5 Registros relacionados dentro de un trabajo

Al cargar un trabajo, se detectan filas relacionadas con misma clave:

- doc_vigilante/dni
- nro_solicitud
- fecha_programacion/fecha

Se guardan en _excel_indices_relacionados para:

- construir objetivos de armas en Fase 2,
- propagar actualizaciones de estado/observaciones a todas las filas relacionadas.

## 7. Flujo normal end-to-end

## 7.1 Preparacion

1. Carga .env.
2. Lee Excel y valida columnas.
3. Obtiene trabajos pendientes segun filtros y orden operativo.
4. Separa trabajos por grupo RUC (SELVA, JV, OTRO).

## 7.2 Login por grupo

1. Valida que existan credenciales del grupo.
2. Abre Chromium visible.
3. Va a login tradicional.
4. Espera recuperacion si hay Service Unavailable (503).
5. Completa credenciales.
6. Resuelve captcha con OCR o modo manual.
7. Envia login y valida por UI autenticada (no solo URL).

## 7.3 Navegacion y configuracion inicial

1. Navega a CITAS -> RESERVAS DE CITAS (fast-path + fallback robusto).
2. Selecciona EXAMEN PARA POLIGONO DE TIRO.

## 7.4 Procesamiento por trabajo

1. Carga registro objetivo desde Excel.
2. Recalcula y adjunta filas relacionadas (_excel_indices_relacionados).
3. Paso 1: seleccion sede/fecha, busca hora y valida cupos.
4. Paso 2: seleccion tipo_operacion, doc_vigilante, SI, nro_solicitud.
5. Fase 2: completa dtTipoLic con pares (tipo, arma) inferidos.
6. Fase 3: captcha + terminos.
7. Genera cita con reintentos rapidos ante fallo captcha/validacion.
8. Marca Excel como Cita Programada (todas las filas relacionadas).
9. Limpia wizard para siguiente trabajo.

## 7.5 Cierre

- imprime resumen final OK/SIN_CUPO/ERROR,
- mantiene navegador abierto para uso manual,
- cierra limpio con Ctrl+C o al finalizar.

## 8. Validaciones clave implementadas

- validacion de login por presencia de UI autenticada,
- deteccion de mensajes de error en login,
- deteccion de Service Unavailable y espera activa,
- confirmacion de selectores PrimeFaces despues de cada seleccion,
- validacion de hora objetivo y cupos libres,
- verificacion de radiobutton seleccionado,
- verificacion de etiquetas finales en combos,
- validacion de seleccion efectiva en tabla dtTipoLic,
- validacion de checkbox de terminos,
- fallbacks para componentes JSF/PrimeFaces inestables.

## 9. Logs operativos y eventos

El flujo reporta eventos de negocio y diagnostico, por ejemplo:

- inicio de script,
- cantidad de pendientes detectados,
- filtro por fecha y registros filtrados,
- grupo RUC en proceso,
- intento de login actual,
- exito/falla de login con tiempo,
- avance por registro y por fase,
- sin cupo,
- errores por registro,
- actualizacion de Excel con cantidad de filas afectadas,
- resumen final o parcial.

Eventos de detencion explicitos:

- interrupcion manual (KeyboardInterrupt),
- ventana/contexto de navegador cerrada,
- fin de ejecucion con navegador cerrado.

## 10. Excepciones y manejo

### 10.1 SinCupoError

Se lanza cuando la hora existe pero no hay cupos disponibles. Resultado:

- incrementa contador SIN_CUPO,
- escribe observacion en Excel,
- continua con el siguiente trabajo.

### 10.2 Errores de procesamiento por registro

Se registran en observaciones con mensajes especificos:

- horario no figura en tabla,
- documento vigilante no disponible para razon social/RUC,
- error generico de procesamiento.

Luego intenta limpiar wizard y continuar.

### 10.3 Cierre de ventana/contexto

Se clasifica como VENTANA_CERRADA y se eleva como interrupcion controlada para detener ejecucion con log explicito.

### 10.4 Interrupcion manual

Se captura KeyboardInterrupt a nivel principal y se imprime resumen parcial:

- tiempo transcurrido,
- acumulados OK/SIN_CUPO/ERROR.

## 11. Escritura en Excel

## 11.1 registrar_sin_cupo_en_excel

- escribe en observaciones/observacion,
- actualiza todas las filas relacionadas,
- fallback por coincidencia de sede+fecha+hora_rango+nro_solicitud.

## 11.2 registrar_cita_programada_en_excel

- escribe estado=Cita Programada,
- actualiza todas las filas relacionadas,
- mismo fallback por coincidencia si no hay indices directos.

## 12. Dependencias

Instalar base:

- playwright
- python-dotenv
- pandas
- openpyxl

OCR opcional:

- pytesseract
- Pillow
- instalacion local de Tesseract OCR

Luego ejecutar instalacion de navegadores de Playwright:

```bash
playwright install
```

## 13. Ejecucion

Desde la raiz del proyecto:

```bash
python pipeline-armas.py
```

## 14. Funciones principales del script

- llenar_login_sel
- obtener_trabajos_pendientes_excel
- cargar_primer_registro_pendiente_desde_excel
- navegar_reservas_citas
- seleccionar_tipo_cita_poligono
- seleccionar_sede_y_fecha_desde_registro
- seleccionar_hora_con_cupo_y_avanzar
- completar_paso_2_desde_registro
- completar_tabla_tipos_arma_y_avanzar
- completar_fase_3_resumen
- generar_cita_final_con_reintento_rapido
- registrar_sin_cupo_en_excel
- registrar_cita_programada_en_excel

## 15. Consideraciones operativas

- El navegador se ejecuta en modo visible (headless=False).
- El captcha puede requerir intervencion manual.
- El flujo puede detenerse por cierre de ventana o interrupcion del operador, dejando trazas claras en log.
- El script esta orientado a operacion asistida, con foco en robustez y trazabilidad.
