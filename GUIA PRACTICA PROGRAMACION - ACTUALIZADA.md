# GUÍA PRÁCTICA ACTUALIZADA (C#) — Proyecto `U9EjercicioEntregable`

> Esta versión está **corregida y completada** contra el código real del entregable
> (`U9EjercicioEntregable-main.zip`). Mantiene el mismo formato de la guía
> original (explicación línea por línea, pensada para principiantes) y agrega:
> el repositorio específico `LibroRepository`, las migraciones, el menú completo
> de `Program.cs`, y todas las correcciones de nombres reales del proyecto.

**Correcciones principales respecto de la guía original:**

| Tema | Guía original decía | Código real del entregable |
|------|---------------------|----------------------------|
| Tipo de `_context` | `private readonly` | **`protected readonly`** (para que `LibroRepository`, la clase hija, pueda usarlo) |
| Nombre del DbContext | `AplicationDbContext` (con error de tipeo) | **`ApplicationDbContext`** |
| Nombres de los `DbSet` | `Autores`, `Libros`, `Categorias` (plural) | **`Autor`, `Libro`, `Categoria` (singular)** |
| Ruta de la base SQLite | `C:\Users\catag\source\repos\...` | **`C:\databases\BaseDatosEjercicios.db`** |
| `EnsureCreated()`/`Migrate()` | se creaba la base desde el código | el proyecto usa **migraciones** (carpeta `Migrations`) |
| Repositorio específico | no existía | **`LibroRepository`** con 6 métodos LINQ |
| Menú | parcial (opciones 1–4…) | **menú completo (opciones 1–15 + 0)**, 15 funciones |

---

## 0. La solución: `AppConsola.slnx`

La **solución** no contiene código: es un archivo simple que agrupa referencias a
uno o más **proyectos**, para que Visual Studio los abra y los compile juntos.

```xml
<Solution>
  <Project Path="AccesoDatos/AccesoDatos.csproj" />
  <Project Path="AppConsola/AppConsola.csproj" />
</Solution>
```

**Línea: `<Project Path="AccesoDatos/AccesoDatos.csproj" />`**
Referencia al proyecto tipo **Biblioteca de Clases**. Es donde viven los Modelos,
el DbContext y los Repositorios. Se usa este tipo de proyecto cuando se quiere
separar la lógica de acceso a datos del programa que la consume (patrón Repositorio).

**Línea: `<Project Path="AppConsola/AppConsola.csproj" />`**
Referencia al proyecto tipo **Aplicación de Consola**: es el programa ejecutable,
el punto de entrada que corre el usuario. Contiene `Program.cs`.

**Estructura general de carpetas dentro de `AccesoDatos`:**

```
AccesoDatos/
├─ Models/           → Autor.cs, Categoria.cs, Libro.cs
├─ Data/             → ApplicationDbContext.cs
├─ Repositories/     → IGenericRepository<T>, GenericRepository<T>, LibroRepository
└─ Migrations/       → InitialMigration (creación de las tablas)
```

Esta organización por carpetas no es obligatoria para compilar, pero refleja en
archivos la separación de responsabilidades del patrón Repositorio.

---

## 1. Models (carpeta `Models`)

Agrupan datos (propiedades) que describen una entidad del sistema.

### 1.1 `Autor.cs`

```csharp
namespace AccesoDatos.Models
{
    public class Autor
    {
        public int Id { get; set; }
        public string Nombre { get; set; }
        public List<Libro> Libros { get; set; } = new();
    }
}
```

**Línea: `namespace AccesoDatos.Models`**
Es como la "carpeta lógica" donde vive esta clase. Sirve para que el compilador no
confunda esta clase con otra que se llame igual en otra parte del proyecto.

**Línea: `public class Autor`**
Es el "molde" de lo que es un Autor. `public` significa que se puede usar desde
cualquier proyecto (por ejemplo, desde `AppConsola`).

**Línea: `public int Id { get; set; }`**
Guarda el número identificador de cada Autor. Entity Framework lo reconoce como
**clave primaria** porque se llama `Id`. Tiene valores incrementales (1, 2, 3…).

**Línea: `public string Nombre { get; set; }`**
Propiedad auto-implementada de tipo `string` (texto). Se usa para cualquier dato
de texto de la entidad.

