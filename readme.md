# [Nombre] [Carné]

## Indicaciones

Recientemente, se utilizó AI para crear un sistema de gestion de una biblioteca, el cual ha generado varios errores, su trabajo es arreglarlo. Dado el siguiente caso de uso, explique y/o resuelva cada problema según se le pida.

---

## Consideraciones

La libreria crea automaticamente un correo con los nombres de la persona

---

## Problemas

### 1. Filtro por autor y género (10%)

QA ha reportado que el endpoint para obtener los libros puede filtrar por **autor** y por **género**, o por cualquiera de los dos de manera individual.

Actualmente:

- Filtrar únicamente por autor funciona correctamente.
- Filtrar únicamente por género funciona correctamente.
- Filtrar por **autor y género al mismo tiempo** provoca que el servidor falle.

**Instrucción:** Explique la causa del problema y resuélvalo.
El error se origina en dos puntos de BookService.getAllBooks. Primero, la llamada tenía los argumentos invertidos: findByAuthorAndGenre(genre, author). Segundo, y más importante, el método derivado en BookRepository estaba declarado como findByAuthorAndGenre(String author, String genre), pero el campo genre de la entidad Book es del tipo enum Genre. Spring Data JPA construye la consulta comparando la propiedad genre (enum) contra un String, lo que produce un error de tipo en tiempo de ejecución y hace fallar el servidor. Filtrar solo por autor funcionaba porque findByAuthor no involucra el enum, y filtrar solo por género también funcionaba porque ahí el valor sí se convertía con Genre.valueOf(genre). Solución que se implemento Se cambió el segundo parámetro del método del repositorio de String a Genre, y en el service se corrigió el orden de los argumentos convirtiendo el género: findByAuthorAndGenre(author, Genre.valueOf(genre)).

---

### 2. Error al volver a prestar un libro (10%)

Un usuario reportó que al pedir prestado el libro **The Selfish Gene**, devolverlo e intentar pedirlo prestado nuevamente, el servidor falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

Error al volver a prestar un libro
Causa: The Selfish Gene tiene available_count = 1 en data.sql. Al prestarlo, el contador baja a 0 y el campo available se pone en false. Al devolverlo, el código incrementaba el contador nuevamente a 1 pero nunca volvía a poner available = true. Por eso, en el segundo préstamo la validación if (!book.isAvailable()) lanzaba "Book is not available", provocando la falla del servidor.
Solución: En la rama de devolución de MovementService.createMovement, después de incrementar el contador se restaura available = true cuando el contador queda mayor que 0.

---

### 3. Cantidad de libros por género (10%)

Existe un endpoint que devuelve la cantidad de libros disponibles por género. Sin embargo, actualmente dicho endpoint falla.

**Instrucción:** Explique la causa del problema y resuélvalo.

Cantidad de libros por género
Causa: El libro The Art of War tiene el campo genre en NULL en data.sql. En getGenresAvailable, al recorrer los libros se ejecuta book.getGenre().name(), lo que produce un NullPointerException al llegar a ese registro y tumba el endpoint.
Solución: Antes de contar, se omiten los libros cuyo género es null, evitando así la invocación de .name() sobre una referencia nula.


---

### 4. Error al consultar un libro por ID (10%)

Un miembro del equipo de frontend reporta que la siguiente llamada falla:

```http
GET /books?id=ed16ed1e-7017-4697-a08a-d28c09a74acf
```

**Instrucción:** Explique la causa del problema.

Error al consultar un libro por ID
Causa: El endpoint GET /books está mapeado al método getAllBooks(author, genre), el cual no declara ningún @RequestParam llamado id. La forma correcta de consultar un libro por su identificador es la ruta con path variable GET /books/{id}, atendida por el método getBookById. Al llamar GET /books?id=ed16ed1e-..., Spring ignora por completo el parámetro id y ejecuta getAllBooks con author = null y genre = null, devolviendo la lista completa de libros en lugar del libro esperado. Por eso, desde el frontend, la llamada "falla": no obtiene el recurso individual que se está solicitando.

---

### 5. Error al crear un libro (10%)

QA ha reportado que el siguiente payload enviado al endpoint `POST /books` provoca un error:

```json
{
  "title": "Clean Code",
  "author": "Robert C. Martin",
  "genre": "classic",
  "isbn": "978-0132350884",
  "available": true,
  "availableCount": 5
}
```

**Instrucción:** Explique la causa del problema.

Error al crear un libro
Causa: En createBook se hace Genre.valueOf(dto.getGenre()) directamente, sin normalizar el texto. Como las constantes del enum Genre están definidas en mayúsculas (CLASSIC), al enviar "genre": "classic" en minúsculas, Genre.valueOf("classic") lanza una IllegalArgumentException, produciendo el error. Cabe notar que el método updateBook sí aplica .toUpperCase() antes de convertir; createBook no lo hace, y esa inconsistencia es la causa del fallo.


---

### 6. Devolución de libros no prestados (20%)

QA ha reportado que un usuario es capaz de devolver libros que nunca ha solicitado en préstamo.

**Instrucción:**

- Confirme si este comportamiento es realmente posible.
- Si es posible, explique la causa y resuelva el problema.
- Si no es posible, explique por qué, haciendo referencia al código correspondiente.

¿Es posible? Sí, el comportamiento es real.
Causa: En MovementService.createMovement, la rama de RETURN únicamente incrementaba el contador del libro y registraba el movimiento, sin verificar que el lector tuviera un préstamo pendiente de ese libro. Como no existía ninguna validación que relacionara al lector con un préstamo previo, cualquier usuario podía devolver un libro que nunca solicitó. Solución: Se agregó el método countByLectorAndBookAndType(Lector, Book, MovementType) en MovementRepository. En la devolución, antes de modificar el libro, se comparan los préstamos y las devoluciones de ese lector para ese libro; si el número de devoluciones es mayor o igual al de préstamos (es decir, no hay ningún préstamo pendiente), se lanza una excepción y se bloquea la operación.

---
