# Explicación para principiantes: ejercicio "Analizar código" (U9)

> Basado en el documento `ANALIZAR CODIGO - RESUELTO.docx` (6 preguntas) y en la
> guía `U9_Analisis_Entregable_y_AnalizarCodigo.md`. Acá vas a ver cada pregunta
> **paso a paso, en lenguaje humano**, como si nunca hubieras leído el código.
> Cualquier término técnico que aparezca se explica ahí mismo.

---

## ¿De qué trata todo esto?

Hay una app de **biblioteca** que guarda Autores, Categorías y Libros en una base
de datos SQLite usando **Entity Framework Core (EF Core)** (la tecnología que
convierte objetos C# en filas de una tabla y viceversa).

El ejercicio "Analizar código" muestra **6 fragmentos de código** y te pregunta
cosas como "¿compila?", "¿qué hace?", "¿está completo?". Algunos fragmentos tienen
**bugs** (errores) que en el proyecto real ya están corregidos: el ejercicio te
pone la versión rota a propósito, para que aprendas a detectar el problema.

Tres ideas que van a aparecer en TODOS los ejercicios:

- **Clave foránea**: un campo que guarda un número con el ID de otra entidad.
  Ej: en `Libro`, el campo `AutorId` guarda "el número del autor".
- **Propiedad de navegación**: un campo que guarda el **objeto completo**
  relacionado. Ej: en `Libro`, el campo `Autor` guarda el autor entero,
  y te deja escribir `libro.Autor.Nombre`.
- **`Include("...")`**: la orden que le damos a EF Core para que **cargue la
  relación** (que no deje la propiedad de navegación en `null`).

---

# PARTE 1: RESOLUCIÓN Y EXPLICACIÓN DE CADA EJERCICIO

## Pregunta 1 — ¿Está completo el modelo `Libro`?

### a) CÓDIGO ORIGINAL

```csharp
public class Libro
{
    public int Id { get; set; }
    public string Titulo { get; set; }
    public int AutorId { get; set; }
}
```

> Enunciado del documento: *"Un libro pertenece a un único autor. ¿Está completo
> este modelo?"*

### b) ¿QUÉ PIDE EL EJERCICIO?

Averiguar si, con estas líneas, el modelo puede representar correctamente que un
libro tiene un autor. O sea: ¿en el código puedo "llegar" desde un libro hasta su
autor?

### c) EXPLICACIÓN PASO A PASO

Vamos línea por línea. Esto NO es un programa que se ejecuta: es una **plantilla**
(clase) que define qué datos tiene un libro.

| Línea | Qué dice | Traducción a español |
|-------|----------|----------------------|
| `public class Libro` | Empieza la plantilla del libro | "Un libro tiene las siguientes casillas..." |
| `public int Id { get; set; }` | Número entero, identificador único | "Cada libro tiene un número para reconocerlo (1, 2, 3...)" |
| `public string Titulo { get; set; }` | Texto | "Y un título escrito con texto" |
| `public int AutorId { get; set; }` | Número entero | "Y el **número** del autor al que pertenece" |

`AutorId` es la **clave foránea**: sirve para que la **base de datos** sepa a qué
autor apunta ese libro. Pero fijate **qué NO hay**: no hay ninguna casilla que sea
"el autor como objeto".

El problema: si yo escribo `libro.Autor.Nombre`, el compilador **me marca error**,
porque `Autor` no existe como propiedad dentro de `Libro`. La clave foránea guarda
el *número*, pero el código necesita además el *objeto* para navegar.

| ¿Qué tengo? | ¿Para qué alcanza? |
|-------------|--------------------|
| `AutorId` (el número) | La base de datos puede relacionar el libro con su autor ✔ |
| ... pero NO `Autor` (el objeto) | El código C# **no puede** hacer `libro.Autor.Nombre` ✘ |

### d) RESULTADO FINAL

**No está completo.** Falta agregar la propiedad de navegación:

```csharp
public Autor Autor { get; set; }
```

(Este es, además, un caso "real": el `Libro.cs` del proyecto entregable tiene
`AutorId` Y `Autor`, más `CategoriaId`, `Categoria` y `Activo`. El ejercicio le
sacó cosas a propósito para que exista la pregunta.)

### e) CONCEPTOS QUE APARECEN

- **Clase**: plantilla que define los datos de una "cosa".
- **Propiedad**: cada casilla de datos que tiene la plantilla.
- **`int` y `string`**: número entero y texto.
- **`{ get; set; }`**: forma corta de decir "esta casilla se puede leer y escribir".
- **Clave foránea**: campo que guarda el ID de otra entidad.
- **Propiedad de navegación**: campo que guarda el objeto relacionado completo.

### f) ⚠️ OJO / ERRORES COMUNES

- **Pensar que `AutorId` alcanza**: alcanza para la base de datos, pero NO para
  el código C#. Se necesitan las dos, no es "una o la otra".
- **Confundir las dos `Autor`**: `AutorId` (mayúscula I, d minúscula) es el número;
  `Autor` es el objeto. Son cosas completamente distintas.
- **Olvidar que `Categoria` y `Activo` también faltan acá**: el ejercicio solo te
  pregunta por la relación con autor. En el proyecto real el modelo tiene más.

---

## Pregunta 2 — `libro.Autor.Nombre` sin `Include`

### a) CÓDIGO ORIGINAL

```csharp
var libros = libroRepository.ObtenerTodos();

foreach (var libro in libros)
{
    Console.WriteLine(libro.Autor.Nombre);
}
```

> Enunciado del documento: *"¿Es correcto el código? Justificar la respuesta.
> ¿Existe algún escenario en el que pueda producirse un error?"*

### b) ¿QUÉ PIDE EL EJERCICIO?

Analizar si el código que listea los libros y muestra "el nombre del autor de cada
libro" es correcto, y si hay algún caso donde se rompa.

### c) EXPLICACIÓN PASO A PASO

- `libroRepository.ObtenerTodos()`: pide a la base **todos** los libros. Pero el
  método genérico hace esto por dentro:

  ```csharp
  _context.Set<T>().AsNoTracking().ToList();
  ```

  No tiene **ningún `Include`**. Y con la configuración que se usa en el curso
  (sin *lazy loading*, lo que en español sería "sin carga perezosa"), las
  propiedades de navegación que **no se pidieron con `Include`** quedan en `null`
  (vacías, "sin valor").

- `foreach (var libro in libros)`: recorre la lista. **Esto sí funciona**.

- `libro.Autor.Nombre`: acá está el problema. Para el primer libro, `libro.Autor`
  es `null`. Y **no se puede escribir `.Nombre` sobre `null`**. El programa se
  **rompe** con un error llamado `NullReferenceException` (la famosa "NullReference": 
  *intentaste tocar algo que no existe*).

Tabla de seguimiento (con 2 libros en la base):

| Línea | Código | `libros` | `libro` actual | `libro.Autor` | ¿Qué se imprime / qué pasa? |
|-------|--------|----------|----------------|---------------|------------------------------|
| 1 | `ObtenerTodos()` | [Libro 1, Libro 2] | - | - | se traen los libros, pero sin relación |
| 3 | `foreach` | - | Libro 1 | `null` | arranca el recorrido |
| 4 | `libro.Autor.Nombre` | - | Libro 1 | `null` | 💥 **CRASH: NullReferenceException** |

> ⚠️ Lo peligroso de este error: **no aparece siempre**. Si la relación ya hubiera
> sido cargada por casualidad en la misma sesión, no fallaría. Por eso el código
> "anda en una prueba rápida" y después explota: no se entiende bien la causa.

**La corrección**: cargar la relación en la misma consulta con `Include`. El
proyecto real lo resuelve con el método `ObtenerTodosCon("Autor")`, que hace
`Include(propiedadRelacionada)` por dentro:

```csharp
var libros = libroRepository.ObtenerTodosCon("Autor");
```

### d) RESULTADO FINAL

**No es correcto en el escenario normal**: con `ObtenerTodos()` (sin `Include`),
`libro.Autor` queda en `null` y el programa lanza `NullReferenceException` lo
primero que toca `libro.Autor.Nombre`. Se corrige usando `Include` (en el proyecto,
`ObtenerTodosCon("Autor")`).

Otras dos formas de cargar relaciones existen (cargarlas después a mano, o activar
*lazy loading*), pero **en este curso/este proyecto no se usan**: la única opción
válida acá es `Include`.

### e) CONCEPTOS QUE APARECEN

- **`foreach`**: bucle que recorre cada elemento de una lista.
- **`null`**: el valor que significa "no hay nada".
- **Excepción (`NullReferenceException`)**: error que rompe el programa al tocar
  algo que es `null`.
- **`Include` (eager loading / carga ansiosa)**: pedir la relación **en la misma
  consulta** para que venga completa.
- **`AsNoTracking`**: le dice a EF Core que no vigile los datos (mejor rendimiento),
  pero no tiene que ver con este error en sí.

### f) ⚠️ OJO / ERRORES COMUNES

- **Asumir que "traer libros" trae también el autor**: no. Con EF Core estándar,
  lo que no se pide con `Include` llega como `null`.
- **Acordarse del error solo cuando "crash"**: este error puede no aparecer siempre,
  por eso hay que entenderlo (no memorizar que "falla").
- **Confundir `null` con cadena vacía**: son distintos. Preguntar
  `if (libro.Autor.Nombre == "")` también explotaría si `Autor` es `null`.
- **Poner `Include` DESPUÉS de traer los datos**: no funciona así; `Include` va
  adentro de la consulta, antes de ejecutarla (antes del `ToList`).

---

## Pregunta 3 — ¿Qué hace esta consulta LINQ?

### a) CÓDIGO ORIGINAL

```csharp
return _context.Libro
    .Where(l => l.Activo)
    .OrderBy(l => l.Titulo)
    .ToList();
```

> Enunciado del documento: *"Explicar qué hace la siguiente consulta."*

### b) ¿QUÉ PIDE EL EJERCICIO?

Explicar con palabras qué devuelve esta cadena de LINQ.

### c) EXPLICACIÓN PASO A PASO

Pensemos que en la tabla `Libro` hay 5 filas:

| Id | Título | Activo |
|----|--------|--------|
| 1 | "Cien años de soledad" | true |
| 2 | "Bajo la misma estrella" | false |
| 3 | "El Principito" | true |
| 4 | "Crónica de una muerte anunciada" | true |
| 5 | "Un libro apagado" | false |

Vamos por cada "eslabón" de la cadena. Fijate que son tres pedazos encadenados
con puntos `.`:

- `_context.Libro`: la puerta de entrada a la tabla de libros. Todavía no se
  ejecutó NADA contra la base: es una "consulta en construcción".
- `.Where(l => l.Activo)`: filtra. `l` representa "cada libro". Se queda solo con
  los que tienen `Activo == true`. En nuestra tabla: los Id 1, 3 y 4.
- `.OrderBy(l => l.Titulo)`: ordena de menor a mayor (A→Z) por título.
  Quedarían → "Cien años de soledad", "Crónica de una muerte anunciada",
  "El Principito".
- `.ToList()`: **convertir el resultado en una lista**. Y acá está el secreto:
  **hasta que no aparece el `.ToList()`, la base de datos NO se toca**.

Esto se llama **ejecución diferida**: las consultas LINQ sobre `_context.Libro` no
se ejecutan con cada línea, sino que van **construyendo la receta completa** y
recién con `.ToList()` le mandan a SQL **una sola pregunta** parecida a:

```sql
SELECT * FROM Libro WHERE Activo = 1 ORDER BY Titulo ASC;
```

O sea: los datos llegan **ya filtrados y ordenados**. No se trae toda la tabla a
memoria para después filtrar en C#.

| Eslabón | ¿Qué le hace a la consulta? | Resultado parcial |
|---------|------------------------------|-------------------|
| `_context.Libro` | "empezamos a construir" | (todavía nada se ejecutó) |
| `.Where(l => l.Activo)` | "dejá solo los activos" | Id 1, 3, 4 |
| `.OrderBy(l => l.Titulo)` | "ordená esos por título A→Z" | "Cien años...", "Crónica...", "El Principito" |
| `.ToList()` | "¡ejecutá y dame la lista final!" | Lista con los 3 libros |

Este detalle de la ejecución diferida es la explicación de por qué conviene que
estas consultas vivan **en el repositorio** (GQL a la base) y no en `Program.cs`
(que traería todo y filtraría en memoria, lento).

### d) RESULTADO FINAL

Devuelve una **lista con todos los libros activos, ordenados por título de forma
ascendente (A–Z)**.

### e) CONCEPTOS QUE APARECEN

- **LINQ**: lenguaje de consultas integrado en C#.
- **Lambda `=>`**: "dado un elemento `l`, evalúo esto".
- **`Where`**: filtrar según una condición.
- **`OrderBy`**: ordenar ascendente.
- **`ToList()`**: materializar ("obtener en concreto") el resultado.
- **Ejecución diferida**: las consultas no corren hasta que se "materializan".
- **`return`**: entregar el resultado a quien llamó el método.

### f) ⚠️ OJO / ERRORES COMUNES

- **`OrderBy` vs `OrderByDescending`**: `OrderBy` es A→Z; `OrderByDescending` es
  Z→A. Confundirlos invierte el resultado.
- **Leer el código como "tres viajes a la base"**: es UN viaje, una sola consulta SQL.
- **Olvidar el `.ToList()`**: sin él, podés ver resultados raros o errores al usar
  la variable, porque nunca se "materializó".
- **Filtrar en memoria después de traer todo**: `ObtenerTodos().Where(...)` hace el
  filtro en C#, trayendo todos los datos antes; es lo que esta consulta evita
  haciendo bien.

---

## Pregunta 4 — `_context` como `private` en la clase base

### a) CÓDIGO ORIGINAL

```csharp
public class GenericRepository<T>
{
    private readonly ApplicationDbContext _context;
}

public class LibroRepository : GenericRepository<Libro>
{
    public List<Libro> ObtenerActivos()
    {
        return _context.Libro
                       .Where(l => l.Activo)
                       .ToList();
    }
}
```

> Enunciado del documento: *"¿Compila este código? ¿Por qué? De haber algún error
> ¿cómo se corregiría?"*

### b) ¿QUÉ PIDE EL EJERCICIO?

Averiguar si el compilador acepta que `LibroRepository` use `_context`, un campo
que pertenece a `GenericRepository<T>` (su clase padre, de la que hereda).

### c) EXPLICACIÓN PASO A PASO

Primero, ¿qué es una **clase padre** y una **clase hija**?

- `GenericRepository<T>` es la clase **padre**: tiene el campo `_context` (el
  "cable" que conecta con la base de datos).
- `LibroRepository : GenericRepository<Libro>` es la clase **hija**: el signo `:`
  significa "heredar de". El hijo recibe todo lo del padre **según lo que el
  padre lo deje ver**.

El punto clave es el modificador `private`:

| Modificador en la clase padre | ¿La clase hija lo ve? | ¿Cualquier otra clase lo ve? |
|-------------------------------|----------------------|------------------------------|
| `public` | Sí | Sí (todo el programa) |
| `protected` | **Sí** | No (solo la familia) |
| `private` | **No, nunca** | No |
| `internal` | Sí | No (solo mismo proyecto) |

`private` significa **"solo visible dentro de esta clase exacta"**. Ni siquiera
las clases hijas lo ven. Es la puerta más cerrada que existe en C#.

Entonces, cuando `LibroRepository` intenta usar `_context`, es como si alguien
tratara de abrir la caja fuerte de su papá... pero el papá le puso una cerradura
`private`: **no tiene la llave**.

El compilador detecta esto **antes de ejecutar nada** y corta con un error, el
`CS0122`: *'GenericRepository<T>._context' no es accesible debido a su nivel de
protección*.

**La corrección**: cambiar `private` por `protected`:

```csharp
protected readonly ApplicationDbContext _context;
```

`protected` es exactamente "visible en la clase padre y en todas sus hijas, pero
no desde afuera". No es tan abierto como `public` (no queremos que `Program.cs`
toque `_context` directamente), ni tan cerrado como `private` (no queremos dejar
a las hijas afuera).

