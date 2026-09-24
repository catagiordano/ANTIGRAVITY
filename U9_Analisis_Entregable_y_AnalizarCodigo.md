# U9 — Proyecto entregable + Ejercicio "Analizar código": explicación completa

Dos materiales, fuertemente relacionados entre sí: el proyecto `U9EjercicioEntregable` es la app real (patrón repositorio genérico + repositorio específico + LINQ), y `ANALIZAR_CODIGO` es un ejercicio de 6 preguntas de interpretación/corrección de código **sobre ese mismo dominio** (a veces mostrando versiones "con bug" de código que en el proyecto real ya está corregido). Por eso conviene leerlos juntos: cada respuesta del ejercicio se puede **verificar contra el código real** del entregable.

---

# PARTE A — Recorrido del proyecto entregable

## A.1 Arquitectura general

```
AppConsola (solucion)
├─ AccesoDatos/                    (biblioteca de clases)
│  ├─ Data/ApplicationDbContext.cs
│  ├─ Models/  -> Autor, Categoria, Libro
│  └─ Repositories/
│     ├─ IGenericRepository<T>
│     ├─ GenericRepository<T>
│     └─ LibroRepository : GenericRepository<Libro>   (repositorio ESPECIFICO)
└─ AppConsola/Program.cs           (menu)
```

Es el mismo patrón ya visto (biblioteca de clases + app de consola + repositorio genérico), con un agregado nuevo importante: **`LibroRepository`**, un repositorio que **hereda** de `GenericRepository<Libro>` y le suma consultas LINQ que son específicas de `Libro` y no tendría sentido poner en el genérico (porque ninguna otra entidad las necesita). Esto es exactamente la aplicación práctica del criterio "¿esta consulta la necesitan todas las entidades, o es específica de una sola?" trabajado antes en esta conversación.

## A.2 `ApplicationDbContext`

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Autor> Autor { get; set; }
    public DbSet<Libro> Libro { get; set; }
    public DbSet<Categoria> Categoria { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlite("Data Source=C:\\databases\\BaseDatosEjercicios.db");
    }
}
```

Un detalle de nomenclatura para tener presente: los `DbSet<T>` están nombrados en **singular** (`Autor`, `Libro`, `Categoria`), igual que la clase que representan, en vez del plural más habitual (`Autores`, `Libros`). Funciona exactamente igual, EF Core no exige plural, es solo una convención de estilo, pero es importante que **el nombre exacto de la propiedad** (`Libro`, con mayúscula) sea el que se usa después en cualquier consulta directa contra el contexto (`_context.Libro`, no `_context.Libros`). Este detalle es, de hecho, la base de uno de los ejercicios propios nuevos (ver C.2).

## A.3 Modelos

`Autor` y `Categoria` tienen cada uno una lista de navegación `List<Libro> Libros` (el lado "muchos" de la relación uno-a-muchos). `Libro` tiene las dos claves foráneas (`AutorId`, `CategoriaId`) junto con sus **propiedades de navegación** (`Autor`, `Categoria`) y un campo `Activo` para **borrado lógico** (nunca se elimina la fila realmente, se marca `Activo = false`).

## A.4 `GenericRepository<T>` — el detalle que conecta con el ejercicio de análisis

```csharp
public class GenericRepository<T> : IGenericRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;

    public GenericRepository()
    {
        _context = new ApplicationDbContext();
    }
    // Agregar, ObtenerTodos, ObtenerTodosCon, ObtenerPorId, Modificar, Eliminar...
}
```

**Este `protected` en `_context` no es casual**, es exactamente la corrección que pide la Pregunta 4 del ejercicio de análisis (ver Parte B.4). En el proyecto real, `_context` **ya está** como `protected readonly`, precisamente para que `LibroRepository`, al heredar de `GenericRepository<Libro>`, pueda usarlo directamente en sus propias consultas.

## A.5 `LibroRepository` — el repositorio específico

```csharp
public class LibroRepository : GenericRepository<Libro>
{
    public List<Libro> ObtenerLibrosPorMasRecientes()
        => _context.Libro.OrderByDescending(l => l.AnioPublicacion).ToList();

    public int ObtenerCantidadLibros()
        => _context.Libro.Count();