**Línea: `public List<Libro> Libros { get; set; } = new();`**
Es el lado "muchos" de la relación **uno-a-muchos**: EF Core va a poner, de forma
automática, todos los `Libro` que pertenezcan a este autor. El `= new();` la
inicializa vacía desde el arranque, para que **nunca sea `null`**.

> **¿Qué pasaría si no estuviera cada cosa?** Si `Id` no se llamara así, EF no
> sabría cuál es su clave primaria y tiraría error al migrar. Si `Libros` no
> tuviera `= new();`, empezaría en `null`, y si en algún lado la recorrieras con
> `foreach`, el programa tiraría `NullReferenceException`.

### 1.2 `Categoria.cs`

Sigue el mismo patrón que `Autor.cs`: `Id`, `Nombre` y una lista de `Libro`.

```csharp
namespace AccesoDatos.Models
{
    public class Categoria
    {
        public int Id { get; set; }
        public string Nombre { get; set; }
        public List<Libro> Libros { get; set; } = new();
    }
}
```

### 1.3 `Libro.cs` — el modelo con las relaciones

```csharp
namespace AccesoDatos.Models
{
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
}
```

**Líneas: `public string Titulo { get; set; }` y `public int AnioPublicacion { get; set; }`**
Datos básicos del libro: título (texto) y año (entero).

**Línea: `public int AutorId { get; set; }`**
Guarda el **número** del autor dueño de este libro. Es la **clave foránea**
(FK = Foreign Key): el dato que realmente se guarda como columna en la tabla
`Libro`. Responde a: "¿qué autor escribió este libro?".

**Línea: `public Autor Autor { get; set; }`**
Es la **propiedad de navegación**: un atajo que permite escribir
`libro.Autor.Nombre` para llegar al autor completo **sin tener que buscarlo a mano**.

**Líneas: `public int CategoriaId { get; set; }` y `public Categoria Categoria { get; set; }`**
Igual que con el autor: el número de la categoría (clave foránea) + el objeto
categoría (propiedad de navegación).

**Línea: `public bool Activo { get; set; } = true;`**
Indica si el libro está "dado de alta". **Arranca en `true`** gracias al `= true;`.
Cuando se "elimina" un libro en la app, no se borra la fila: este campo pasa a
`false` (**borrado lógico**). Es distinto de `Autor`/`Categoria`: no es una
relación, es un dato propio del libro.

> **Regla de oro del modelo:** clave foránea (`AutorId`, un número) y propiedad de
> navegación (`Autor`, un objeto) son **dos cosas distintas y hacen falta las dos**:
> una para que la base de datos relacione las tablas, otra para que el código C#
> pueda hacer `libro.Autor.Nombre`.

---

## 2. El DbContext (carpeta `Data`)

```csharp
using Microsoft.EntityFrameworkCore;
using AccesoDatos.Models;

namespace AccesoDatos.Data
{
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
}
```

**Línea: `using Microsoft.EntityFrameworkCore;`**
Importa las clases de Entity Framework Core que se usan acá: `DbContext`,
`DbSet<T>` y `DbContextOptionsBuilder`. Se usa siempre que se trabaje con EF Core.

**Línea: `using AccesoDatos.Models;`**
Importa las clases del modelo (`Autor`, `Libro`, `Categoria`) para poder usarlas
como tipo dentro de los `DbSet<T>`.

**Línea: `public class ApplicationDbContext : DbContext`**
Declara la clase que representa la **conexión con la base de datos**, heredando de
`DbContext`. Al heredar, esta clase obtiene automáticamente toda la funcionalidad
de EF Core (seguimiento de cambios, generación de SQL, migraciones…). Se usa **una
sola** clase de este tipo por base de datos.

> ⚠️ Nota sobre el nombre: en el código real está bien escrito
> `ApplicationDbContext`. Si lo escribís con el error de tipeo
> `AplicationDbContext`, el proyecto **no compila** (ninguna clase con ese nombre
> existiría). Verificá que coincida el nombre en los tres lugares: la clase, el
> `GenericRepository` y el `Program.cs`.

