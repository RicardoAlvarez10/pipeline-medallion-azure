# Documento funcional — Pipeline Medallion en Azure

**Autor:** Ricardo Alvarez
**Versión:** 1.0

## 1. Problema de negocio

La operación cuenta con archivos CSV separados que describen ventas diarias en distintas tiendas, junto con tres tablas maestras (clientes, productos, tiendas). La información llega desnormalizada y sin validar. Los equipos de marketing, comercial y dirección necesitan un único conjunto de datos confiable y agregado para responder preguntas como:

- ¿Cuáles son los productos más vendidos?
- ¿Qué categorías concentran el mayor monto de ventas?
- ¿Cómo evolucionan las ventas día a día por punto de venta?

El pipeline resuelve la ingesta, integración, limpieza y agregación de esos datos para que las respuestas se obtengan con una consulta SQL.

## 2. Alcance

Dentro:

- Lectura de los 4 CSV fuente desde Azure Blob Storage.
- Persistencia en Delta Lake (Bronze).
- Limpieza, deduplicación, casts y enriquecimiento con joins (Silver).
- Generación de tres tablas analíticas (Gold).
- Registro en Unity Catalog.

Fuera (roadmap):

- Detección automática de calidad de datos con alertas.
- Integración con Power BI.
- Captura de cambios (CDC) sobre maestras.

## 3. Fuentes de datos

| Archivo | Filas aprox. | Descripción | Campos clave |
|---------|--------------|-------------|--------------|
| `Ventas_diarias.csv` | 60 | Tabla de hechos. Una fila por venta. | fecha, tienda_id, producto_id, cliente_id, cantidad, precio_unitario |
| `Clientes.csv` | 20 | Maestra de clientes. | cliente_id, nombre, email, ciudad |
| `Productos.csv` | 20 | Maestra de productos. | producto_id, nombre_producto, categoria |
| `tiendas.csv` | 20 | Maestra de tiendas. | tienda_id, nombre_tienda, ciudad |

## 4. Arquitectura técnica

Arquitectura Medallion en tres capas:

1. **Bronze** — Captura los datos tal como llegan. Todas las columnas como `string` para evitar pérdida por tipos inválidos.
2. **Silver** — Aplica reglas de calidad, transforma tipos y enriquece la tabla de hechos con datos descriptivos.
3. **Gold** — Materializa agregaciones específicas por caso de uso.

| Componente | Rol |
|------------|-----|
| Azure Blob Storage | Repositorio de archivos crudos. |
| Azure Data Factory | Orquestación (schedule, triggers). |
| Azure Databricks | Motor de transformación (PySpark + SQL). |
| Delta Lake | Formato de almacenamiento transaccional. |
| Unity Catalog | Gobierno de datos: schemas, tablas y permisos. |

## 5. Flujo de procesamiento

```
[Blob Storage]
       │
       ▼
[01_Bronze_Ingesta]              spark.read.csv → write delta → CREATE TABLE bronze_*
       │
       ▼
[02_Silver_Limpieza]             dropna + dropDuplicates + casts + 3 joins → silver_ventas
       │
       ▼
[03_Gold_Agregaciones]           groupBy + agg → gold_ventas_por_tienda
                                                gold_ventas_por_categoria
                                                gold_top_productos
```

## 6. Reglas de calidad de datos

| # | Regla | Implementación |
|---|-------|----------------|
| 1 | No se aceptan filas con nulos en ventas | `df.dropna()` |
| 2 | Una venta no puede duplicarse | `dropDuplicates(["fecha","tienda_id","producto_id","cliente_id"])` |
| 3 | `cantidad` debe ser entero | `cast("int")` |
| 4 | `precio_unitario` debe ser decimal | `cast("double")` |
| 5 | `fecha` debe ser fecha válida `yyyy-MM-dd` | `to_date(col("fecha"), "yyyy-MM-dd")` |
| 6 | Integridad referencial | Inner joins contra clientes, productos y tiendas |

Las filas que incumplen alguna regla no se cargan en Silver. La traza queda disponible releyendo la tabla Bronze.

## 7. Salidas analíticas

| Tabla Gold | Pregunta de negocio | Granularidad |
|------------|---------------------|--------------|
| `gold_ventas_por_tienda` | ¿Cuánto vendió cada tienda cada día? | fecha × tienda |
| `gold_ventas_por_categoria` | ¿Qué categorías concentran el mayor monto? | categoría |
| `gold_top_productos` | ¿Cuáles son los productos más vendidos? | producto |

Métricas comunes: `total_unidades = SUM(cantidad)` y `total_ventas = SUM(cantidad * precio_unitario)`.

## 8. Seguridad

- Credenciales: la Access Key del Storage Account no se versiona. En el repositorio se usa el placeholder `TU_KEY_AQUI`. Se recomienda Databricks Secret Scopes respaldados por Azure Key Vault.
- Unity Catalog: las tablas se separan en tres schemas (`bronze_clase10`, `silver_clase10`, `gold_clase10`) para permitir permisos diferenciados por capa.
- Datos sensibles: el email de clientes se enmascara en el dataset de muestra.

## 9. Operación

- Reprocesamiento: cada notebook escribe con `mode("overwrite")` y `overwriteSchema=true`. Re-ejecutable sin estado intermedio.
- Orden: Bronze → Silver → Gold.
- Monitoreo: cada notebook imprime el conteo de filas por paso. Una caída inesperada indica problemas de calidad.
- Time travel: las tablas Delta exponen versiones consultables con `DESCRIBE HISTORY` y `VERSION AS OF`.

## 10. Roadmap

1. Orquestar los tres notebooks en un pipeline de Azure Data Factory con schedule trigger diario.
2. Incorporar aserciones de calidad (Great Expectations o módulo propio).
3. Conectar la capa Gold a Power BI.
4. Migrar Bronze a `mode("append")` para conservar el histórico crudo.