### d) RESULTADO FINAL

**No compila** (error `CS0122`). `_context` es `private` → invisible para
`LibroRepository`. Se corrige declarándolo **`protected`** (que es exactamente lo
que el proyecto real ya tiene).

### e) CONCEPTOS QUE APARECEN

- **Herencia (`:`)**: una clase hija que hereda de una clase padre.
- **Modificadores de acceso**: `public`, `protected`, `private`, que deciden
  quién puede ver cada cosa.
- **Campo**: variable declarada a nivel de clase (acá `_context`).
- **`readonly`**: el campo solo se puede asignar una vez (en el constructor).
- **Error de compilación**: el compilador rechaza el programa; no se puede correr.

### f) ⚠️ OJO / ERRORES COMUNES

- **Pensar que "heredar" significa "ver todo"**: no. Heredás lo que el modificador
  de acceso te deje ver.
- **Elegir `public` "para que ande"**: compila, pero es mala práctica: `_context`
  es un detalle interno que el resto del programa no debería tocar. La opción casi
  automática para "compartir con las hijas" es `protected`.
- **Confundir `private` con `protected`**: `private` = solo esta clase;
  `protected` = esta clase + sus hijas.
- **No leer el error del compilador**: el mensaje `CS0122` dice exactamente cuál
  es el problema de acceso.

