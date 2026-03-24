# ARMAS-SUCAMEC

Automatización asistida del flujo de programación de citas en SEL (SUCAMEC), enfocada en el proceso **EXAMEN PARA POLIGONO DE TIRO**.

Este repositorio implementa un flujo operativo de extremo a extremo **hasta el paso previo a generar una cita real**.

## 1. Proposito del proyecto

Reducir trabajo manual repetitivo en la programacion de citas, manteniendo control humano donde el proceso lo requiere.

En su estado actual, el script:

- toma datos desde Excel,
- completa automaticamente los pasos del wizard en SEL,
- valida selecciones criticas,
- se detiene antes de cualquier accion final de emision/generacion definitiva de cita.

## 2. Alcance funcional actual

El MVP ya cubre estas capacidades:

- login en autenticacion tradicional (con OCR y fallback manual para captcha),
- navegacion robusta a CITAS -> RESERVAS DE CITAS,
- seleccion de tipo de cita EXAMEN PARA POLIGONO DE TIRO,
- reserva de cupo por sede, fecha y hora con validacion de cupos,
- completado del Paso 2 (tipo de operacion, documento vigilante, solicitud SI, nro solicitud),
- completado de tabla de armas en Fase 2 usando `tipo_arma` + `arma` del Excel,
- completado de Fase 3 (captcha resumen + aceptacion de terminos),
- permanencia del navegador abierto para validacion operativa manual.

## 3. Limite operativo intencional

El flujo se ejecuta **hasta antes de generar una cita real**.

Eso significa que:

- no se ejecuta accion final de confirmacion/emision de cita,
- no se automatiza el cierre transaccional final,
- el operador conserva el control del ultimo tramo sensible.

## 4. Arquitectura y stack

- Python 3
- Playwright (API sync)
- pandas + openpyxl (lectura Excel)
- pytesseract + Pillow (OCR opcional)
- dotenv (.env para configuracion)

Enfoque adoptado:

- Excel local como fuente de verdad para iterar rapido,
- selectores PrimeFaces con validaciones post-accion,
- estrategia de fallback para componentes JSF inestables,
- automatizacion asistida en vez de full unattended.

## 5. Estructura de datos (Excel)

Columnas requeridas por el flujo actual:

- `sede`
- `fecha`
- `hora_rango`
- `tipo_operacion`
- `doc_vigilante` (o `dni` como fallback)
- `nro_solicitud`
- `tipo_arma`
- `arma`
- `estado` (se procesa `Pendiente`)

Columnas opcionales/operativas:

- `id_registro`
- `observacion`
- `fecha_programacion` (si existe, se usa para agrupar registros relacionados)

Regla de procesamiento actual:

- se toma el primer registro `Pendiente`,
- se agrupan registros del mismo documento y misma fecha de programacion,
- de ese grupo se construyen los objetivos de arma para Fase 2.

## 6. Flujo completo implementado

### 6.1 Inicio y carga de datos

1. Carga variables de entorno.
2. Lee Excel y valida columnas.
3. Obtiene primer registro pendiente.
4. Normaliza fecha (`dd/mm/yyyy`) y hora (`HH:MM-HH:MM`).
5. Construye objetivos de arma para la tabla `dtTipoLic`.

### 6.2 Login SEL

1. Abre navegador Chromium visible.
2. Ingresa a login tradicional.
3. Completa tipo doc, numero, usuario y clave.
4. Resuelve captcha con OCR base (primera lectura valida por intento) o modo manual.
5. Envia login y valida URL de acceso.

### 6.3 Navegacion a Reservas

1. Intenta fast-path al item `RESERVAS DE CITAS`.
2. Si no confirma, usa flujo estandar PanelMenu (expandir CITAS y click en submenu).
3. Verifica que se haya cargado `gestionCitasForm`.

### 6.4 Seleccion de tipo de cita

1. Abre SelectOneMenu `Cita para`.
2. Selecciona `EXAMEN PARA POLIGONO DE TIRO`.
3. Valida etiqueta final del combo.

### 6.5 Reserva de cupos (Paso 1)

1. Selecciona `sede` y `fecha` desde Excel.
2. Busca `hora_rango` en tabla de programacion.
3. Verifica cupos libres > 0.
4. Marca radiobutton de la fila.
5. Avanza con boton `Siguiente`.

### 6.6 Datos de solicitud (Paso 2)

1. Selecciona `tipo_operacion` por coincidencia flexible.
2. Resuelve autocomplete de `doc_vigilante` con fallback.
3. Fuerza `Seleccione Solicitud = SI`.
4. Selecciona `nro_solicitud` por token numerico del Excel.

### 6.7 Tabla de armas (Fase 2)

1. Localiza filas de `dtTipoLic`.
2. Activa celda editable de arma.
3. Selecciona arma segun pares `(tipo_fila, arma_objetivo)` derivados del Excel.
4. Valida seleccion efectiva en cada fila.
5. Avanza con `botonSiguiente3`.

### 6.8 Resumen y terminos (Fase 3)

1. Espera panel de resumen.
2. Resuelve captcha del resumen con la logica base del login.
3. Configuracion actual para Fase 3:
   - sin refresh automatico,
   - sin filtro de ambiguedad,
   - sin limite de intentos OCR (`max_intentos=None`).
4. Escribe captcha o deriva a ingreso manual si OCR no aplica.
5. Marca checkbox de terminos y valida estado activo.

### 6.9 Punto de corte del MVP

Al terminar Fase 3, el flujo imprime tiempos y deja el navegador abierto para accion manual.

No ejecuta la accion final de generar/emitir cita.

## 7. Funciones clave del script

Las funciones principales se encuentran en `prueba-8.py`:

- `llenar_login_sel`: orquestador principal del flujo.
- `cargar_primer_registro_pendiente_desde_excel`: carga y prepara datos del Excel.
- `navegar_reservas_citas`: navegacion robusta al modulo de reservas.
- `seleccionar_tipo_cita_poligono`: seleccion de tipo de cita.
- `seleccionar_sede_y_fecha_desde_registro`: completado del Paso 1.
- `seleccionar_hora_con_cupo_y_avanzar`: seleccion de hora con control de cupos.
- `completar_paso_2_desde_registro`: completado de datos de solicitud.
- `completar_tabla_tipos_arma_y_avanzar`: mapeo de armas en Fase 2.
- `completar_fase_3_resumen`: captcha de resumen y terminos.
- `solve_captcha_ocr_base` y `solve_captcha_ocr`: motor OCR base estilo login.

## 8. Configuracion requerida (.env)

Variables de entorno usadas:

- `TIPO_DOC` (default: `RUC`)
- `NUMERO_DOCUMENTO`
- `USUARIO_SEL`
- `CLAVE_SEL`
- `EXCEL_PATH` (default: `data/programaciones-armas.xlsx`)

## 9. Dependencias

Dependencias base:

- `playwright`
- `python-dotenv`
- `pandas`
- `openpyxl`

OCR opcional:

- `pytesseract`
- `Pillow`
- instalacion local de Tesseract OCR

## 10. Ejecucion

Desde la carpeta del proyecto:

```bash
python prueba-8.py
```

## 11. Manejo de errores y robustez

El script incorpora:

- reintentos globales de login,
- validaciones de estado tras cada seleccion critica,
- fallbacks para componentes PrimeFaces (paneles, autocomplete, celdas editables),
- mensajes de log orientados a diagnostico operativo.

## 12. Estado del proyecto

Estado actual: **MVP funcional avanzado**.

Listo para operar flujo asistido hasta el resumen final, con control humano antes de generar una cita real.