    public int ObtenerCantidadLibrosActivos()
        => _context.Libro.Count(l => l.Activo);

    public Libro? ObtenerLibroPorId(int id)
        => _context.Libro.FirstOrDefault(l => l.Id == id);

    public List<Libro> ObtenerLibrosOrdenadosPorTitulo()
        => _context.Libro.OrderBy(l => l.Titulo).ToList();

    public bool ExistenLibrosActivos()
        => _context.Libro.Any(l => l.Activo);
}
```

Cada uno de estos seis métodos es una consulta LINQ **traducida directamente a SQL** por EF Core (no se trae todo a memoria y se filtra después: `Where`, `Count`, `Any`, `OrderBy` se ejecutan del lado de la base de datos gracias a que `_context.Libro` es un `IQueryable<Libro>`, no una `List<Libro>` todavía). Esto es lo que en la clase de consulta se identificó como el criterio correcto: consultas específicas de una sola entidad, resueltas en un repositorio propio, no en `Program.cs`.

**Por qué `LibroRepository` puede usar `_context` sin declararlo de nuevo:** porque lo heredó de `GenericRepository<Libro>`, exactamente igual que en el ejemplo de herencia visto antes (`LibroFisico`/`LibroDigital` heredando de `Libro`). El constructor de `GenericRepository<T>` ya crea el `ApplicationDbContext` y lo guarda en `_context`; `LibroRepository` no tiene ni necesita su propio constructor.

## A.6 `Program.cs` — dos tipos de variable, a propósito

```csharp
IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();
IGenericRepository<Categoria> categoriaRepository = new GenericRepository<Categoria>();
LibroRepository libroRepository = new LibroRepository();
```

Fijate la diferencia: `autorRepository` y `categoriaRepository` están tipados con la **interfaz genérica** (`IGenericRepository<T>`), porque para `Autor` y `Categoria` **alcanza** con las operaciones básicas (`Agregar`, `ObtenerTodos`, etc.). `libroRepository`, en cambio, está tipado directamente como `LibroRepository` (la clase concreta), **porque el programa necesita llamar a métodos que no están en la interfaz genérica** (`ObtenerLibrosPorMasRecientes`, `ObtenerCantidadLibros`, etc.). Si `libroRepository` estuviera declarado como `IGenericRepository<Libro>`, ninguna de esas llamadas compilaría — es exactamente el mismo principio que la Pregunta 6 del ejercicio de análisis (Parte B.6) y que la Pregunta 2 de "interpretación de código" del Segundo Examen (Camión/Vehículo): **el tipo de la variable decide qué miembros ves, no lo que el objeto real "sepa hacer".**

---

# PARTE B — Ejercicio "Analizar código": explicación profunda de cada pregunta

## B.1 Pregunta 1 — Falta la propiedad de navegación `Autor`

**Código del ejercicio:**
```csharp
public class Libro
{
    public int Id { get; set; }
    public string Titulo { get; set; }
    public int AutorId { get; set; }
}
```

**Consigna:** un libro pertenece a un único autor, ¿está completo este modelo?

**Respuesta oficial:** no, falta `public Autor Autor { get; set; }`.

**Por qué, en profundidad:** `AutorId` es la **clave foránea** (un `int`, el dato "crudo" que se guarda en la columna de la tabla `Libro`). Eso alcanza para que la base de datos sepa a qué autor pertenece cada libro, pero **no alcanza** para que el código C# pueda **navegar** esa relación como objeto. Sin `public Autor Autor { get; set; }` (la **propiedad de navegación**):

- No podés escribir `libro.Autor.Nombre` en ningún lado del código, el compilador directamente no reconoce `Autor` como miembro de `Libro`.
- **`Include("Autor")` no tiene forma de funcionar.** El método `Include` necesita el nombre de una **propiedad de navegación** para saber qué relación cargar; sin esa propiedad declarada, no hay nada que EF Core pueda "incluir" para esta relación.

**Verificación contra el proyecto real:** el `Libro.cs` del entregable **sí** tiene la propiedad completa:
```csharp
public int AutorId { get; set; }
public Autor Autor { get; set; }
public int CategoriaId { get; set; }
public Categoria Categoria { get; set; }
```
Es decir, el ejercicio muestra una versión **recortada a propósito** (le sacaron `Autor`, `CategoriaId`, `Categoria` y `Activo`) para que la pregunta tenga sentido, el proyecto entregable es la versión ya corregida.

> ⚠️ **Punto que vale en el final:** clave foránea (`AutorId`, un dato escalar) y propiedad de navegación (`Autor`, un objeto) **son dos cosas distintas y ambas hacen falta** para que la relación funcione completamente en EF Core: una para la base de datos, otra para poder navegar el objeto en C#.

## B.2 Pregunta 2 — `libro.Autor.Nombre` sin `Include`

**Código del ejercicio:**
```csharp
var libros = libroRepository.ObtenerTodos();