**Líneas: `public DbSet<Autor> Autor { get; set; }` (y `Libro`, `Categoria`)**
Cada `DbSet<T>` representa **una tabla** dentro de la base de datos. EF Core, al
ver esta propiedad, sabe que debe crear una tabla `Autor` con columnas
equivalentes a las propiedades de la clase `Autor`.

**Detalle de nomenclatura:** los `DbSet` están nombrados en **singular**
(`Autor`, `Libro`, `Categoria`), igual que la clase que representan. Funciona
igual que el plural; EF Core no lo exige. Pero es importante usar **el nombre
exacto de la propiedad** después: `_context.Libro` (no `_context.Libros`). C# es
**sensible a mayúsculas**: `Libro` y `libro` son identificadores distintos.

**Línea: `protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)`**
Método que se ejecuta para configurar la conexión. `override` significa "reemplazo
el comportamiento que traía de fábrica" y `protected` permite que lo use solo la
clase (y sus hijas, como el repositorio que trabaja con él).

**Línea: `optionsBuilder.UseSqlite("Data Source=C:\\databases\\BaseDatosEjercicios.db");`**
Elige el motor SQLite y la ruta del **archivo** de la base de datos. Ojo con el
`\\`: en un `string` de C#, `\\` representa **una** barra literal `\`. La ruta
completa es `C:\databases\BaseDatosEjercicios.db`.

---

## 3. Repositorio Genérico (carpeta `Repositories`)

### 3.1 La interfaz: `IGenericRepository.cs`

```csharp
using AccesoDatos.Models;
using System;
using System.Collections;
using System.Collections.Generic;
using System.Text;

namespace AccesoDatos.Repositories
{
    public interface IGenericRepository<T> where T : class
    {
        void Agregar(T entidad);
        List<T> ObtenerTodos();
        List<T> ObtenerTodosCon(string propiedadRelacionada);
        T ObtenerPorId(int id);
        void Modificar(T entidad);
        void Eliminar(object id);
    }
}
```

**Línea: `public interface IGenericRepository<T> where T : class`**
Declara una **interfaz**: un contrato que solo define **QUÉ** métodos debe tener
cualquier clase que la implemente, sin decir **CÓMO**. El `<T>` es un parámetro de
**tipo genérico**: esta interfaz no está atada a una sola entidad, sirve para
cualquier. `where T : class` es una restricción: obliga a que `T` sea siempre una
**clase** (un tipo por referencia), nunca un tipo primitivo como `int` o `bool`.

**Los 6 métodos:** son las operaciones base que debe cumplir cualquier repositorio:

| Método | ¿Qué hace? |
|--------|------------|
| `void Agregar(T entidad)` | dar de alta un registro |
| `List<T> ObtenerTodos()` | traer todos los registros |
| `List<T> ObtenerTodosCon(string propiedadRelacionada)` | traer todos + cargar una relación (`Include`) |
| `T ObtenerPorId(int id)` | buscar uno por su Id |
| `void Modificar(T entidad)` | actualizar un registro |
| `void Eliminar(object id)` | borrar un registro |

### 3.2 La implementación: `GenericRepository.cs`

```csharp
using AccesoDatos.Data;
using AccesoDatos.Models;
using Microsoft.EntityFrameworkCore;

namespace AccesoDatos.Repositories
{
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

        public List<T> ObtenerTodos()
        {
            return _context.Set<T>()
                           .AsNoTracking()
                           .ToList();
        }

        public List<T> ObtenerTodosCon(string propiedadRelacionada)
        {
            return _context.Set<T>()
                           .Include(propiedadRelacionada)
                           .AsNoTracking()
                           .ToList();
        }

        public T ObtenerPorId(int id)
        {
            return _context.Set<T>().Find(id);
        }

        public void Modificar(T entidad)
        {
            _context.Set<T>().Update(entidad);
            _context.SaveChanges();
        }

        public void Eliminar(object id)
        {
            var entidad = _context.Set<T>().Find(id);

            if (entidad != null)
            {
                _context.Set<T>().Remove(entidad);
                _context.SaveChanges();
            }
        }
    }
}
```

