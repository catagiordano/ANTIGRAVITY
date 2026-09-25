# Leido
# GUÍA PRÁCTICA ACTUALIZADA (C#) — Proyecto `U9EjercicioEntregable`

> Esta versión está **corregida y completada** contra el código real del entregable
> (`U9EjercicioEntregable-main.zip`). Mantiene el mismo formato de la guía
> original (explicación línea por línea, pensada para principiantes) y ahora cubre
> **el 100% del código del proyecto**, de punta a punta.
>
> **Qué hay en cada archivo, y dónde se explica en esta guía:**
>
> | Archivo del proyecto | Líneas | Sección de la guía |
> |---------------------|--------|--------------------|
> | `AccesoDatos.csproj` | 17 | 0.3 |
> | `AppConsola.csproj` | 26 | 0.3 |
> | `AppConsola.slnx` | 4 | 0.1 |
> | `Models/Autor.cs` | 13 | 1.1 |
> | `Models/Categoria.cs` | 13 | 1.2 |
> | `Models/Libro.cs` | 18 | 1.3 |
> | `Data/ApplicationDbContext.cs` | 17 | 2 |
> | `Repositories/IGenericRepository.cs` | 20 | 3.1 |
> | `Repositories/GenericRepository.cs` | 60 | 3.2 |
> | `Repositories/LibroRepository.cs` | 53 | 3.3 |
> | `Migrations/*InitialMigration.cs` | 92 | 4 |
> | `Program.cs` | 431 | 5.1 – 5.11 |
>
> **Además de explicar el código**, la guía incluye los comandos de consola que
> hacen funcionar el proyecto (sección 7), el esquema de la base de datos (8), los
> errores más frecuentes y cómo se solvean (9), una autoevaluación de 14
> preguntas de examen con respuesta (10) y un glosario (11).

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

## 0. La solución, los proyectos y su configuración

### 0.1 La solución: `AppConsola.slnx`

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

> `.slnx` es el formato **nuevo** de solución que trae .NET 10. En las versiones
> anteriores el archivo se llamaba `AppConsola.sln` y tenía muchísimas líneas de
> GUIDs; el `.slnx` es legible y ocupa 4 líneas. Las dos cosas son la misma idea.

### 0.2 Estructura real de carpetas del proyecto

```
AppConsola/                              ← carpeta de la solución
├─ AppConsola.slnx                       ← la solución (agrupa, no programa)
│
├─ AccesoDatos/                          ← proyecto: Biblioteca de Clases
│  ├─ AccesoDatos.csproj                 ← configuración de este proyecto
│  ├─ Models/
│  │   ├─ Autor.cs
│  │   ├─ Categoria.cs
│  │   └─ Libro.cs
│  ├─ Data/
│  │   └─ ApplicationDbContext.cs
│  ├─ Repositories/
│  │   ├─ IGenericRepository.cs
│  │   ├─ GenericRepository.cs
│  │   └─ LibroRepository.cs
│  └─ Migrations/
│      ├─ 20260916173340_InitialMigration.cs           ← los cambios (Up / Down)
│      ├─ 20260916173340_InitialMigration.Designer.cs  ← metadatos
│      └─ ApplicationDbContextModelSnapshot.cs         ← "foto" del modelo
│
└─ AppConsola/                           ← proyecto: Aplicación de Consola
   ├─ AppConsola.csproj
   └─ Program.cs                         ← 431 líneas: menú + 15 funciones
```

Esta organización por carpetas no es obligatoria para compilar, pero refleja en
los archivos la separación de responsabilidades del patrón Repositorio. Fijate que
**el nombre de la carpeta de la solución y el del proyecto de consola son iguales**
(`AppConsola`): es una coincidencia de nombres, no un error.

### 0.3 Los archivos `.csproj`: la configuración de cada proyecto ⭐ (nuevo)

El `.csproj` es el "documento de identidad" del proyecto: dice **qué versión de .NET
usa**, **qué paquetes tiene instalados** y **a qué otros proyectos referencia**.
No tiene lógica de negocio, pero sin él el proyecto no compila.

**`AccesoDatos/AccesoDatos.csproj` (completo):**

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.11" />
  </ItemGroup>

  <ItemGroup>
    <Folder Include="Migrations\" />
  </ItemGroup>

</Project>
```

**Línea: `<Project Sdk="Microsoft.NET.Sdk">`**
Declara que este proyecto usa el SDK de .NET moderno (estilo "declarativo", sin
listas de archivos compilados).

**Línea: `<TargetFramework>net10.0</TargetFramework>`**
La **versión de .NET** con la que se compila. Todo el proyecto (solución, proyectos,
migraciones) tiene que usar la misma, si no aparece el error `NETSDK1045`.

**Línea: `<ImplicitUsings>enable</ImplicitUsings>`** ⭐ (explica algo del código)
Activa los `using` **automáticos**. Por eso `Program.cs` usa `Console` y
`List<T>` **sin escribir `using System;` ni `using System.Collections.Generic;`**.
Si este fuera `disable`, el programa no compilaría. (Por eso en los archivos de
`Repositories` los `using` **sí** están escritos: se agregaron a mano, son
redundantes pero inofensivos.)

**Línea: `<Nullable>enable</Nullable>`** ⭐ (explica el `?` del código)
Activa el análisis de nulos. Gracias a esto el compilador **obliga** a marcar los
tipos que pueden ser `null`, y por eso existe `Libro? ObtenerLibroPorId(int id)`:
el `?` **no es decorativo**: es una promesa de que puede no encontrar el registro.

**Línea: `<PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" ... />`**
El **paquete NuGet** que trae Entity Framework Core con el motor SQLite. Sin esta
línea, `using Microsoft.EntityFrameworkCore;` daría error `CS0246` (no existe el
nombre) y `UseSqlite` no existiría. Es el mismo número de versión en los dos
proyectos: EF Core no mezcla versiones.