foreach (var libro in libros)
{
    Console.WriteLine(libro.Autor.Nombre);
}
```

**Consigna:** ¿es correcto? ¿Hay algún escenario en el que produzca error?

**Respuesta oficial:** la solución es usar `ObtenerTodosCon("Autor")`, que aplica `Include`.

**Por qué, en profundidad, el escenario de error exacto:** `ObtenerTodos()` (el método genérico) hace esto:
```csharp
public List<T> ObtenerTodos()
    => _context.Set<T>().AsNoTracking().ToList();
```
No tiene **ningún** `Include`. Con la configuración estándar de EF Core (sin *lazy loading* habilitado, que es la configuración que se usa en todo este curso), las propiedades de navegación que no se pidieron explícitamente con `Include` **quedan en `null`** después de traer los datos. Entonces:

- El `foreach` recorre bien la lista de libros (eso sí funciona).
- Pero **la primera vez** que se ejecuta `libro.Autor.Nombre` sobre un libro cuyo `Autor` no fue cargado, el programa lanza `NullReferenceException`, porque `libro.Autor` es `null`, y no se puede acceder a `.Nombre` sobre `null`.

Este es un error que **no aparece siempre**: si por casualidad EF Core sí hubiera cargado esa relación (por ejemplo, si la entidad ya estuviera *trackeada* en memoria de una consulta anterior en el mismo `_context`, cosa que acá tampoco pasa, porque además se usa `AsNoTracking()`), no fallaría. Eso lo hace más peligroso todavía: **el código puede "andar" en una prueba rápida y fallar en otro momento**, si no se entiende bien la causa.

**Las tres formas de cargar una relación en EF Core** (para tener el panorama completo):
1. **Eager loading** (`Include`, lo que corresponde acá): se pide la relación en la misma consulta.
2. **Explicit loading** (`_context.Entry(libro).Reference(l => l.Autor).Load()`): se carga la relación después, a demanda, para una entidad puntual.
3. **Lazy loading**: la relación se carga sola, automáticamente, la primera vez que se accede, pero requiere el paquete `Microsoft.EntityFrameworkCore.Proxies`, activar `UseLazyLoadingProxies()`, y que las propiedades de navegación sean `virtual`. **No se usa en este curso** (el proyecto no tiene ese paquete ni esa configuración), así que en este contexto la única opción válida es *eager loading* con `Include`.

**Verificación contra el proyecto real:** `Program.cs`, en `MostrarLibros()`, hace exactamente lo correcto:
```csharp
var libros = libroRepository.ObtenerTodosCon("Autor");
...
$"Autor: {libro.Autor.Nombre}"
```

> ⚠️ **Punto que vale en el final:** si en el examen ves `objeto.PropiedadDeNavegacion.Algo` después de una consulta que **no** tiene `Include`, sospechá de `NullReferenceException` como primera hipótesis.

## B.3 Pregunta 3 — Explicar la consulta

**Código del ejercicio:**
```csharp
return _context.Libro
    .Where(l => l.Activo)
    .OrderBy(l => l.Titulo)
    .ToList();