**Línea: `protected readonly ApplicationDbContext _context;`**
Guarda la conexión a la base de datos:
- **`protected`**: visible en esta clase **y en sus hijas** (por eso
  `LibroRepository` puede usarlo). No es `private` (que ni las hijas lo verían) ni
  `public` (que cualquiera podría tocar la conexión).
- **`readonly`**: una vez asignado (en el constructor), su valor no puede volver a
  cambiar.

**Línea: `public class GenericRepository<T> : IGenericRepository<T> where T : class`**
Declara la clase concreta que implementa la interfaz. El `: IGenericRepository<T>`
le dice al compilador: "esta clase promete cumplir todo lo que la interfaz exige".
La idea es **reutilizar exactamente el mismo código** de acceso a datos con
cualquier entidad futura (Autor, Categoria, Libro…) sin escribir un repositorio
por cada una.

**Línea: `public GenericRepository() { _context = new ApplicationDbContext(); }`**
Es el **constructor**: el bloque que se ejecuta automáticamente cada vez que se
hace `new GenericRepository<T>()`. Deja el objeto listo antes de usarse: crea la
conexión y la guarda en `_context`.

**Línea: `_context.Set<T>().Add(entidad); _context.SaveChanges();`**
- `Set<T>()`: le pide al contexto la **tabla** que corresponde al tipo `T`
  (si `T` es `Autor`, devuelve el `DbSet<Autor>` definido en el DbContext).
- `Add(entidad)`: solo **anota** que hay algo nuevo para guardar (no toca la base).
- `SaveChanges()`: es el que **escribe** en el archivo `.db`. Sin esta línea,
  el dato nunca llegaría a la base.

**Línea: `_context.Set<T>().AsNoTracking().ToList();`**
Trae todos los registros de esa tabla. `AsNoTracking()` le dice a EF "no hace falta
que vigiles estos objetos", lo que hace la consulta más rápida cuando solo se lee.

**Línea: `_context.Set<T>().Include(propiedadRelacionada).AsNoTracking().ToList();`**
Igual que `ObtenerTodos()`, pero además le pide que traiga **pegada** una relación
(por ejemplo, el `Autor` de cada `Libro`). **Sin el `Include`, `libro.Autor`
vendría en `null`.**

**Línea: `_context.Set<T>().Find(id);`**
Busca un solo registro por su **clave primaria**. Si no lo encuentra devuelve
`null`. Por eso **siempre** se chequea `if (entidad != null)` después de llamarlo.

**Línea: `_context.Set<T>().Update(entidad); _context.SaveChanges();`**
Le dice a EF "este objeto cambió, actualizá la base" y `SaveChanges()` hace el
`UPDATE` real. **Importantísimo:** `Update()` adjunta y marca la entidad aunque
venga de una consulta con `AsNoTracking()`; no depende del seguimiento automático.

**Línea: `var entidad = ...Find(id); if (entidad != null) { ...Remove(entidad); ...SaveChanges(); }`**
Busca el registro y, si existe, lo **borra de verdad** (físicamente) de la base.
> **CUIDADO:** este método existe en el genérico, pero la app **nunca lo usa para
> libros**: los libros se "eliminan" de forma **lógica** (`Activo = false`) para
> no perder el historial.

### 3.3 El repositorio específico: `LibroRepository.cs` ⭐ (nuevo)

```csharp
using AccesoDatos.Models;
using System.Collections.Generic;
using System.Linq;

namespace AccesoDatos.Repositories
{
    public class LibroRepository : GenericRepository<Libro>
    {
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
    }
}
```

**Línea: `public class LibroRepository : GenericRepository<Libro>`**
Es un repositorio que **hereda** del genérico: tiene todos los métodos de
`IGenericRepository<T>` (Agregar, ObtenerTodos, Modificar…) **y** agrega consultas
**específicas de Libro**. Por eso puede usar `_context` sin declararlo otra vez:
lo heredó (por eso es `protected`, no `private`). Criterio de diseño: una consulta
que solo necesita `Libro` vive acá, no en el genérico ni en `Program.cs`.

Ahora, los 6 métodos, que son consultas **LINQ** (LINQ = lenguaje de consultas
integrado; los métodos se traducen a SQL y se ejecutan en la base de datos):