**Línea: `<Folder Include="Migrations\" />`**
Marcador para que Visual Studio muestre la carpeta aunque esté vacía. No afecta la
compilación.

**`AppConsola/AppConsola.csproj` (completo):**

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.11">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.11" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="10.0.11">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\AccesoDatos\AccesoDatos.csproj" />
  </ItemGroup>

</Project>
```

**Línea: `<OutputType>Exe</OutputType>`**
Dice que este proyecto produce un **programa ejecutable**. `AccesoDatos.csproj`
**no** tiene esta línea: por eso es una biblioteca y no se puede "correr"; solo
guarda clases que otro proyecto usa.

**Línea: `Microsoft.EntityFrameworkCore.Design`** ⭐ (la clave de las migraciones)
Es el paquete que permite **generar migraciones** desde la consola
(`dotnet ef migrations add`). Vive en el proyecto de consola, no en la biblioteca,
porque las migraciones son una herramienta de **desarrollo**, no de ejecución.

**Línea: `Microsoft.EntityFrameworkCore.Tools`**
Agrega el menú *"Entity Framework Core > PMC"* (Package Manager Console) dentro de
Visual Studio, que es la otra forma de crear migraciones sin salir del IDE.

**Línea: `<PrivateAssets>all</PrivateAssets>`**
Le dice a NuGet: "este paquete es solo mío, no lo compartas con los proyectos que
me referencien". Evita que `AccesoDatos` herede herramientas que no necesita.

**Línea: `<ProjectReference Include="..\AccesoDatos\AccesoDatos.csproj" />`** ⭐
**La línea que une los dos proyectos.** Le dice a `AppConsola`: "usá también el
código de `AccesoDatos`". Sin esta línea, el `using AccesoDatos.Models;` del
`Program.cs` daría error `CS0246: The type or namespace name 'AccesoDatos' could not
be found`, aunque la carpeta exista.

> **Diferencia clave:** `PackageReference` = paquete externo (NuGet, se descarga de
> internet). `ProjectReference` = proyecto tuyo (ya está en la solución, solo se
> "apunta"). Los archivos `.cs` de un proyecto referenciado **no se copian**: el
> compilador los lee desde donde están.

### 0.4 Cómo se construyó el proyecto (de cero, con comandos) ⭐ (nuevo)

```bash
# 1. Crear la solución
dotnet new sln -n AppConsola

# 2. Crear los dos proyectos DENTRO de la carpeta de la solución
dotnet new classlib -n AccesoDatos -o AccesoDatos
dotnet new console  -n AppConsola   -o AppConsola

# 3. Meter los proyectos en la solución
dotnet sln add AccesoDatos/AccesoDatos.csproj
dotnet sln add AppConsola/AppConsola.csproj

# 4. Vincular los proyectos (crea el <ProjectReference>)
dotnet add AppConsola reference AccesoDatos

# 5. Instalar EF Core
dotnet add AccesoDatos package Microsoft.EntityFrameworkCore.Sqlite
dotnet add AppConsola   package Microsoft.EntityFrameworkCore.Design
dotnet add AppConsola   package Microsoft.EntityFrameworkCore.Tools
```

**Ojo con el orden:** el paso 4 es el que genera la línea `ProjectReference` del
`.csproj`. Si se hace al revés, el proyecto no encuentra al otro y no compila.
Todos los comandos se ejecutan **desde la carpeta de la solución**.

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

El menú que ve el usuario, literal:

```csharp
bool continuar = true;