```

**Respuesta oficial:** obtiene todos los libros activos, ordenados por título ascendente, y devuelve el resultado como lista.

**Por qué, en profundidad, lo que no se ve a simple vista:** este método no ejecuta "tres pasos" contra la base de datos (uno por cada línea). Hasta que no aparece `.ToList()`, **no se ejecuta ninguna consulta SQL todavía**, `_context.Libro` es un `IQueryable<Libro>`, y cada `.Where(...)` / `.OrderBy(...)` que se le encadena **no filtra ni ordena nada en memoria**, sino que va **construyendo una expresión** que representa la consulta completa. Recién cuando aparece `.ToList()` (lo que se llama **ejecución diferida**, *deferred execution*), EF Core traduce **toda la cadena junta** a una única sentencia SQL equivalente a:
```sql
SELECT * FROM Libro WHERE Activo = 1 ORDER BY Titulo ASC;
```
y la ejecuta una sola vez contra la base de datos, trayendo ya filtrados y ordenados solo los datos que hacen falta, no toda la tabla para después filtrar en C#.

**Por qué esto importa tanto en el examen:** es la justificación técnica exacta de por qué "una consulta LINQ específica se resuelve en el repositorio, no trayendo todo con `ObtenerTodos()` y filtrando después en `Program.cs`" (tema visto en la clase de consulta): si en cambio se hiciera `libroRepository.ObtenerTodos().Where(l => l.Activo)`, el `.Where` se ejecutaría **en memoria, en C#, después de haber traído absolutamente todos los libros de la tabla**, mucho menos eficiente cuantos más libros haya.

**Verificación contra el proyecto real:** este es literalmente el cuerpo de `LibroRepository.ObtenerCantidadLibrosActivos()` combinado con `ObtenerLibrosOrdenadosPorTitulo()`, el ejercicio junta ambos criterios (activo + orden) en una sola consulta de ejemplo.

## B.4 Pregunta 4 — `_context` como `private` en la clase base

**Código del ejercicio:**
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

**Consigna:** ¿compila? ¿Hay algún escenario de error?

**Respuesta oficial:** no compila. `_context` es `private`, así que no es visible desde `LibroRepository`. La solución es declararlo `protected`.

**Por qué, en profundidad, repasando la tabla de modificadores de acceso en herencia (ya vista antes en esta conversación):**

| Modificador en la clase base | ¿La clase hija lo ve? |
|---|---|
| `public` | Sí |
| `protected` | Sí, es específicamente para esto |
| `internal` | Sí, si está en el mismo proyecto |
| `private` | **No, nunca**, ni siquiera las clases hijas |

`private` es el nivel de acceso más restrictivo que existe en C#: significa "solo visible dentro de esta clase exacta, ni siquiera las que hereden de ella". Por eso, aunque `LibroRepository : GenericRepository<Libro>` herede de la clase base, **no puede ver** su campo `_context` si está declarado `private`, el compilador tira `CS0122: 'GenericRepository<T>._context' is inaccessible due to its protection level`.

El fix es cambiar el modificador a `protected`, que es exactamente "visible en la clase base y en toda su descendencia", ni tan cerrado como `private` (que ni las hijas ven), ni tan abierto como `public` (que vería cualquier clase del programa, incluido `Program.cs`, lo cual sería mala práctica: `_context` es un detalle interno de cómo el repositorio accede a los datos, no algo que el resto del programa debería tocar directamente).

**Verificación contra el proyecto real:** el `GenericRepository.cs` del entregable **ya tiene** `protected readonly ApplicationDbContext _context;`, es la versión corregida, coincide exactamente con lo que pide la respuesta del ejercicio.

> ⚠️ **Punto que vale en el final:** cuando una clase base va a tener clases hijas que necesitan reutilizar un campo o método internamente (no desde afuera del todo el sistema), la elección casi automática es `protected`, ni `private` ni `public`.

## B.5 Pregunta 5 — LINQ mal ubicado en `Program.cs`

**Código del ejercicio:**
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

**Respuesta oficial:** funciona, pero la lógica de filtrado quedó mal ubicada en `Program.cs` en vez de en un Repository; además `ObtenerTodos()` no trae la relación con `Autor`.

**Ampliando (la respuesta oficial junta dos problemas distintos, separémoslos):**

**Problema 1, de diseño (no impide que compile ni que corra):** el filtro `.Where(l => l.Activo)` se está resolviendo **en Program.cs**, trayendo primero **todos** los libros (`ObtenerTodos()`, sin filtrar en la base), y recién después descartando los inactivos **en memoria**, en C#. Es exactamente el caso "mal" del criterio de diseño LINQ visto antes: esta consulta debería vivir en `LibroRepository` (por ejemplo, reutilizando `ObtenerCantidadLibrosActivos` o un nuevo método `ObtenerLibrosActivos()`), para que el filtro se resuelva en SQL y no se traiga de la base más de lo necesario.

**Problema 2, un error real en tiempo de ejecución (esto es lo más importante para el examen, y la respuesta oficial lo menciona de pasada):** `ObtenerTodos()` no tiene `Include`, así que `libro.Autor` va a ser `null` para cada libro, y la línea `$"{libro.Titulo} - {libro.Autor.Nombre}"` va a lanzar **`NullReferenceException`** en el primer libro que recorra el `foreach`. Es el mismo mecanismo exacto que la Pregunta 2 (B.2): acá se repite a propósito, con el agregado del problema de diseño, para practicar que **un mismo fragmento de código puede tener más de un problema a la vez**, y hay que identificarlos todos, no conformarse con encontrar el primero.

## B.6 Pregunta 6 — la interfaz no tiene el método específico

**Código del ejercicio:**
```csharp
IGenericRepository<Libro> libroRepository = new GenericRepository<Libro>();
var libros = libroRepository.ObtenerLibrosMasRecientes();
```

**Respuesta oficial:** no compila, porque la variable está tipada como `IGenericRepository<Libro>`, y esa interfaz no declara `ObtenerLibrosMasRecientes()`. La solución es usar `LibroRepository libroRepository = new LibroRepository();`.

**Por qué, en profundidad, el mismo principio del upcasting, aplicado a interfaces:** `IGenericRepository<T>` solo declara `Agregar`, `ObtenerTodos`, `ObtenerTodosCon`, `ObtenerPorId`, `Modificar`, `Eliminar`. `ObtenerLibrosMasRecientes()` es un método que **solo existe en `LibroRepository`**, no en la interfaz ni en `GenericRepository<T>`. Aunque el objeto creado en memoria (`new GenericRepository<Libro>()`) fuera, hipotéticamente, un `LibroRepository` (en este caso ni siquiera lo es, se instancia `GenericRepository<Libro>`, no `LibroRepository`), **el compilador solo mira el tipo de la variable** (`IGenericRepository<Libro>`) para decidir qué miembros dejarte escribir. El error es `CS1061: 'IGenericRepository<Libro>' no contiene una definición para 'ObtenerLibrosMasRecientes'`.

Es exactamente el mismo mecanismo que la Pregunta 2 de "interpretación de código" del Segundo Examen (`Vehiculo camion = new Camion(...)`, donde `camion.CapacidadAdicional` no compilaba), ahí era una clase abstracta, acá es una interfaz, pero **la regla es idéntica**: el tipo de la variable limita qué ves, sin importar el tipo real del objeto.

**Verificación contra el proyecto real:** `Program.cs` declara exactamente `LibroRepository libroRepository = new LibroRepository();` (tipo concreto, no la interfaz), que es la corrección que pide la respuesta.

> ⚠️ **Punto que vale en el final:** esta regla ("el tipo de la variable decide qué se puede escribir, no el tipo real del objeto") es la misma en herencia de clases, en clases abstractas y en interfaces. Si un examen te la pregunta con cualquiera de las tres, el razonamiento es idéntico.

---

# PARTE C — Ejercicios propios de nivel igual o superior

Mismo estilo "analizar código", sobre el mismo dominio (`Libro`/`Autor`/`Categoria`/`LibroRepository`), para seguir practicando. Solución colapsada.

## C.1 (mismo nivel) — `Include` con la propiedad equivocada

```csharp
var libros = libroRepository.ObtenerTodosCon("Categoria");