---

## Pregunta 5 — LINQ mal ubicado en `Program.cs`

### a) CÓDIGO ORIGINAL

```csharp
IGenericRepository<Libro> libroRepository = new GenericRepository<Libro>();

var libros = libroRepository
                .ObtenerTodos()
                .Where(l => l.Activo)
                .ToList();

foreach (var libro in libros)
{
    Console.WriteLine($"{libro.Titulo} - {libro.Autor.Nombre}");
}
```

> Enunciado del documento: *"Supongamos que el siguiente código se encuentra en
> Program.cs. ¿Es correcto?"* (la respuesta oficial: "funciona, pero la lógica de
> filtrado quedó mal ubicada, y `ObtenerTodos()` no obtiene la relación con Autor")

### b) ¿QUÉ PIDE EL EJERCICIO?

Analizar si el código compila/corre y detectar **todos** los problemas que tenga.

### c) EXPLICACIÓN PASO A PASO

Este fragmento tiene **DOS problemas distintos**. El error típico de principiante
es encontrar UNO y dar por terminado el análisis. Acá tienen que ver los dos.

**¿Compila? ¿Corre?** Compila sin problemas (el `.Where` sobre una `List<Libro>`
es válido). Corre... hasta el primer `Console.WriteLine`, donde se rompe.

