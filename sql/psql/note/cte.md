在 PostgreSQL 中，CTE（Common Table Expressions，通用表表达式）是一个用于在单个查询中定义临时结果集的语法结构。这些临时结果集在查询的主语句中可以被多次引用。CTE 是通过 `WITH` 关键字引入的。

使用 CTE 可以使查询更具可读性，尤其是在需要多次引用同一结果集或在复杂查询中分步骤处理数据时。

### 基本语法

```sql

WITH cte_name AS (

    -- 定义 CTE 的查询

    SELECT column1, column2

    FROM some_table

    WHERE condition

)

-- 使用 CTE 的查询

SELECT column1, column2

FROM cte_name

WHERE condition;

```

### 示例 1：简单 CTE

以下示例展示了如何使用 CTE 计算部门员工的平均工资，并查询工资高于平均工资的员工。

```sql

WITH avg_salaries AS (

    SELECT department, AVG(salary) AS avg_salary

    FROM employees

    GROUP BY department

)

SELECT e.name, e.salary, a.avg_salary

FROM employees e

JOIN avg_salaries a ON e.department = a.department

WHERE e.salary > a.avg_salary;

```

### 示例 2：递归 CTE

递归 CTE 用于处理层次结构或递归关系，例如员工和管理层关系树。递归 CTE 由两部分组成：锚点成员和递归成员。

以下示例展示了如何使用递归 CTE 计算公司中的员工层级：

```sql

WITH RECURSIVE employee_hierarchy AS (

    -- 锚点成员

    SELECT id, name, manager_id, 1 AS level

    FROM employees

    WHERE manager_id IS NULL

    UNION ALL

    -- 递归成员

    SELECT e.id, e.name, e.manager_id, eh.level + 1

    FROM employees e

    JOIN employee_hierarchy eh ON e.manager_id = eh.id

)

SELECT id, name, manager_id, level

FROM employee_hierarchy

ORDER BY level;

```

### 示例 3：多次引用 CTE

以下示例展示了如何在一个查询中多次引用 CTE 结果集。

```sql

WITH department_totals AS (

    SELECT department, COUNT(*) AS employee_count, SUM(salary) AS total_salary

    FROM employees

    GROUP BY department

)

-- 查询部门的员工总数和工资总额

SELECT department, employee_count, total_salary

FROM department_totals

UNION ALL

-- 查询平均工资

SELECT department, NULL AS employee_count, total_salary / employee_count AS avg_salary

FROM department_totals;

```

### 组合多个 CTE

你可以在一个 `WITH` 语句中定义多个 CTE，并在主查询中引用这些 CTE。

```sql

WITH cte1 AS (

    SELECT column1, column2

    FROM table1

    WHERE condition1

), cte2 AS (

    SELECT column3, column4

    FROM table2

    WHERE condition2

)

SELECT c1.column1, c2.column3

FROM cte1 c1

JOIN cte2 c2 ON c1.column2 = c2.column4;

```

### 完整示例

假设有一个员工表 `employees`，包含以下数据：

```sql

CREATE TABLE employees (

    id SERIAL PRIMARY KEY,

    name VARCHAR(100),

    department VARCHAR(100),

    salary NUMERIC,

    manager_id INT

);

INSERT INTO employees (name, department, salary, manager_id) VALUES

('Alice', 'Engineering', 60000, NULL),

('Bob', 'Engineering', 55000, 1),

('Charlie', 'HR', 50000, NULL),

('David', 'HR', 45000, 3),

('Eve', 'Engineering', 50000, 1);

```

#### 使用 CTE 计算平均工资并查询高于平均工资的员工：

```sql

WITH avg_salaries AS (

    SELECT department, AVG(salary) AS avg_salary

    FROM employees

    GROUP BY department

)

SELECT e.name, e.salary, a.avg_salary

FROM employees e

JOIN avg_salaries a ON e.department = a.department

WHERE e.salary > a.avg_salary;

```

#### 使用递归 CTE 计算员工层级：

```sql

WITH RECURSIVE employee_hierarchy AS (

    SELECT id, name, manager_id, 1 AS level

    FROM employees

    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.name, e.manager_id, eh.level + 1

    FROM employees e

    JOIN employee_hierarchy eh ON e.manager_id = eh.id

)

SELECT id, name, manager_id, level

FROM employee_hierarchy

ORDER BY level;

```

通过这些示例，你可以看到 CTE 在简化复杂查询和处理递归数据结构方面的强大功能。