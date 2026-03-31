# n8n-operational-workflows

**Sistema de nómina automatizado para pymes Ecuador** — workflows en n8n que reemplazan 4+ horas de trabajo manual en Excel por una ejecución de 30 segundos.

> ⚠️ Los datos de empresa, empleados y credenciales han sido anonimizados. Los folder IDs de Google Drive son placeholders. Ver [Configuración](#configuración) para adaptarlo a tu entorno.

---

## El problema que resuelve

En la mayoría de pymes ecuatorianas, la nómina se procesa así:

1. RRHH abre varios archivos Excel (novedades, descuentos, fondos de reserva, quirografarios)
2. Los consolida manualmente, fila por fila
3. Calcula IESS, décimos, fondos de reserva según la normativa vigente
4. Genera el archivo bancario TXT para la transferencia
5. Todo esto: 3-4 horas por mes, con alto riesgo de error humano

Este sistema lo hace en **~30 segundos**, con trazabilidad completa y cero errores de digitación.

---

## Arquitectura del sistema

El sistema está compuesto por **3 workflows** que se ejecutan en secuencia:

```
[1. Setup DB]  →  [2. Carga de empleados]  →  [3. Cálculo de nómina]
   (una vez)        (cuando hay cambios)         (mensual)
```

### Workflow 1 — Setup DB / Schema nómina
Crea la estructura de base de datos en PostgreSQL. Se ejecuta **una sola vez** al instalar el sistema.

- Crea la tabla `empleados` con todos los campos necesarios para la normativa ecuatoriana (IESS, décimos, fondos de reserva, rubros variables, descuentos fijos)
- Genera índices para búsquedas rápidas por cédula, empresa y estado
- Soporta multi-empresa (`codigo_empresa`)

### Workflow 2 — Carga de empleados (Google Drive)
Sincroniza el archivo maestro de empleados desde Google Drive a PostgreSQL.

**Flujo:**
```
Trigger manual
  → Configuración (empresa, folder)
  → Listar carpeta en Google Drive
  → Filtrar archivo por nombre de empresa (ej: empleados_EMPRESA.xlsx)
  → Descargar y leer archivo Excel
  → Limpiar y normalizar datos
  → Upsert a PostgreSQL (INSERT o UPDATE según cédula)
  → Consulta de verificación
```

**Lógica de upsert:** Si el empleado ya existe (mismo número de cédula), actualiza todos los campos excepto el salario base si el nuevo valor es 0. Esto evita sobreescribir datos con hojas incompletas.

### Workflow 3 — Cálculo de nómina (Google Drive)
El corazón del sistema. Procesa la nómina mensual completa.

**Entradas (carpetas Google Drive):**
| Carpeta | Contenido |
|---------|-----------|
| `novedades/` | Excel con horas extra, ausencias, ingresos del mes |
| `descuentos/` | Excel con descuentos variables del período |
| `ajustes/` | Excel con ajustes manuales a fondos de reserva |
| `pdfs/` | PDFs de IESS (quirografarios + fondos de reserva) |

**Flujo principal:**
```
Trigger manual
  → Configurar período (fecha_nomina: YYYY-MM, empresa)
  → Resolver mes y palabras clave de búsqueda
  → [paralelo] Listar y filtrar: Novedades / Descuentos / Ajustes / PDFs
  → Descargar y leer todos los archivos
  → Ensamblar: Fondos de reserva + Quirografarios + Multas + Préstamos
              + Anticipos + Seguro accidentes + Salud SA + Crediflash
              + Novedades mensuales
  → Unir con datos de empleados en DB
  → Normalizar cédulas (10 dígitos, ceros a la izquierda)
  → Cálculo de nómina (sueldos, IESS personal 9.45%, beneficios, descuentos)
  → Limpieza y validación de resultado
  → Exportar a Excel
```

**Lógica de búsqueda de archivos:** El sistema resuelve automáticamente variantes del nombre del mes (ej: para `2025-06` busca `junio`, `jun`, `06`, `2025-06`, `202506`, `junio_2025`, etc.) para ser tolerante a diferentes convenciones de nomenclatura.

---

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Orquestación | n8n (self-hosted) |
| Base de datos | PostgreSQL |
| Almacenamiento | Google Drive |
| Lectura Excel | n8n Spreadsheet File node |
| Lenguaje lógica | JavaScript (nodos Code de n8n) |
| Output | Excel (.xlsx) listo para revisión y archivo bancario |

---

## Resultado

| Métrica | Antes | Después |
|---------|-------|---------|
| Tiempo de procesamiento | 3-4 horas | ~30 segundos |
| Errores de digitación | Frecuentes | Cero |
| Trazabilidad | Ninguna | Log completo por ejecución |
| Archivos de entrada | Consolidación manual | Carga automática desde Drive |
| Multi-empresa | No | Sí (parámetro `codigo_empresa`) |

---

## Requisitos

- n8n v1.x (self-hosted o cloud)
- PostgreSQL 13+
- Cuenta de Google con acceso a Google Drive
- Credenciales OAuth2 configuradas en n8n para Google Drive

---

## Configuración

### 1. Base de datos
Importa y ejecuta `Setup_DB_Schema_nomina.json` en n8n. Solo se ejecuta una vez.

Asegúrate de tener una credencial de PostgreSQL configurada en n8n con los datos de tu servidor.

### 2. Google Drive
En cada workflow, reemplaza los placeholders por los IDs reales de tus carpetas:

| Placeholder | Descripción |
|-------------|-------------|
| `FOLDER_ID_EMPLEADOS` | Carpeta con el archivo maestro de empleados |
| `FOLDER_ID_NOVEDADES` | Carpeta con novedades mensuales |
| `FOLDER_ID_DESCUENTOS` | Carpeta con descuentos del período |
| `FOLDER_ID_AJUSTES` | Carpeta con ajustes a fondos de reserva |
| `FOLDER_ID_PDFS` | Carpeta con PDFs del IESS |

Para obtener el ID de una carpeta en Google Drive: abre la carpeta, copia el fragmento de la URL después de `folders/`.

### 3. Nodo de Configuración (inicio de cada workflow)
Edita el nodo **"Configuracion"** o **"Configuracion del periodo"** al inicio de cada workflow:

```
empresa:       → Código de tu empresa (ej: "MIEMPRESA")
fecha_nomina:  → Período a procesar (formato: "YYYY-MM", ej: "2025-06")
```

### 4. Formato del archivo de empleados
El archivo Excel de empleados debe tener columnas con estos nombres (no sensible a mayúsculas ni tildes):

`identificacion`, `nombre`, `fecha_nacimiento`, `fecha_contratacion`, `salario_base`, `estado`, `id_banco`, `cuenta_bancaria`, `decimos_mensual`, y rubros adicionales según aplique.

---

## Estructura del repositorio

```
n8n-operational-workflows/
├── workflows/
│   ├── 01_Setup_DB_Schema_nomina.json
│   ├── 02_Carga_de_empleados.json
│   └── 03_Calculo_de_nomina.json
├── docs/
│   └── estructura_excel_empleados.md   (próximamente)
└── README.md
```

---

## Notas de seguridad

- Los workflows exportados **no contienen credenciales reales** — n8n exporta solo los IDs de credencial, que no funcionan en otro servidor
- Los folder IDs de Google Drive han sido reemplazados por placeholders descriptivos
- Los datos de empresa y empleados son ficticios o han sido eliminados
- Se recomienda usar variables de entorno en n8n para los IDs de carpeta en producción

---

## Contexto

Este sistema fue construido para automatizar la nómina de ~200 empleados distribuidos en múltiples empresas simultáneamente. Forma parte de un stack más amplio de automatización operativa que incluye dashboards de visualización financiera y workflows de reportes con aprobación humana.

---

*Parte del portafolio de [William Rosales](https://github.com/datawilly) — Arquitecto de Automatización Operativa*
