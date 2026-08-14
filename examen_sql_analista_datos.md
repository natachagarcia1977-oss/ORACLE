# Examen Técnico — Analista de Datos Jr.
## Resolución paso a paso (SQL para Oracle)

Antes de empezar, aclaro algunos conceptos que vas a necesitar para entender todo lo que viene. Si ya los conocés, podés saltar directo a la Q1.

---

## 🧠 Conceptos previos

- **Tabla de "puesta en escena" (staging)**: es una tabla intermedia donde se cargan datos ya transformados, lista para que otra área (en este caso RRHH) los use, sin tocar las tablas originales.
- **"De sesión" (Global Temporary Table - GTT)**: en Oracle existe un tipo especial de tabla llamada `GLOBAL TEMPORARY TABLE`. Su estructura (columnas) es permanente y la ven todos los usuarios, pero los **datos** que carga cada usuario/sesión son privados: nadie más los ve, y desaparecen cuando termina la sesión (o el commit, según cómo se configure). Es ideal para cálculos temporales como este informe.
- **Función de ventana (window function)**: es una función que hace un cálculo (como un promedio) **sin colapsar las filas**. Es decir, te da el promedio del departamento, pero seguís viendo cada empleado en su propia fila. Se reconoce por la cláusula `OVER (...)`.

---

## Q1) Estructura del Staging

### Paso 1: ¿Qué tipo de tabla necesito?

El enunciado pide una tabla "de puesta en escena de **sesión**". Eso es literalmente la definición de una `GLOBAL TEMPORARY TABLE` en Oracle. La sintaxis básica es:

```sql
CREATE GLOBAL TEMPORARY TABLE nombre_tabla (
    columna1 tipo,
    columna2 tipo
)
ON COMMIT PRESERVE ROWS;
```

- `ON COMMIT PRESERVE ROWS` → los datos se mantienen durante toda la sesión (hasta que cierres la conexión), aunque hagas `COMMIT`.
- La otra opción sería `ON COMMIT DELETE ROWS` (borra los datos apenas hacés commit), pero como queremos generar un informe y que los datos persistan mientras trabajamos, usamos `PRESERVE ROWS`.

### Paso 2: Traducir cada columna del enunciado a SQL

| Columna pedida | Tipo Oracle | Por qué |
|---|---|---|
| ID de empleado | `NUMBER PRIMARY KEY` | Es un número y es la clave que identifica cada fila de forma única |
| nombre_completo | `VARCHAR2(100)` | Texto, se va a calcular en el INSERT (Q2), acá solo definimos el "molde" |
| nombre_del_departamento | `VARCHAR2(50)` | Texto |
| salario_anual | `NUMBER` | Numérico, se calcula en el INSERT |
| diff_vs_dept_avg | `NUMBER` | Numérico, se calcula en el INSERT |
| nivel_de_antigüedad | `VARCHAR2(20)` | Texto tipo 'Junior' / 'Intermedio' / 'Senior' |
| fecha_de_carga | `DATE DEFAULT SYSDATE` | Si no le mandás un valor al insertar, Oracle pone la fecha/hora actual sola |

`SYSDATE` es una función interna de Oracle que devuelve la fecha y hora del servidor en el momento exacto de la operación (acá, del INSERT).

### Paso 3: Código final de la Q1

```sql
CREATE GLOBAL TEMPORARY TABLE gtt_employee_summary (
    employee_id        NUMBER          PRIMARY KEY,
    full_name           VARCHAR2(100),
    department_name     VARCHAR2(50),
    annual_salary        NUMBER,
    diff_vs_dept_avg    NUMBER,
    seniority_level      VARCHAR2(20),
    fecha_de_carga       DATE DEFAULT SYSDATE
)
ON COMMIT PRESERVE ROWS;
```

> 📌 Nota: usé los nombres de columna en inglés (`employee_id`, `full_name`, etc.) porque son los que aparecen explícitamente en el enunciado de la Q2 (`employee_id`, `full_name`, `department_name`, `annual_salary`, `diff_vs_dept_avg`, `seniority_level`). La tabla de la Q1 solo te muestra el nombre "traducido" para que entiendas qué representa cada columna, pero deben coincidir con los nombres reales que vas a usar al insertar.

---

## Q2) Carga de Staging con Columnas Analíticas

Ahora hay que llenar esa tabla con datos reales, transformados, usando un `INSERT INTO ... SELECT`. Este comando hace dos cosas a la vez: **lee** datos de otras tablas (`employees`, `departments`) y los **inserta** ya transformados en `gtt_employee_summary`.

Vamos requisito por requisito.

### Paso 1: `employee_id` — sin transformar

Se trae tal cual desde `employees`:

```sql
e.employee_id
```

### Paso 2: `full_name` — concatenar nombre y apellido

En Oracle, para pegar textos se usa el operador `||` (doble pipe). Ponemos un espacio `' '` en el medio:

```sql
e.first_name || ' ' || e.last_name AS full_name
```

### Paso 3: `department_name` — sin perder empleados sin departamento