Vamos por partes.

**Problema 1 (diseño): el filtro se hizo en `Program.cs`, no en el Repository.**

- `.ObtenerTodos()` trae de la base **TODOS** los libros (activos e inactivos),
  porque el filtro se aplica **después**, recién cuando el método devolvió todo.
- `.Where(l => l.Activo)` al estar **después** de `ObtenerTodos()` se ejecuta
  **en memoria, en C#**: la base ya mandó todos los registros y recién acá se
  descartan los inactivos.
- Es lo mismo que pedir que te manden 500 libros por correo para tirar 300 acá,
  en vez de pedir solo los 200 que necesitás.
- La consulta debería vivir en `LibroRepository` (por ejemplo, un método
  `ObtenerLibrosActivos()`), para que el filtro se resuelva en SQL, en la base.

**Problema 2 (error real en tiempo de ejecución): falta el `Include` del autor.**

- `ObtenerTodos()` no usa `Include`, así que `libro.Autor` llega `null` (mismo
  mecanismo de la Pregunta 2).
- En el `foreach`, la línea `$"{libro.Titulo} - {libro.Autor.Nombre}"` va a
  explotar con `NullReferenceException` en el primer libro.

Tabla de seguimiento:

| Línea | Código | ¿Qué pasa? |
|-------|--------|------------|
| 1 | `new GenericRepository<Libro>()` | se crea el repositorio (vía interfaz) |
| 4 | `.ObtenerTodos()` | trae TODOS los libros (activos e inactivos), sin autor |
| 5 | `.Where(l => l.Activo)` | filtra **en memoria** los activos |
| 6 | `.ToList()` | lista final |
| 9 | `foreach` | arranca el recorrido |
| 10 | `$"...{libro.Autor.Nombre}"` | 💥 CRASH: `NullReferenceException` (`Autor` es `null`) |