**`OrderByDescending(l => l.AnioPublicacion)` — libros más recientes**
Ordena los libros por año, de **mayor a menor** (los más nuevos primero).
`l => l.AnioPublicacion` es una *lambda*: "para cada libro `l`, mirá su año".

**`Count()` — cantidad total**
Cuenta **todos** los libros y devuelve un `int`.

**`Count(l => l.Activo)` — cantidad de activos**
Cuenta **solo** los libros que cumplen la condición (que `Activo == true`).
La lambda es el filtro del conteo.

**`FirstOrDefault(l => l.Id == id)` — buscar por ID**
Devuelve el **primer** libro cuyo `Id` sea el pedido, o `null` si no existe. Por
eso el tipo de retorno es `Libro?` (el `?` marca que puede devolver `null`).

**`OrderBy(l => l.Titulo)` — ordenar por título**
Ordena de menor a mayor (A→Z) por el título. `OrderByDescending` hubiera sido Z→A.

**`Any(l => l.Activo)` — ¿existen activos?**
Devuelve `true` si hay **al menos un** libro activo; `false` si no hay ninguno.
Devuelve `bool`, no una lista: es una pregunta "sí/no".

| Método | Respuesta que da |
|--------|------------------|
| `ObtenerLibrosPorMasRecientes()` | lista ordenada por año desc. |
| `ObtenerCantidadLibros()` | número total |
| `ObtenerCantidadLibrosActivos()` | número de activos |
| `ObtenerLibroPorId(id)` | un `Libro` o `null` |
| `ObtenerLibrosOrdenadosPorTitulo()` | lista ordenada A→Z |
| `ExistenLibrosActivos()` | `true`/`false` |

---

## 4. Migraciones (carpeta `Migrations`) ⭐ (nuevo)

Las **migraciones** son el mecanismo de EF Core para ir dejando la base de datos al
día con los modelos: cada migración es un "historial" de cambios que transforma el
esquema (crear tablas, columnas, llaves…). Se genera con comandos como
`dotnet ef migrations add InitialMigration` y se aplica con
`dotnet ef database update`.

La migración `20260916173340_InitialMigration.cs` (código autogenerado por EF)
contiene dos métodos:

- `Up(MigrationBuilder migrationBuilder)`: **aplica** los cambios. Acá crea las
  tablas `Autor`, `Categoria` y `Libro`.
- `Down(MigrationBuilder migrationBuilder)`: **revierte** los cambios (las borra).

Lo importante que se ve en `Up`:

```csharp
table.PrimaryKey("PK_Autor", x => x.Id);                 // clave primaria
table.Column<int>(type: "INTEGER", nullable: false)
    .Annotation("Sqlite:Autoincrement", true);           // Id autoincremental

table.ForeignKey(
    name: "FK_Libro_Autor_AutorId",
    column: x => x.AutorId,
    principalTable: "Autor",
    principalColumn: "Id",
    onDelete: ReferentialAction.Cascade);                // relación 1-a-muchos
```

Esto corresponde **exactamente** a lo que los modelos declaraban:
- `AutorId` y `CategoriaId` son columnas y además **llaves foráneas** (apuntan a
  la tabla `Autor`/`Categoria`).
- `onDelete: Cascade`: si se borra el autor, se borran sus libros. **Pregunta de
  examen frecuente.**

> Relación con la guía original: la versión vieja explicaba `EnsureCreated()`
> (crear la base al vuelo). El entregable usa **migraciones**, que son la forma
> ordenada de mantener la base cuando el modelo crece. Son dos caminos válidos;
> el proyecto real usa el de migraciones.

---

## 5. `Program.cs` (el menú completo)

### 5.1 Los `using` y los repositorios

```csharp
using AccesoDatos.Models;
using AccesoDatos.Repositories;

IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();
IGenericRepository<Categoria> categoriaRepository = new GenericRepository<Categoria>();
LibroRepository libroRepository = new LibroRepository();
```

**Línea: `using AccesoDatos.Models;` y `using AccesoDatos.Repositories;`**
Le dicen a C# "voy a usar cosas de estas bibliotecas/namespaces". Así se puede
escribir `Autor` en vez del nombre completo `AccesoDatos.Models.Autor`.

**Líneas de los repositorios — fijate que hay DOS tipos de variable, a propósito:**

