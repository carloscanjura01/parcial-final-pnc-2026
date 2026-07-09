# Carlos Canjura Paz 00021622

## Indicaciones

Recientemente, se utilizó AI para crear un sistema de gestion de una biblioteca, el cual ha generado varios errores, su trabajo es arreglarlo. Dado el siguiente caso de uso, explique y/o resuelva cada problema según se le pida.

---

## Consideraciones

La libreria crea automaticamente un correo con los nombres de la persona

---

## Problemas

### 1. Filtro por autor y género (10%)

Causa: el fallo se origina en dos defectos que se combinan en la capa de servicio y la capa de repositorio.

- Argumentos invertidos: en `BookService.getAllBooks` la llamada es `findByAuthorAndGenre(genre, author)`, es decir, se envía el género donde el repositorio espera el autor y viceversa.
- Tipo de dato incorrecto: el método derivado del repositorio se declaró como `findByAuthorAndGenre(String author, String genre)`, pero en la entidad Book el atributo genre es el enum `Genre`. Al ejecutar la consulta, Hibernate intenta enlazar un String a un parámetro de tipo enum y lanza `IllegalArgumentException: Parameter value did not match expected type Genre`, que el manejador global de excepciones traduce en un error 500.

Los filtros individuales sí funcionan porque `findByAuthor` recibe correctamente un String y `findByGenre` recibe el enum tras la conversión con `Genre.valueOf(genre)`. Solo la combinación de ambos filtros pasa por el método defectuoso.

Solución: corregir la firma del repositorio para que reciba el enum y corregir el orden y la conversión en el servicio.

```java

List<Book> findByAuthorAndGenre(String author, Genre genre);


if (author != null && genre != null) {
    return bookRepository.findByAuthorAndGenre(
        author, Genre.valueOf(genre.toUpperCase()));
}
```

---

### 2. Error al volver a prestar un libro (10%)

Causa: el libro The Selfish Gene tiene un único ejemplar (`availableCount = 1`). El flujo de préstamo y devolución maneja de forma asimétrica la bandera `available`:

- Al prestarlo, el contador baja de 1 a 0 y el código marca `available = false` correctamente.
- Al devolverlo, el contador sube de 0 a 1, pero la rama de devolución nunca restablece `available = true`. El estado queda inconsistente: hay 1 ejemplar en existencia pero el libro figura como no disponible.

En el segundo intento de préstamo, la validación `!book.isAvailable()` se cumple y se lanza `RuntimeException("Book is not available")`, que se convierte en un error 500 en el cliente.

Solución: restablecer la disponibilidad en la rama de devolución, ya que después de incrementar el contador siempre existe al menos un ejemplar.

```java
} else {
    book.setAvailableCount(book.getAvailableCount() + 1);
    book.setAvailable(true);
}
```

---

### 3. Cantidad de libros por género (10%)

Causa: en los datos semilla (`data.sql`), el libro The Art of War fue insertado con `genre = NULL`. El método `getGenresAvailable()` recorre todos los libros e invoca `book.getGenre().name()` sin validar nulos. Al llegar a ese registro se produce un `NullPointerException`, y el endpoint responde 500. Esto es posible porque la columna genre de la entidad no está marcada como `nullable = false`, a diferencia del resto de campos.

Solución: proteger el conteo contra géneros nulos, omitiendo esos registros (o agrupándolos bajo una categoría especial si el negocio lo requiere).

```java
for (Book book : books) {
    if (book.getGenre() == null) {
        continue;
    }
    String genreName = book.getGenre().name();
    countByGenre.put(genreName,
        countByGenre.getOrDefault(genreName, 0L) + 1);
}
```

---

### 4. Error al consultar un libro por ID (10%)

Causa: es un error de uso del contrato de la API, no un defecto interno. El endpoint de consulta por identificador está definido con una variable de ruta (path variable):

```java
@GetMapping("/{id}")
public ResponseEntity<Book> getBookById(@PathVariable UUID id)
```

La forma correcta de invocarlo es `GET /books/ed16ed1e-7017-4697-a08a-d28c09a74acf`. La llamada del frontend, `GET /books?id=...`, no coincide con esa ruta: Spring la enruta al endpoint de listado `GET /books`, que solo declara los parámetros opcionales `author` y `genre`. El parámetro `id` no está declarado en ese método, por lo que se ignora silenciosamente y la respuesta es la lista completa de libros en lugar del libro solicitado. Desde la perspectiva del frontend la llamada "falla" porque no recibe el recurso esperado, aunque el servidor responda 200.

---

### 5. Error al crear un libro (10%)

Causa: el payload envía `"genre": "classic"` en minúsculas, pero el enum define las constantes en mayúsculas (CLASSIC, CRIME, etc.). El método `createBook` realiza la conversión directa:

```java
book.setGenre(Genre.valueOf(dto.getGenre()));
```

`Genre.valueOf("classic")` lanza `IllegalArgumentException: No enum constant Genre.classic` porque `valueOf` exige coincidencia exacta con el nombre de la constante, incluyendo mayúsculas. El manejador global convierte la excepción en un error 500.

Es notable la inconsistencia del código generado por IA: el método `updateBook` sí normaliza con `dto.getGenre().toUpperCase()` antes de `valueOf`, pero `createBook` omite esa normalización. La corrección consistiría en aplicar la misma normalización en `createBook` (o validar el género y responder 400 Bad Request ante valores inválidos).

---

### 6. Devolución de libros no prestados (20%)

Confirmación: Sí, el comportamiento es posible.

Causa: el endpoint `POST /movements/return` delega en `createMovement(dto, MovementType.RETURN)`, el cual únicamente valida que el lector exista (búsqueda por correo) y que el libro exista (búsqueda por ISBN). No existe ninguna verificación de que el lector tenga un préstamo (BORROWING) activo sobre ese libro. En consecuencia, cualquier lector puede "devolver" cualquier libro cuantas veces quiera, y cada devolución incrementa `availableCount`, inflando el inventario indefinidamente y corrompiendo la integridad de los datos.

Solución: antes de procesar una devolución, comparar la cantidad de préstamos contra la cantidad de devoluciones de ese lector para ese libro. Si no hay préstamos pendientes, se rechaza la operación. Se agrega un método derivado al repositorio de movimientos:

```java

long countByLectorAndBookAndType(Lector lector, Book book,
                                 MovementType type);

long borrows = movementRepository
    .countByLectorAndBookAndType(lector, book,
        MovementType.BORROWING);
long returns = movementRepository
    .countByLectorAndBookAndType(lector, book,
        MovementType.RETURN);
if (borrows <= returns) {
    throw new RuntimeException(
        "Lector has no active borrowing for this book");
}
```