**La corrección completa** (diseño + error):

```csharp
// En LibroRepository:
public List<Libro> ObtenerLibrosActivos()
{
    return _context.Libro
                   .Include(l => l.Autor)
                   .Where(l => l.Activo)
                   .ToList();
}
```

### d) RESULTADO FINAL

Compila y arranca, pero tiene **dos problemas**:
1. La lógica de filtrado está mal ubicada en `Program.cs` en vez del Repository
   (filtra en memoria, trayendo de más a la red).
2. `ObtenerTodos()` no carga `Autor` → `NullReferenceException` al imprimir
   `libro.Autor.Nombre`.

### e) CONCEPTOS QUE APARECEN

- **Interfaz**: un "tipo de variable" que muestra solo los métodos del contrato.
- **`$"..."`**: texto con interpolación (huecos `{ }` que se rellenan con valores).
- **Filtrado en memoria vs en base de datos**: dónde se resuelve el `Where`.
- **Capas del programa**: `Program.cs` (menú, muestra resultados) vs los
  `Repository` (consultan la base). Cada cosa en su lugar.
- **`Include`**: cargar la relación para no fallar.

### f) ⚠️ OJO / ERRORES COMUNES

- **Conformarse con un solo problema**: el enunciado pide analizar el código
  completo; hay que listar TODOS los problemas (acá: uno de diseño y un crash).
- **No distinguir "no compila" de "se rompe al correr"**: este código compila y
  se rompe en ejecución. La Pregunta 4 era lo contrario (ni compila).
- **Creer que `.Where` después de `.ToList()` es igual a filtrar en SQL**: no,
  eso ya es memoria.
