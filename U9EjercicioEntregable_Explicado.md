# Interpretación del entregable: AppConsola (C# + Entity Framework Core)

> Guía de estudio para principiantes absolutos. Acá vamos a leer el código del
> proyecto `U9EjercicioEntregable` **línea por línea** y traducirlo a lenguaje
> humano. Al final de cada ejercicio hay **ejercicios de práctica del mismo nivel**.

---

## ¿Qué es este proyecto en una frase?

Es una **aplicación de consola** (una app que corre en la ventana negrita y se
maneja con el teclado) para administrar una **biblioteca**: podés registrar
Autores, Categorías y Libros, listarlos, modificarlos, eliminarlos y consultar
datos con **LINQ** (una forma de hacer preguntas a colecciones de datos usando
un lenguaje parecido al inglés: `Where` = dónde, `OrderBy` = ordenar por, etc.).

Y todo lo que guardás queda **almacenado en una base de datos SQLite** (un
archivo de base de datos) usando una tecnología llamada **Entity Framework Core**.
Pero no te asustes: para interpretar el código no hace falta saber todo eso,
solo entender qué hace cada línea.

---

# PARTE 1: RESOLUCIÓN Y EXPLICACIÓN DE CADA EJERCICIO

## Ejercicio 1 — El menú: `while` + `switch`

### a) CÓDIGO ORIGINAL

