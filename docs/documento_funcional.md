# Documento funcional — Pipeline Medallion en Azure

**Autor:** Ricardo Alvarez
**Diplomatura:** Data Engineer en Azure
**Fecha:** Mayo 2026
**Versión:** 1.0

---

## 1. Problema de negocio

La empresa cuenta con un conjunto de archivos CSV que describen su operación de ventas diarias en distintas tiendas. La información llega de fuentes heterogéneas (sistema transaccional, catálogo maestro de clientes, catálogo de productos y catálogo de tiendas) y no está unificada ni validada. Los equipos de marketing, comercial y dirección necesitan un único conjunto de datos confiable y agregado para poder responder preguntas como:

- ¿Cuáles son los productos más vendidos?
- ¿Qué categorías concentran el mayor monto de ventas?
- ¿Cómo evolucionan las ventas día a día por punto de venta?

Este pipeline resuelve la **ingesta, integración, limpieza y agregación** de esos datos para que las respuestas puedan obtenerse con una consulta SQL.

## 2. Alcance

Quedan dentro del alcance:

- Lectura de los 4 CSV fuentes desde Azure Blob Storage.
- Persistencia incremental en Delta Lake (Bronze).
- Limpieza, deduplicación, casts y enriquecimiento con joins (Silver).
- Generación de tres tablas analíticas para consumo de negocio (Gold).
- Registro de todas las tablas en Unity Catalog.

Quedan fuera del alcance (mejoras a futuro):

- Detección y notificación automática de calidad de datos (alertas por umbrales).
- Integración con Power BI o herramienta de BI similar.
- Captura de cambios (CDC) sobre las tablas maestras.

## 3. Fuentes de datos

| Archivo | Filas aprox. | Descripción | Campos clave |
|---------|--------------|-------------|--------------|
| `Ventas_diarias.csv` | 60 | Tabla de hechos. Una fila por venta. | fecha, tienda_id, producto_id, cliente_id, cantidad, precio_unitario |
| `Clientes.csv` | 20 | Maestra de clientes. | cliente_id, nombre, email, ciudad |
| `Productos.csv` | 20 | Maestra de productos. | producto_id, nombre_producto, categoria |
| `tiendas.csv` | 20 | Maestra de tiendas. | tienda_id, nombre_tienda, ciudad |

Todos los archivos se ingestan desde el container `medallion` del Storage Account `stdiploventas` (West US 2).

## 4. Arquitectura técnica

El pipeline se implementa sobre la arquitectura **Medallion**:

1. **Bronze** — Captura los datos exactamente como llegan. Todas las columnas se mantienen como `string` para evitar pérdida por tipos inválidos. Permite reconstruir cualquier capa superior re-ejecutando los pasos siguientes.
2. **Silver** — Aplica las reglas de calidad de datos, transforma tipos y enriquece la tabla de hechos con los datos descriptivos de las maestras. Es la fuente "única de verdad" para analítica.
3. **Gold** — Materializa agregaciones específicas para responder preguntas de negocio. Cada tabla Gold corresponde a un caso de uso documentado.

Componentes Azure utilizados:

| Componente | Rol |
|------------|-----|
| Azure Blob Storage | Repositorio de archivos crudos. |
| Azure Data Factory | Orquestador del pipeline (planificación, triggers). |
| Azure Databricks | Motor de transformación (PySpark + SQL). |
| Delta Lake | Formato de almacenamiento transaccional. |
| Unity Catalog | Gobierno de datos: schemas, tablas y permisos. |

## 5. Flujo de procesamiento

```
[Blob Storage]                         (CSV crudos)
       │
       ▼
[01_Bronze_Ingesta.ipynb]              spark.read.csv → write delta → CREATE TABLE bronze_*
       │
       ▼
[02_Silver_Limpieza_Enriquecimiento]   dropna + dropDuplicates + casts + 3 joins → silver_ventas
       │
       ▼
[03_Gold_Agregaciones.ipynb]           groupBy + agg → gold_ventas_por_tienda
                                                     → gold_ventas_por_categoria
                                                     → gold_top_productos
```

## 6. Reglas de calidad de datos (capa Silver)

| # | Regla | Implementación | Por qué |
|---|-------|----------------|---------|
| 1 | No se aceptan filas con nulos en `ventas_diarias` | `df.dropna()` | Una venta sin fecha, cantidad o id no aporta información analítica. |
| 2 | Una venta no puede duplicarse | `dropDuplicates(["fecha","tienda_id","producto_id","cliente_id"])` | Evita doble conteo por reintentos o cargas repetidas. |
| 3 | `cantidad` debe ser entero | `cast("int")` | Garantiza agregaciones numéricas correctas. |
| 4 | `precio_unitario` debe ser decimal | `cast("double")` | Necesario para calcular `total = cantidad * precio_unitario`. |
| 5 | `fecha` debe ser una fecha válida `yyyy-MM-dd` | `to_date(col("fecha"), "yyyy-MM-dd")` | Permite particionar y filtrar por rangos temporales. |
| 6 | Integridad referencial | Inner joins contra `clientes`, `productos`, `tiendas` | Una venta a un cliente/producto/tienda inexistente queda excluida. |

Las filas que incumplen alguna regla **no se cargan en Silver**. Pueden re-inspeccionarse releyendo la tabla Bronze (que conserva el dato crudo).

## 7. Salidas analíticas (capa Gold)

| Tabla Gold | Pregunta de negocio | Granularidad |
|------------|---------------------|--------------|
| `gold_ventas_por_tienda` | ¿Cuánto vendió cada tienda cada día? | fecha × tienda |
| `gold_ventas_por_categoria` | ¿Qué categorías concentran el mayor monto? | categoría |
| `gold_top_productos` | ¿Cuáles son los productos más vendidos en monto? | producto |

Cada tabla incluye dos métricas: `total_unidades` (suma de `cantidad`) y `total_ventas` (suma de `cantidad * precio_unitario`).

## 8. Seguridad

- **Credenciales**: la Access Key del Storage Account no debe versionarse. En este repositorio el placeholder `TU_KEY_AQUI` indica donde debe pegarse. En producción se recomienda usar **Databricks Secret Scopes** respaldados por **Azure Key Vault**.
- **Permisos de Unity Catalog**: las tablas se separan en tres schemas (`bronze_clase10`, `silver_clase10`, `gold_clase10`) para poder dar permisos diferenciados por capa.
- **Datos sensibles**: el campo `email` de la tabla de clientes se enmascara en el dataset de muestra (`data/sample/clientes_sample.csv`).

## 9. Operación

- **Reprocesamiento**: cada notebook escribe con `mode("overwrite")` y `overwriteSchema=true`. Es seguro re-ejecutarlos individualmente.
- **Ordenamiento**: Bronze debe correr antes que Silver; Silver antes que Gold.
- **Monitoreo**: cada notebook imprime el conteo de filas tras cada paso. Una caída inesperada de filas entre Bronze y Silver indica problemas de calidad de datos.
- **Time travel**: al estar todo en Delta, cada operación de escritura genera una nueva versión visible con `DESCRIBE HISTORY` y consultable con `VERSION AS OF n`.

## 10. Próximos pasos (roadmap)

1. **Orquestación con ADF**: encadenar los tres notebooks en un pipeline de Data Factory con un schedule trigger diario.
2. **Pruebas de calidad**: incorporar Great Expectations o un módulo de aserciones que detenga el pipeline ante una caída de filas mayor al X%.
3. **Visualización**: conectar la capa Gold a Power BI para dashboards autoservicio.
4. **Histórico**: cambiar a `mode("append")` en Bronze para conservar archivos históricos.
