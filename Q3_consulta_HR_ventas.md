# Q3) Empleados del departamento de Ventas ("Sales")

## Enunciado

Sobre el esquema HR, escribir una consulta en Oracle que muestre el número de identificación (`employee_id`) de todos los empleados que pertenecen al departamento de Ventas ("Sales").

## Solución con JOIN

```sql
SELECT e.employee_id
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE d.department_name = 'Sales';
```

**Cómo funciona:**
- `JOIN departments d ON e.department_id = d.department_id` conecta cada empleado con su departamento usando la clave foránea `department_id`.
- `WHERE d.department_name = 'Sales'` filtra solo los que pertenecen a Ventas.

## Solución alternativa con subconsulta

Si el examen pide estilo "clásico" sin `JOIN` explícito:

```sql
SELECT employee_id
FROM employees
WHERE department_id = (
    SELECT department_id
    FROM departments
    WHERE department_name = 'Sales'
);
```

> **Nota:** este enfoque solo funciona bien si el nombre del departamento es único (en el esquema HR estándar de Oracle sí lo es). Si hubiera más de un departamento con ese nombre, conviene usar `IN` en lugar de `=`.