- **$` con `Author` minúscula o nombres típicos**: en el proyecto real es
  `libro.Autor.Nombre` (el objeto navegación completo).

---

## Pregunta 6 — la interfaz no tiene el método específico

### a) CÓDIGO ORIGINAL

```csharp
IGenericRepository<Libro> libroRepository = new GenericRepository<Libro>();
var libros = libroRepository.ObtenerLibrosMasRecientes();
```

> Enunciado del documento: *"¿Compila este código? ¿Por qué? De haber algún error
> ¿cómo se corregiría?"*

### b) ¿QUÉ PIDE EL EJERCICIO?

Averiguar si el código puede compilar, sabiendo que el método
`ObtenerLibrosMasRecientes()` **no está** en la interfaz `IGenericRepository<T>`.

### c) EXPLICACIÓN PASO A PASO

Para entender, separo las dos líneas:

```csharp
IGenericRepository<Libro> libroRepository = new GenericRepository<Libro>();
//  ^tipo de la variable              ^objeto real que se crea
```

**La regla de oro**: cuando llamás a un método, el compilador **no mira qué objeto
se creó** (`new GenericRepository<Libro>()`). Mira **el tipo de la variable**
(`IGenericRepository<Libro>`).

- `libroRepository` está **declarada** como `IGenericRepository<Libro>`.
- Esa interfaz solo anuncia estos métodos: `Agregar`, `ObtenerTodos`,
  `ObtenerTodosCon`, `ObtenerPorId`, `Modificar`, `Eliminar`.
- `ObtenerLibrosMasRecientes()` **no está en esa lista**.

Es como si la variable llevara una "carta de presentación" que enumera lo que
"sabe hacer". El método nuevo no está en la carta → **el compilador no te deja
escribirlo**, y tira el error `CS1061`: *'IGenericRepository<Libro>' no contiene
una definición para 'ObtenerLibrosMasRecientes'*.

Y ojo: aun si el objeto real fuera un `LibroRepository` (que sí tiene el método),
el resultado sería el mismo, porque **el compilador mira la variable, no el objeto**.
"El tipo de la variable decide qué se puede escribir."

| Plano de la película | ¿Qué "se sabe"? |
|----------------------|------------------|
| El objeto real (`new GenericRepository<Libro>()`) | "sabe" los 6 métodos genéricos (y si fuera LibroRepository, sabría más) |
| El tipo declarado (`IGenericRepository<Libro>`) | solo muestra los 6 del contrato → `ObtenerLibrosMasRecientes` NO ES VISIBLE |

**La corrección**: declarar la variable con el tipo concreto `LibroRepository`:

```csharp
LibroRepository libroRepository = new LibroRepository();
var libros = libroRepository.ObtenerLibrosMasRecientes();
```

Por eso el `Program.cs` real, cuando necesita los métodos específicos, declara
`LibroRepository libroRepository = new LibroRepository();` directamente, sin usar
la interfaz para ese caso.

> Esta misa regla ("el tipo declarado limita lo que podés ver") vale para
> herencia de clases, clases abstractas e interfaces: es siempre la misma idea.

### d) RESULTADO FINAL

**No compila** (error `CS1061`). La variable está tipada como
`IGenericRepository<Libro>`, que no contiene `ObtenerLibrosMasRecientes()`. Se
corrige tipando la variable como `LibroRepository`:

```csharp
LibroRepository libroRepository = new LibroRepository();
```

### e) CONCEPTOS QUE APARECEN

- **Interfaz**: contrato con los métodos que las clases deben implementar.
- **Tipo declarado de la variable**: lo que el compilador usa para permitir o no
  cada llamada.
- **Error de compilación `CS1061`**: "ese tipo no tiene ese miembro".
- **Método específico**: un método que vive solo en una clase concreta
  (no en la interfaz ni en la clase genérica).

### f) ⚠️ OJO / ERRORES COMUNES

- **Pensar "pero el objeto sí lo tiene"**: da igual. El compilador mira la
  **variable**, no el objeto.
- **Confundir interfaz con clase concreta**: la interfaz es solo la lista; la
  clase aporta los métodos extra, que se ven solo si la variable es el tipo correcto.
- **Asumir que "genérico" significa "tiene todo"**: `GenericRepository<T>` solo
  tiene lo genérico (de ahí su nombre). Las cosas específicas de cada entidad van
  en repositorios como `LibroRepository`.

---

# PARTE 2: EJERCICIOS SIMILARES EN NIVEL

Mismo estilo "analizar código", sobre el mismo dominio (`Libro`/`Autor`/
`Categoria`/repositorios). Resolvélos ANTES de destapar la solución.

## Ejercicios nivel de la Pregunta 1 (modelos / propiedades de navegación)

**N1.1 — ¿Está completo este modelo para navegar de un libro a su categoría?**

```csharp
public class Libro
{
    public int Id { get; set; }
    public string Titulo { get; set; }
    public int AutorId { get; set; }
    public Autor Autor { get; set; }
    public int CategoriaId { get; set; }
}
```

*Consigna:* el modelo tiene la relación con `Autor` completa. ¿Alcanza con
`CategoriaId` para poder escribir `libro.Categoria.Nombre`?

<details>
<summary>Ver solución</summary>

**No.** `CategoriaId` es la clave foránea (el número), pero falta la propiedad de
navegación `public Categoria Categoria { get; set; }`. Sin ella, `libro.Categoria`
no existe para el compilador y el `Include("Categoria")` no tendría de qué cargar.
Es el mismo caso exacto de la Pregunta 1, pero con `Categoria`.
</details>

**N1.2 — La relación vista desde el otro lado (Autor → Libros)**

```csharp
public class Autor
{
    public int Id { get; set; }
    public string Nombre { get; set; }
}
```

*Consigna:* una regla del negocio dice que "un autor puede tener muchos libros".
Con este modelo, ¿podés escribir `autor.Libros` para obtener la lista de libros
de un autor? ¿Qué propiedad falta y qué tipo tendría?

<details>
<summary>Ver solución</summary>

**No puede.** Falta la lista de navegación inversa:

```csharp
public List<Libro> Libros { get; set; } = new();
```

Es el lado "muchos" de la relación uno-a-muchos. Sin ella no podés recorrer los
libros de un autor desde C#. (En el proyecto real, `Autor` y `Categoria` tienen
esa `List<Libro> Libros`.)
</details>

## Ejercicios nivel de la Pregunta 2 (`Include`)

**N2.1 — Se cargó la relación equivocada**

```csharp
var libros = libroRepository.ObtenerTodosCon("Categoria");

