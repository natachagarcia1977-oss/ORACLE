
**Ejercicio Técnico — Analista de Datos Jr.**

Q1) Estructura del Staging

El área de Recursos Humanos solicitó al equipo de Analytics un informe de auditoría salarial. Usted, como analista junior, debe desarrollar el código SQL para definir la estructura de una tabla de staging de sesión (gtt_employee_summary), pensada para alojar un resumen desnormalizado y pre-calculado de empleados a partir de la tabla employees.

gtt_employee_summary:

| Columna          | Tipo                 | Restricción / Rol                                                                |
| ---------------- | -------------------- | -------------------------------------------------------------------------------- |
| employee_id      | NUMBER               | PRIMARY KEY                                                                      |
| full_name        | VARCHAR2(100)        | Campo a calcular                                                                 |
| department_name  | VARCHAR2(50)         | Nombre del departamento                                                          |
| annual_salary    | NUMBER               | Campo a calcular                                                                 |
| diff_vs_dept_avg | NUMBER               | Campo a calcular                                                                 |
| seniority_level  | VARCHAR2(20)         | Campo a calcular                                                                 |
| load_date        | DATE DEFAULT SYSDATE | Se autocompleta con la fecha/hora del INSERT si no se especifica explícitamente. |

Q2)  Carga de Staging con Columnas Analíticas

Ya definio la estructura de `gtt_employee_summary` como tabla de staging de sesión. El área de Analytics ahora requiere que complete la **carga inicial** de esa tabla a partir de la fuente transaccional (`employees`, `departments`), aplicando las transformaciones de negocio acordadas con RRHH. Desarrolle el `INSERT INTO ... SELECT` que puebla `gtt_emp_summary`, cumpliendo los siguientes requerimientos funcionales:

1. **Identificación del empleado**: `employee_id` debe tomarse sin transformación desde la fuente.
2. `full_name` debe construirse concatenando nombre y apellido del empleado, separados por un espacio.
3. **Denormalización de departamento**: `department_name` debe incorporarse desde la tabla `departments`, contemplando que **pueden existir empleados sin departamento asignado** (la solución no debe excluir esos registros del resultado).
4. `annual_salary` debe representar el salario anualizado del empleado, a partir de su remuneración mensual.
5. `diff_vs_dept_avg` debe expresar la diferencia entre el salario anual del empleado y el salario anual promedio de su propio departamento, sin recurrir a una subconsulta agregada por separado (es decir, resuelto en la misma pasada del `SELECT` mediante funciones de ventana).   
6. `seniority_level` debe clasificar al empleado como `'Junior'`, `'Intermedio'` o `'Senior'` según su antigüedad en años completos, calculada a partir de su fecha de contratación:
    - Menos de 5 años → `'Junior'`
    - Entre 5 y 14 años → `'Intermedio'`
    - 15 años o más → `'Senior'`


Q3) Sobre el esquema HR, escribir una consulta en Oracle que muestre el número de identificación 
(employee_id) de todos los empleados que pertenecen al departamento de Ventas ("Sales").