Acá está el punto clave del ejercicio. Si usás un `JOIN` normal (`INNER JOIN`), Oracle **elimina** de los resultados a cualquier empleado cuyo `department_id` sea `NULL` o no coincida con ningún departamento. El enunciado dice explícitamente que eso no debe pasar.

La solución es usar `LEFT JOIN`: trae **todos** los empleados de `employees`, y si no tienen departamento coincidente en `departments`, simplemente pone `NULL` en `department_name` en vez de borrar la fila.

```sql
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
```

Y seleccionamos:
```sql
d.department_name
```

### Paso 4: `annual_salary` — anualizar el salario mensual

Si el salario en `employees` es mensual, para anualizarlo simplemente lo multiplicás por 12 (12 meses):

```sql
e.salary * 12 AS annual_salary
```

### Paso 5: `diff_vs_dept_avg` — diferencia contra el promedio del departamento, con función de ventana

Este es el requisito más avanzado. Piden la diferencia entre:
- el salario anual del empleado, y
- el promedio de salario anual **de su departamento**

...pero **sin usar una subconsulta aparte** (por ejemplo, algo como `(SELECT AVG(...) FROM employees WHERE department_id = e.department_id)`). En cambio, hay que resolverlo con una **función de ventana**, en la misma consulta.

La función de ventana que necesitamos es:

```sql
AVG(e.salary * 12) OVER (PARTITION BY e.department_id)
```

¿Qué hace esto?
- `AVG(e.salary * 12)` → calcula el promedio del salario anual.
- `OVER (PARTITION BY e.department_id)` → le dice a Oracle: "no calcules un solo promedio general, calculá **un promedio por cada departamento**", pero sin agrupar ni perder filas (a diferencia de `GROUP BY`, acá seguís viendo cada empleado individualmente).

Entonces la diferencia queda:

```sql
(e.salary * 12) - AVG(e.salary * 12) OVER (PARTITION BY e.department_id) AS diff_vs_dept_avg
```

### Paso 6: `seniority_level` — clasificar por antigüedad en años completos

Primero hay que calcular cuántos años completos pasaron desde `hire_date` (fecha de contratación) hasta hoy. Para eso usamos `MONTHS_BETWEEN`, que devuelve la diferencia en **meses** entre dos fechas, y después la dividimos por 12 para pasarla a años. Usamos `FLOOR` para quedarnos solo con los años **completos** (redondea siempre hacia abajo, así no cuenta un año que todavía no se cumplió del todo):

```sql
FLOOR(MONTHS_BETWEEN(SYSDATE, e.hire_date) / 12)
```

Después aplicamos la clasificación con `CASE WHEN`, que funciona como un "si... entonces... si no...":

```sql
CASE
    WHEN FLOOR(MONTHS_BETWEEN(SYSDATE, e.hire_date) / 12) < 5  THEN 'Junior'
    WHEN FLOOR(MONTHS_BETWEEN(SYSDATE, e.hire_date) / 12) BETWEEN 5 AND 14 THEN 'Intermedio'
    ELSE 'Senior'
END AS seniority_level
```

- Menos de 5 años → `Junior`
- Entre 5 y 14 años (ambos incluidos) → `Intermedio`
- 15 años o más → cae en el `ELSE`, o sea `Senior`

### Paso 7: Código final de la Q2

```sql
INSERT INTO gtt_employee_summary (
    employee_id,
    full_name,
    department_name,
    annual_salary,
    diff_vs_dept_avg,
    seniority_level
)
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS full_name,
    d.department_name,
    e.salary * 12 AS annual_salary,
    (e.salary * 12) - AVG(e.salary * 12) OVER (PARTITION BY e.department_id) AS diff_vs_dept_avg,
    CASE
        WHEN FLOOR(MONTHS_BETWEEN(SYSDATE, e.hire_date) / 12) < 5  THEN 'Junior'
        WHEN FLOOR(MONTHS_BETWEEN(SYSDATE, e.hire_date) / 12) BETWEEN 5 AND 14 THEN 'Intermedio'
        ELSE 'Senior'
    END AS seniority_level
FROM employees e
LEFT JOIN departments d
    ON e.department_id = d.department_id;
```

> Fijate que **no** incluimos `fecha_de_carga` en el `INSERT`: al dejarla afuera de la lista de columnas, Oracle aplica automáticamente el `DEFAULT SYSDATE` que definimos en la Q1. Eso es justamente lo que pide el enunciado ("se autocompleta... si no se especifica").

---

## ✅ Resumen de las herramientas SQL usadas

| Herramienta | Para qué la usamos |
|---|---|
| `GLOBAL TEMPORARY TABLE ... ON COMMIT PRESERVE ROWS` | Crear la tabla de staging de sesión |
| `DEFAULT SYSDATE` | Autocompletar fecha de carga |
| `\|\|` | Concatenar texto (nombre + apellido) |
| `LEFT JOIN` | No perder empleados sin departamento |
| `AVG(...) OVER (PARTITION BY ...)` | Promedio por grupo sin perder el detalle de cada fila (función de ventana) |
| `MONTHS_BETWEEN` + `FLOOR` | Calcular años completos de antigüedad |
| `CASE WHEN ... THEN ... ELSE ... END` | Clasificar en Junior / Intermedio / Senior |