foreach (var libro in libros)
{
    Console.WriteLine($"{libro.Titulo} - {libro.Autor.Nombre}");
}
```

*Consigna:* `ObtenerTodosCon("Categoria")` incluye la relación con `Categoria`.
¿Compila? ¿Y qué pasa en tiempo de ejecución en la línea de `Console.WriteLine`?

<details>
<summary>Ver solución</summary>

**Compila** (`ObtenerTodosCon` acepta cualquier texto y el compilador no valida el
nombre de la propiedad). **Falla al correr** con `NullReferenceException`: el
`Include` pidió `"Categoria"`, pero el código intenta leer `libro.Autor.Nombre`, y
`Autor` NO se cargó → queda `null`. El `Include` tiene que pedir la relación que
realmente vas a usar: `ObtenerTodosCon("Autor")`.
</details>

**N2.2 — Uso de DOS relaciones a la vez**

```csharp
var libros = libroRepository.ObtenerTodosCon("Autor");

foreach (var libro in libros)
{
    Console.WriteLine($"{libro.Titulo} - Autor: {libro.Autor.Nombre} - Categoría: {libro.Categoria.Nombre}");
}
```

*Consigna:* ahora se muestra el autor Y la categoría. ¿Con `Include("Autor")`
alcanza, o también acá hay un escenario de error?

<details>
<summary>Ver solución</summary>

**También hay error**: `Autor` está cargado (eso funciona), pero `Categoria` no se
incluyó y queda en `null`. La línea explota con `NullReferenceException` al tocar
`libro.Categoria.Nombre`. Para mostrar ambas necesitás que `ObtenerTodosCon`
pueda cargar más de un `Include` (el `GenericRepository` actual solo acepta un
string; habría que extenderlo para que reciba varias propiedades).
</details>

## Ejercicios nivel de la Pregunta 3 (consultas LINQ)

**N3.1 — Combina dos filtros y un orden**

```csharp
return _context.Libro
    .Where(l => l.Activo && l.AnioPublicacion > 2000)
    .OrderByDescending(l => l.AnioPublicacion)
    .ToList();
```

*Consigna:* explicá en una oración qué devuelve esta consulta. ¿Qué hace el `&&`?

<details>
<summary>Ver solución</summary>

Devuelve los libros que estén **activos Y con año de publicación mayor a 2000**,
ordenados por año de **mayor a menor** (los más nuevos primero). El `&&` es "Y":
las dos condiciones deben cumplirse a la vez para que el libro pase el filtro.
Cuidado: `OrderByDescending` ordena descendente; con `OrderBy` sería al revés.
</details>

**N3.2 — Contar inactivos**

```csharp
return _context.Libro
    .Where(l => !l.Activo)
    .Count();
```

*Consigna:* ¿qué devuelve este método cuando hay 6 libros y 2 de ellos inactivos?
¿Qué significa el `!` delante de `l.Activo`?

<details>
<summary>Ver solución</summary>

Devuelve **2**. El `Where` se queda solo con los libros que NO están activos
(`!` invierte el valor: `!true` = `false`), y el `Count()` cuenta cuántos pasaron
el filtro. Fijate que acá el filtro se hace antes del recuento, en la base de
datos, y no se trae nada de más.
</details>

## Ejercicios nivel de la Pregunta 4 (acceso private/protected)

**N4.1 — ¿Sabés corregir el mismo bug con otra entidad?**

```csharp
public class GenericRepository<T>
{
    private readonly ApplicationDbContext _context;
}

public class AutorRepository : GenericRepository<Autor>
{
    public int CantidadDeAutores()
    {
        return _context.Autor.Count();
    }
}
```

*Consigna:* ¿compila? ¿Cómo lo corregís? ¿Por qué `protected` es mejor que
`public` para este caso?

<details>
<summary>Ver solución</summary>

**No compila** (mismo `CS0122`: `_context` es `private`). Se corrige cambiándolo a
`protected readonly ApplicationDbContext _context;`. `protected` es mejor que
`public` porque `_context` es un detalle interno de los repositorios: las clases
hijas deben poder usarlo, pero el resto del programa (por ejemplo `Program.cs`)
no debería tocarlo directamente.
</details>

**N4.2 — Un método protegido en acción**

```csharp
public class GenericRepository<T>
{
    protected List<T> Ejecutar(Func<IQueryable<T>, List<T>> consulta)
    {
        // ...ejecuta la consulta y devuelve una lista...
    }
}