foreach (var libro in libros)
{
    Console.WriteLine($"{libro.Titulo} - {libro.Autor.Nombre}");
}
```

**Consigna:** ¿compila? ¿Corre sin errores?

<details>
<summary>Ver solución</summary>

**Compila perfecto** (`ObtenerTodosCon` acepta cualquier `string`, el compilador no valida el nombre de la propiedad). **Falla en tiempo de ejecución** con `NullReferenceException` en `libro.Autor.Nombre`, porque el `Include` que se pidió fue `"Categoria"`, no `"Autor"`, se cargó la relación equivocada. `libro.Categoria` sí estaría disponible acá, pero no se usa. Corrección: `ObtenerTodosCon("Autor")`, o si hicieran falta las dos relaciones a la vez, EF Core permite encadenar más de un `Include` (no con este repositorio genérico tal como está, que solo admite un string, sería necesario extender `ObtenerTodosCon` para aceptar varias propiedades, o usar el patrón de ruta con puntos si fuera una relación anidada).
</details>

## C.2 (mismo nivel) — Sensibilidad a mayúsculas en el nombre del `DbSet`

```csharp
public class ApplicationDbContext : DbContext
{
    public DbSet<Libro> libro { get; set; }   // notese la minuscula
}
```
```csharp
public class LibroRepository : GenericRepository<Libro>
{
    public List<Libro> ObtenerActivos()
        => _context.Libro.Where(l => l.Activo).ToList();
}
```

**Consigna:** ¿compila este `LibroRepository`?

<details>
<summary>Ver solución</summary>

**No compila.** C# es **sensible a mayúsculas y minúsculas** (*case-sensitive*): `libro` (la propiedad tal como quedó declarada en `ApplicationDbContext`) y `Libro` (como se la intenta usar en `LibroRepository`) son **dos identificadores completamente distintos** para el compilador, aunque un humano los lea como "la misma palabra". El error es `CS1061: 'ApplicationDbContext' no contiene una definición para 'Libro'` (con mayúscula), el compilador ni siquiera sugiere automáticamente que quisiste decir `libro`, aunque algunos IDEs sí lo hagan como ayuda visual. Corrección: usar `_context.libro` (respetando el nombre exacto tal como fue declarado), o, mejor práctica, renombrar la propiedad del `DbContext` a `Libro` (con mayúscula, PascalCase, como corresponde a una propiedad pública en C#) y usarla así en todos lados.
</details>

## C.3 (nivel superior) — Ocultamiento de campo (`field hiding`) entre clase base e hija

```csharp
public class GenericRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    public GenericRepository() { _context = new ApplicationDbContext(); }
}