```csharp
IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();
IGenericRepository<Categoria> categoriaRepository = new GenericRepository<Categoria>();
LibroRepository libroRepository = new LibroRepository();
```

- `autorRepository` y `categoriaRepository` están tipados con la **interfaz
  genérica**: para Autor y Categoría **alcanza** con las operaciones básicas
  (`Agregar`, `ObtenerTodos`, …).
- `libroRepository` está tipado como **`LibroRepository`** (la clase concreta),
  porque el programa necesita llamar a métodos que **NO están en la interfaz**:
  `ObtenerLibrosMasRecientes`, `ObtenerCantidadLibros`, etc.
- **Regla**: *el tipo de la variable decide qué métodos podés llamar*. Si
  `libroRepository` estuviera declarado como `IGenericRepository<Libro>`, ninguna
  de las llamadas a los métodos LINQ compilaría (error `CS1061`).

### 5.2 El bucle del menú: `while` + `switch`

```csharp
bool continuar = true;

while (continuar)
{
    Console.WriteLine("1. Alta Autor");
    // ... opciones 2 a 15 ...
    Console.WriteLine("0. Salir");

    Console.Write("Seleccione una opción: ");
    string opcion = Console.ReadLine();

    Console.Clear();

    switch (opcion)
    {
        case "1": AltaAutor(); break;
        // ...cases 2 a 15...
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

**Línea: `bool continuar = true;`**
Cajita que controla si el menú sigue mostrándose. Mientras sea `true`, se repite un
ciclo: mostrar menú, leer opción, ejecutar. Apenas el usuario elige `0`, pasa a
`false` y el bucle termina.

**Línea: `switch (opcion)`**
Sirve para decidir qué hacer según el valor leído. Si eligió `"1"` (con comillas,
porque `ReadLine` siempre devuelve texto), llama a `AltaAutor()`; `"2"` →
`AltaCategoria()`; etc. `case "0"` apaga el bucle; `default` avisa opción inválida.

**Las opciones del menú real:**

| Opción | Función | Qué hace |
|--------|---------|----------|
| 1 | `AltaAutor()` | registra un autor |
| 2 | `AltaCategoria()` | registra una categoría |
| 3 | `AltaLibro()` | registra un libro (pide autor + categoría) |
| 4 | `MostrarAutores()` | lista autores |
| 5 | `MostrarCategorias()` | lista categorías |
| 6 | `MostrarLibros()` | lista libros **activos** |
| 7 | `ModificarLibro()` | cambia el título de un libro |
| 8 | `EliminarLibro()` | eliminación lógica (`Activo = false`) |
| 9 | `ModificarAutor()` | cambia el nombre de un autor |
| 10 | `MostrarLibrosMasRecientes()` | libros por año desc. |
| 11 | `MostrarCantidadLibros()` | total de libros |
| 12 | `MostrarCantidadLibrosActivos()` | cantidad de activos |
| 13 | `BuscarLibroPorId()` | busca un libro por ID |
| 14 | `MostrarLibrosOrdenadosPorTitulo()` | libros por título A→Z |
| 15 | `VerificarLibrosActivos()` | ¿existen activos? sí/no |
| 0 | — | salir |

### 5.3 Una función típica de alta: `AltaAutor()`

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

**Línea: `void AltaAutor()`**
Define una función llamada `AltaAutor`. `void` = no devuelve ningún resultado.
Las funciones locales se definen después del código principal y se llaman cuando
hace falta.

**Líneas: `Autor autor = new Autor { Nombre = Console.ReadLine() };`**
- `Console.ReadLine()` lee el texto que escribió el usuario.
- `new Autor { Nombre = ... }` fabrica un objeto `Autor` y, en el mismo momento,
  le asigna el nombre (el `{ ... }` es un **inicializador de objetos**).
- `Autor autor = ...` guarda en la cajita `autor` ese objeto.

**Línea: `autorRepository.Agregar(autor);`**
Envía el autor al repositorio para guardarlo en la base de datos (es el
`Add` + `SaveChanges` del genérico).

**Línea: `PresioneParaContinuar();`**
Pausa el programa hasta que el usuario toque una tecla y limpia la pantalla. Se
repite en **casi todas** las funciones del menú.

> El mismo patrón se repite con `AltaCategoria()` y `AltaLibro()`. La diferencia
> de `AltaLibro()` es que, además del título y el año, pide elegir **autor y
> categoría**: lista las opciones (con su `Id`) y guarda los IDs elegidos en
> `AutorId` y `CategoriaId`, dejando `Activo = true` al crear el `Libro`.

### 5.4 Listar con `foreach` y mostrar con `$"..."`: `MostrarAutores()`

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

- `var autores = ...`: `ObtenerTodos()` devuelve una lista de autores; `var` es una
  cajita "que se entera sola" del tipo.
- `foreach (var autor in autores)`: "por cada autor de la lista, hacé esto".
- `$"ID: {autor.Id} | Nombre: {autor.Nombre}"`: texto con **interpolación**. Las
  llaves `{ }` son huecos que se rellenan con el valor de cada autor.

### 5.5 Listar libros solo activos — `MostrarLibros()` (el más didáctico)

```csharp
void MostrarLibros()
{
    Console.WriteLine("===== LISTADO DE LIBROS =====");

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

    Console.WriteLine("=============================");

    PresioneParaContinuar();
}
```

- `ObtenerTodosCon("Autor")`: igual que `ObtenerTodos()`, pero **incluye** la
  relación con `Autor` (el `Include` del genérico). **Sin esto, `libro.Autor`
  sería `null`** y `libro.Autor.Nombre` rompería el programa.
- `!libros.Any()`: `Any()` pregunta "¿hay al menos un libro?"; el `!` lo niega.
  Entonces `!libros.Any()` es "¿no hay ningún libro?" → si es `true`, avisa.
- `libros.Where(l => l.Activo)`: filtro por los libros **activos**, para no
  mostrar los "borrados".

### 5.6 Eliminación lógica — `EliminarLibro()`

```csharp
void EliminarLibro()
{
    MostrarLibros();

    Console.Write("Ingrese el ID del libro: ");
    string idLibroABorrar = Console.ReadLine();

    if (idLibroABorrar == null || idLibroABorrar == "")
    {
        Console.WriteLine("No fue ingresado ningun ID de libro");
        PresioneParaContinuar();
    }
    else
    {
        int id = int.Parse(idLibroABorrar);

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
    }

    PresioneParaContinuar();
}
```

- **No se borra la fila**: se marca `Activo = false` y se guarda con `Modificar`
  (un `UPDATE`). Por eso se llama *eliminación lógica*: el libro deja de aparecer
  en los listados que filtran `l.Activo`, pero el registro sigue en la tabla.
- `int.Parse(...)`: convierte el texto que escribió el usuario a número.
  (Ojo: si escribiera letras, `int.Parse` rompería el programa.)
- `libroRepository.ObtenerPorId(id)` usa `Find` y puede devolver `null`; por eso
  se verifica `if (libro != null)` antes de tocarlo.

### 5.7 Las funciones de consulta (opciones 10–15)

Leen datos a través de los métodos LINQ de `LibroRepository` (las explicaciones
detalladas están en la sección 3.3):

```csharp
void MostrarLibrosMasRecientes()
{
    Console.WriteLine("===== LIBROS MÁS RECIENTES =====");
    foreach (var libro in libroRepository.ObtenerLibrosPorMasRecientes())
    {
        Console.WriteLine($"{libro.Titulo} - {libro.AnioPublicacion}");
    }
    PresioneParaContinuar();
}

