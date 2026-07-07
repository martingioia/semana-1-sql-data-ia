# 📊 Semana 1 — Datos, SQL y la IA como copiloto

Evidencia práctica del bloque de introducción a SQL de la capacitación **Data + IA**. Se trabajó sobre el dataset **[Online Retail](https://www.kaggle.com/datasets/carrie1/ecommerce-data)** (Kaggle), un catálogo de transacciones de una tienda online del Reino Unido, usando **SQLite** (vía SQLiteOnline) como motor de práctica.

## 🗂️ Dataset

| Columna | Descripción |
|---|---|
| `invoiceno` | Número de factura/pedido |
| `stockcode` | Código del producto |
| `description` | Descripción del producto |
| `quantity` | Cantidad vendida (negativa = devolución) |
| `invoicedate` | Fecha y hora de la compra |
| `unitprice` | Precio unitario |
| `customerid` | ID del cliente (puede ser nulo) |
| `country` | País del cliente |

> Nota: al importar el CSV a SQLiteOnline, los nombres de columna quedaron en minúscula (`customerid` en vez de `CustomerID`), a diferencia del archivo original.

---

## Día 3 — GROUP BY, HAVING, JOIN

**Consigna:** escribir una query que cuente cuántos pedidos hizo cada cliente, ordenados de mayor a menor.

```sql
SELECT 
    customerid, 
    COUNT(DISTINCT invoiceno) AS total_pedidos
FROM online_retail
WHERE customerid IS NOT NULL
GROUP BY customerid
ORDER BY total_pedidos DESC;
```

**Por qué `COUNT(DISTINCT invoiceno)` y no `COUNT(invoiceno)`:** cada fila del dataset representa un producto dentro de un pedido, no un pedido completo. Sin `DISTINCT`, se contarían líneas de producto en vez de pedidos reales.

**Resultado (top 5):**

| customerid | total_pedidos |
|---|---|
| 14911 | 248 |
| 12748 | 224 |
| 17841 | 169 |
| 14606 | 128 |
| 15311 | 118 |

---

## Día 4 — La IA como copiloto (validación)

**Consigna:** validar el resultado de una query, sin confiar a ciegas en lo generado.

Se tomó el resultado del cliente con más pedidos (`14911`, con 248) y se verificó con una query independiente, aislando solo ese cliente:

```sql
SELECT COUNT(DISTINCT invoiceno) 
FROM online_retail 
WHERE customerid = 14911;
```

**Resultado:** `248` — coincide exactamente con el valor obtenido en la query agrupada del Día 3, confirmando que la lógica del `GROUP BY` es correcta.

---

## Día 5 — Mini-desafío: 5 preguntas de negocio

### 1. Top 10 clientes que más gastaron en total

```sql
SELECT 
    customerid,
    ROUND(SUM(quantity * unitprice), 2) AS total_gastado
FROM online_retail
WHERE customerid IS NOT NULL
GROUP BY customerid
ORDER BY total_gastado DESC
LIMIT 10;
```

| customerid | total_gastado |
|---|---|
| 14646 | 279,489.02 |
| 18102 | 256,438.49 |
| 17450 | 187,482.17 |
| 14911 | 132,572.62 |

**Hallazgo:** el cliente `14911`, que tenía más *cantidad* de pedidos (Día 3), no es el que más *gastó* en total — aparece 4to en gasto. Esto indica que hacía pedidos frecuentes pero de menor monto promedio, mientras que `14646` concentró más valor en menos operaciones.

### 2. Producto más vendido en cantidad

```sql
SELECT 
    description,
    SUM(quantity) AS unidades_vendidas
FROM online_retail
WHERE quantity > 0
GROUP BY description
ORDER BY unidades_vendidas DESC
LIMIT 10;
```

**Resultado:** *"PAPER CRAFT, LITTLE BIRDIE"* con 80,995 unidades.

**Hallazgo (calidad de datos):** en el top 10 apareció una fila con 32,547 unidades vendidas pero **descripción vacía** — evidencia de inconsistencias en el dataset que en un caso real ameritarían una limpieza previa antes de confiar en este ranking.

### 3. Cantidad de pedidos por país

```sql
SELECT 
    country,
    COUNT(DISTINCT invoiceno) AS total_pedidos
FROM online_retail
GROUP BY country
ORDER BY total_pedidos DESC;
```

| country | total_pedidos |
|---|---|
| United Kingdom | 23,494 |
| Germany | 603 |
| France | 461 |
| EIRE | 360 |

**Hallazgo:** el dataset está fuertemente concentrado en el Reino Unido (más de 38 veces el segundo país), consistente con que la tienda es británica.

### 4. Ticket promedio por pedido

```sql
SELECT 
    ROUND(AVG(total_por_pedido), 2) AS ticket_promedio
FROM (
    SELECT 
        invoiceno,
        SUM(quantity * unitprice) AS total_por_pedido
    FROM online_retail
    WHERE quantity > 0
    GROUP BY invoiceno
) AS pedidos;
```

**Resultado:** `$513.54` de ticket promedio por pedido.

**Por qué la subconsulta:** promediar `quantity * unitprice` directamente daría el promedio por *línea de producto*, no por pedido. Fue necesario calcular primero el total de cada pedido (subconsulta interna) y recién ahí promediar esos totales (consulta externa).

### 5. Mes con más ventas en total

```sql
SELECT 
    strftime('%Y-%m', invoicedate) AS mes,
    ROUND(SUM(quantity * unitprice), 2) AS ventas_totales
FROM online_retail
WHERE quantity > 0
GROUP BY mes
ORDER BY ventas_totales DESC
LIMIT 5;
```

> Nota: `strftime` es una función específica de SQLite; en Postgres el equivalente sería `TO_CHAR(invoicedate, 'YYYY-MM')`.

---

## 🛠️ Herramientas utilizadas

- **SQLiteOnline** — motor de práctica sin instalación
- **Claude** — copiloto para diseño de queries y validación de resultados

## 📌 Aprendizajes clave de la semana

- La diferencia entre contar *filas* y contar *entidades únicas* (`COUNT` vs `COUNT DISTINCT`) es crítica cuando una tabla tiene granularidad de detalle (línea de producto) distinta a la pregunta de negocio (pedido).
- Validar resultados con una query independiente y más simple es una forma rápida y confiable de detectar errores de lógica.
- Los datasets reales tienen problemas de calidad (nulos, descripciones vacías, inconsistencias) que aparecen naturalmente al hacer análisis — vale la pena reportarlos, no solo ignorarlos.