```csharp
bool continuar = true;

while (continuar)
{
    Console.WriteLine("1. Alta Autor");
    // ... más opciones de menú ...

    Console.Write("Seleccione una opción: ");
    string opcion = Console.ReadLine();

    Console.Clear();

    switch (opcion)
    {
        case "1":
            AltaAutor();
            break;

        case "0":
            continuar = false;
            Console.WriteLine("Aplicación finalizada.");
            break;

        default:
            Console.WriteLine("Opción inválida.");
            PresioneParaContinuar();
            break;
    }
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender qué hace el programa desde que arranca: ¿por qué se repite el menú?,
¿qué pasa cuando el usuario escribe un número?, ¿qué hace que el menú "termine"?

### c) EXPLICACIÓN PASO A PASO

Empecemos desde cero. Piensen en el programa como un **restaurante que atiende
las 24 horas**: siempre está esperando que un cliente pida algo. Cuando el cliente
le da una orden, la cocina la prepara... y después el restaurante **vuelve a
esperar la próxima orden**. Solo cierra cuando el dueño dice "apagamos todo".

Eso es exactamente el `while (continuar)`. `while` significa "mientras",
`continuar` significa "seguir".

- Línea `bool continuar = true;`: creamos una "cajita" llamada `continuar`
  que solo puede guardar dos valores: `true` (verdadero) o `false` (falso).
  La encendemos con `true` porque el programa **todavía no quiere terminar**.

- `while (continuar)`: mientras `continuar` sea `true`, se repite todo lo que
  está adentro de las llaves `{ }`. Es como decir: "**mientras el semáforo esté
  en verde, seguí manejando**".

- Dentro del bucle se imprime el menú con `Console.WriteLine(...)` y después
  preguntamos con `Console.Write("Seleccione una opción: ")`.

  > Diferencia importantísima:
  > - `WriteLine` escribe una línea y **salta a la siguiente**.
  > - `Write` escribe y **deja el cursor en la misma línea**.

- `string opcion = Console.ReadLine();`: la línea `Console.ReadLine()` hace una
  **pausa y espera que el usuario escriba algo y apriete ENTER**. Todo lo que
  escribió (por ejemplo "3") se guarda en la cajita `opcion`. OJO: siempre se
  guarda como **texto** (`string`), aunque el usuario haya escrito un número.

- `Console.Clear();`: limpia la pantalla para que el menú vuelva a verse prolijo.

- Luego viene el `switch (opcion)`. El `switch` funciona como un **portero de
  boliche**: mira lo que hay en la cajita `opcion` y decide a qué caso mandarlo.

  - Si `opcion` es `"1"` (el texto "1") → llama a la función `AltaAutor()`.
  - Si `opcion` es `"0"` → le asigna `false` a `continuar` y dice
    "Aplicación finalizada. **Ojo: acá la cajita cambió a false**".
  - `default` (por defecto) → es el caso "no era ninguno de los anteriores".
    Por ejemplo si el usuario escribió "hola". Se imprime "Opción inválida".

Cuando el `switch` termina, el bucle `while` vuelve a **chequear la cajita
`continuar`**:

| Línea | Código                    | continuar | opcion | ¿Qué se ve en pantalla? |
|-------|---------------------------|-----------|--------|-------------------------|
| 1     | `bool continuar = true;`  | `true`    | -      | - |
| 10    | `while (continuar)`       | `true`    | -      | se entra al menú |
| 39    | `Console.Write(...)`      | `true`    | -      | "Seleccione una opción: " |
| 40    | `ReadLine()` → "3"        | `true`    | "3"    | (el usuario escribe) |
| 44    | `switch (opcion)`         | `true`    | "3"    | - |
| 54    | `case "3": AltaLibro()`   | `true`    | "3"    | menu de alta de libro |
| 10    | vuelve al `while`         | `true`    | -      | se muestra el menú otra vez |
| 40    | `ReadLine()` → "0"        | `true`    | "0"    | (el usuario escribe) |
| 106   | `case "0":`               | `false`   | "0"    | "Aplicación finalizada." |
| 10    | `while (continuar)`       | `false`   | -      | **el bucle termina** |

### d) RESULTADO FINAL

El programa muestra el menú **una y otra vez** hasta que el usuario elige `0`.
Cuando elige `0`, `continuar` pasa a `false` y el bucle `while` se detiene:
la aplicación termina.

### e) CONCEPTOS QUE APARECEN

- **Variable**: una cajita con nombre que guarda un valor.
- **bool**: tipo de dato que solo vale `true` o `false`.
- **string**: tipo de dato que guarda texto.
- **Bucle `while`**: repite un bloque mientras la condición sea verdadera.
- **`switch` / `case`**: estructura de decisión múltiple, "si es esto, hacé esto".
- **`break`**: marca el final de cada caso del `switch`.
- **Función/método**: un bloque de código con nombre que se puede llamar (invocar)
  cuando se necesita. Ej: `AltaAutor()`.

### f) ⚠️ OJO / ERRORES COMUNES

- **`case "1"` con las comillas**. El usuario escribió el *texto* `"1"`, por eso
  el case usa comillas. Si ponés `case 1:` (sin comillas) nunca va a coincidir,
  porque `opcion` es un `string`, no un número.
- **`=` no es lo mismo que `==`**. En el `while` se **evalúa** con lo que ya vale
  la cajita (`continuar` es `bool`, no hace falta comparar). Escribir
  `while (continuar == false)` también funciona; escribir `while (continuar = true)`
  es un **error lógico**: asigna, no compara.
- **Olvidarse del `break`**: sin `break`, el switch "se cae" al siguiente caso
  (en C#, incluso da error de compilación en muchos casos).
- **`ReadLine()` siempre devuelve texto**: aunque el usuario escriba "7", en la
  variable `opcion` queda el texto `"7"`.

---

## Ejercicio 2 — Alta de un Autor (crear un objeto y guardarlo)

### a) CÓDIGO ORIGINAL

```csharp
void AltaAutor()
{
    Console.Write("Nombre del autor: ");

    Autor autor = new Autor
    {
        Nombre = Console.ReadLine()
    };

    autorRepository.Agregar(autor);

    Console.WriteLine("Autor registrado correctamente.");

    PresioneParaContinuar();
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender cómo se crea un "Autor" a partir de lo que tipea el usuario y cómo se
guarda en la base de datos.

### c) EXPLICACIÓN PASO A PASO

- `void AltaAutor()`: `void` significa "vacío" → esta función **no devuelve nada**,
  solo hace su trabajo. El paréntesis vacío `()` significa que no recibe datos
  de entrada.

- `Console.Write("Nombre del autor: ")`: preguntamos al usuario (nota: con
  `Write`, no `WriteLine`, para que escriba a la derecha de la pregunta).

- `Autor autor = new Autor { Nombre = Console.ReadLine() };`: vamos a desarmarla
  en tres pedazos:

  1. `Console.ReadLine()` → espera que el usuario escriba "Gabriel García Márquez"
     (por ejemplo) y devuelve ese texto.
  2. `new Autor { Nombre = ... }` → **fabrica** un objeto nuevo de tipo `Autor`.
     Los corchetes `{ }` son un **inicializador de objetos**: le asigné un valor
     a la propiedad `Nombre` *en el mismo momento* en que lo fabrico.
  3. `Autor autor = ...` → a esa "ficha de autor" recién fabricada la guardo en
     la cajita `autor`.

  Traducción: *"Hace una ficha nueva de autor y escribile su nombre"*.

- `autorRepository.Agregar(autor);`: le pasamos la ficha al **repositorio**
  (el encargado de hablar con la base de datos) con el método `Agregar`. Este
  método inserta el autor en la base de datos.

- `Console.WriteLine("Autor registrado correctamente.");`: avisa que salió todo bien.

- `PresioneParaContinuar();`: pausa (`Console.ReadKey()`) hasta que el usuario
  toque una tecla y limpia la pantalla. (Podés verlo al final del `Program.cs`).

| Línea | Código | entrada del usuario | objeto `autor` | base de datos |
|-------|--------|---------------------|----------------|---------------|
| 1     | `Write(...)` | - | - | - |
| 6     | `ReadLine()` | "Gabriel García Márquez" | - | - |
| 7     | `new Autor { Nombre = ... }` | - | Nombre = "Gabriel García Márquez" | - |
| 9     | `autorRepository.Agregar(autor)` | - | Nombre = "... (id asignado)" | ✔ autor insertado |

### d) RESULTADO FINAL

El usuario escribe un nombre, se crea un objeto `Autor`, se guarda en la base de
datos y se imprime "Autor registrado correctamente."

### e) CONCEPTOS QUE APARECEN

- **Clase**: un molde/plantilla que define qué datos tiene una cosa (un Autor).
- **Objeto**: un ejemplar concreto creado a partir del molde.
- **`new`**: palabra clave para crear un objeto nuevo.
- **Propiedad**: un dato que vive dentro del objeto (ej. `Autor.Nombre`).
- **Inicializador de objetos**: llenar propiedades dentro de las llaves al crear.
- **Método que no devuelve nada (`void`)**.
- **Repositorio**: capa que se encarga de guardar/leer datos.

### f) ⚠️ OJO / ERRORES COMUNES

- **Crear el objeto pero no asignarle el nombre**: `new Autor()` sin las llaves
  deja la propiedad `Nombre` vacía y en la base aparecería un autor sin nombre.
- **Confundir la clase con el objeto**: `Autor` (con mayúscula) es la plantilla;
  `autor` (con minúscula) es la cajita que guarda la ficha concreta.
- **Olvidar `Agregar`**: si creás el objeto pero nunca lo pasás al repositorio,
  se pierde nada más termina la función.

---

## Ejercicio 3 — Listar elementos con `foreach` y texto con `$`

### a) CÓDIGO ORIGINAL

```csharp
void MostrarAutores()
{
    Console.WriteLine("===== AUTORES =====");

    var autores = autorRepository.ObtenerTodos();

    foreach (var autor in autores)
    {
        Console.WriteLine(
            $"ID: {autor.Id} | Nombre: {autor.Nombre}");
    }

    PresioneParaContinuar();
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender cómo se recorren todos los autores guardados para imprimirlos, uno por
línea.

### c) EXPLICACIÓN PASO A PASO

- `var autores = autorRepository.ObtenerTodos();`
  - `ObtenerTodos()` pregunta a la base de datos **"dame todos los autores"** y
    devuelve una **lista** (una colección con varios elementos).
  - `var` es una "cajita que se entera sola del tipo": no hace falta que diga
    `List<Autor>` porque el compilador lo deduce de `ObtenerTodos()`.

- `foreach (var autor in autores)`: `foreach` significa **"por cada uno"**.
  Traducción: *"por cada autor que haya en la lista `autores`, hacé lo que viene
  entre las llaves"*. Es como repartir cartas: vas carta por carta y mirás cada una.

  - El `foreach` no necesita decir cuántas veces repetir: él solo itera hasta que
    la lista se termina.

- Dentro del `foreach`:
  ```csharp
  Console.WriteLine($"ID: {autor.Id} | Nombre: {autor.Nombre}");
  ```
  El `$` al inicio de un texto lo convierte en **texto con huequitos** (`{ }`).
  Donde va `{autor.Id}` se reemplaza por el ID real de ese autor, y donde va
  `{autor.Nombre}` se reemplaza por su nombre. Esto se llama **interpolación de
  cadenas**.

  Ejemplo si en la lista hay dos autores:
  | Iteración | `autor` | texto armado que imprime |
  |-----------|---------|--------------------------|
  | 1ª        | Id=1, Nombre="Gabriel García Márquez"  | `ID: 1 \| Nombre: Gabriel García Márquez` |
  | 2ª        | Id=2, Nombre="Isabel Allende"          | `ID: 2 \| Nombre: Isabel Allende` |
  | 3ª        | ya no hay más → el foreach **termina** | - |

### d) RESULTADO FINAL

Se imprime un título `===== AUTORES =====` y después una línea por cada autor,
así: `ID: 1 | Nombre: Gabriel García Márquez`. Si no hay autores, no se imprime
ninguna línea (solo el título).

### e) CONCEPTOS QUE APARECEN

- **Lista (`List<T>`)**: colección ordenada que guarda varios elementos.
- **`foreach`**: bucle pensado para recorrer cada elemento de una colección.
- **`var`**: "ni idea del tipo, deducilo vos, compilador".
- **Interpolación `$"..."`**: texto con huecos que se rellenan con valores.

### f) ⚠️ OJO / ERRORES COMUNES

- **Usar el plural como variable individual**: en el ciclo, la variable es
  `autor` (un solo autor), no `autores` (toda la lista). `foreach (var autor in autores)`.
- **Olvidar el `$`**: si ponés `"ID: {autor.Id}"` sin `$`, el programa imprime
  literalmente el texto `{autor.Id}` en vez del valor.
- **Confundir `foreach` con `for`**: `for` usa un índice (`i = 0; i < n; i++`) y
  sirve cuando necesitás saber "en qué posición estoy". `foreach` es más simple:
  solo te da cada elemento.
- **Los `{ }` del `$"` no se escapan fácil**: si el texto necesita llaves
  literales, hay que duplicarlas (`{{` y `}}`). Para un principiante, no.

---

## Ejercicio 4 — Filtrar libros activos: `!libros.Any()` y `Where(...)`

### a) CÓDIGO ORIGINAL

```csharp
var libros = libroRepository.ObtenerTodosCon("Autor");

if (!libros.Any())
{
    Console.WriteLine("No existen libros registrados.");
}
else
{
    foreach (var libro in libros.Where(l => l.Activo))
    {
        Console.WriteLine(
            $"ID: {libro.Id} | " +
            $"Título: {libro.Titulo} | " +
            $"Año: {libro.AnioPublicacion} | "+
            $"Autor: {libro.Autor.Nombre}");
    }
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender qué hace `.Any()`, qué significa el `!` delante, y cómo se quedan solo
los libros *activos* usando `Where`.

### c) EXPLICACIÓN PASO A PASO

- `ObtenerTodosCon("Autor")`: devuelve todos los libros **incluyendo** la info
  del autor de cada uno (porque los libros guardan `AutorId`, no el nombre; el
  método le pega el objeto `Autor` completo para poder mostrar `libro.Autor.Nombre`).

- `libros.Any()`: `Any()` pregunta **"¿hay por lo menos UN elemento?"**.
  Devuelve `true` si hay al menos un libro, `false` si la lista está vacía.

- `!libros.Any()`: el `!` significa **NO**, es la **negación**. Entonces
  `!libros.Any()` se lee: *"¿es cierto que NO hay ni un solo libro?"*.

  - Si es `true` (no hay libros) → imprime "No existen libros registrados."
  - Si es `false` (sí hay libros) → entra al `else` y los muestra.

  Traducción completa del `if`: *"si la lista está vacía, avisá; si no, mostralos"*.

- `libros.Where(l => l.Activo)`: filtrar con LINQ.
  - `Where` recorre cada libro `l`.
  - `=>` (lambda, "va hacia") es una abreviatura de función: *"dado un libro l..."*.
  - `l.Activo` evalúa la propiedad `Activo` de ese libro. Si es `true`, el libro
    **pasa el filtro**; si es `false`, **se descarta**.
  - El resultado: solo los libros con `Activo == true`.

  Supongamos estos libros:
  | Libro | Título | Activo | ¿Pasa el filtro `l => l.Activo`? |
  |-------|--------|--------|----------------------------------|
  | 1 | "Cien años de soledad"  | `true`  | ✔ sí |
  | 2 | "Un libro eliminado"     | `false` | ✘ no |
  | 3 | "La casa de los espíritus" | `true` | ✔ sí |

  El `foreach` mostrará solo los libros **1 y 3**.

### d) RESULTADO FINAL

Si no hay libros → "No existen libros registrados." Si hay libros, se imprimen
**solo los que están activos**, uno por línea, con ID, Título, Año y Autor.

### e) CONCEPTOS QUE APARECEN

- **LINQ**: lenguaje de consultas integrado en C# (ej. `Where`, `Any`).
- **`Any()`**: "¿hay al menos uno?".
- **Operador de negación `!`**: invierte un booleano.
- **`Where(...)`**: filtra elementos según una condición.
- **Lambda `=>`**: "dado un parámetro, devuelvo una condición".
- **Propiedad nullable / de navegación**: `libro.Autor` trae el objeto Autor
  completo gracias a `ObtenerTodosCon("Autor")`.

### f) ⚠️ OJO / ERRORES COMUNES

- **Leer `!libros.Any()` al revés**: mucho ojo con el "NO". Sería un error típico
  pensar "si hay libros, entro al if". Acá se entra al if precisamente cuando
  **NO** hay libros.
- **Confundir `Any()` con contar**: `Any()` solo responde "¿hay o no hay?";
  `Count()` dice cuántos hay. Para el mensaje "¿existen o no existen?" conviene
  `Any()`.
- **Usar `=` en vez de `==` dentro de la lambda**: `l.Activo = true` **asigna**
  (y rompe el filtro); lo correcto es `l.Activo == true` o simplemente `l.Activo`.
- **`Where` con `Activo = false` al revés**: se usa `Where(l => l.Activo)` para
  activos; para inactivos sería `Where(l => !l.Activo)` (¡de nuevo el NO!).

---

## Ejercicio 5 — Eliminación lógica (borrar sin borrar)

### a) CÓDIGO ORIGINAL

```csharp
var libro = libroRepository.ObtenerPorId(id);

if (libro != null)
{
    libro.Activo = false;

    libroRepository.Modificar(libro);

    Console.WriteLine("Libro eliminado lógicamente.");
}
else
{
    Console.WriteLine("Libro no encontrado.");
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender qué pasa cuando el usuario elige "Eliminar Libro": **no se borra de la
base de datos**, sino que se "apaga" usando el campo `Activo`.

### c) EXPLICACIÓN PASO A PASO

- `ObtenerPorId(id)`: va a buscar el libro con ese ID. Si lo encuentra, devuelve
  el objeto `Libro`; si no lo encuentra, devuelve `null` (nada, "vacío").

- `if (libro != null)`: `!=` significa "es distinto de". Entonces:
  *"si `libro` NO es vacío..."* → o sea, **si existía**, entrar.

  > Ojo, hay un `!=` NO (`!`+`=`) y un `==`. `!=` es "distinto de". Es la única
  > forma de preguntar "¿esto existe?" porque no podés hacer `if (libro)`, el
  > booleano lo construís vos con la comparación.

- `libro.Activo = false;`: **apagamos** el libro. No lo eliminamos: lo marcamos
  como inactivo. Por eso se llama **eliminación lógica** (se elimina "de forma
  lógica/para el negocio", no "físicamente" de la tabla).

- `libroRepository.Modificar(libro);`: guarda el cambio (que `Activo` ahora es
  `false`) en la base de datos. Es un **UPDATE**, no un DELETE.

- Si `libro == null` (no se encontró) → se imprime "Libro no encontrado.".

| Estado | Objeto `libro` | `libro.Activo` | base de datos |
|--------|----------------|----------------|---------------|
| Existe | ✔ objeto completo | `true` → se cambia a **`false`** | se actualiza |
| No existe | `null` | - | no se toca nada, mensaje de error |

### d) RESULTADO FINAL

Si el libro existe, queda con `Activo = false` y deja de aparecer en los listados
que filtran con `Where(l => l.Activo)`. Pero el registro **sigue existiendo** en
la base de datos.

### e) CONCEPTOS QUE APARECEN

- **`null`**: el valor que significa "no hay nada".
- **`!=` / `==`**: "distinto de" / "igual a".
- **Eliminación lógica**: apagar un registro con un campo booleano en vez de borrarlo.
- **`Modificar`/`Update`**: actualizar un registro existente.
- **Validación de existencia**: preguntar `if (objeto != null)` antes de usar el objeto.

### f) ⚠️ OJO / ERRORES COMUNES

- **`if (libro != null)` vs `if (libro)`**: en C# no podés meter un objeto suelto
  en un `if`; tenés que escribir la comparación completa.
- **Borrar de verdad por error**: si usaramos el método `Eliminar(id)` del
  repositorio general, el registro se **sacaría para siempre**. Acá se eligió
  marcarlo como inactivo para no perder datos.
- **Confundir `!=` con la negación `!`**: `libro != null` es comparación. Si
  escribís `libro !null`, es error de sintaxis.
- **`null` también va para objetos**: no solo para string. Cualquier variable que
  guarde un objeto puede estar "vacía" (`null`).

---

## Ejercicio 6 — LINQ: ordenar y contar libros

### a) CÓDIGO ORIGINAL

```csharp
public List<Libro> ObtenerLibrosPorMasRecientes()
{
    return _context.Libro
                   .OrderByDescending(l => l.AnioPublicacion)
                   .ToList();
}

public int ObtenerCantidadLibros()
{
    return _context.Libro
                   .Count();
}

public int ObtenerCantidadLibrosActivos()
{
    return _context.Libro
                   .Count(l => l.Activo);
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender cómo LINQ ordena los libros de más reciente a más viejo y cómo cuenta
libros (todos, o solo los activos).

### c) EXPLICACIÓN PASO A PASO

**Primer método: ordenar por año descendente.**

- `_context.Libro`: es el "expediente" de la tabla de libros (el repositorio
  tiene acceso a la base de datos por medio de este `_context`).
- `.OrderByDescending(l => l.AnioPublicacion)`: ordena la lista. La lambda
  `l => l.AnioPublicacion` le dice "mirá el año de cada libro y ordená según eso",
  y `Descending` significa "de mayor a menor" → **los más nuevos primero**.
- `.ToList()`: convierte el resultado de vuelta en una lista normal (porque
  LINQ trabaja con consultas "diferidas", el `ToList()` "materializa" el resultado).

**Segundo método: contar todos.**

- `.Count()` sin parámetro: cuenta **cuántos libros hay en total** y devuelve un `int`.

**Tercer método: contar solo los activos.**

- `.Count(l => l.Activo)`: cuenta **solo los libros que cumplen la condición**
  (que `Activo` sea `true`). Fíjense que la lambda es "lo mismo" que en `Where`.

Ejemplo con 4 libros (2 activos):
| Libro | Título | Año | Activo |
|-------|--------|-----|--------|
| 1 | "La casa de los espíritus" | 1982 | true |
| 2 | "Cien años de soledad"     | 1967 | true |
| 3 | "Borrado"                  | 2001 | false |
| 4 | "El amor en los tiempos..."| 1985 | false |

- `ObtenerLibrosPorMasRecientes()` → orden: Libro 3 (2001), Libro 4 (1985), Libro 1 (1982), Libro 2 (1967). *(Ojo: ordena por año, aunque el libro 3 esté inactivo.)*
- `ObtenerCantidadLibros()` → **4**.
- `ObtenerCantidadLibrosActivos()` → **2**.

### d) RESULTADO FINAL

- `ObtenerLibrosPorMasRecientes` devuelve la lista ordenada por año descendente.
- `ObtenerCantidadLibros` devuelve el total.
- `ObtenerCantidadLibrosActivos` devuelve cuántos tienen `Activo == true`.

### e) CONCEPTOS QUE APARECEN

- **`OrderByDescending`**: ordenar descendente (de mayor a menor / de más nuevo a más viejo).
- **`Count()`**: contar elementos.
- **`Count(predicado)`**: contar elementos que cumplen una condición.
- **Método que devuelve valor**: `return` "entrega" el resultado al que llamó.
- **`int`**: tipo de dato para números enteros.

### f) ⚠️ OJO / ERRORES COMUNES

- **`OrderBy` vs `OrderByDescending`**: el primero ordena de menor a mayor
  (az → zz), el segundo al revés. Si confundís los dos, los libros "más recientes"
  saldrían los más viejos.
- **Olvidar el `.ToList()`**: algunos LINQ funcionan "flojitos" (deferidos); sin
  materializar, después podés tener sorpresas al reutilizar la lista.
- **`Count()` no solo cuenta libros**: si le pasás la lambda, cuenta los que
  cumplen. Definir bien la intención antes de escribir.
- **El parámetro de la lambda**: `l` puede llamarse como quieras (`x`, `libro`).
  Lo que importa es que sea **uno** de los elementos, no toda la colección.

---

## Ejercicio 7 — LINQ: buscar un libro y verificar si existen activos

### a) CÓDIGO ORIGINAL

```csharp
public Libro? ObtenerLibroPorId(int id)
{
    return _context.Libro
                   .FirstOrDefault(l => l.Id == id);
}

public List<Libro> ObtenerLibrosOrdenadosPorTitulo()
{
    return _context.Libro
                   .OrderBy(l => l.Titulo)
                   .ToList();
}

public bool ExistenLibrosActivos()
{
    return _context.Libro
                   .Any(l => l.Activo);
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender cómo se busca un libro puntual por su ID, el orden alfabético por
título, y cómo responder sí/no si existe al menos un libro activo.

### c) EXPLICACIÓN PASO A PASO

**Primer método: buscar por ID.**

- `FirstOrDefault(l => l.Id == id)`: recorre los libros buscando **el primero**
  cuyo `Id` sea `id`. `First` = "primero"; `OrDefault` = "**o si no, el valor por
  defecto**" (que para objetos es `null`).
- El tipo de retorno es `Libro?`: el sigo `?` después de un tipo dice "este
  método **puede devolver `null`**" (por si el libro no existe). Por eso en el
  `Program.cs` se puede preguntar `if (libro == null)`.

**Segundo método: ordenar por título.**

- `.OrderBy(l => l.Titulo)`: ordena **ascendente** (de la A a la Z) por el título.
  A diferencia de `OrderByDescending`, este es el orden "normal" del abecedario.

**Tercer método: ¿hay activos?**

- `.Any(l => l.Activo)`: recorre los libros y pregunta "¿**al menos uno** tiene
  `Activo == true`?". Devuelve `true` o `false`.

Con los mismos 4 libros de antes:
| Consulta | Resultado |
|----------|-----------|
| `ObtenerLibroPorId(2)` | → Libro "Cien años de soledad" |
| `ObtenerLibroPorId(99)` | → `null` (no existe) |
| `ObtenerLibrosOrdenadosPorTitulo()` | → orden alfabético por título |
| `ExistenLibrosActivos()` | → `true` (hay 2 activos) |

### d) RESULTADO FINAL

- Se encuentra el libro con el ID pedido, o `null` si no existe.
- Los libros salen en orden alfabético de título.
- `ExistenLibrosActivos` responde `true`/`false` según haya o no al menos un libro activo.

### e) CONCEPTOS QUE APARECEN

- **`FirstOrDefault`**: "dame el primero que cumpla, o `null` si no hay ninguno".
- **`OrderBy`**: ordenar ascendente.
- **`Any(predicado)`**: "¿hay al menos uno que cumpla?" → devuelve `bool`.
- **Tipos nullable `Libro?`**: tipos que admiten `null`.
- **`==`** al comparar valores numéricos (`l.Id == id`).

### f) ⚠️ OJO / ERRORES COMUNES

- **`First` (sin OrDefault) lanza error**: si usás solo `First` y no hay ninguna
  coincidencia, el programa **se rompe** (excepción). `FirstOrDefault` es la
  versión segura.
- **Confundir `==` con `=`**: dentro de `l.Id == id` es comparación. Un solo `=`
  es asignación y ahí no va.
- **No pensar en `null`**: si el método puede devolver `null` y nadie lo pregunta,
  después `libro.Titulo` reventaría al acceder a un libro que no existe.
- **`Any` y `Where` son distintos**: `Where` devuelve una **lista filtrada**;
  `Any` devuelve **true/false**. Si decís "quiero saber si hay activos", usás
  `Any`, no `Where`.

---

## Ejercicio 8 — Genéricos, interfaces y herencia en los repositorios

### a) CÓDIGO ORIGINAL

```csharp
public interface IGenericRepository<T> where T : class
{
    void Agregar(T entidad);
    List<T> ObtenerTodos();
    List<T> ObtenerTodosCon(string propiedadRelacionada);
    T ObtenerPorId(int id);
    void Modificar(T entidad);
    void Eliminar(object id);
}

public class GenericRepository<T> : IGenericRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;

    public GenericRepository()
    {
        _context = new ApplicationDbContext();
    }

    public void Agregar(T entidad)
    {
        _context.Set<T>().Add(entidad);
        _context.SaveChanges();
    }

    // ... otros métodos ...

    public List<T> ObtenerTodos()
    {
        return _context.Set<T>()
                       .AsNoTracking()
                       .ToList();
    }
}

public class LibroRepository : GenericRepository<Libro>
{
    // ... métodos especiales de Libro con LINQ ...
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender por qué un solo repositorio "genérico" puede trabajar con Autor,
Categoría y Libro a la vez, y por qué Libro necesita además un repositorio propio.

### c) EXPLICACIÓN PASO A PASO

**¿Qué es la `<T>`? (la letra genérica)**

- `IGenericRepository<T>` significa: "este contrato sirve para **cualquier tipo
  T**". En el programa, al crear el repositorio se dice cuál es el tipo:

  ```csharp
  IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();
  IGenericRepository<Categoria> categoriaRepository = new GenericRepository<Categoria>();
  LibroRepository libroRepository = new LibroRepository();
  ```

  Es como una **plantilla de documento**: la misma plantilla se rellena con "Autor"
  o con "Categoría". El `<T>` se cambia por el tipo real en cada uso.

- `where T : class`: una restricción que dice "T tiene que ser una **clase**"
  (un molde de objeto), no un número ni un texto.

**¿Qué es una interfaz?**

- `interface` es un **contrato**: lista QUÉ métodos tiene que tener una clase,
  pero sin decir CÓMO los hace. `IGenericRepository<T>` promete que cualquiera
  que la use tendrá `Agregar`, `ObtenerTodos`, `ObtenerPorId`, etc.

**¿Qué hace la clase genérica?**

- `GenericRepository<T> : IGenericRepository<T>` dice "esta clase cumple con el
  contrato". La ventaja: el método `Agregar(T entidad)` funciona **igual** para
  Autor, Categoría o Libro, porque todos son clases. No hay que copiar/pegar el
  mismo código 3 veces.

- `_context.Set<T>()`: le dice a Entity Framework "tratá esta tabla/entidad de
  tipo T". Por eso con un solo método manejamos tres tablas distintas.

**¿Por qué Libro necesita su propio repositorio?**

- `LibroRepository : GenericRepository<Libro>`: hereda todo lo genérico **y le
  agrega** métodos exclusivos de libros: `ObtenerLibrosPorMasRecientes`,
  `ObtenerCantidadLibrosActivos`, etc. Esto se llama **herencia** (el hijo tiene
  todo lo del padre, más lo suyo).

| Concepto | Analogía |
|----------|----------|
| Interfaz `IGenericRepository<T>` | El contrato/checklist de lo que hay que implementar |
| Clase `GenericRepository<T>` | La fábrica que hace el trabajo con cualquier tipo T |
| `GenericRepository<Autor>` | La fábrica "especializada" en Autores |
| `LibroRepository : GenericRepository<Libro>` | Fábrica de libros: hereda el trabajo genérico + métodos propios |

### d) RESULTADO FINAL

Con una sola implementación genérica se cubren tres entidades (Autor, Categoría,
Libro), y Libro extiende esa base con consultas específicas. Menos código
duplicado y más facilidad para agregar entidades nuevas.

### e) CONCEPTOS QUE APARECEN

- **Genérico `<T>`**: código que funciona con varios tipos sin repetirlo.
- **Interfaz**: contrato de métodos que una clase debe implementar.
- **Herencia `: Base`**: una clase "hija" que hereda de otra "padre".
- **`protected`**: acceso solo para la clase y sus hijas (por eso
  `LibroRepository` puede usar `_context`).
- **`readonly`**: el campo no se puede reasignar después del constructor.

### f) ⚠️ OJO / ERRORES COMUNES

- **La T no existe "de verdad"**: `<T>` no es un tipo concreto; si usás
  `GenericRepository` sin decir `<Autor>`, al compilar exige que aclares el tipo.
- **Confundir interface con clase**: una interfaz **no implementa** nada, solo
  declara. Tratar de darle cuerpo a un método ahí da error.
- **`protected` vs `private`**: si `_context` fuera `private`, `LibroRepository`
  (la hija) **no podría usarlo**; por eso está `protected`.
- **No confundir `Set<T>()` con la tabla**: `_context.Set<T>` es la "puerta de
  acceso" a la tabla de tipo T, no el tipo en sí.

---

## Ejercicio 9 — El modelo `Libro` y sus relaciones

### a) CÓDIGO ORIGINAL

```csharp
public class Libro
{
    public int Id { get; set; }
    public string Titulo { get; set; }
    public int AnioPublicacion { get; set; }
    public int AutorId { get; set; }
    public Autor Autor { get; set; }
    public int CategoriaId { get; set; }
    public Categoria Categoria { get; set; }
    public bool Activo { get; set; } = true;
}
```

### b) ¿QUÉ PIDE EL EJERCICIO?

Entender cómo se "dibuja" en código la ficha de un libro y qué significan los IDs
y las propiedades que apuntan a Autor/Categoría.

### c) EXPLICACIÓN PASO A PASO

- `public class Libro { }`: definimos el **molde** de libro. Todo libro tiene los
  campos que están dentro de las llaves.

- Cada línea es una **propiedad** = un dato del libro.

  - `public int Id { get; set; }` → número entero, Identificador único (1, 2, 3...).
    Cada libro tiene un Id distinto. Suele ser la **clave primaria** de la tabla.
  - `public string Titulo { get; set; }` → texto con el título.
  - `public int AnioPublicacion { get; set; }` → número entero con el año.
  - `public bool Activo { get; set; } = true;` → verdadero/falso. Y fíjense el
    `= true` al final: **arranca en verdadero** por defecto. Todo libro nuevo nace
    "activo", y solo se apaga con la eliminación lógica (ejercicio 5).

- Las **relaciones** (lo más difícil de visualizar): un libro pertenece a un autor
  y a una categoría.

  - `public int AutorId { get; set; }` → guarda **el número de ID del autor**.
    Es la *llave foránea*.
  - `public Autor Autor { get; set; }` → guarda **el objeto Autor completo**
    (para poder escribir `libro.Autor.Nombre` sin tener que buscarlo de nuevo).
  - Igual con `CategoriaId` y `Categoria`.

  Se llaman **propiedades de navegación**: te "dejan navegar" de un libro a su
  autor directamente.

| Propiedad | Tipo | ¿Qué guarda? |
|-----------|------|--------------|
| `Id` | int | el número identificador único del libro |
| `Titulo` | string | el título (texto) |
| `AnioPublicacion` | int | el año |
| `AutorId` | int | el ID del autor al que pertenece (llave foránea) |
| `Autor` | Autor | el autor completo (navegación) |
| `CategoriaId` | int | el ID de la categoría (llave foránea) |
| `Categoria` | Categoria | la categoría completa (navegación) |
| `Activo` | bool | si está "dado de alta" o "eliminado lógicamente" (`true` por defecto) |

En la migración (el archivo que crea las tablas en la base de datos) se ve lo
mismo: la tabla `Libro` tiene `AutorId`, `CategoriaId` y llaves foráneas
(`ForeignKey`) que conectan a las tablas `Autor` y `Categoria`.

### d) RESULTADO FINAL

El modelo `Libro` representa la ficha completa de un libro e incluye las llaves
foráneas (`AutorId`, `CategoriaId`) y las propiedades de navegación (`Autor`,
`Categoria`) que permiten acceder a los datos relacionados sin buscarlos aparte.

### e) CONCEPTOS QUE APARECEN

- **Clase / modelo**: la plantilla de una "cosa" del problema.
- **Propiedad**: dato del objeto.
- **Clave primaria**: el `Id` único que identifica cada registro.
- **Llave foránea**: un `...Id` que apunta a otra tabla.
- **Propiedad de navegación**: objeto relacionado ("saltar" de libro a autor).
- **Valor por defecto**: `= true` al declarar le da un valor inicial a `Activo`.

### f) ⚠️ OJO / ERRORES COMUNES

- **Guardar solo el ID y olvidar la navegación (o al revés)**: se necesitan
  **las dos**: `AutorId` para guardar, `Autor` para mostrar.
- **Confundir `Id` con `AutorId`**: `Id` es del propio libro; `AutorId` es de OTRO.
- **Olvidar el `public`**: las propiedades sin `public` no se ven desde afuera y
  Entity Framework ni las reconoce.
- **`{ get; set; }` para principiantes**: es la forma corta de una propiedad
  de lectura/escritura. No es una función; es "esta casilla se puede leer y
  escribir".

---

# PARTE 2: EJERCICIOS SIMILARES EN NIVEL

Abajo tenés ejercicios del **mismo nivel** que los resueltos. La idea es que
deduzcas la salida **sin ejecutar** el código (o ejecutándolo después para
comprobar).

## Práctica 1 (sobre el menú / `while` + `switch`)

**E1.** ¿Cuántas veces se imprime "Paso por acá" y qué imprime este código?

```csharp
int n = 0;
while (n < 3)
{
    Console.WriteLine("Paso por acá");
    n = n + 1;
}
Console.WriteLine($"Terminé, n vale {n}");
```

**E2.** La siguiente línea tiene un error. ¿Por qué nunca entra al `case`?

```csharp
string opcion = Console.ReadLine(); // el usuario escribe: 2
switch (opcion)
{
    case 2:
        Console.WriteLine("Elegiste la opción 2");
        break;
    default:
        Console.WriteLine("Opción inválida");
        break;
}
```

**E3.** ¿Qué diferencia hay si el usuario tipea "0" en este `switch`? ¿Cuál es el
valor final de `encendido`?

```csharp
bool encendido = true;
while (encendido)
{
    string tecla = Console.ReadLine();
    switch (tecla)
    {
        case "0": encendido = false; break;
        default:  break;
    }
}
```

## Práctica 2 (sobre crear objetos)

**E1.** Completá el objeto para que quede un autor con nombre "Jorge Luis Borges".
¿Qué hace `Agregar`?

```csharp
Autor autor = new Autor
{
    // ¿qué va acá?
};
autorRepository.Agregar(autor);
```

**E2.** ¿Qué imprimen estos `WriteLine` en orden? (el usuario escribe "Julio Cortázar")

```csharp
Console.Write("Nombre: ");
string nombre = Console.ReadLine();
Autor a = new Autor { Nombre = nombre };
Console.WriteLine(a.Nombre.Length);
Console.WriteLine(a.Nombre.ToUpper());
```

**E3.** ¿Qué pasa si borramos la línea `autorRepository.Agregar(autor);`? ¿Sigue
existendo el autor en la base de datos?

## Práctica 3 (sobre `foreach` y `$`)

**E1.** ¿Qué se imprime exactamente?

```csharp
var numeros = new List<int> { 10, 20, 30 };
foreach (var n in numeros)
{
    Console.WriteLine($"Número {n * 2}");
}
```

**E2.** ¿Está bien o está mal? ¿Qué pasa si quito el `$`?

```csharp
var nombres = new List<string> { "Ana", "Beto" };
foreach (var nombre in nombres)
{
    Console.WriteLine("Hola {nombre}");
}
```

**E3.** Si la lista está vacía, ¿cuántas líneas imprime este código y cuál?

```csharp
var vacio = new List<Autor>();
Console.WriteLine("Inicio");
foreach (var a in vacio)
{
    Console.WriteLine(a.Nombre);
}
Console.WriteLine("Fin");
```

## Práctica 4 (sobre `Any()` / `Where()`)

**E1.** ¿Qué imprime el `if`? Hay libros, pero todos con `Activo == false`.

```csharp
var libros = new List<Libro> { /* libros todos inactivos */ };
if (!libros.Any())
{
    Console.WriteLine("No hay libros");
}
else if (!libros.Any(l => l.Activo))
{
    Console.WriteLine("Hay libros, pero ninguno activo");
}
else
{
    Console.WriteLine("Hay al menos uno activo");
}
```

**E2.** Solo con estas tres líneas, ¿qué libros mostraría el `foreach`?

```csharp
var datos = new List<Libro>
{
    new Libro { Id = 1, Titulo = "A", Activo = true  },
    new Libro { Id = 2, Titulo = "B", Activo = false },
    new Libro { Id = 3, Titulo = "C", Activo = true  }
};
foreach (var l in datos.Where(l => !l.Activo))
{
    Console.WriteLine(l.Titulo);
}
```

**E3.** Escribí una condición con `Where` para quedarte con los libros del año 2020
y activos. (Pista: hay que combinar dos condiciones. LinQ permite `l => A && B`.)

## Práctica 5 (sobre eliminación lógica)

**E1.** ¿Qué imprime este código? ¿La base de datos sigue teniendo el libro?

```csharp
var libro = repo.ObtenerPorId(7); // existe, Activo = true
if (libro != null)
{
    libro.Activo = false;
    repo.Modificar(libro);
    Console.WriteLine("Baja lógica");
}
else
{
    Console.WriteLine("No encontrado");
}
```

**E2.** ¿Qué problema tiene este código? ¿Por qué puede explotar?

```csharp
var libro = repo.ObtenerPorId(100); // NO existe
Console.WriteLine(libro.Titulo);
```

**E3.** En el `Program.cs`, ¿por qué antes de eliminar se llama a `MostrarLibros()`?
¿Qué le está mostrando al usuario para que pueda elegir el ID correcto?

## Práctica 6 (sobre ordenar y contar con LINQ)

**E1.** Con la lista `{ "Casa", "Avión", "Barco" }`, ¿qué orden imprime esto?

```csharp
foreach (var s in lista.OrderBy(x => x))
{
    Console.WriteLine(s);
}
foreach (var s in lista.OrderByDescending(x => x))
{
    Console.WriteLine(s);
}
```

**E2.** Dado: 5 libros, 3 activos. ¿Cuánto devuelve cada uno?

```csharp
repo.ObtenerCantidadLibros();        // ?
repo.ObtenerCantidadLibrosActivos(); // ?
```

**E3.** ¿Qué valor devuelve `Count()` si `lista` está vacía? ¿Y `Any()`?

## Práctica 7 (sobre buscar y verificar)

**E1.** ¿Qué devuelve `ObtenerLibroPorId(3)` si el libro 3 no existe? ¿Qué tipo de
retorno lo permite?

**E2.** ¿Por qué conviene usar `FirstOrDefault` en vez de `First` para buscar por ID?

**E3.** En `ExistenLibrosActivos`, ¿qué pasa si todos los libros tienen `Activo = false`?
¿Y si no hay ningún libro en la tabla?

## Práctica 8 (sobre genéricos e interfaces)

**E1.** ¿Podría crearse un repositorio genérico para una clase nueva `Editorial`
si solo existiera esa clase? ¿Qué habría que escribir para usarlo con Editorial?

**E2.** ¿Por qué `LibroRepository` puede usar `_context` si el campo está
declarado en `GenericRepository<T>`? ¿Y qué pasaría si fuera `private`?

**E3.** ¿Un `interface` puede tener código "por dentro" (implementación)? ¿Sí o no,
y por qué?

## Práctica 9 (sobre el modelo)

**E1.** Agregá al modelo `Libro` una propiedad `Editorial` (texto) con valor
por defecto `""`. ¿Es coherente con el resto del modelo?

**E2.** ¿Qué diferencia hay entre `AutorId` y `Autor` en el modelo `Libro`? ¿Cuándo
se usa cada uno en el `Program.cs`?

**E3.** Si un libro está con `Activo = false`, ¿cuántas líneas del menú del
`Program.cs` van a omitirlo? (Revisá los métodos que filtran con `.Activo`.)

---

# PARTE 3: SOLUCIONES DE LOS EJERCICIOS DE PRÁCTICA

> Consejo para estudiar: **intentá resolverlos ANTES de mirar acá.** Después
> compará, y si te equivocaste, buscá en cuál concepto estuvo el error (esa es
> la parte que más enseña).

## Soluciones Práctica 1 (menú / `while` + `switch`)

**E1.** Imprime `"Paso por acá"` **3 veces** y después `"Terminé, n vale 3"`.

| n al iniciar | ¿n < 3? | ¿imprime? | n al terminar |
|--------------|---------|-----------|---------------|
| 0 | sí | "Paso por acá" | 1 |
| 1 | sí | "Paso por acá" | 2 |
| 2 | sí | "Paso por acá" | 3 |
| 3 | **no** | - (sale del while) | - |

Cuando `n` llega a 3, `n < 3` da `false`, el `while` se corta y se imprime una
última vez con el valor final de `n` → `"Terminé, n vale 3"`.

**E2.** **`case 2:` sin comillas** está mal. `opcion` es un `string` (texto), y
el usuario escribió el texto `"2"`. Al comparar textos, cada `case` tiene que
ser un texto también: `case 2:` busca un número. El programa cae al `default`
y siempre imprime `"Opción inválida"`, incluso escribiendo 2.
**Corrección:** usar `case "2":`.

**E3.** El usuario tipea `"0"` → entra al `case "0"` → `encendido = false` → el
`while` corta. El valor final de `encendido` es **`false`**. (No se imprime
ningún mensaje porque en este `switch` no hay `WriteLine`.)

## Soluciones Práctica 2 (crear objetos)

**E1.**

```csharp
Autor autor = new Autor
{
    Nombre = "Jorge Luis Borges"
};
```

`Agregar(autor)` inserta ese objeto en la base de datos (hace un `INSERT` en la
tabla de autores). Sin esa línea, el autor solo existiría en la memoria del
programa mientras dura la función.

**E2.** Imprime:
1. `14` → `a.Nombre` es `"Julio Cortázar"` (cuenta J-u-l-i-o + espacio + C-o-r-t-á-z-a-r = 14). `Length` da la cantidad de caracteres.
2. `JULIO CORTÁZAR` → `.ToUpper()` convierte todo a mayúsculas.

**E3.** El autor **NO** se guarda. El objeto se creó en memoria, pero nunca se
pasó al repositorio (`Agregar`). Cuando la función termina, el objeto se pierde.
Queda "flotando" nada más durante la ejecución.

## Soluciones Práctica 3 (`foreach` y `$`)

**E1.** Tres líneas:

```
Número 20
Número 40
Número 60
```

Porque adentro del `{ }` se interpola `n * 2` (10×2, 20×2, 30×2).

**E2.** **Está mal** (para lo que probablemente querés). Como **no** hay `$` antes
del texto, los `{ }` no son huecos: se imprimen **literales**. Sale dos veces el
texto `Hola {nombre}`. Con el `$`, saldría `Hola Ana` y `Hola Beto`.

**E3.** Imprime **2 líneas**: `Inicio` y `Fin`. El `foreach` sobre una lista vacía
no ejecuta el cuerpo ni una sola vez (no hay elementos que recorrer).

## Soluciones Práctica 4 (`Any()` / `Where()`)

**E1.** Imprime `"Hay libros, pero ninguno activo"`. El primer `if` no entra
(porque `libros.Any()` es `true`, y con `!` queda `false`). El segundo `if`
(`!libros.Any(l => l.Activo)`) sí entra: hay libros, pero ninguno cumple
`Activo == true`.

**E2.** Solo el **ID 2 ("B")** → `Where(l => !l.Activo)` se queda con los que
NO están activos. Imprime `B`.

**E3.**

```csharp
var resultado = datos.Where(l => l.AnioPublicacion == 2020 && l.Activo);
```

El `&&` significa "Y": el libro tiene que cumplir las dos condiciones a la vez.

## Soluciones Práctica 5 (eliminación lógica)

**E1.** Imprime `"Baja lógica"`. El libro **sigue existiendo en la base de datos**,
pero con `Activo = false`. Por eso el "baja" es *lógica*: el registro no se borra,
se apaga.

**E2.** `ObtenerPorId(100)` devuelve **`null`** (ese libro no existe). Después,
`libro.Titulo` intenta leer una propiedad de un objeto que es `null` → el programa
**se rompe** con un error `NullReferenceException`. Siempre hay que preguntar
`if (libro != null)` antes de usar el objeto.

**E3.** `MostrarLibros()` lista los libros **con sus IDs** para que el usuario
vea qué ID elegir y no tenga que adivinarlo de memoria. Dato extra: como
`MostrarLibros()` solo muestra libros activos (`Where(l => l.Activo)`), un libro
ya apagado no aparece ahí para ser "re-eliminado", que es lo que se busca: se
puede eliminar lógicamente una vez.

## Soluciones Práctica 6 (ordenar y contar)

**E1.** Primer `foreach` (ascendente): `Avión`, `Barco`, `Casa`.
Segundo `foreach` (descendente): `Casa`, `Barco`, `Avión`.
`OrderBy` va en orden normal de letras y `OrderByDescending` al revés.

**E2.** `ObtenerCantidadLibros()` → **5** (cuenta todo). `ObtenerCantidadLibrosActivos()` → **3** (solo los que cumplen `Activo == true`).

**E3.** `Count()` sobre una lista vacía devuelve **`0`**. `Any()` devuelve **`false`**
(no hay ni un elemento). Notá la diferencia: `Count()` dice "cuántos", `Any()`
dice "¿hay o no hay?".

## Soluciones Práctica 7 (buscar y verificar)

**E1.** Devuelve **`null`**. Esto es posible porque el tipo de retorno está
declarado como **`Libro?`** (con el signo `?`, que marca que la variable puede
valer "nada"). Ese `null` es lo que el `Program.cs` verifica con
`if (libro == null)`.

**E2.** Porque `First` **lanza una excepción (rompe el programa)** si no encuentra
nada. `FirstOrDefault` es la versión "segura": devuelve `null` (en vez de explotar)
cuando no hay coincidencia, y así podemos controlarlo con un `if`.

**E3.** Devuelve **`false`** en los dos casos:
- Si todos los libros están con `Activo = false` → ningún libro cumple el predicado.
- Si la tabla está vacía → no hay ni un elemento que evaluar (`Any` sobre lista vacía es `false`).

O sea: `Any` nunca "miente"; solo responde si encontró al menos un `true`.

## Soluciones Práctica 8 (genéricos / interfaces)

**E1.** **Sí.** Solo hace falta que `Editorial` sea una clase (cumple la
restricción `where T : class`) y crear el repositorio así:

```csharp
IGenericRepository<Editorial> editorialRepository = new GenericRepository<Editorial>();
```

Con el mismo código genérico funcionan Autor, Categoría, Libro o Editorial: esa
es la ventaja de los genéricos.

**E2.** Porque `_context` está declarado como **`protected`**: eso permite que la
clase y **sus descendientes** (las que heredan, como `LibroRepository`) lo usen.
Si fuera `private`, solo `GenericRepository<T>` podría tocarlo, y
`LibroRepository` daría **error de compilación** al intentar usarlo.

**E3.** **No.** La interfaz es solo el *contrato*: declara los métodos
(`void Agregar(...)`, etc.) pero **sin cuerpo**. La implementación (el código
dentro de las llaves) la escribe la clase que la implementa. Si una interfaz
tuviera cuerpo, dejaría de ser interfaz.

## Soluciones Práctica 9 (modelo)

**E1.**

```csharp
public string Editorial { get; set; } = "";
```

**Sí, es coherente**: mantiene el mismo patrón (propiedad pública, `string`, y un
valor por defecto como hace `Activo`). Acá el valor por defecto evita que quede
`null` y rompa el programa al imprimirlo.

**E2.**
- `AutorId` es la **llave foránea**: un número que guarda *cuál* autor. Se usa al
  **guardar** (en `AltaLibro`: `AutorId = autorId`).
- `Autor` es la **propiedad de navegación**: el objeto completo, que se usa para
  **leer** datos del autor (en `MostrarLibros`: `libro.Autor.Nombre`).
- Regla mental: el `...Id` se guarda; la propiedad sin `Id` se muestra.

**E3.** Dos opciones del menú lo "omitirían":
- **Opción 6 (Ver Libros)**: `MostrarLibros()` filtra con `Where(l => l.Activo)`.
- **Opción 12 (Cantidad de libros activos)**: el `Count(l => l.Activo)` no lo cuenta.

Las demás no filtran por `Activo`: el 10 (más recientes), 13 (buscar por ID),
14 (ordenados por título) y 15 (verificar) lo ven/responden como corresponde.