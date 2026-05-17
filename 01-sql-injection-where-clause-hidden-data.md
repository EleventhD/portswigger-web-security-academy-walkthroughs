## 1. Información general

| Campo | Detalle |
|---|---|
| Plataforma | PortSwigger Web Security Academy |
| Learning Path | SQL Injection |
| Categoría | SQL Injection en cláusula `WHERE` |
| Dificultad | Apprentice / Basic |

---

## 2. Objetivo del laboratorio

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de productos. Cuando el usuario selecciona una categoría, la aplicación ejecuta una consulta SQL como la siguiente:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

Para resolver el laboratorio, se debe realizar un ataque de inyección SQL que provoque que la aplicación muestre uno o más productos no publicados.

### 2.1 Primera observación del objetivo

Para este laboratorio, el objetivo es claro: modificar la petición HTTP para lograr que la aplicación muestre productos no publicados, es decir, productos que normalmente no deberían mostrarse porque no cumplen la siguiente condición:

```sql
released = 1
```

---


## 3. Reconocimiento

### 3.1 Primer vistazo

El primer vistazo al laboratorio nos presenta un aplicativo sencillo. Se muestra un listado de distintos productos y una sección de búsqueda o filtrado. Cerca de esta sección podemos encontrar filtros prefabricados que permiten navegar entre distintas categorías.

<img width="1920" height="1200" alt="Screenshot_2026-05-14_23_39_48" src="https://github.com/user-attachments/assets/4f9ffedd-90fc-4c43-a03a-b7b51b41ab31" />

En la barra de direcciones del navegador podemos observar parte de la petición HTTP que se envía al backend:

```http
GET /filter?category=Accessories HTTP/2
```

De acuerdo con la descripción del laboratorio, este parámetro es utilizado por la aplicación para construir una consulta SQL similar a la siguiente:

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

Aquí ya podemos identificar cuál es la parte de la consulta donde debemos realizar la inyección. Dentro de la petición HTTP, el valor controlado por el usuario se encuentra en el parámetro:

```http
category=Accessories
```

De acuerdo con el comportamiento del aplicativo, el valor dentro de `category` cambia para cumplir la función de filtrado por categoría. Adicionalmente, la aplicación aplica de forma implícita un segundo filtro:

```sql
released = 1
```

Este segundo filtro limita los resultados únicamente a productos publicados.

---

### 3.2 SQL Injection

El parámetro `category` es controlado por el usuario y parece ser utilizado directamente por la aplicación para construir la consulta SQL del filtro de productos.

Como el valor `released` no es modificable directamente desde la petición HTTP, debemos aplicar la inyección dentro del parámetro `category`.

El objetivo es alterar la lógica de la consulta SQL para que la aplicación ignore la condición:

```sql
AND released = 1
```

Para ello, probamos inyectando una comilla simple y un comentario SQL después del valor de la categoría:

```http
GET /filter?category=Accessories'-- HTTP/2
```

La consulta SQL resultante quedaría conceptualmente de la siguiente manera:

```sql
SELECT * FROM products WHERE category = 'Accessories'--' AND released = 1
```

El objetivo de esta inyección es cerrar el valor de `category` utilizando una comilla simple `'` y después comentar todo lo que siga en la consulta mediante `--`.

De esta manera, la parte final de la consulta deja de ejecutarse:

```sql
AND released = 1
```

<img width="1920" height="1200" alt="Screenshot_2026-05-14_23_56_30" src="https://github.com/user-attachments/assets/f95f338d-905d-4189-bae7-6b2c1f71ecf7" />

---

### 3.3 Resultado obtenido

Al probar esta petición HTTP modificada, logramos obtener lo siguiente:

<img width="1920" height="1200" alt="Screenshot_2026-05-14_23_59_03" src="https://github.com/user-attachments/assets/8603ec20-b1ce-4992-8a2d-bfb84df5bbe9" />

Con esto podemos dar por cumplido el objetivo del laboratorio. Logramos visualizar productos dentro de la tienda que inicialmente no podíamos ver debido a que no correspondían al valor:

```sql
released = 1
```

---

## 4. Explicación del laboratorio

Este laboratorio demuestra una vulnerabilidad básica de **SQL Injection** dentro de una cláusula `WHERE`.

La aplicación permite filtrar productos por categoría mediante el parámetro `category`, el cual es enviado en la petición HTTP:

```http
GET /filter?category=Accessories HTTP/2
```

El problema ocurre porque la aplicación aparentemente toma el valor de este parámetro y lo concatena directamente dentro de una consulta SQL, sin validar ni parametrizar adecuadamente la entrada del usuario.

La consulta original esperada sería similar a esta:

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

Esta consulta tiene dos condiciones principales:

| Condición | Propósito |
|---|---|
| `category = 'Accessories'` | Mostrar únicamente productos de la categoría seleccionada |
| `released = 1` | Mostrar únicamente productos publicados |

El objetivo del laboratorio es lograr que se muestren productos no publicados. Para conseguirlo, no necesitamos modificar directamente el valor de `released`, ya que este campo no está expuesto en la petición HTTP. En su lugar, debemos alterar la lógica de la consulta desde el parámetro que sí controlamos: `category`.

El payload utilizado fue:

```sql
Accessories'--
```

Este payload funciona de la siguiente manera:

| Parte del payload | Función |
|---|---|
| `Accessories` | Valor legítimo de la categoría |
| `'` | Cierra la cadena SQL original |
| `--` | Comenta el resto de la consulta SQL |

Al enviar el payload, la consulta queda conceptualmente así:

```sql
SELECT * FROM products WHERE category = 'Accessories'--' AND released = 1
```

Todo lo que aparece después de `--` es tratado como comentario por la base de datos. Por lo tanto, la condición:

```sql
AND released = 1
```

queda anulada y ya no se aplica al resultado final.