void MostrarCantidadLibros()
{
    Console.WriteLine("===== CANTIDAD TOTAL DE LIBROS =====");
    Console.WriteLine($"Cantidad: {libroRepository.ObtenerCantidadLibros()}");
    PresioneParaContinuar();
}

void MostrarCantidadLibrosActivos()
{
    Console.WriteLine("===== CANTIDAD DE LIBROS ACTIVOS =====");
    Console.WriteLine($"Cantidad: {libroRepository.ObtenerCantidadLibrosActivos()}");
    PresioneParaContinuar();
}

void BuscarLibroPorId()
{
    Console.Write("Ingrese ID del libro: ");
    int id = int.Parse(Console.ReadLine());

    var libro = libroRepository.ObtenerLibroPorId(id);

    if (libro == null)
    {
        Console.WriteLine("Libro no encontrado.");
    }
    else
    {
        Console.WriteLine($"Título: {libro.Titulo} | Año: {libro.AnioPublicacion}");
    }
    PresioneParaContinuar();
}

void MostrarLibrosOrdenadosPorTitulo()
{
    Console.WriteLine("===== LIBROS ORDENADOS POR TÍTULO =====");
    foreach (var libro in libroRepository.ObtenerLibrosOrdenadosPorTitulo())
    {
        Console.WriteLine($"{libro.Titulo} - {libro.AnioPublicacion}");
    }
    PresioneParaContinuar();
}

