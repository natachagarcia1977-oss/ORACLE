# `get_hired_employees`

Procedimiento almacenado en PL/SQL que cuenta la cantidad de empleados contratados dentro de un rango de años determinado.

## Descripción

El procedimiento recibe un año de inicio y un año de finalización, valida que el rango sea correcto y, si lo es, calcula cuántos empleados de la tabla `employees` fueron contratados (`hire_date`) dentro de ese rango de años. El resultado se imprime mediante `dbms_output.put_line`.

## Firma

```sql
PROCEDURE get_hired_employees(
    p_start_year IN NUMBER,
    p_end_year   IN NUMBER
)
```

## Parámetros

| Parámetro       | Tipo   | Modo | Descripción                                  |
|-----------------|--------|------|-----------------------------------------------|
| `p_start_year`  | NUMBER | IN   | Año de inicio del rango de búsqueda.          |
| `p_end_year`    | NUMBER | IN   | Año de finalización del rango de búsqueda.    |

## Variables internas

| Variable  | Tipo   | Descripción                                                   |
|-----------|--------|-----------------------------------------------------------------|
| `v_count` | NUMBER | Almacena la cantidad de empleados contratados en el rango dado. |

## Lógica del procedimiento

1. Valida que `p_start_year` no sea mayor que `p_end_year`.
   - Si la validación falla, se imprime un mensaje de error indicando que el año de inicio no puede ser mayor al de finalización, y el procedimiento finaliza.
2. Si la validación es correcta:
   - Se ejecuta una consulta sobre la tabla `employees`, filtrando por el año extraído de la columna `hire_date` mediante `EXTRACT(YEAR FROM hire_date)`, dentro del rango `[p_start_year, p_end_year]`.
   - Se almacena el resultado en `v_count`.
   - Se imprime el valor de `v_count` mediante `dbms_output.put_line`.

## Requisitos previos

- Debe existir la tabla `employees` con una columna `hire_date` de tipo `DATE` (o compatible con `EXTRACT(YEAR FROM ...)`).
- El usuario que ejecuta el procedimiento debe tener permisos `SELECT` sobre la tabla `employees` y permisos de ejecución (`EXECUTE`) sobre el procedimiento.
- Para ver la salida de `dbms_output.put_line`, debe habilitarse la salida del servidor (por ejemplo, `SET SERVEROUTPUT ON` en SQL*Plus o SQL Developer).

## Código fuente

```sql
CREATE OR REPLACE PROCEDURE get_hired_employees(
    p_start_year IN NUMBER,
    p_end_year   IN NUMBER
) IS
    v_count NUMBER;
BEGIN
    IF (p_start_year > p_end_year) THEN
        dbms_output.put_line('El año de inicio no puede ser mayor al año de finalización');
    ELSE
        SELECT count(*)
          INTO v_count
          FROM employees
         WHERE EXTRACT(YEAR FROM hire_date) BETWEEN p_start_year AND p_end_year;

        dbms_output.put_line(v_count);
    END IF;
END get_hired_employees;
/
```

## Ejemplo de uso

```sql
SET SERVEROUTPUT ON;

BEGIN
    get_hired_employees(2010, 2015);
END;
/
```

**Salida esperada (ejemplo):**
```
42
```

## Ejemplo de caso inválido

```sql
BEGIN
    get_hired_employees(2020, 2015);
END;
/
```

**Salida esperada:**
```
El año de inicio no puede ser mayor al año de finalización
```

## Consideraciones y posibles mejoras

- **Manejo de excepciones**: el procedimiento no captura excepciones (por ejemplo, `NO_DATA_FOUND` o errores de tipo de dato). Se recomienda agregar un bloque `EXCEPTION` para mayor robustez.
- **Uso de `dbms_output`**: es útil para pruebas y depuración, pero no es apropiado para uso en producción o integraciones. Se podría reemplazar por un parámetro de salida (`OUT NUMBER`) para que el valor sea utilizable por otros programas.
- **Validación de parámetros**: no se valida que los años sean valores razonables (por ejemplo, años negativos o futuros). Podría agregarse validación adicional según las reglas de negocio.
- **Rendimiento**: si la tabla `employees` es muy grande, considerar un índice funcional sobre `EXTRACT(YEAR FROM hire_date)` para optimizar la consulta.

## Historial de cambios

| Versión | Fecha       | Descripción              |
|---------|-------------|---------------------------|
| 1.0     | -           | Versión inicial del procedimiento. |