while (continuar)
{
    Console.WriteLine("1. Alta Autor");
    Console.WriteLine("2. Alta Categoría");
    Console.WriteLine("3. Alta Libro");
    Console.WriteLine();

    Console.WriteLine("4. Ver Autores");
    Console.WriteLine("5. Ver Categorías");
    Console.WriteLine("6. Ver Libros");
    Console.WriteLine();

    Console.WriteLine("7. Modificar Libro");
    Console.WriteLine("8. Eliminar Libro");
    Console.WriteLine("9. Modificar Autor");
    Console.WriteLine();

    // Opciones LINQ - entregable 3.
    Console.WriteLine("10. Ver libros más recientes");
    Console.WriteLine("11. Cantidad total de libros");
    Console.WriteLine("12. Cantidad de libros activos");
    Console.WriteLine("13. Buscar libro por ID");
    Console.WriteLine("14. Ver libros ordenados por título");
    Console.WriteLine("15. Verificar si existen libros activos");
    Console.WriteLine();

    Console.WriteLine("0. Salir");
    Console.WriteLine();

    Console.Write("Seleccione una opción: ");
    string opcion = Console.ReadLine();

    Console.Clear();

    switch (opcion)
    {
        case "1":
            AltaAutor();
            break;

        // ... cases 2 a 15: exactamente la misma forma, uno por función ...
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

> Los 15 `case` son **idénticos en forma** (`case "N":` → llamada → `break;`), por
> eso no se repiten acá: el mapa completo de opción → función está en la tabla de
> abajo y el código de cada función, en 5.3 – 5.10. Fijate que el comentario
> `// Opciones LINQ - entregable 3.` separa el bloque: el **menú mismo está
> dividido en 3 entregables**, y el último bloque es el que agrega `LibroRepository`.

**Línea: `bool continuar = true;`**
Cajita que controla si el menú sigue mostrándose. Mientras sea `true`, se repite un
ciclo: mostrar menú, leer opción, ejecutar. Apenas el usuario elige `0`, pasa a
`false` y el bucle termina.

**Línea: `Console.Write(...)` vs `Console.WriteLine(...)`**
La **primera** no baja de línea; la **segunda** sí. Por eso los textos del menú y
del pedido usan `WriteLine` (quedan uno debajo del otro) y los pedidos de dato
(`"Seleccione una opción: "`, `"Título: "`) usan `Write`, para que el cursor se
quede en la misma línea esperando que el usuario escriba.

**Línea: `string opcion = Console.ReadLine();`**
`ReadLine()` **devuelve texto**, por eso la variable es `string` y los `case` comparan
con `"1"`, `"2"`… **con comillas**. Si se escribiera `case 1:` (sin comillas) nunca
entraría por ahí y el `default` avisaría siempre "Opción inválida".

**Línea: `Console.Clear();`**
Borra la pantalla **después** de leer, para que la función que viene muestre su
resultado en una pantalla limpia. Es el "<kbd>cls</kbd>" de la consola.

**Línea: `switch (opcion)`**
Sirve para decidir qué hacer según el valor leído. Si eligió `"1"`, llama a
`AltaAutor()`; `"2"` → `AltaCategoria()`; etc. `case "0"` apaga el bucle; `default`
avisa opción inválida. El `break;` es **obligatorio**: sin él, C# sigue ejecutando
el `case` siguiente ("caída" de casos) y terminaría llamando a todas las funciones.

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

> El mismo patrón (leer → `new` con inicializador → `Agregar` → avisar →
> `PresioneParaContinuar()`) se repite en **todas** las altas. Los ejemplos
> completos están en las secciones 5.8 y 5.9, sin volver a explicar
> `Console.ReadLine()` ni el inicializador de objetos.

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

`Console.ReadKey()` **espera una tecla** (por eso el programa "se congela" hasta que
el usuario siga); `Console.Clear()` borra lo que había, para que cada pantalla
arranque limpia. No necesita `using System;` gracias a `ImplicitUsings` (ver 0.3).

### 5.8 Las dos altas que faltan ⭐ (nuevo)

**`AltaCategoria()`** — es `AltaAutor()` con otro tipo de objeto: por eso no hace
falta volver a explicarla línea por línea.

```csharp
void AltaCategoria()
{
    Console.Write("Nombre de la categoría: ");

    Categoria categoria = new Categoria
    {
        Nombre = Console.ReadLine()
    };

    categoriaRepository.Agregar(categoria);

    Console.WriteLine("Categoría registrada correctamente.");

    PresioneParaContinuar();
}
```

**`AltaLibro()`** — la más larga del menú, porque un libro **depende** de un autor
y de una categoría: tiene que dejar que el usuario los elija.

```csharp
void AltaLibro()
{
    Console.Write("Título: ");
    string titulo = Console.ReadLine();

    Console.Write("Año publicación: ");
    int anio = int.Parse(Console.ReadLine());

    Console.WriteLine();
    Console.WriteLine("Autores disponibles:");

    foreach (var autor in autorRepository.ObtenerTodos())
    {
        Console.WriteLine(
            $"ID: {autor.Id} - {autor.Nombre}");
    }

    Console.Write("Seleccione el ID del autor: ");
    int autorId = int.Parse(Console.ReadLine());

    Console.WriteLine();
    Console.WriteLine("Categorías disponibles:");

    foreach (var categoria in categoriaRepository.ObtenerTodos())
    {
        Console.WriteLine(
            $"ID: {categoria.Id} - {categoria.Nombre}");
    }

    Console.Write("Seleccione el ID de la categoría: ");
    int categoriaId = int.Parse(Console.ReadLine());

    Libro libro = new Libro
    {
        Titulo = titulo,
        AnioPublicacion = anio,
        AutorId = autorId,
        CategoriaId = categoriaId,
        Activo = true
    };

    libroRepository.Aggregar(libro);

    Console.WriteLine("Libro registrado correctamente.");

    PresioneParaContinuar();
}
```

Lo que hay que mirar acá:

| Línea | Por qué importa |
|-------|-----------------|
| `string titulo = Console.ReadLine();` | se lee **primero** a una variable, porque el `ReadLine` se ejecuta una sola vez: si se escribieran 4 veces seguidas dentro del inicializador, las preguntas saldrían en orden inverso al del código y sería imposible acertar |
| `int anio = int.Parse(Console.ReadLine());` | `int.Parse` (o `Convert.ToInt32`) convierte el **texto** del usuario a número. Sin esto, `AnioPublicacion = "2020"` ni siquiera compila |
| `foreach (...) { $"ID: {autor.Id} - {autor.Nombre}" }` | el **menú de selección**: sin mostrar el `Id`, el usuario no tendría forma de saber qué número escribir |
| `AutorId = autorId` | guarda **el número**, no el objeto `autor`. Esta es la clave foránea (ver 1.3) |
| `Activo = true` | explícito **aunque** la propiedad ya arranque en `true` (ver 1.3). Queda a la vista que el libro nace activo |

> **Dato importante:** `AltaLibro()` **no** asigna `Autor = autor` ni
> `Categoria = categoria`. Guarda solo los `Id`. La navegación
> (`libro.Autor.Nombre`) se resuelve **después**, cuando se consulta con
> `Include`. Si se asignara el objeto entero, EF Core intentaría insertar un autor
> duplicado.

### 5.9 `MostrarCategorias()` ⭐ (nuevo)

Es `MostrarAutores()` (ver 5.4) cambiando el repositorio y el título. Se muestra
por separado para dejar claro que **cada entidad tiene su propio listado** y que
todos usan el mismo `ObtenerTodos()` del genérico.

```csharp
void MostrarCategorias()
{
    Console.WriteLine("===== CATEGORÍAS =====");

    var categorias = categoriaRepository.ObtenerTodos();

    foreach (var categoria in categorias)
    {
        Console.WriteLine(
            $"ID: {categoria.Id} | Nombre: {categoria.Nombre}");
    }

    PresioneParaContinuar();
}
```

> Fijate el detalle: el genérico tiene **un solo** `ObtenerTodos()`, pero como la
> variable está declarada como `IGenericRepository<Categoria>`, el mismo método
> devuelve `List<Categoria>` y no `List<Autor>`. Eso es el **genérico**: un código,
> muchos tipos.

### 5.10 Modificar: buscar → cambiar → guardar ⭐ (nuevo)

**`ModificarAutor()`** y **`ModificarLibro()`** son el mismo método con distinto
tipo de entidad. Se explican una sola vez:

```csharp
void ModificarAutor()
{
    MostrarAutores();                              // 1. mostrar para elegir

    Console.Write("Ingrese el ID del autor: ");
    int id = int.Parse(Console.ReadLine());

    var autor = autorRepository.ObtenerPorId(id);  // 2. buscar (puede ser null)

    if (autor != null)                             // 3. ¿existe?
    {
        Console.Write("Nuevo nombre: ");
        autor.Nombre = Console.ReadLine();          // 4. cambiar la propiedad

        autorRepository.Modificar(autor);           // 5. guardar (UPDATE)

        Console.WriteLine("Autor modificado correctamente.");
    }
    else
    {
        Console.WriteLine("Autor no encontrado.");
    }

    PresioneParaContinuar();
}

void ModificarLibro()
{
    MostrarLibros();

    Console.Write("Ingrese el ID del libro: ");
    int id = int.Parse(Console.ReadLine());

    var libro = libroRepository.ObtenerPorId(id);

    if (libro != null)
    {
        Console.Write("Nuevo título: ");
        libro.Titulo = Console.ReadLine();

        libroRepository.Modificar(libro);

        Console.WriteLine("Libro modificado correctamente.");
    }
    else
    {
        Console.WriteLine("Libro no encontrado.");
    }

    PresioneParaContinuar();
}
```

**Los 5 pasos son siempre los mismos** (y son los mismos que en `EliminarLibro`,
5.6, salvo que ahí en vez de cambiar un dato se cambia `Activo` a `false`):

1. **Mostrar** la lista, para que el usuario vea los `Id` válidos.
2. **Pedir** el `Id` y convertirlo con `int.Parse`.
3. **Buscar** con `ObtenerPorId(id)` → puede devolver `null`.
4. **Comprobar** `if (x != null)` antes de tocarlo (si no, `NullReferenceException`).
5. **Cambiar** la propiedad y **`Modificar(x)`** → `Update()` + `SaveChanges()`.

> **¿Por qué `Modificar` y no `Add`?** Porque el registro **ya existe**: `Add`
> intentaría insertar una fila nueva (y fallaría por la clave primaria
> duplicada). `Update` genera un `UPDATE`. Es el mismo motivo por el que
> `EliminarLibro` usa `Modificar`: no está creando ni borrando nada, solo
> cambiando el valor de `Activo`.

### 5.11 El mapa completo de `Program.cs` ⭐ (nuevo)

Con esto está el **100% del `Program.cs`** (431 líneas) repartido en esta guía:

| # | Parte del archivo | Dónde se explica |
|---|------------------|------------------|
| 1 | `using` + los 3 repositorios | 5.1 |
| 2 | `while` + el `switch` con los 15 `case` | 5.2 |
| 3 | `AltaAutor()` | 5.3 |
| 4 | `AltaCategoria()` | 5.8 |
| 5 | `AltaLibro()` | 5.8 |
| 6 | `MostrarAutores()` | 5.4 |
| 7 | `MostrarCategorias()` | 5.9 |
| 8 | `MostrarLibros()` | 5.5 |
| 9 | `ModificarAutor()` | 5.10 |
| 10 | `ModificarLibro()` | 5.10 |
| 11 | `EliminarLibro()` | 5.6 |
| 12–17 | Las 6 funciones de consulta (opciones 10–15) | 5.7 |
| 18 | `PresioneParaContinuar()` | 5.7 |

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

Y el camino inverso, cuando el usuario **escribe**:

```
pantalla → ReadLine → int.Parse → new Libro → repo.Agregar → _context.Add → SaveChanges → archivo .db
```

**Las dos direcciones usan el mismo camino de acceso a datos.** Por eso las
operaciones de lectura y de escritura están en los mismos repositorios: no hay
"otro camino" que se pueda olvidar.

---

## 7. Comandos de consola que se usan en este proyecto ⭐ (nuevo)

Son los mismos que se mostraron en 0.4, pero aplicados al día a día. Se escriben en la
**consola de comandos** (o en la *Terminal* de Visual Studio) con la carpeta de la
solución como ubicación.

### 7.1 Compilar y correr

| Comando | Qué hace |
|---------|----------|
| `dotnet restore` | descarga los paquetes NuGet declarados en los `.csproj` |
| `dotnet build` | compila los dos proyectos (sin abrir la app) |
| `dotnet run --project AppConsola` | compila y **ejecuta** la consola |
| `dotnet clean` | borra `bin/` y `obj/` cuando algo se rompe raro |

> `dotnet run` se ejecuta **desde la solución**: la app **no tiene** `Main`
> propio visible, porque en .NET 10 el punto de entrada es un `Program.cs` con
> código de nivel superior (top-level statements) — por eso puede usar `using` y
> declarar variables **sueltas**, sin `class Program`. Esto también explica por
> qué las funciones se definen **después** del `while` y aun así se pueden llamar.

### 7.2 Migraciones (el paso que más cuesta)

Primero, una sola vez por máquina, hace falta instalar la herramienta global:

```bash
dotnet tool install --global dotnet-ef
```

| Comando | Qué hace |
|---------|----------|
| `dotnet ef migrations add InitialMigration` | compara los modelos con la base y **escribe** el archivo de migración en `Migrations/` |
| `dotnet ef database update` | **aplica** las migraciones pendientes: crea el archivo `.db` con las tablas |
| `dotnet ef migrations remove` | borra la última migración (si todavía no la aplicaste) |
| `dotnet ef migrations list` | muestra cuáles están aplicadas y cuáles faltan |
| `dotnet ef database update 0` | vuelve atrás, a la base vacía |

Los tres comandos de EF se ejecutan **desde el proyecto que tiene el paquete
`Design`**, o sea `AppConsola`:

```bash
dotnet ef database update --project AppConsola
```

**El flujo completo cuando cambiás un modelo** (por ejemplo, agregás
`Editorial` a `Libro`):

```
1. Editás Libro.cs          → el modelo cambió
2. dotnet ef migrations add EditorialEnLibro   → EF escribe qué hay que cambiar
3. Revisás el Up() de la migración             → opcional, pero conviene
4. dotnet ef database update                   → se aplica de verdad
```

> **Regla de oro:** `migrations add` **no toca la base**, solo escribe el archivo
> con los cambios. `database update` es el que modifica el `.db`. Si te saltás el
> paso 2 y solo hacés el 4, la base no cambia.

### 7.3 Las 3 capas de "guardar" — no confundir

| Qué hago | Qué pasa |
|----------|----------|
| `Add(entidad)` | EF **anota** en memoria: todavía no se escribió nada |
| `SaveChanges()` | EF **escribe** en el archivo `.db` (envía un INSERT/UPDATE/DELETE) |
| `database update` | Se **crea o modifica la estructura** de las tablas (no los datos) |

Los dos primeros pasan **dentro del repositorio**; el tercero es un comando
externo. `SaveChanges()` sin `Add` no hace nada; `Add` sin `SaveChanges` se
descarta al cerrar el programa.

---

## 8. El esquema de la base de datos (qué queda en el `.db`) ⭐ (nuevo)

Este es el resultado final: **3 tablas** unidas por **2 relaciones**.

```
┌─────────────────────┐         ┌─────────────────────┐
│       Autor         │         │      Categoria      │
├─────────────────────┤         ├─────────────────────┤
│ PK Id       INTEGER │         │ PK Id       INTEGER │
│    Nombre   TEXT    │         │    Nombre   TEXT    │
└─────────────────────┘         └─────────────────────┘
           │                                   │
           │ 1                                 │ 1
           │                                   │
           │ N                                 │ N
┌───────────────────────────────────────────────────────┐
│                      Libro                            │
├───────────────────────────────────────────────────────┤
│ PK  Id              INTEGER  autoincremental          │
│     Titulo          TEXT                              │
│     AnioPublicacion INTEGER                           │
│ FK  AutorId         INTEGER  → Autor.Id     (Cascade) │
│ FK  CategoriaId     INTEGER  → Categoria.Id (Cascade) │
│     Activo          INTEGER  (bool: 0 / 1)             │
└───────────────────────────────────────────────────────┘
```

**Cómo leer este diagrama:**

- `PK` = **clave primaria** (Primary Key): el `Id`, único e irrepetible.
- `FK` = **clave foránea** (Foreign Key): un número que **apunta** a otra tabla.
- `1` arriba / `N` abajo = relación **uno-a-muchos**: un autor tiene N libros; un
  libro pertenece a **un solo** autor.
- `Cascade` = si se borra el autor, sus libros se borran solos (ver 4).
- `Activo INTEGER 0/1` = un `bool` de SQLite: no tiene tipo `BOOLEAN`, se guarda
  como número. Por eso la columna existe aunque en C# se declare `bool`.

**Equivalencias entre la clase C# y la tabla SQLite:**

| Propiedad en C# | Tipo C# | Cómo queda en la tabla | Por qué |
|-----------------|---------|------------------------|---------|
| `Id` | `int` | `INTEGER` autoincremental | EF Core le agrega `Autoincrement` solo (ver 4) |
| `Nombre` | `string` | `TEXT` | cualquier texto |
| `AnioPublicacion` | `int` | `INTEGER` | número entero |
| `Activo` | `bool` | `INTEGER` (0 / 1) | SQLite no tiene tipo booleano |
| `Autor` | `Autor` | **no existe** | es propiedad de navegación: no es una columna (ver 1.3) |
| `Libros` | `List<Libro>` | **no existe** | es la lista inversa: tampoco es columna |

> **Las dos filas marcadas "no existe" son la clave de todo el modelo:** en la
> tabla `Libro` hay **3 columnas** (`Id`, `Titulo`, `AnioPublicacion`) más 2 FK y
> 1 bool, pero la clase `Libro` tiene **7 propiedades**. Las que no son columnas
> (las de navegación y la lista) existen **solo en memoria**, se llenan con
> `Include` o automáticamente cuando EF navega la relación.

**Índices** (`IX_Libro_AutorId`, `IX_Libro_CategoriaId`, que se ven en la
migración): son estructuras de búsqueda para encontrar rápido los libros de un autor.
No son datos ni reglas: solo aceleran las consultas.

---

## 9. Errores frecuentes y cómo se solvean ⭐ (nuevo)

Los 10 errores que más aparecen trabajando en este proyecto, con su causa real.

| # | Error | Causa real | Cómo se arregla |
|---|-------|-----------|-----------------|
| 1 | `CS0246: no se encuentra el tipo o namespace 'AccesoDatos'` | falta el `<ProjectReference>` en `AppConsola.csproj` | `dotnet add AppConsola reference AccesoDatos` |
| 2 | `CS0246: no se encuentra 'UseSqlite' / 'DbContext'` | falta el paquete EF Core | `dotnet add package Microsoft.EntityFrameworkCore.Sqlite` |
| 3 | `CS0246: no se encuentra 'Console'` | `ImplicitUsings` en `disable` | ponerlo en `enable`, o escribir `using System;` |
| 4 | `NullReferenceException` en `libro.Autor.Nombre` | faltó el `Include("Autor")` en la consulta | usar `ObtenerTodosCon("Autor")` (ver 5.5) |
| 5 | `NullReferenceException` al usar lo que devuelve `ObtenerPorId` | se usó el resultado sin chequear | `if (x != null) { ... } else { ... }` (ver 5.10) |
| 6 | `FormatException` / el programa se cae al escribir un ID | `int.Parse` con letras o con `Enter` solo | usar `int.TryParse` y avisar si falla |
| 7 | `SqliteException: no such table: Libro` | nunca se corrieron las migraciones | `dotnet ef database update` (ver 7.2) |
| 8 | `SqliteException: database is locked` | el programa quedó abierto con la base tomada | cerrar la consola / el `.db` y volver a correr |
| 9 | `CS1061: 'IGenericRepository<Libro>' no tiene 'ObtenerLibrosPorMasRecientes'` | la variable está tipada con la **interfaz** | declararla como `LibroRepository` (ver 5.1) |
| 10 | `InvalidOperationException: no such entity` al hacer `Add` con un `Id` ya usado | se usó `Add` para modificar | usar `Modificar` (ver 5.10) |

> Los errores **4 y 5** son los más difíciles de ver de entrada: el programa
> compila perfectamente y **solo falla cuando corre**. Por eso la guía insiste
> tanto con el `Include` y con el `if (x != null)`: son las dos protecciones que
> separan "anda en el ejemplo del profe" de "anda siempre".

**El improvement rápido que evita el error 6** (no está en el entregable, pero
conviene saberlo):

```csharp
Console.Write("Ingrese el ID del libro: ");

if (int.TryParse(Console.ReadLine(), out int id))
{
    // se puede seguir
}
else
{
    Console.WriteLine("El ID debe ser un número.");
    PresioneParaContinuar();
}
```

`int.TryParse` **no se rompe**: devuelve `true`/`false` en vez de tirar excepción.
Cambiar `Parse` por `TryParse` es, casi siempre, la mejor mejora posible en una
consola.

---

## 10. Autoevaluación: preguntas de examen ⭐ (nuevo)

Para verificar que la guía se entendió de verdad. Las respuestas están al final de
cada pregunta, tapadas con `<details>` para no hacer trampa.

<details>
<summary><b>1. ¿Qué pasaría si en <code>Autor.cs</code> la propiedad se llamara <code>ID</code> en vez de <code>Id</code>?</b></summary>

EF Core reconoce la clave primaria por convención **solo** si se llama `Id`. Con
`ID` (o `IdAutor`, o `Codigo`) EF no la reconoce y hay que configurarla a mano con
`[Key]` en `OnModelCreating`. Sin eso, la migración falla o crea la tabla sin clave
primaria.
</details>

<details>
<summary><b>2. ¿Cuál es la diferencia entre <code>AutorId</code> y <code>Autor</code>?</b></summary>

`AutorId` (int) es la **clave foránea**: la columna que realmente se guarda en la
tabla `Libro`. `Autor` (objeto) es la **propiedad de navegación**: existe solo en
memoria y sirve para escribir `libro.Autor.Nombre`. Hacen falta las dos: la
primera para la base, la segunda para el código.
</details>

<details>
<summary><b>3. ¿Por qué <code>GenericRepository</code> declara <code>protected readonly</code> y no <code>private</code>?</b></summary>

Porque `LibroRepository` **hereda** de `GenericRepository<Libro>` y usa `_context`
en sus consultas LINQ. Con `private`, la clase hija no podría verla y el proyecto
no compilaría. Y `readonly` impide que alguien reasigne la conexión por error.
</details>

<details>
<summary><b>4. Si <code>libroRepository</code> estuviera declarado como <code>IGenericRepository&lt;Libro&gt;</code>, ¿qué pasaría?</b></summary>

Compilaría, pero **ninguna** llamada a `ObtenerLibrosPorMasRecientes()`,
`ObtenerCantidadLibros()`, etc. compilaría: daría error `CS1061`, porque esos
métodos no están en la interfaz. El tipo de la variable decide qué métodos existen.
</details>

<details>
<summary><b>5. ¿Qué diferencia hay entre <code>Add</code> y <code>SaveChanges</code>?</b></summary>

`Add` solo **anota** la entidad en memoria (la cola de cambios). `SaveChanges` es
el que **escribe** en el archivo `.db`. Sin `SaveChanges`, el dato se pierde al
cerrar el programa.
</details>

<details>
<summary><b>6. ¿Qué diferencia hay entre <code>ObtenerPorId</code> (genérico) y <code>ObtenerLibroPorId</code> (específico)?</b></summary>

Los dos buscan por `Id` y los dos pueden devolver `null`. La diferencia es de
propósito: `ObtenerPorId` es parte del **contrato** de cualquier repositorio
(funciona con Autor, Categoria, Libro); `ObtenerLibroPorId` es una **consulta de
negocio** que solo tiene sentido para libros, así que vive en `LibroRepository`.
`Find` busca primero en memoria antes de ir a la base; `FirstOrDefault` siempre
consulta. Para buscar por clave primaria, `Find` es más rápido.
</details>

<details>
<summary><b>7. ¿Qué es el borrado lógico y por qué <code>EliminarLibro()</code> no llama a <code>Eliminar()</code>?</b></summary>

Borrado lógico es marcar `Activo = false` en vez de borrar la fila. `EliminarLibro`
no llama a `Eliminar` porque el entregable quiere **conservar el historial** de los
libros dados de baja. La fila sigue existiendo, pero todos los listados la filtran
con `l.Activo`. Fijate que `Eliminar()` **sí existe** en el repositorio genérico:
lo que no se usa para libros, no es que no exista.
</details>

<details>
<summary><b>8. ¿Qué hace <code>Include("Autor")</code> y qué pasa si no se pone?</b></summary>

Es un `Include` dentro de `ObtenerTodosCon`: le pide a EF que además de los datos
del libro traiga el **autor** relacionado (un `LEFT JOIN`). Si no se pone,
`libro.Autor` llega en `null` y `libro.Autor.Nombre` tira `NullReferenceException`.
</details>

<details>
<summary><b>9. ¿Para qué sirven <code>Up</code> y <code>Down</code> de una migración?</b></summary>

`Up` **aplica** los cambios (crea tablas, columnas, llaves). `Down` los
**revierte** (borra tablas). Sirven para poder volver atrás si algo salió mal:
`dotnet ef database update 0` ejecuta todos los `Down` en orden inverso.
</details>

<details>
<summary><b>10. ¿Qué significa <code>onDelete: ReferentialAction.Cascade</code>?</b></summary>

Que si se borra la fila de `Autor`, los `Libro` que apuntan a él se borran
**automáticamente**. Es lo que garantiza que no queden libros "huérfanos" sin
autor. El contrario sería `Restrict` (o `NoAction`), que impediría borrar el autor
mientras tenga libros asociados.
</details>

<details>
<summary><b>11. ¿Qué pasa si cambiás el nombre de la propiedad <code>Nombre</code> a <code>Apellido</code> en <code>Autor.cs</code>?</b></summary>

Hay que generar una migración nueva: `dotnet ef migrations add RenombrarColumna` y
luego `dotnet ef database update`. EF detecta el cambio como "se llama `Apellido`
la columna que antes era `Nombre`". Si solo cambiás el código y no hacés la
migración, el programa sigue leyendo la columna vieja.
</details>

<details>
<summary><b>12. ¿Por qué <code>ObtenerTodos()</code> usa <code>AsNoTracking()</code> y <code>ObtenerPorId()</code> no?</b></summary>

`AsNoTracking` le dice a EF que no haga memoria de los objetos, porque la consulta
es solo de **lectura**: sale más rápido. `ObtenerPorId` **no** lo usa porque su
resultado se va a **modificar** (en `ModificarLibro` y `EliminarLibro`) y EF
necesita saber si el objeto cambió para generar el `UPDATE`.
</details>

<details>
<summary><b>13. ¿Por qué el proyecto <code>AccesoDatos</code> no se puede ejecutar?</b></summary>

Porque su `.csproj` **no tiene** `<OutputType>Exe</OutputType>`. Es una
**biblioteca de clases**: no tiene punto de entrada, solo expone clases (`public`)
que otro proyecto —`AppConsola`, que sí tiene el `Exe`— utiliza.
</details>

<details>
<summary><b>14. ¿Qué pasaría si borrás el archivo <code>.db</code> y volvés a correr la app?</b></summary>

El archivo no se crea solo, aunque el código esté bien. Va a saltar
`SqliteException: no such table: Autor`, porque el archivo lo crea
`dotnet ef database update` (ver 7.2). El `.cs` no crea la base: solo la migra.
</details>

---

## 11. Glosario rápido ⭐ (nuevo)

| Término | Significado en una línea |
|---------|--------------------------|
| **Solution / `.slnx`** | archivo que **agrupa** proyectos; no tiene código |
| **Project / `.csproj`** | un proyecto; sí tiene código y su configuración |
| **Model / Entidad** | clase que representa una tabla (`Autor`, `Libro`) |
| **DbContext** | la clase que habla con la base de datos |
| **DbSet\<T\>** | una propiedad del DbContext = **una tabla** |
| **Migración** | historial de cambios del esquema (`Up` / `Down`) |
| **Interfaz** | contrato: dice **qué** métodos hay, no **cómo** se hacen |
| **Genérico** `<T>` | código que sirve para muchos tipos |
| **Clase hija** | clase que **hereda** otra y reutiliza su código |
| **`protected`** | visible en la clase y en sus hijas |
| **`readonly`** | se puede asignar una vez; después no cambia |
| **`static`** | pertenece a la clase, no a un objeto |
| **LINQ** | consultas con estilo natural (`Where`, `OrderBy`, `Count`) que se traducen a SQL |
| **Lambda** `x => x.campo` | "para cada `x`, hacé..." |
| **`Include`** | traé una relación junto con el dato principal |
| **`AsNoTracking()`** | no memorices el objeto (más rápido para leer) |
| **`Find` vs `FirstOrDefault`** | buscar por clave primaria vs buscar con un filtro |
| **`Add` vs `Update` vs `Remove`** | insertar / actualizar / borrar (siempre con `SaveChanges`) |
| **FK / clave foránea** | número que apunta a otra tabla |
| **Propiedad de navegación** | el objeto relacionado; no es columna |
| **Relación 1-a-N** | un autor (1) tiene muchos libros (N) |
| **`Cascade`** | al borrar el padre, se borran los hijos |
| **Borrado lógico** | marcar `Activo = false` en vez de borrar la fila |
| **`Nullable enable`** | el compilador obliga a marcar los tipos que pueden ser `null` |
| **`ImplicitUsings`** | agrega los `using` comunes solo |
| **`ProjectReference`** | este proyecto usa el código de otro proyecto |
| **`PackageReference`** | este proyecto usa un paquete de NuGet |

---

## Chuleta resumen de la guía

- **Solución**: agrupa proyectos; no tiene código.
- **`.csproj`**: versión de .NET (`net10.0`), paquetes (`PackageReference`) y
  proyectos relacionados (`ProjectReference`). Sin el `ProjectReference`, la
  biblioteca no se ve desde la consola.
- **`ImplicitUsings` / `Nullable`**: explican por qué no hay `using System;` y por
  qué existe el `?` en `Libro?`.
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
  revierte; las FK con `Cascade` borran en cadena. `migrations add` **escribe el
  archivo**, `database update` **toca la base**: son dos pasos distintos.
- **Program.cs**: `while` + `switch` para el menú; variables tipadas con la
  interfaz cuando alcanzan las operaciones base, y con `LibroRepository` cuando se
  necesitan los métodos LINQ (el tipo de la variable decide qué se puede llamar).
- **Los 5 pasos de toda modificación**: mostrar → pedir Id → buscar → chequear
  `null` → cambiar y `Modificar`.
- **Borrado lógico**: `Activo = false` con `Modificar`, nunca `Eliminar`.
- **Los 2 errores invisibles**: el `Include` que falta y el `null` sin chequear.
  Compilan bien y fallan recién cuando el programa corre.