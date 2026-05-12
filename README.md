# Pipeline de Datos — Arquitectura Medallion en Azure

![Arquitectura](architecture.png)

## Descripción

Pipeline end-to-end que ingesta archivos CSV de ventas diarias desde Azure Blob Storage, los procesa en Azure Databricks aplicando la arquitectura Medallion (Bronze / Silver / Gold) sobre Delta Lake y los expone como tablas analíticas listas para consumo de BI.

## Tecnologías

- Azure Blob Storage
- Azure Data Factory
- Azure Databricks
- Delta Lake
- PySpark
- Unity Catalog

## Arquitectura

| Capa | Descripción |
|------|-------------|
| Bronze | Ingesta de CSV crudos. Todas las columnas como `string`, sin transformaciones. |
| Silver | Limpieza, deduplicación, casts de tipo y enriquecimiento con joins contra maestras. |
| Gold | Agregaciones por dimensión de negocio (tienda, categoría, producto). |

Cada capa se escribe con `mode("overwrite")` y es reproducible de forma independiente.

## Estructura del repositorio

```
pipeline-medallion-azure/
├── notebooks/
│   ├── 01_Bronze_Ingesta.ipynb
│   ├── 02_Silver_Limpieza_Enriquecimiento.ipynb
│   └── 03_Gold_Agregaciones.ipynb
├── data/
│   └── sample/
│       ├── clientes_sample.csv
│       ├── productos_sample.csv
│       ├── tiendas_sample.csv
│       └── ventas_diarias_sample.csv
├── docs/
│   └── documento_funcional.md
├── architecture.png
├── .gitignore
└── README.md
```

## Validaciones (capa Silver)

1. Eliminación de filas con nulos en la tabla de hechos (`dropna`).
2. Deduplicación por clave de negocio: `fecha + tienda_id + producto_id + cliente_id`.
3. Casts de tipo: `fecha → date`, `cantidad → int`, `precio_unitario → double`.
4. Integridad referencial: inner joins contra `clientes`, `productos` y `tiendas`. Las ventas con referencias inexistentes quedan excluidas.

## Resultados

> *[CAPTURA: gold_top_productos — ranking por total_ventas]*

> *[CAPTURA: gold_ventas_por_categoria — unidades y ventas por categoría]*

> *[CAPTURA: gold_ventas_por_tienda — granularidad diaria por tienda]*

## Cómo ejecutar

1. Clonar el repositorio.
2. Crear un Storage Account y un container `medallion` en Azure. Cargar los CSV de `data/sample/` (o datos propios con el mismo schema).
3. En un workspace de Azure Databricks con Unity Catalog, crear un Volume y ajustar la ruta `BASE_VOL` del notebook Bronze.
4. En `01_Bronze_Ingesta.ipynb` reemplazar `TU_KEY_AQUI` por la Access Key del Storage Account.
5. Ejecutar los notebooks en orden: Bronze → Silver → Gold.
6. Consultar la capa Gold desde el Catalog Explorer o vía SQL:
   ```sql
   SELECT * FROM dbx_diplo_ricardo.gold_clase10.gold_top_productos LIMIT 10;
   ```

## Autor

**Ricardo Alvarez**

[linkedin.com/in/ricardo-andres-alvarez](https://www.linkedin.com/in/ricardo-andres-alvarez/) · [ricardoalvarez913@gmail.com](mailto:ricardoalvarez913@gmail.com)