void VerificarLibrosActivos()
{
    Console.WriteLine("===== VERIFICAR LIBROS ACTIVOS =====");

    if (libroRepository.ExistenLibrosActivos())
    {
        Console.WriteLine("Existen libros activos.");
    }
    else
    {
        Console.WriteLine("No existen libros activos.");
    }
    PresioneParaContinuar();
}
```

Casos didácticos para pensar:

| Método llamado | Tipo que devuelve | ¿Cómo se usa el resultado? |
|----------------|-------------------|----------------------------|
| `ObtenerLibrosPorMasRecientes()` | `List<Libro>` | se recorre con `foreach` |
| `ObtenerCantidadLibros()` | `int` | se interpola en el mensaje |
| `ObtenerCantidadLibrosActivos()` | `int` | se interpola en el mensaje |
| `ObtenerLibroPorId(id)` | `Libro?` (o `null`) | se verifica `if (libro == null)` |
| `ObtenerLibrosOrdenadosPorTitulo()` | `List<Libro>` | se recorre con `foreach` |
| `ExistenLibrosActivos()` | `bool` | se mete directo en un `if` |

La ayuda de `PresioneParaContinuar()`:

```csharp
void PresioneParaContinuar()
{
    Console.WriteLine();
    Console.WriteLine("Presione una tecla para continuar...");
    Console.ReadKey();
    Console.Clear();
}
```

---

## 6. Repaso: el flujo completo de una consulta

Cuando el usuario elige la opción 10 (libros más recientes):

1. El `switch` del `while` llama a `MostrarLibrosMasRecientes()`.
2. Esa función llama a `libroRepository.ObtenerLibrosPorMasRecientes()`.
3. Como `libroRepository` es `LibroRepository`, el método usa `_context` (heredado
   y `protected`) con `OrderByDescending(...)`.
4. EF Core traduce eso a SQL (`SELECT * FROM Libro ORDER BY AnioPublicacion DESC`),
   lo ejecuta **una sola vez** y entrega los datos ya ordenados.
5. La función recorre la lista y la muestra; `PresioneParaContinuar()` pausa.

```
switch → función del menú → LibroRepository (LINQ) → _context (SQLite) → pantalla
```

---

## Chuleta resumen de la guía

- **Solución**: agrupa proyectos; no tiene código.
- **Models**: clases con `Id` (clave primaria), datos y **relaciones**
  (clave foránea + propiedad de navegación + lista del lado "muchos").
- **DbContext**: una clase por base; `DbSet<T>` por entidad; `OnConfiguring`
  define la conexión SQLite.
- **Repositorio genérico**: 6 operaciones base reutilizables para cualquier `T`.
- **`protected readonly`**: la conexión se comparte con las clases hijas pero no
  se reasigna ni se toca desde afuera.
- **`LibroRepository`**: hereda el genérico y suma las consultas **específicas** de
  libro (LINQ), cada consulta en el lugar que corresponde.
- **`Include`**: sin él, las relaciones vienen en `null` → `NullReferenceException`.
- **Migrations**: historial que crea/actualiza las tablas; `Up` aplica, `Down`
  revierte; las FK con `Cascade` borran en cadena.
- **Program.cs**: `while` + `switch` para el menú; variables tipadas con la
  interfaz cuando alcanzan las operaciones base, y con `LibroRepository` cuando se
  necesitan los métodos LINQ (el tipo de la variable decide qué se puede llamar).
- **Borrado lógico**: `Activo = false` con `Modificar`, nunca `Eliminar`.