public class LibroRepository : GenericRepository<Libro>
{
    private readonly ApplicationDbContext _context = new ApplicationDbContext();

    public List<Libro> ObtenerActivos()
        => _context.Libro.Where(l => l.Activo).ToList();
}
```

**Consigna:** ¿compila? Si compila, ¿hay algún problema real de todos modos?

<details>
<summary>Ver solución</summary>

**Compila**, pero con una advertencia del compilador (`CS0108: 'LibroRepository._context' oculta el miembro heredado 'GenericRepository<T>._context'. Use la palabra clave new si el ocultamiento era intencional.`). Esto es **ocultamiento de campo** (*field hiding*), el mismo mecanismo que el ocultamiento de métodos con `new` visto antes, aplicado acá a un campo en vez de a un método: `LibroRepository` declara su **propio** `_context`, que es un objeto **completamente distinto** del `_context` heredado de `GenericRepository<Libro>`, ahora existen **dos instancias separadas** de `ApplicationDbContext` dentro del mismo objeto `LibroRepository`. Dentro de `ObtenerActivos()`, `_context` se refiere al campo **propio** de `LibroRepository` (el más "cercano" en la búsqueda de nombres), no al heredado. En la práctica esto no rompe `ObtenerActivos()` en sí (usa su propio contexto, consistente), pero es una fuente de bugs sutiles si en algún momento se mezclan operaciones que deberían compartir el mismo contexto (por ejemplo, si `Agregar`, heredado, usa el contexto de la base, y `ObtenerActivos`, propio, usa el contexto redeclarado, necesitaran ver los mismos cambios sin guardar todavía). La corrección es simplemente **no** redeclarar `_context` en `LibroRepository`: ya lo tiene disponible, heredado y `protected`, tal como está resuelto en el proyecto real.
</details>

## C.4 (nivel superior, implementación) — Agregar un método nuevo a `LibroRepository`

**Consigna:** implementá en `LibroRepository` un método `ObtenerLibrosPorAutor(int autorId)` que devuelva todos los libros activos de un autor puntual, con el autor ya cargado (para poder mostrar `libro.Autor.Nombre` sin riesgo de `NullReferenceException`).

<details>
<summary>Ver solución</summary>

```csharp
public List<Libro> ObtenerLibrosPorAutor(int autorId)
{
    return _context.Libro
                   .Include(l => l.Autor)
                   .Where(l => l.AutorId == autorId && l.Activo)
                   .ToList();
}
```

Puntos a justificar si te lo piden en el examen:
- `Include(l => l.Autor)` (acá con lambda, ya que se está trabajando directo con `_context.Libro`, no con el método genérico `ObtenerTodosCon(string)`) carga la relación para evitar el error de la Pregunta 2/5.
- El filtro combina **dos condiciones** en el mismo `Where` (`AutorId == autorId && l.Activo`), para no devolver libros dados de baja lógicamente.
- Es un método **específico de `Libro`**, así que corresponde que viva en `LibroRepository`, no en el repositorio genérico ni resuelto a mano en `Program.cs`, mismo criterio de diseño de toda esta guía.
- Falta agregar la línea en el menú de `Program.cs` (`case "16":` más una función local que pida el ID de autor y llame a este método) para que sea utilizable desde la app, parte de la implementación completa si el examen pide "intégralo al programa".
</details>

## C.5 (nivel superior) — `AsNoTracking()` y `Modificar()`, ¿son compatibles?

```csharp
var libro = libroRepository.ObtenerPorId(5);
libro.Titulo = "Nuevo título";
libroRepository.Modificar(libro);
```

**Consigna:** `ObtenerPorId` usa `Find`, no `AsNoTracking`, pero otros métodos del repositorio sí usan `AsNoTracking()`. Si una entidad se hubiera obtenido con un método que usa `AsNoTracking()`, y después se llamara a `Modificar(entidad)`, ¿el cambio se guardaría igual?

<details>
<summary>Ver solución</summary>

**Sí, se guardaría igual**, y es importante entender por qué. `AsNoTracking()` le dice a EF Core "traé estos datos, pero no los sigas vigilando para detectar cambios automáticamente" (mejora el rendimiento de las consultas de solo lectura, como los distintos `ObtenerTodos...` y reportes). Eso significa que si modificaras el objeto en memoria y llamaras directamente a `_context.SaveChanges()` sin nada más, EF Core **no se enteraría** del cambio (porque no lo estaba vigilando). Pero el método `Modificar` del repositorio genérico no depende de ese seguimiento automático:
```csharp
public void Modificar(T entidad)
{
    _context.Set<T>().Update(entidad);
    _context.SaveChanges();
}
```
`.Update(entidad)` **adjunta explícitamente** la entidad al contexto y la marca como `Modified` en ese mismo momento, sin importar si venía de una consulta con o sin tracking, es una operación independiente que no necesita que la entidad haya sido "vigilada" desde que se la trajo. Por eso el patrón `ObtenerAlgo() (con AsNoTracking) → modificar en memoria → Modificar(entidad)` es completamente válido y es, de hecho, el que usa todo el proyecto entregable.
</details>

---

## Resumen rápido (chuleta) de esta guía

- Clave foránea (`AutorId`) y propiedad de navegación (`Autor`) son **dos cosas distintas**; ambas hacen falta para poder navegar la relación en código.
- Sin `Include`, una propiedad de navegación queda en `null` → `NullReferenceException` al primer acceso.
- LINQ (`Where`/`OrderBy`/...) sobre `_context.Set<T>()` no ejecuta nada hasta `.ToList()` (ejecución diferida); todo se traduce a **una sola** consulta SQL.
- `private` en la clase base = invisible hasta para las clases hijas; para compartir con hijas, `protected`.
- El tipo **declarado** de una variable (interfaz, clase abstracta o clase base) limita qué miembros podés usar sin castear, sin importar el tipo real del objeto, regla idéntica en herencia, clases abstractas e interfaces.
- C# es sensible a mayúsculas/minúsculas: `Libro` y `libro` son identificadores distintos.
- Redeclarar un campo heredado en una clase hija con el mismo nombre lo **oculta** (`field hiding`), generando dos instancias separadas en vez de compartir una, el compilador avisa con `CS0108`.
- `AsNoTracking()` afecta solo el seguimiento automático de cambios; `Update()` dentro de `Modificar` adjunta y marca la entidad igual, sin depender de ese seguimiento.