public class CategoriaRepository : GenericRepository<Categoria>
{
    public List<Categoria> CategoriasConLibros()
    {
        return Ejecutar(c => c.Where(x => x.Libros.Any()).ToList());
    }
}
```

*Consigna:* `Ejecutar` está `protected`. ¿`CategoriaRepository` puede llamarlo?
¿Y podría llamarlo una clase que NO herede de `GenericRepository<T>` (como
`Program`)? ¿Por qué?

<details>
<summary>Ver solución</summary>

**Sí puede** llamarlo `CategoriaRepository`, porque al heredar recibe los miembros
`protected` (son para la "familia"). **No puede** llamarlo `Program`, porque no
hereda de `GenericRepository<T>` y el `protected` solo se ve dentro de la clase
y sus hijos. Es exactamente el "ni tan cerrado ni tan abierto" de la Pregunta 4.
</details>

## Ejercicios nivel de la Pregunta 5 (lógica mal ubicada)

**N5.1 — El conteo también tiene su lugar**

```csharp
// En Program.cs:
var total = libroRepository.ObtenerTodos().Count();
Console.WriteLine($"Hay {total} libros.");
```

*Consigna:* ¿compila y corre? ¿Por qué está mal ubicado? ¿Dónde debería vivir esta
cuenta, y qué método del proyecto real ya la resuelve?

<details>
<summary>Ver solución</summary>

**Compila y corre. Pero**: para contar se traen a memoria TODOS los libros
(`ObtenerTodos()` sin filtros) y se cuentan en C#. La cuenta es una consulta de
agregado que debe resolverse en la base, en `LibroRepository`. El proyecto real
ya lo resuelve con `ObtenerCantidadLibros()` → `_context.Libro.Count()`.
</details>

**N5.2 — Ordenar en Program.cs**

```csharp
// En Program.cs:
foreach (var libro in libroRepository.ObtenerTodos().OrderBy(l => l.AnioPublicacion))
{
    Console.WriteLine(libro.Titulo);
}
```

*Consigna:* ¿qué problema de diseño tiene este código, además de que `ObtenerTodos`
no carga ninguna relación? (No hace falta que explotes por el `Autor`: acá solo se
imprime `Titulo`, que sí viene cargado.)

<details>
<summary>Ver solución</summary>

El `OrderBy` se aplica **en memoria** después de traer TODOS los libros; el orden
debería pedirse en la base, dentro del repositorio (el proyecto real tiene
`ObtenerLibrosOrdenadosPorTitulo()` y `ObtenerLibrosPorMasRecientes()`). Traer
todo solo para ordenarlo es ineficiente cuando la tabla crece. Y como detalle
apartes: acá no hay `NullReferenceException` porque no se toca ninguna propiedad
de navegación.
</details>

## Ejercicios nivel de la Pregunta 6 (interfaz vs clase concreta)

**N6.1 — El objeto SÍ es el correcto... y aun así falla**

```csharp
IGenericRepository<Libro> libroRepository = new LibroRepository();
var libros = libroRepository.ObtenerLibrosMasRecientes();
```

*Consigna:* acá el objeto real ES un `LibroRepository` (tiene el método). ¿Compila
igual? ¿Qué principio de la Pregunta 6 se está aplicando?

<details>
<summary>Ver solución</summary>

**No compila igual.** El objeto real da igual: el compilador mira el tipo de la
variable (`IGenericRepository<Libro>`), que no declara `ObtenerLibrosMasRecientes`. Es
el principio exacto de la Pregunta 6: "el tipo declarado decide qué se puede
escribir, no lo que el objeto sabe hacer". Corrección: declarar la variable como
`LibroRepository`.
</details>

**N6.2 — ¿Y con Autor?**

```csharp
IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();
autorRepository.Agregar(autor);   //  ✔ método del contrato
var todos = autorRepository.ObtenerTodos();  //  ✔ método del contrato
```

*Consigna:* ¿por qué acá SÍ compila todo, a diferencia de la Pregunta 6? ¿Qué
pasaría si intentaras llamar a `autorRepository.DarDeBaja(logico)` inventando un
método que solo existiera en una clase `AutorRepository` que no está en la interfaz?

<details>
<summary>Ver solución</summary>

Compila porque `Agregar` y `ObtenerTodos` **están en la interfaz** `IGenericRepository<T>`
(son parte del contrato genérico). Si inventaras `autorRepository.DarDeBaja(...)` (método
que no está en la interfaz), el compilador tira `CS1061`, el mismo error de la
Pregunta 6: la variable solo "muestra" lo que su tipo declara.
</details>

---

## Resumen rápido (chuleta)

- **Clave foránea (`AutorId`) ≠ propiedad de navegación (`Autor`)**: la primera es
  el número que guarda la base; la segunda, el objeto que te deja navegar en C#.
  Se necesitan las dos.
- **Sin `Include`, la relación queda `null`** → `NullReferenceException` al primer
  acceso. Además, el `Include` tiene que cargar **la propiedad que vas a usar**.
- **LINQ no ejecuta nada hasta `.ToList()`**: `Where`/`OrderBy`/`Count` se
  traducen a **una sola** consulta SQL y se resuelven en la base.
- **Las consultas específicas viven en el repositorio**, no en `Program.cs`
  (filtrar/ordenar/contar en memoria trayendo todo = lento y mal ubicado).
- **`private` ni las hijas lo ven**; para compartir con clases hijas → `protected`
  (ni `private` ni `public`).
- **El tipo declarado de la variable limita lo que podés escribir**, sin importar
  el objeto real. Vale igual en herencia, clases abstractas e interfaces.
- **Un fragmento puede tener varios problemas a la vez**: buscá todos (ej. uno de
  diseño + uno de `null` en la Pregunta 5).