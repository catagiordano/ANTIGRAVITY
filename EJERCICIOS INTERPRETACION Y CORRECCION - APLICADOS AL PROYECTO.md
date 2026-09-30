# BANCO DE EJERCICIOS — INTERPRETACIÓN Y CORRECCIÓN DE C# APLICADO AL PROYECTO
## `U9EjercicioEntregable` — AppConsola + AccesoDatos (C# / .NET / Entity Framework Core)

> Todos los fragmentos usan **los nombres reales del proyecto**: `Libro`, `Autor`, `Categoria`,
> `ApplicationDbContext`, los `DbSet` singulares (`_context.Libro`, `_context.Autor`,
> `_context.Categoria`), `GenericRepository<T>`, `IGenericRepository<T>` y `LibroRepository`
> de `AccesoDatos/Repositories/`.
>
> Los que están marcados **NO COMPILA** no se pueden ejecutar: son de análisis.
> Los demás se pueden copiar en `AppConsola/AppConsola/Program.cs` para probarlos de a uno.

**Nivel 1** = una línea o un concepto · **Nivel 2** = un método o una clase completa ·
**Nivel 3** = integrador (POO + LINQ + Repositorio + EF Core).

---

## Mapa ejercicio → archivo real del proyecto

| Ej. | Tema | Dónde tocar el código real |
|---|---|---|
| 1 | `Include` / N+1 / `NullReferenceException` | `GenericRepository.cs:23` vs `:30` — `Program.cs:232` |
| 2 | `private` vs `protected` en la clase base | `GenericRepository.cs:10` — `LibroRepository.cs:12` |
| 3 | `static` vs instancia | `GenericRepository.cs:12-15` (un contexto por repositorio) |
| 4 | Interfaz implementada incompleta | `IGenericRepository.cs` (20 líneas) |
| 5 | `struct` en un `foreach` | filtrado de `Libro` en `Program.cs:240` |
| 6 | Conversiones, división entera, `int.Parse` | `Program.cs:156`, `:168`, `:321` |
| 7 | Integrador: repositorio mal formado | `LibroRepository.cs` (agregar un método) |
| 8 | `Equals`/`GetHashCode` en `Libro` | `Models/Libro.cs` (agregar una propiedad) |
| 9 | Orden de construcción, campo `static` | `ApplicationDbContext.OnConfiguring` |
| 10 | División entera y `decimal` | `LibroRepository.cs:18-29` (agregar un promedio) |
| 11 | `break` / `continue` | `Program.cs:10` (el `while` del menú) |
| 12 | Polimorfismo y `base` | `Program.cs:242-246` (la línea del `$` interpolado) |
| 13 | LINQ encadenado | `LibroRepository.cs:10-50` |
| 14 | Recursividad | `Models/Libro.cs` (validar `Titulo` y `AnioPublicacion`) |
| 15 | Eventos y `delegate` | `Program.cs:191` (alta de libro) |
| 16 | `null`, `??`, `TryParse` | `Program.cs:156`, `:312-321` |
| 17 | Ejecución diferida | `LibroRepository.cs:12-14` y `:38-41` |
| 18 | Patrón Strategy | `Program.cs:242-246` (formatear la salida) |
| 19 | `Include` / `ThenInclude` | `GenericRepository.cs:30-36` |
| 20 | Colección en clase abstracta | `Models/Autor.cs:11` (`List<Libro> Libros`) |
| 21 | Integrador: servicio de aplicación | `Program.cs:150-196` (`AltaLibro`) |
| 22 | Instanciación de repositorios | `Program.cs:4-6` |
| 23 | Modificadores de acceso entre proyectos | `AccesoDatos` → `AppConsola` |
| 24 | Interfaz vs clase abstracta | `IGenericRepository.cs` + `GenericRepository.cs` |
| 25 | `virtual` / `override` / `sealed` / `new` | `Models/*.cs` |

---

# CATEGORÍA A — CORRECCIÓN DE CÓDIGO

## Ejercicio 1 — Nivel 1 — Relación no cargada: `NullReferenceException` en el listado de libros

**Enunciado:** En `Program.cs` la opción 6 "Ver Libros" funciona. ¿Por qué la opción 10 "Ver libros más recientes" **no** muestra el autor? ¿Y qué pasaría si en la opción 6 se usara `ObtenerTodos()` en vez de `ObtenerTodosCon("Autor")`? Justificar y corregir.

```csharp
// ===== Program.cs, opción 6: FUNCIONA =====
void MostrarLibros()
{
    var libros = libroRepository.ObtenerTodosCon("Autor");   // <-- Include

    foreach (var libro in libros.Where(l => l.Activo))
    {
        Console.WriteLine(
            $"ID: {libro.Id} | " +
            $"Título: {libro.Titulo} | " +
            $"Año: {libro.AnioPublicacion} | " +
            $"Autor: {libro.Autor.Nombre}");        // <-- Autor viene cargado
    }
}

// ===== Program.cs, opción 10: NO MUESTRA EL AUTOR =====
void MostrarLibrosMasRecientes()
{
    foreach (var libro in libroRepository.ObtenerLibrosPorMasRecientes())
    {
        Console.WriteLine($"{libro.Titulo} - {libro.AnioPublicacion}");
    }
}

// ===== Y si la opción 6 usara este listado? =====
var libros = libroRepository.ObtenerTodos();

foreach (var libro in libros)
{
    Console.WriteLine(libro.Autor.Nombre);          // <-- NullReferenceException
}
```

```csharp
// AccesoDatos/Repositories/GenericRepository.cs (código real del proyecto)
public List<T> ObtenerTodos()                                   // línea 23
{
    return _context.Set<T>()
                   .AsNoTracking()
                   .ToList();
}

public List<T> ObtenerTodosCon(string propiedadRelacionada)      // línea 30
{
    return _context.Set<T>()
                   .Include(propiedadRelacionada)
                   .AsNoTracking()
                   .ToList();
}
```

**// --- RESPUESTA ---**

**¿Compila?** Sí, las dos versiones compilan. El problema no es de compilación, es de **datos**.

**Qué hace `ObtenerTodos()`:** ejecuta `SELECT * FROM Libro`. La columna `AutorId` viene, pero la **propiedad de navegación `Libro.Autor`** queda en `null`, porque no se pidió información de la tabla `Autor`. En el `foreach`, `libro.Autor` es `null` y `libro.Autor.Nombre` revienta:

```text
System.NullReferenceException: Object reference not set to an instance of an object.
```

**Qué hace `ObtenerTodosCon("Autor")`:** el `Include("Autor")` agrega el `JOIN` con la tabla `Autor`, y entonces `libro.Autor` **sí** es un objeto con sus datos. Por eso la opción 6 funciona.

**Por qué la opción 10 no muestra el autor:** no es un error: `ObtenerLibrosPorMasRecientes()` (`LibroRepository.cs:10`) **no pide** el autor, solo hace `OrderByDescending(l => l.AnioPublicacion)`. Para mostrarlo hay que agregar el `Include`.

**¿Cuándo NO falla aunque falte el `Include`?** Si el `DbContext` tuviera **Lazy Loading** activado (`UseLazyLoadingProxies()`), la primera vez funcionaría, pero EF Core lanzaría **una consulta extra por cada libro**: el famoso **problema N+1**. Con 500 libros, 501 consultas. En este proyecto **no** hay proxies, así que falla siempre.

**Código corregido:**

```csharp
public List<Libro> ObtenerLibrosPorMasRecientes()
{
    return _context.Libro
                   .Include(l => l.Autor)              // JOIN con Autor
                   .AsNoTracking()                     // solo lectura
                   .OrderByDescending(l => l.AnioPublicacion)
                   .ToList();
}
```

**Variante con navegación inversa (cargar los libros desde el autor):**

```csharp
var autores = _context.Autor
                         .Include(a => a.Libros)          // colección uno-a-muchos
                         .AsNoTracking()
                         .ToList();

// Ahora sí: foreach (var libro in autor.Libros) ...
```

**Frase clave del parcial:** la clave foránea (`AutorId`) y la propiedad de navegación (`Autor`) son dos cosas distintas. `AutorId` es una **columna**; `Autor` es un **objeto** que solo existe si alguien pidió el `Include`. Por eso `Models/Libro.cs` declara las dos.

---

## Ejercicio 2 — Nivel 1 — `private` vs `protected` en la clase base (error CS0122)

**Enunciado:** ¿Compila este código? ¿Por qué? ¿Cómo se corregiría? Es el error más caro de un parcial de POO.

```csharp
// AccesoDatos/Repositories/GenericRepository.cs  (versión con ERROR)
public class GenericRepository<T> : IGenericRepository<T> where T : class
{
    private readonly ApplicationDbContext _context;      // <-- private

    public GenericRepository()
    {
        _context = new ApplicationDbContext();
    }

    public List<T> ObtenerTodos() => _context.Set<T>().AsNoTracking().ToList();
}

// AccesoDatos/Repositories/LibroRepository.cs
public class LibroRepository : GenericRepository<Libro>
{
    public List<Libro> ObtenerLibrosPorMasRecientes()
    {
        return _context.Libro                            // <-- ERROR DE COMPILACIÓN
                        .OrderByDescending(l => l.AnioPublicacion)
                        .ToList();
    }
}
```

**// --- RESPUESTA ---**

**¿Compila?** No.

```text
error CS0122: '_context' is inaccessible due to its protection level
```

**Por qué:** `private` significa "visible **solo dentro de la clase que lo declara**". La herencia no es una excepción: una clase hija **no puede tocar** los miembros privados de su clase padre. El `_context` de `GenericRepository<T>` es un detalle interno de esa clase: se escribe en su constructor y se usa en sus propios métodos. `LibroRepository` es una clase **distinta**, así que solo puede usar lo que el padre decida exponer (`public` o `protected`).

**Código corregido (una línea):**

```csharp
public class GenericRepository<T> : IGenericRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;     // <-- protected
    ...
}
```

**Tabla de modificadores (vale memorizarla):**

| Modificador | Se ve en… | ¿Alcanza para la herencia? |
|---|---|---|
| `private` | Solo la misma clase | No |
| `private protected` | Solo clases derivadas **del mismo ensamblado** | Sí, y más restrictivo |
| `protected` | La misma clase **y** sus derivadas | **Sí — es lo correcto acá** |
| `internal` | Cualquier clase **del mismo ensamblado**, tenga o no herencia | No alcanza |
| `protected internal` | Las derivadas de cualquier sitio **o** cualquiera del mismo ensamblado | Sí, pero abre de más |
| `public` | Cualquiera, desde cualquier parte | Sí |

**Dos argumentos para justificar el `protected`:**

1. **Comunicación mínima:** la clase padre publica exactamente lo que las hijas necesitan, ni una línea más. `GenericRepository<T>` se puede modificar por dentro sin romper a `LibroRepository`.
2. **Alternativa equivalente:** si se dejara en `private`, la única forma de que `LibroRepository` acceda al contexto sería un método `protected` del padre, por ejemplo `protected IQueryable<T> Consulta() => _context.Set<T>().AsNoTracking();`. Diseño equivalente, peor ergonomía.

**Ojo con la tentación de `public`:** `public ApplicationDbContext Contexto => _context;` compila y "funciona", pero expone el `DbContext` a `AppConsola`, que podría escribir en la base sin pasar por el repositorio: se rompe el patrón Repositorio.

---

## Ejercicio 3 — Nivel 1 — `static` mal usado y tres contextos de base de datos

**Enunciado:** ¿Compila el código? ¿Qué imprime una vez corregido? Justificar también el diseño de `Program.cs`.

```csharp
// AccesoDatos/Repositories/ContadorConsultas.cs  (NO COMPILA)
public class ContadorConsultas
{
    private int _consultas;
    private static int _totalGeneral;

    public void RegistrarConsulta(string operacion)
    {
        _consultas++;
        _totalGeneral++;
    }

    public int Consultas() => _consultas;

    public static void Mostrar(ContadorConsultas contador)
    {
        Console.WriteLine(contador.Consultas);        // línea problemática
    }

    public static int TotalGeneral => _totalGeneral;
}

// --- Uso ---
var a = new ContadorConsultas();
var b = new ContadorConsultas();

a.RegistrarConsulta("ObtenerTodos");
b.RegistrarConsulta("Agregar");

ContadorConsultas.Mostrar(a);
ContadorConsultas.Mostrar(b);
Console.WriteLine(ContadorConsultas.TotalGeneral);
```

**// --- RESPUESTA ---**

**¿Compila?** No.

```text
error CS0120: An object reference is required for the non-static field, method, or property
'ContadorConsultas.Consultas()'
```

**Por qué:** `static` significa "pertenece a la **clase**, no a las instancias". Un método `static` **no tiene `this`**: no existe ningún objeto al que aplicarse, así que no puede invocar `_consultas` (que es de instancia) sin que le pasen un objeto. El parámetro `contador` está ahí… pero solo se lo **pasa**, no lo **usa**: le faltan los paréntesis de invocación.

**Código corregido:**

```csharp
public static void Mostrar(ContadorConsultas contador)
{
    Console.WriteLine(contador.Consultas());     // <-- se invoca SOBRE el objeto
}
```

**Salida:**
```text
1
1
2
```

- `a.Consultas()` → 1 (registró una consulta).
- `b.Consultas()` → 1 (también una). `_consultas` es de **instancia**: cada objeto tiene la suya.
- `TotalGeneral` → 2. `_totalGeneral` es `static`: **una sola copia compartida** por todos los objetos de la clase.

**La diferencia clave:** `static` = compartido por toda la clase (y por todo el programa si es `public`); sin `static` = una copia por objeto.

**Aplicado al proyecto real:** en `Program.cs` hay **tres** repositorios y cada uno crea **su propio** contexto en el constructor:

```csharp
IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();            // contexto 1
IGenericRepository<Categoria> categoriaRepository = new GenericRepository<Categoria>(); // contexto 2
LibroRepository libroRepository = new LibroRepository();                               // contexto 3
```

Eso son **tres contextos** contra la misma base SQLite. Funciona porque SQLite es un archivo, pero en un servidor real con 200 usuarios cada `new ApplicationDbContext()` abriría una conexión nueva. La corrección de diseño es **inyección de dependencias**: un solo contexto, inyectado por constructor.

```csharp
// Esto exige AGREGAR el constructor con parámetro a GenericRepository<T>:
// el proyecto real solo tiene el constructor sin parámetros (línea 12).
using var context = new ApplicationDbContext();
IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>(context);
IGenericRepository<Categoria> categoriaRepository = new GenericRepository<Categoria>(context);
LibroRepository libroRepository = new LibroRepository(context);
```

**Lección para el parcial:** `static` se usa para lo que es único y compartido (un contador global, una configuración, una fábrica). Un contador de consultas **por repositorio** no debería ser `static`; un contador global de la aplicación, sí.

---

## Ejercicio 4 — Nivel 2 — La interfaz `IGenericRepository<T>` implementada a medias

**Enunciado:** ¿Compila este código? ¿Qué métodos le faltan? Completar la implementación.

```csharp
// AccesoDatos/Repositories/IGenericRepository.cs  (código real, 20 líneas)
public interface IGenericRepository<T> where T : class
{
    void Agregar(T entidad);
    List<T> ObtenerTodos();
    List<T> ObtenerTodosCon(string propiedadRelacionada);
    T ObtenerPorId(int id);
    void Modificar(T entidad);
    void Eliminar(object id);
}

// AccesoDatos/Repositories/ProductoRepository.cs  (NO COMPILA)
using AccesoDatos.Models;

public class ProductoRepository : IGenericRepository<Producto>
{
    private readonly ApplicationDbContext _context;

    public ProductoRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public List<Producto> ObtenerTodos() => _context.Producto.AsNoTracking().ToList();

    public void Agregar(Producto entidad)
    {
        _context.Producto.Add(entidad);
        _context.SaveChanges();
    }
}
```

**// --- RESPUESTA ---**

**¿Compila?** No. Faltan **cuatro** métodos de los seis:

```text
error CS0535: 'ProductoRepository' does not implement interface member
'IGenericRepository<Producto>.ObtenerTodosCon(string)'
error CS0535: 'ProductoRepository' does not implement interface member
'IGenericRepository<Producto>.ObtenerPorId(int)'
error CS0535: 'ProductoRepository' does not implement interface member
'IGenericRepository<Producto>.Modificar(Producto)'
error CS0535: 'ProductoRepository' does not implement interface member
'IGenericRepository<Producto>.Eliminar(object)'
```

**Por qué:** una interfaz es un **contrato de cumplimiento total**. Si una clase dice `: IGenericRepository<T>`, el compilador exige **los seis** métodos, con **exactamente** la misma firma (mismo nombre, mismos parámetros, mismo tipo de retorno). No existen métodos "opcionales" en una interfaz: el que la implementa promete que cualquiera que la use puede llamar a cualquiera de los seis.

**Código corregido:**

```csharp
public class ProductoRepository : IGenericRepository<Producto>
{
    private readonly ApplicationDbContext _context;

    public ProductoRepository(ApplicationDbContext context) => _context = context;

    public void Agregar(Producto entidad)
    {
        _context.Producto.Add(entidad);
        _context.SaveChanges();
    }

    public List<Producto> ObtenerTodos() => _context.Producto.AsNoTracking().ToList();

    public List<Producto> ObtenerTodosCon(string propiedadRelacionada)
        => _context.Producto.Include(propiedadRelacionada).AsNoTracking().ToList();

    public Producto ObtenerPorId(int id) => _context.Producto.Find(id);

    public void Modificar(Producto entidad)
    {
        _context.Producto.Update(entidad);
        _context.SaveChanges();
    }

    public void Eliminar(object id)
    {
        var entidad = _context.Producto.Find(id);

        if (entidad != null)
        {
            _context.Producto.Remove(entidad);
            _context.SaveChanges();
        }
    }
}
```

**Respuesta de diseño (la que vale en el parcial):** no hace falta escribir esta clase. El proyecto ya tiene `GenericRepository<T>`, que implementa **todo** el contrato para **cualquier** entidad. `ProductoRepository` solo debería existir si `Producto` necesita consultas que el genérico no puede expresar (un `Where` con `Join`, una proyección, un `GroupBy`). Ahí se crea un **repositorio específico** que **hereda** del genérico y **agrega** métodos, sin reimplementar el CRUD:

```csharp
public class ProductoRepository : GenericRepository<Producto>
{
    public ProductoRepository(ApplicationDbContext context) : base(context) { }

    public int ObtenerStockTotal() => _context.Producto.Sum(p => p.Stock);
}
```

**Variante moderna (C# 8+):** un método con implementación por defecto en la interfaz, para no obligar a escribirlo en todas las clases:

```csharp
public interface IGenericRepository<T> where T : class
{
    void Agregar(T entidad);
    List<T> ObtenerTodos();
    List<T> ObtenerTodosCon(string propiedadRelacionada);
    T ObtenerPorId(int id);
    void Modificar(T entidad);

    void Eliminar(object id)
    {
        throw new NotSupportedException($"La entidad {typeof(T).Name} no admite borrado.");
    }
}
```

---

## Ejercicio 5 — Nivel 2 — `struct` dentro de un `foreach`: error de compilación y mutaciones perdidas

**Enunciado:** Se quiere calcular en qué posición de la estantería (fila/columna) queda cada libro. ¿Compila? ¿Qué falla?

```csharp
public struct Posicion
{
    public int Fila;
    public int Columna;

    public void Mover(int filas, int columnas)
    {
        Fila += filas;
        Columna += columnas;
    }

    public override string ToString() => $"(F{Fila}, C{Columna})";
}

var posiciones = new List<Posicion> { new Posicion(), new Posicion() };
int filas = 5;

foreach (var pos in posiciones)
{
    pos = new Posicion { Fila = 1, Columna = 1 };     // línea 1
    pos.Mover(filas, 2);                              // línea 2
}

foreach (var pos in posiciones)
    Console.WriteLine(pos);                           // línea 3

// Contraste: con una clase sí se persiste el cambio
var libros = new List<Libro>
{
    new Libro { Id = 1, Titulo = "Rayuela" },
    new Libro { Id = 2, Titulo = "Ficciones" }
};

var primero = libros[0];                              // línea 4
primero.Titulo = "Rayuela (2da edición)";             // línea 5
Console.WriteLine(primero.Titulo);
```

**// --- RESPUESTA ---**

**¿Compila?** No. Hay **dos** problemas distintos: uno de compilación y otro de lógica.

**Problema 1 (compilación) — línea 1:**

```text
error CS1656: Cannot assign to 'pos' because it is a 'foreach iteration variable'
```

La variable de iteración del `foreach` es de **solo lectura**. Está pensada para *leer* la colección, no para usarla de acumulador. Para eso está el `for`.

**Problema 2 (lógica) — línea 2:** aunque la línea 2 compila, **no cambia nada en la lista**. `Posicion` es un **`struct`**, o sea un **tipo valor**. En el `foreach`, `pos` es una **copia** del elemento de la lista, no el elemento. `Mover()` modifica la copia, y al terminar la vuelta esa copia se descarta. La salida real (eliminando la línea 1) es:

```text
(F0, C0)
(F0, C0)
Rayuela (2da edición)
```

Nunca `(F6, C3)`.

**La trampa que casi todos caen:** si el `Console.WriteLine(pos)` está **dentro** del `foreach`, la salida cambia a `(F5, C2)` dos veces. No es que el código esté bien: es que se está **imprimiendo la copia ya modificada**. Por eso el ejercicio pone el `WriteLine` **después** del ciclo: ahí se lee el estado **real** de la lista, que sigue intacto.

**La línea 5 sí funciona:** `libros[0]` devuelve la referencia real (porque `Libro` es `class`, un tipo referencia) y `Titulo` queda modificado. `Console.WriteLine(libros[0].Titulo)` también imprime el título nuevo: son el mismo objeto.

**Resumen de la regla:**

| Tipo | Qué copia el `foreach` | ¿Se puede modificar? |
|---|---|---|
| `class` (tipo referencia) | La **referencia** al objeto | Sí, se modifica el objeto original |
| `struct` (tipo valor) | Una **copia** del valor | No afecta al original |

**Código corregido — opción 1: que `Mover` devuelva el valor (la forma recomendada)**

```csharp
public struct Posicion
{
    public int Fila;
    public int Columna;

    // Devuelve una NUEVA posicion: no muta laoriginal
    public Posicion Mover(int filas, int columnas)
        => new Posicion { Fila = Fila + filas, Columna = Columna + columnas };

    public override string ToString() => $"(F{Fila}, C{Columna})";
}

var posiciones = new List<Posicion> { new Posicion(), new Posicion() };

for (int i = 0; i < posiciones.Count; i++)
{
    var p = new Posicion { Fila = 1, Columna = 1 };
    posiciones[i] = p.Mover(5, 2);        // <- la clave: reasignar el resultado
}
```

```text
(F6, C3)
(F6, C3)
```

**Ojo con esto, que es un error muy frecuente:**

```csharp
posiciones[i].Mover(5, 2);      // NO cambia nada
```

Aunque se esté indexando, `posiciones[i]` devuelve una **copia** del struct, `Mover` la muta y la copia se descarta. Con un `struct` mutable hay que **reasignar** el resultado, sí o sí.

**Código corregido — opción 2: `record struct` inmutable (la más idiomática)**

```csharp
public readonly record struct Posicion(int Fila, int Columna)
{
    public Posicion Mover(int filas, int columnas) => new(Fila + filas, Columna + columnas);
    public override string ToString() => $"(F{Fila}, C{Columna})";
}

var posiciones = new List<Posicion> { new(0, 0), new(0, 0) };

for (int i = 0; i < posiciones.Count; i++)
    posiciones[i] = posiciones[i].Mover(5, 2);

foreach (var pos in posiciones)
    Console.WriteLine(pos);
```

```text
(F5, C2)
(F5, C2)
```

**Un detalle de C# que nivel 2 suele preguntar:** ¿se puede usar `ref` para mutar in-place? Sí, pero **solo con arreglos**, no con el indexer de `List<T>`:

```csharp
var arr = new Posicion[2];
ref Posicion slot = ref arr[0];   // funciona: el indexer de los arreglos devuelve ref
slot.Mover(5, 2);                  // sí modifica arr[0]

// En un List<T>:
// error CS0206: A non ref-returning property or indexer may not be used as an out or ref value
ref Posicion slot2 = ref lista[0];
```

**Aplicado al proyecto:** `List<Libro> Libros { get; set; } = new();` en `Models/Autor.cs` contiene **clases**, así que `foreach (var libro in autor.Libros) libro.Titulo = "x";` sí modifica los libros reales. Si `Libro` fuera `struct`, ese `foreach` sería una pérdida de datos silenciosa: el tipo de error más difícil de detectar, porque **no da ningún error de compilación ni de ejecución**.

---

## Ejercicio 6 — Nivel 2 — Conversiones, división entera y `int.Parse` con entrada vacía

**Enunciado:** ¿Compilan estas líneas? ¿Qué imprime el código corregido?

```csharp
// (A) Promedio de años de publicación
int sumaAnios = 1967 + 1963 + 1955 + 1944;
int cantidad = 4;
int promedioAnios = sumaAnios / cantidad;             // (A)

// (B) Sumar un decimal a un entero
double promedio = 6.5;
int total = sumaAnios + promedio;                     // (B)

// (C) Cargar el año desde la consola (Program.cs:156)
int anio = int.Parse(Console.ReadLine());              // (C)

// (D) Sigla del tipo de documento
int anioLibro = 1990;
char sigla = "L";                                     // (D)

// (E) Convertir el año a texto y de vuelta
int copia = (int)anioLibro.ToString();                 // (E)

// (F) Descuento sobre un precio
double precio = 199.99;
decimal totalConDescuento = precio * 0.90m;            // (F)
```

**// --- RESPUESTA ---**

| Línea | ¿Compila? | Motivo |
|---|---|---|
| (A) | Sí | `int / int` = **división entera**: `7829 / 4` = **1957**, se descarta el `,25`. |
| (B) | **No** | `promedio` es `double`, así que el `int` se promociona a `double` y la suma vale `7835.5`. Guardar un `double` en un `int` **nunca** es implícito. **error CS0266** |
| (C) | Sí, **pero puede fallar** | Compila, pero si el usuario solo aprieta Enter, `Console.ReadLine()` devuelve `""` y `int.Parse("")` lanza **`FormatException`**. Es un bug real del `Program.cs:156`. (Devolver `null` es otro caso: ver la nota de abajo.) |
| (D) | **No** | `"L"` es un `string`; no hay conversión implícita `string` → `char`. **error CS0266** |
| (E) | **No** | El cast `(int)` solo es válido si existe una conversión de tipos. `string` → `int` no existe. **error CS0030** |
| (F) | **No** | `double * decimal` no es una operación válida (no se pueden mezclar). **error CS0019** |

**La diferencia entre `""` y `null` (pregunta clásica):**

| Entrada | Qué devuelve `Console.ReadLine()` | Qué lanza `int.Parse` |
|---|---|---|
| El usuario apretó Enter | `""` (string vacío, **no** es `null`) | **`FormatException`** |
| Redirección de un archivo vacío / `Ctrl+Z` en Windows (fin de flujo) | `null` | **`ArgumentNullException`** |

Y `catch (FormatException)` **no** alcanza para `null`: la excepción se escapa y el programa se cae igual. Si se quiere cubrir todo, hace falta `catch (ArgumentNullException)` además, o directamente `int.TryParse`, que maneja los dos casos devolviendo `false`.

**Código corregido:**

```csharp
int sumaAnios = 1967 + 1963 + 1955 + 1944;      // 7829
int cantidad = 4;

// (A) Para obtener el promedio real hay que convertir antes de dividir
int promedioAnios = (int)(sumaAnios / (double)cantidad);      // 1957
double promedioReal = sumaAnios / (double)cantidad;           // 1957.25

// (B)
double promedio = 6.5;
double total = sumaAnios + promedio;                        // 7835.5
int totalRedondeado = (int)Math.Round(total);               // 7836

// (C) La forma segura de leer un número
int anio;
while (!int.TryParse(Console.ReadLine(), out anio))
{
    Console.WriteLine("Año inválido, ingrese un número.");
}

// (D)
char sigla = "L"[0];                                      // 'L'
char sigla2 = char.Parse("L");                            // 'L'

// (E)
int anioLibro = 1990;
int copia = int.Parse(anioLibro.ToString());               // 1990
int copia2 = (int)anioLibro;                              // 1990 (sin pasar por el string)

// (F)
decimal precio = 199.99m;                                  // sufijo m = decimal
decimal totalConDescuento = precio * 0.90m;                // 179.9910
decimal redondeado = Math.Round(totalConDescuento, 2);     // 179.99
```

**Salida de las líneas corregidas que imprimen:**
```text
1957
1957.25
7835.5
7836
1990
1990
179.9910
179.99
```

**Sobre el `199.9910`:** el `decimal` de C# guarda **la escala** (los decimales) además del valor. `199.99` tiene escala 2 y `0.90` también, así que el producto conserva 4 decimales. El valor matemático es `179.991`; el `0` de más es información de precisión, no un número diferente. Por eso `Math.Round(..., 2)` sí devuelve `179.99`.

**Los cuatro conceptos que hay que explicar:**

1. **División entera.** `int / int` descarta la parte decimal. Para dividir de verdad, **al menos un operando** debe ser `double`, `float` o `decimal`. Por eso el cast va en el denominador: `(double)cantidad`.
2. **Jerarquía de conversiones implícitas:** `byte < short < int < long < float < double < decimal`. El camino nunca se "deshace": de `double` a `int` **siempre** hace falta un cast explícito.
3. **`199.99m`:** la `m` minúscula indica `decimal`. Sin ella sería `double`, y `double` **no representa exactamente** los decimales (recuerda `0.1 + 0.2 != 0.3`). Para precios, `decimal` **nunca** `double`.
4. **`int.Parse` vs `int.TryParse`:** `int.Parse` lanza excepción si falla; `int.TryParse` devuelve `false` y deja el valor en `0`. En un programa de consola con entrada del usuario, **`TryParse` siempre**.

**Corrección real sugerida en `Program.cs`:** el patrón `int.Parse(Console.ReadLine())` aparece en las líneas **156**, **168**, **180**, **260**, **286**, **321** y **379** (esta última con el valor leído en la 312).

```csharp
int anio;
while (!int.TryParse(Console.ReadLine(), out anio) || anio < 1450 || anio > 2100)
{
    Console.Write("Año inválido. Ingrese un número entre 1450 y 2100: ");
}
```

**Boxing / unboxing (pregunta clásica de examen):** `object caja = anio;` envuelve el `int` en un objeto (boxing: pasa a vivir en el heap). `int copia = (int)caja;` desenvuelve (unboxing). Si la caja contiene otra cosa, `InvalidCastException`.

---

## Ejercicio 7 — Nivel 3 (integrador) — Un método mal formado en `LibroRepository`

**Enunciado:** Se quiere agregar a `LibroRepository` un método que devuelva los títulos de los libros activos junto con el nombre de su autor. ¿Compila? ¿Qué defectos tiene?

```csharp
// AccesoDatos/Repositories/LibroRepository.cs  (NO COMPILA)
public class LibroRepository : GenericRepository<Libro>
{
    private readonly ApplicationDbContext _context = new ApplicationDbContext();

    public List<Libro> ObtenerActivosConAutor()
    {
        var libros = _context.Libro
                        .Where(l => l.Activo)
                        .AsNoTracking()
                        .ToList();

        return libros.Select(l => l.Autor.Nombre).ToList();     // línea problemática
    }
}
```

**// --- RESPUESTA ---**

**¿Compila?** No.

```text
error CS0266: Cannot implicitly convert type 'List<string>' to 'List<Libro>'
(an explicit conversion exists)
```

El método declara devolver `List<Libro>` pero el `return` entrega un `List<string>` (el `Select` proyecta cada `Libro` a su nombre de autor). `List<T>` es **invariante**: no hay conversión implícita de `List<string>` a `List<Libro>`, aunque ambos sean listas. Habría que forzar con `Cast<Libro>()`, y en runtime fallaría al convertir cada elemento: eso es un parche de emergencia, no una solución.

**Defectos acumulados, aunque se arreglara la firma:**

1. **Error de lógica → `NullReferenceException`.** No hay `Include`. Después de `AsNoTracking()` y de materializar con `ToList()`, la navegación `l.Autor` es `null` contra la base real. `l.Autor.Nombre` revienta en el `Select`, no en el `foreach`.
2. **Pérdida de información.** El método se llama `ObtenerActivosConAutor` pero devuelve **solo nombres**: el `Libro` que pidió el llamador se descarta, y si después quiere mostrar `libro.Titulo` ya no lo tiene.
3. **Responsabilidad invertida.** Un repositorio devuelve **entidades**, no texto. Proyectar a un objeto de lectura es decisión de la **capa de aplicación**, no del acceso a datos.
4. **Nuevo contexto por método.** `new ApplicationDbContext()` dentro del método es un **antipatrón**: cada llamada abre y cierra un contexto. El contexto se recibe por constructor (una vez) y lo administra quien lo creó.
5. **Se ignora el genérico.** La clase hereda de `GenericRepository<Libro>` y sin embargo **no usa ninguno** de sus métodos: reimplementa el acceso a datos a mano.

**Código corregido (repositorio de solo lectura + DTO):**

```csharp
// 1) El DTO de salida: inmutable, definido con record
public record LibroActivo(int Id, string Titulo, int Anio, string Autor);

// 2) Repositorio de SOLO LECTURA, con la proyección al servidor
public class LibroReadRepository
{
    private readonly ApplicationDbContext _context;

    public LibroReadRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public List<LibroActivo> ObtenerActivosConAutor()
    {
        return _context.Libro
                      .AsNoTracking()                       // solo lectura
                      .Where(l => l.Activo)                 // WHERE Activo = 1
                      .Include(l => l.Autor)                // JOIN Autor
                      .OrderBy(l => l.Titulo)               // ORDER BY Titulo
                      .Select(l => new LibroActivo(         // se traduce a SQL
                          l.Id,
                          l.Titulo,
                          l.AnioPublicacion,
                          l.Autor!.Nombre))
                      .ToList();
    }
}
```

**Y el `Program.cs` consumidor:**

```csharp
var repoLectura = new LibroReadRepository(context);

foreach (var item in repoLectura.ObtenerActivosConAutor())
{
    Console.WriteLine($"{item.Titulo} ({item.Anio}) - {item.Autor}");
}
```

**Dos mensajes clave para el parcial:**

- **`GenericRepository<T>`** sirve para el CRUD estándar de cualquier entidad. Cuando aparece un `Where` con relaciones, una proyección o un `GroupBy`, se crea un **repositorio específico de lectura** que hereda del genérico y **agrega** consultas, sin duplicar el CRUD.
- **`AsNoTracking()`** solo tiene sentido en consultas de lectura: sin él, cada entidad materializada queda **rastreada** en el `Change Tracker`. Alguien podría modificarla por error y un `SaveChanges()` posterior guardaría cambios no deseados.

---

## Ejercicio 8 — Nivel 3 (integrador) — `Libro` usado como clave de `Dictionary`

**Enunciado:** Se cuenta cuántos libros hay por autor, usando el `Libro` como clave del diccionario. ¿Compila? ¿Qué imprime? ¿Cuál es el error conceptual y cómo se corrige?

```csharp
var libros = new List<Libro>
{
    new Libro { Id = 1, Titulo = "Rayuela",   AnioPublicacion = 1963, AutorId = 1 },
    new Libro { Id = 1, Titulo = "Rayuela",   AnioPublicacion = 1963, AutorId = 1 },
    new Libro { Id = 2, Titulo = "Ficciones", AnioPublicacion = 1944, AutorId = 2 }
};

var porId = new Dictionary<Libro, int>();
porId[libros[0]] = 1;
porId[libros[1]] = 1;

Console.WriteLine(porId.Count);

var porTitulo = new Dictionary<string, int>();
porTitulo[libros[0].Titulo] = 1;
porTitulo[libros[1].Titulo] = 2;

Console.WriteLine(porTitulo.Count);
Console.WriteLine(porTitulo[libros[0].Titulo]);
```

**// --- RESPUESTA ---**

**¿Compila?** Sí, sin un solo error, y no lanza excepción.

**Salida:**
```text
2
1
2
```

**El error conceptual:** las dos primeras líneas coinciden porque usan `libros[0]` y `libros[1]`… pero son **objetos distintos en memoria**. Un `Dictionary<TKey, TValue>` decide si dos claves son la misma usando `EqualityComparer<Libro>.Default`, y para una clase que **no** redefine `Equals`/`GetHashCode` eso se resuelve a la comparación **por referencia** (dirección de memoria). Resultado: 2 entradas donde el alumno esperaba 1.

En cambio, el segundo diccionario usa `string` como clave, y el `==` de `string` **sí compara valor** (carácter a carácter): `"Rayuela" == "Rayuela"` es `true`, así que queda una sola entrada, con valor 2 (la última asignación pisa la anterior).

**Consecuencia real en este proyecto:** sin `AsNoTracking()`, EF Core **rastrea** las entidades, y si el mismo `Libro` se carga en dos consultas del mismo contexto, EF devuelve **la misma instancia** (por eso a veces "funciona"). Pero con `AsNoTracking()`, con dos contextos distintos, o con entidades creadas a mano, aparecen **instancias diferentes de la misma fila**, y cualquier lógica de "contar únicos" da resultados falsos. Es una fuente clásica de errores intermitentes.

**Código corregido — opción 1 (la recomendada: la clave natural, el `Id`)**

```csharp
var cantidadPorId = new Dictionary<int, int>();
cantidadPorId[libros[0].Id] = 1;
cantidadPorId[libros[1].Id] = 1;

Console.WriteLine(cantidadPorId.Count);      // 1
```

**Código corregido — opción 2 (redefinir la identidad de la entidad)**

```csharp
public class Libro
{
    public int Id { get; set; }
    public string Titulo { get; set; }
    public int AnioPublicacion { get; set; }

    public override bool Equals(object? obj)
        => obj is Libro otro && Id == otro.Id;

    public override int GetHashCode() => Id.GetHashCode();
}
```

**Opción 3 (C# 9+, la más limpia):**

```csharp
public record Libro(int Id, string Titulo, int AnioPublicacion, int AutorId);
```

`record` genera automáticamente `Equals`, `GetHashCode`, `ToString` y los operadores `==` / `!=` **por valor**. (Ojo: para mapear una entidad de EF Core con `record` hay que usar un constructor y propiedades con `init`.)

**Contrato que no se puede romper:** si dos objetos son iguales (`Equals` devuelve `true`), **obligatoriamente** deben devolver el mismo `GetHashCode()`. Si se cambia el `Id` de un objeto que ya está en el diccionario, la clave "envejece" y el `Dictionary` deja de encontrarla. Por eso las entidades de EF se usan como clave por su `Id`, que **no cambia**.

---

# CATEGORÍA B — ¿QUÉ IMPRIME?

## Ejercicio 9 — Nivel 1 — Campos `static`, constructor estático y orden de ejecución

**Enunciado:** ¿Qué imprime exactamente? Explicar el orden.

```csharp
public class Conexion
{
    public static int InstanciasCreadas;
    public string Descripcion { get; }

    static Conexion()
    {
        Console.WriteLine("1. Constructor estático");
        InstanciasCreadas = 100;
    }

    public Conexion(string descripcion)
    {
        InstanciasCreadas++;
        Descripcion = descripcion;
        Console.WriteLine($"2. Nueva conexión: {descripcion}");
    }

    public override string ToString() => $"[{Descripcion}] total={InstanciasCreadas}";
}

var c1 = new Conexion("Autores");
var c2 = new Conexion("Libros");

Console.WriteLine(c1);
Console.WriteLine(c2);
Console.WriteLine(Conexion.InstanciasCreadas);
```

**// --- RESPUESTA ---**

**Salida:**
```text
1. Constructor estático
2. Nueva conexión: Autores
2. Nueva conexión: Libros
[Autores] total=102
[Libros] total=102
102
```

**Justificación, paso a paso:**

- El **constructor estático** (`static Conexion()`) se ejecuta **una sola vez**, de manera automática, **antes** de la primera creación de un objeto de esa clase (o antes del primer acceso a un miembro `static`). Imprime su línea y deja `InstanciasCreadas = 100`. Aunque en el archivo estuviera escrito más abajo, igual se ejecuta primero.
- `InstanciasCreadas` es un **campo `static`**: existe en **una única copia compartida** por todos los objetos de la clase. Por eso `c1` ve el valor que dejó `c2` (102), aunque se muestre `c1` después de crear los dos.
- `Descripcion` es de **instancia**: cada objeto tiene el suyo (`"Autores"` en uno, `"Libros"` en el otro).
- `Console.WriteLine(c1)` invoca automáticamente `ToString()` (el compilador detecta que el objeto tiene `ToString()` y lo llama).
- Al final, `Conexion.InstanciasCreadas` es 102.

**Contraste que hay que saber:**

| | Sin `static` | Con `static` |
|---|---|---|
| Cuántas copias hay | Una por objeto | Una sola para toda la clase |
| Se reinicia al crear un objeto | Sí | No |
| Pertenece a… | La instancia | La clase |

**Aplicado al proyecto:** `ApplicationDbContext` es una clase normal (sin `static`), y por eso `GenericRepository<T>` crea **uno por repositorio** (`new ApplicationDbContext()` en su constructor). Si `ApplicationDbContext` fuera `static`, todos los repositorios compartirían el mismo contexto y el patrón Repositorio con interfaz quedaría sin sentido.

---

## Ejercicio 10 — Nivel 1 — Divisiones, `decimal` y promedios con datos reales

**Enunciado:** ¿Qué imprime? Justificar cada línea, sobre todo las que sorprendan.

```csharp
int anio1 = 1967;
int anio2 = 1963;
int anio3 = 1955;
int anio4 = 1944;

int cantidad = 4;
int suma = anio1 + anio2 + anio3 + anio4;

Console.WriteLine(suma / cantidad);
Console.WriteLine((double)suma / cantidad);
Console.WriteLine(suma % cantidad);

decimal precioPromedio = 24500.50m;
decimal descuento = 0.15m;

Console.WriteLine(precioPromedio * (1 - descuento));
Console.WriteLine(Math.Round(precioPromedio * (1 - descuento), 2));

double d = 0.1 + 0.2;
Console.WriteLine(d);
Console.WriteLine(d == 0.3);
Console.WriteLine(d > 0.3);

Console.WriteLine(1963 / 100);
Console.WriteLine(1963 / 100.0);
```

**// --- RESPUESTA ---**

**Salida:**
```text
1957
1957.25
1
20825.4250
20825.42
0.30000000000000004
False
True
19
19.63
```

**Justificación:**

| Línea | Resultado | Explicación |
|---|---|---|
| `suma / cantidad` | `1957` | Los dos son `int` → **división entera**: 7829 / 4 = 1957,25 y se descarta el `,25`. **El error clásico.** |
| `(double)suma / cantidad` | `1957.25` | Con **un solo** operando convertido a `double`, el `int` se promociona y la división pasa a ser real. |
| `suma % cantidad` | `1` | Resto de la división entera: 7829 = 4 × 1957 + 1. |
| `precioPromedio * (1 - descuento)` | `20825.4250` | `decimal` es exacto en base 10: 24500.50 × 0.85 = 20825.425, y además **guarda la escala**: 2 decimales × 2 decimales = 4, así que imprime `20825.4250`. |
| `Math.Round(…, 2)` | **`20825.42`** | Redondeo a 2 decimales, pero con la regla del **empate al par** (`MidpointRounding.ToEven`, el valor por defecto). El tercer decimal es `5` exacto, así que hay empate: se decide mirando si el dígito anterior es par o impar. Aquí es `2` (par), así que **se mantiene**: `20825.42`. |
| `d` | `0.30000000000000004` | `double` es binario (IEEE 754): ni 0.1 ni 0.2 son representables de forma exacta, y la suma arrastra el error. |
| `d == 0.3` | `False` | El literal `0.3` es **otro** `double` con su propio error: no son el mismo valor bit a bit. **Comparar `double` con `==` es un error de lógica.** |
| `d > 0.3` | `True` | El error de la suma hace que `d` sea ligeramente mayor. |
| `1963 / 100` | `19` | Ambos literales son enteros → división entera. |
| `1963 / 100.0` | `19.63` | `100.0` es `double` → división real. |

**El redondeo al par, que es la línea que más confunde.** `Math.Round` **no** redondea "medio hacia arriba" como hace la calculadora escolar. Redondea al valor **par** más cercano en caso de empate:

| Expresión | Resultado | Por qué |
|---|---|---|
| `Math.Round(2.5m, 0)` | `2` | Empate: el dígito es `2`, que es par → se queda |
| `Math.Round(3.5m, 0)` | `4` | Empate: el dígito es `3`, que es impar → sube al par más cercano |
| `Math.Round(-2.5m, 0)` | `-2` | Ídem, hacia el par |
| `Math.Round(20825.4250m, 2)` | `20825.42` | Empate en el tercer decimal, y `2` es par |
| `Math.Round(20825.4260m, 2)` | `20825.43` | Sin empate (el tercer decimal es `6`): sube |

Si se necesita el "medio hacia arriba" de la calculadora, hay que pedirlo explícitamente:

```csharp
Math.Round(20825.4250m, 2, MidpointRounding.AwayFromZero);   // 20825.43
```

En contabilidad y facturación eso es exactamente lo que se quiere, y por eso conviene **no** confiarse del `Math.Round` por defecto para sumar importes.

**Reglas para memorizar:**

- **Para un promedio de enteros, el cast va en el denominador** (o en ambos): `suma / (double)cantidad`.
- **Para dinero, `decimal` con sufijo `m`** y redondeo explícito con `Math.Round`.
- **Nunca comparar `float`/`double` con `==`:** usar `Math.Abs(a - b) < 0.000001`.
- **`int.Parse("19.63")` lanza `FormatException`:** el separador de miles y el punto decimal no coinciden con el formato regional.

**Aplicado al proyecto:** si se agrega al `LibroRepository` un método `ObtenerPromedioAnio()` que devuelva `int` con `Sum(...) / Count(...)`, el resultado **pierde la parte decimal**. Hay que castear: `(int)(promedio)` da 1957, o `Math.Round(promedio)` da 1957. Conviene devolver `double` si se quiere el valor exacto.

---

## Ejercicio 11 — Nivel 1 — `break`, `continue` y el menú del `Program.cs`

**Enunciado:** ¿Qué imprime? ¿En qué vuelta se detiene cada ciclo y por qué? **Para que la salida sea determinista, el enunciado declara la entrada que escribe el usuario: `3`, `11`, `99`, `0`.**

```csharp
// Réplica del menú de Program.cs, con validaciones
// ENTRADA del usuario, en este orden: 3 → 11 → 99 → 0
int opcion;
while (true)
{
    Console.Write("Seleccione una opción: ");
    opcion = int.Parse(Console.ReadLine());      // el usuario escribe: 3, 11, 99, 0

    if (opcion == 0) break;                        // A: salir del programa
    if (opcion < 1 || opcion > 15) continue;        // B: opción inválida

    Console.WriteLine("Ejecutando opción " + opcion);

    if (opcion == 3)
    {
        for (int i = 1; i <= 5; i++)
        {
            if (i == 2) continue;                   // C
            if (i == 4) break;                      // D
            Console.WriteLine("  item " + i);
        }
    }
}

Console.WriteLine("Fin");

foreach (var letra in "Rayuela")
{
    if (letra == 'y') continue;                    // E
    if (letra == 'l') break;                       // F
    Console.WriteLine(letra);
}
```

**// --- RESPUESTA ---**

**Salida:**
```text
Seleccione una opción: Ejecutando opción 3
  item 1
  item 3
Seleccione una opción: Ejecutando opción 11
Seleccione una opción: Seleccione una opción: Fin
R
a
u
e
```

**Justificación, vuelta por vuelta:**

- **Vuelta 1 (`opcion = 3`):** no es 0 (no hay `break`) y está entre 1 y 15 (no hay `continue`), así que imprime "Ejecutando opción 3". Como es 3, entra al `for`:
  - `i = 1`: ni 2 ni 4 → imprime "item 1".
  - `i = 2`: `continue` salta el `WriteLine` **pero la vuelta sí ocurrió** (el `i++` se ejecuta igual).
  - `i = 3`: imprime "item 3".
  - `i = 4`: `break` **abandona el `for` por completo**. El `i++` tampoco se ejecuta: la vuelta 5 nunca se alcanza. Por eso nunca se ve "item 5".
- **Vuelta 2 (`opcion = 11`):** imprime "Ejecutando opción 11" y **no** entra al `for` (el `if` compara contra 3).
- **Vuelta 3 (`opcion = 99`):** `continue` salta directo a la condición del `while`, así que **no imprime nada**. Pero el `Console.Write` del prompt ya se había ejecutado: por eso en la salida se ven **dos prompts pegados en la misma línea** y ningún "Ejecutando opción 99".
- **Vuelta 4 (`opcion = 0`):** entra por el `break` y **abandona el `while`**. Se imprime "Fin".
- **El `foreach` sobre `"Rayuela"`:** un `string` es `IEnumerable<char>`. `R` imprime, `a` imprime, `y` dispara `continue` (no imprime, pero la vuelta sigue), `u` imprime, `e` imprime, `l` dispara `break` y el recorrido termina. Nunca llega a la `a` final.

**El detalle clave del `foreach`:** tanto `continue` como `break` están **antes** del `WriteLine`, así que la letra que dispara cualquiera de los dos **no se imprime**. Si el `break` se escribiera después del `WriteLine`, la `l` sí se vería. **El orden de las condiciones y de las instrucciones no es decorativo: cambia la salida.**

**La trampa del enunciado (importante para el parcial):** si en lugar de `opcion = int.Parse(Console.ReadLine())` se escribe `opcion = 11;` (para "simular" al usuario), el programa **se cuelga**: `while (true)` vuelve siempre al principio, `opcion` vuelve a ser 11, nunca es 0 y nunca está fuera de rango. No hay salida posible. Un ejercicio con `while (true)` **siempre** tiene que declarar de dónde sale el valor, o es un bucle infinito.

**Y ojo con este `int.Parse(Console.ReadLine())`:** es exactamente el bug del ejercicio 6 (C) y del `Program.cs:156`. Si el usuario escribe `abc` en lugar de un número, se lanza `FormatException` y el programa se cae. La versión segura es `int.TryParse`.

**La diferencia con el menú real (que hay que aclarar):** el `Program.cs` no usa `while (true)` sino `while (continuar)`, con `bool continuar = true` en la línea 8 y el `while` en la 10. La opción "0. Salir" no usa `break`: asigna `continuar = false` en la línea 107, y el `while` se da por terminated en la condición. La diferencia no es cosmética: con `while (continuar)` el `break` **no es necesario**, y si alguien dejara un `break` suelto se terminaría el menú antes de tiempo.

**Reglas:**

| Palabra clave | Efecto | Vuelve a |
|---|---|---|
| `continue` | Salta el resto de **esta vuelta** | La condición del ciclo |
| `break` | **Termina** el ciclo | La línea después del ciclo |
| `return` | Sale del **método** completo | La línea después del método |

En un `for`, el `continue` ejecuta **primero** el `i++` y después vuelve a evaluar la condición. En un `while`, vuelve directo a la condición. En un `do...while`, el `continue` salta **a la condición**, y el bloque se ejecuta al menos una vez.

---

## Ejercicio 12 — Nivel 2 — Polimorfismo: `virtual`, `override`, `base` y el `ToString`

**Enunciado:** ¿Qué imprime? ¿Qué método se ejecuta en cada línea y por qué?

```csharp
public abstract class Publicacion
{
    protected string _titulo;
    protected int _anio;

    public Publicacion(string titulo, int anio)
    {
        _titulo = titulo;
        _anio = anio;
    }

    public string Titulo => _titulo;
    public int Anio => _anio;

    public abstract string Tipo();                 // cada tipo define el suyo
    public virtual string Estado() => "En catálogo";
    public string Ficha() => $"{Tipo()} '{_titulo}' ({_anio}) - {Estado()}";

    public override string ToString() => Ficha();
}

public class Libro : Publicacion
{
    public int Paginas { get; }

    public Libro(string titulo, int anio, int paginas) : base(titulo, anio)
    {
        Paginas = paginas;
    }

    public override string Tipo() => "Libro";
}

public class Revista : Publicacion
{
    public int Numero { get; }

    public Revista(string titulo, int anio, int numero) : base(titulo, anio)
    {
        Numero = numero;
    }

    public override string Tipo() => "Revista";
    public override string Estado() => "Agotada";
}

var publicaciones = new List<Publicacion>
{
    new Revista("Time", 2020, 12),
    new Libro("Rayuela", 1963, 420),
    new Libro("Ficciones", 1944, 174)
};

foreach (var p in publicaciones)
{
    Console.WriteLine(p);                          // usa ToString()
}
```

**// --- RESPUESTA ---**

**Salida:**
```text
Revista 'Time' (2020) - Agotada
Libro 'Rayuela' (1963) - En catálogo
Libro 'Ficciones' (1944) - En catálogo
```

**Justificación — el mecanismo del despacho virtual:**

1. `ToString()` y `Ficha()` están escritos **una sola vez**, en `Publicacion`. El `ToString()` llama a `Ficha()`, que llama a `Tipo()` y a `Estado()`.
2. El compilador genera esas llamadas de forma "virtual": no se resuelven en compilación, se resuelven **en ejecución** contra el **tipo real** del objeto. Por eso el primer objeto imprime "Revista" y no "Publicacion", aunque la variable esté declarada como `Publicacion`.
3. `Revista.Estado()` sobrescribe (`override`) al `virtual` de la base → "Agotada". Los dos `Libro` **no** lo sobrescriben → usan el `virtual` de la base → "En catálogo".
4. `Tipo()` es `abstract`: **obligatorio**. Si `Libro` o `Revista` no lo implementaran, el compilador daría error CS0534 al declararlos.

**Los conceptos que hay que nombrar si el parcial los pide:**

| Concepto | Dónde aparece |
|---|---|
| **Tipo estático** vs **tipo dinámico** | `Publicacion` (estático) vs `Revista`/`Libro` (dinámico) |
| **Conversión implícita base ← derivada** | `new Libro(...)` guardado en un `List<Publicacion>` |
| **`base(...)`** | `Libro(…) : base(titulo, anio)` — el constructor base se ejecuta **primero** |
| **`override`** | `Tipo()` y `Estado()`: la versión de la derivada reemplaza a la de la base |
| **`base.Metodo()`** | Permite llamar a la implementación base desde la derivada |
| **`protected`** | `_titulo` y `_anio` son accesibles desde las clases derivadas |
| **`ToString()`** | Todos los objetos lo heredan de `object`; `WriteLine` lo invoca |

**Aplicado al proyecto:** en `Program.cs:242-246` hay un solo `WriteLine` con 4 campos y no hay jerarquía. Si mañana aparece `Revista` y `Audiolibro`, ese `WriteLine` se va a llenar de `if (libro is Revista r)`. La solución OO es exactamente esta: un `abstract class Publicacion` del que hereden `Libro` y `Revista`, y un **único** recorrido que imprima `p`. Menos código, menos `if`, y agregar un tipo nuevo no obliga a tocar el recorrido.

---

## Ejercicio 13 — Nivel 2 — LINQ encadenado sobre la lista de libros

**Enunciado:** ¿Qué imprime? Ordenar las operaciones y justificar cada valor.

```csharp
var libros = new List<Libro>
{
    new Libro { Id = 1, Titulo = "Cien años de soledad", AnioPublicacion = 1967, AutorId = 1, Activo = true },
    new Libro { Id = 2, Titulo = "Rayuela",             AnioPublicacion = 1963, AutorId = 2, Activo = true },
    new Libro { Id = 3, Titulo = "Pedro Páramo",       AnioPublicacion = 1955, AutorId = 3, Activo = false },
    new Libro { Id = 4, Titulo = "Ficciones",          AnioPublicacion = 1944, AutorId = 2, Activo = true }
};

var catalogo = libros
    .Where(l => l.Activo)
    .OrderBy(l => l.Titulo)
    .Select(l => new
    {
        Id = l.Id,
        Titulo = l.Titulo,
        Decada = (l.AnioPublicacion / 10) * 10
    })
    .ToList();

foreach (var item in catalogo)
    Console.WriteLine($"{item.Id} | {item.Titulo} | {item.Decada}");

var masReciente = libros.OrderByDescending(l => l.AnioPublicacion).FirstOrDefault();
Console.WriteLine(masReciente?.Titulo ?? "(no hay)");

var masAntiguo = libros.OrderBy(l => l.AnioPublicacion).FirstOrDefault();
Console.WriteLine(masAntiguo?.Titulo);

Console.WriteLine(libros.Count(l => l.Activo));
Console.WriteLine(libros.Any(l => l.AnioPublicacion > 1950));
Console.WriteLine(libros.Sum(l => l.AnioPublicacion));
Console.WriteLine(libros.Max(l => l.AnioPublicacion) - libros.Min(l => l.AnioPublicacion));

var porEstado = libros.GroupBy(l => l.Activo);
foreach (var grupo in porEstado)
    Console.WriteLine($"Activo={grupo.Key}: {grupo.Count()} libro(s)");
```

**// --- RESPUESTA ---**

**Salida:**
```text
1 | Cien años de soledad | 1960
4 | Ficciones | 1940
2 | Rayuela | 1960
Cien años de soledad
Ficciones
3
True
7829
23
Activo=True: 3 libro(s)
Activo=False: 1 libro(s)
```

**Justificación paso a paso:**

1. `Where(l => l.Activo)` deja 3 libros. **"Pedro Páramo"** queda afuera (tiene `Activo = false`: es el borrado lógico de `Program.cs:327`).
2. `OrderBy(l => l.Titulo)` ordena alfabéticamente: "Cien años de soledad", "Ficciones", "Rayuela".
3. `Select` crea **objetos anónimos** (sin clase declarada; el compilador genera una tipo interno con las propiedades `Id`, `Titulo` y `Decada`). La década usa **división entera**: `1967 / 10` = 196, `196 * 10` = **1960**; `1944 / 10` = 194 → **1940**; `1963 / 10` = 196 → **1960**.
4. `ToList()` **materializa** la cadena: sin esta llamada, la consulta se reevalúa en cada recorrido.
5. `OrderByDescending(...).FirstOrDefault()` → el de mayor año: **"Cien años de soledad"** (1967). El `?.` evita la excepción si la lista estuviera vacía, y el `??` pone el texto alternativo.
6. `OrderBy(...).FirstOrDefault()` → el **más antiguo**: "Ficciones" (1944).
7. `Count(l => l.Activo)` → **3**.
8. `Any(l => l.AnioPublicacion > 1950)` → devuelve `bool`: hay varios → **True**.
9. `Sum(l => l.AnioPublicacion)` → 1967 + 1963 + 1955 + 1944 = **7829** (sobre la lista **completa**: cada consulta es independiente del `Where` anterior).
10. `Max - Min` → 1967 − 1944 = **23** (los años entre el más nuevo y el más viejo).
11. `GroupBy(l => l.Activo)` agrupa por `true` y por `false`. En cada grupo, `grupo.Key` es el valor agrupado (el `bool`) y `grupo.Count()` la cantidad: **3 activos** y **1 inactivo**.

**Advertencia sobre esa última salida:** el **orden de los grupos no está garantizado** por la especificación de LINQ. En esta ejecución concreta salen `True` antes que `False` porque el primer elemento de la lista es activo, y la implementación agrupa en orden de primera aparición, pero eso es un **detalle de implementación**, no una garantía. Si el orden importa (en un informe, por ejemplo), hay que pedirlo explícitamente:

```csharp
foreach (var grupo in porEstado.OrderBy(g => g.Key))
    Console.WriteLine($"Activo={grupo.Key}: {grupo.Count()} libro(s)");
// Ahora sí, siempre: Activo=False primero, Activo=True después.
```

Lo mismo pasa con `SelectMany`, `Distinct` y `Union`: la salida es determinista solo si se termina en un `OrderBy`/`ThenBy`.

**Los errores que evalúa el parcial:**

- Poner `AsEnumerable()` antes de un `Where` sobre una entidad de EF Core: corta la traducción a SQL y filtra en memoria. En el proyecto real, `LibroRepository` **no** lleva `AsEnumerable()` en ningún lado, y por eso EF traduce todo a SQL.
- Confundir `First()` con `FirstOrDefault()`: `First()` lanza `InvalidOperationException` si la lista está vacía.
- Suponer que un `Where` "consume" la lista original: `libros` sigue intacta. Cada cadena LINQ es una consulta nueva e independiente.
- `OrderBy` sobre `Titulo` depende de la culture configurada: el orden de tildes, eñes y mayúsculas puede variar. Si importa, hay que pasar un `StringComparer` explícito.

---

## Ejercicio 14 — Nivel 2 — Recursividad aplicada a datos del proyecto

**Enunciado:** ¿Qué imprime? Explicar cómo funciona cada recursión y cuál es el problema de la versión recursiva.

```csharp
// Requiere: using System.Linq;
static int ContarVocales(string texto)
{
    if (string.IsNullOrEmpty(texto)) return 0;                 // caso base
    char c = char.ToLower(texto[0]);
    bool esVocal = "aeiou".Contains(c);
    return (esVocal ? 1 : 0) + ContarVocales(texto.Substring(1));
}

static int ContarVocalesEficiente(string texto, int indice)
{
    if (indice >= texto.Length) return 0;                      // caso base
    char c = char.ToLower(texto[indice]);
    return ("aeiou".Contains(c) ? 1 : 0) + ContarVocalesEficiente(texto, indice + 1);
}

static bool EsPalindromo(string texto)
{
    if (texto.Length < 2) return true;                          // caso base
    if (char.ToLower(texto[0]) != char.ToLower(texto[^1])) return false;
    return EsPalindromo(texto.Substring(1, texto.Length - 2));
}

static bool EsPalindromoNormalizado(string texto)
{
    var limpio = new string(texto
        .Where(char.IsLetterOrDigit)
        .Select(char.ToLower)
        .ToArray());
    return EsPalindromo(limpio);
}

static int SumaDigitos(int numero)
{
    if (numero == 0) return 0;
    return numero % 10 + SumaDigitos(numero / 10);
}

static long Factorial(int n)
{
    if (n <= 1) return 1;
    return n * Factorial(n - 1);
}

Console.WriteLine(ContarVocales("Rayuela"));
Console.WriteLine(ContarVocalesEficiente("Rayuela", 0));
Console.WriteLine(EsPalindromo("Anita lava la tina"));
Console.WriteLine(EsPalindromo("Anitalavalatina"));
Console.WriteLine(EsPalindromoNormalizado("Anita lava la tina"));
Console.WriteLine(EsPalindromo("Rayuela"));
Console.WriteLine(SumaDigitos(1963));
Console.WriteLine(Factorial(5));
Console.WriteLine(Factorial(21));
```

**// --- RESPUESTA ---**

**Salida:**
```text
4
4
False
True
True
False
19
120
-4249290049419214848
```

**Justificación:**

- `ContarVocales("Rayuela")`: R(**no**) a(sí) y(**no**) u(sí) e(sí) l(**no**) a(sí) → **4** vocales. La `y` cuenta porque el código solo mira si el carácter está en `"aeiou"`. El método va quitando la primera letra con `Substring(1)`: `"Rayuela"` → `"ayuela"` → … → `""` (caso base = 0).
- `ContarVocalesEficiente("Rayuela", 0)`: mismo resultado (**4**), pero avanza con un índice en lugar de copiar el string.
- `EsPalindromo("Anita lava la tina")` → **False**, y esa es la sorpresa del ejercicio. La frase "se lee igual al derecho y al revés" solo si **se ignoran los espacios**, y esta versión **no los ignora**: compara `'A'` con `'a'`, ok; luego `'n'` con `'n'`… hasta llegar a los espacios del medio, que no coinciden con letras. Falla ahí y devuelve `false`.
- `EsPalindromo("Anitalavalatina")` → **True**: sin espacios ni mayúsculas, sí es palíndromo.
- `EsPalindromoNormalizado("Anita lava la tina")` → **True**: primero **limpia** (deja solo letras y dígitos, en minúscula) y después aplica el mismo algoritmo. Esa es la forma correcta de comparar dos títulos.
- `EsPalindromo("Rayuela")`: `'R'` vs `'a'` son distintos → **False** en la primera llamada, sin seguir.
- `SumaDigitos(1963)`: 1963 % 10 = 3, recursiona con 196; 6, con 19; 9, con 1; 1, con 0 (caso base). 3+6+9+1 = **19**.
- `Factorial(5)` = 5 × 4 × 3 × 2 × 1 = **120**.
- `Factorial(21)` = 21!. El máximo de un `long` es ≈ 9,22 × 10¹⁸ y 21! ≈ 5,1 × 10¹⁹: no entra. Como los enteros usan contexto **unchecked** por defecto, el valor **da la vuelta** y aparece negativo. Dentro de `checked { Factorial(21) }` se lanzaría `OverflowException`.

**La estructura obligatoria de una función recursiva:**

1. **Caso base:** la condición que **detiene** la recursión. Sin él → `StackOverflowException`.
2. **Caso recursivo:** el llamado a sí misma con un problema **más chico** (un carácter menos, un número más chico, n−1).

**Problemas de la versión recursiva (hay que mencionarlos):**

| Versión | Problema |
|---|---|
| `ContarVocales` con `Substring(1)` | **O(n²)**: en cada paso copia el string entero. Con un título de 10.000 caracteres es lentísimo. |
| `ContarVocalesEficiente` con índice | **O(n)** y sin copias: es la versión correcta. |
| `Factorial` recursivo | 21 llamadas apiladas. Con `n = 100.000` la pila se agota. La versión iterativa no usa memoria. |
| `Fibonacci` recursivo | **Exponencial** (≈ 1,618ⁿ). Se resuelve con memoización o con un `for`. |

**Versiones iterativas equivalentes:**

```csharp
static int ContarVocalesIterativo(string texto)
{
    int total = 0;
    foreach (char c in texto.ToLower())
        if ("aeiou".Contains(c)) total++;
    return total;
}

static long FactorialIterativo(int n)
{
    long resultado = 1;
    for (int i = 2; i <= n; i++) resultado *= i;
    return resultado;
}
```

**Aplicado al proyecto:** `ContarVocales(Titulo)` serviría para un **buscador** de títulos dentro de `LibroRepository`; `SumaDigitos(AnioPublicacion)` sirve para clasificar o validar un año de publicación; `Factorial` no tiene sentido aquí, pero es el ejemplo canónico para explicar el desbordamiento de pila y el `checked`.

---

## Ejercicio 15 — Nivel 2 — Eventos y `delegate`: avisar cuando se da de alta un libro

**Enunciado:** ¿Qué imprime? ¿En qué orden se invocan los handlers? ¿Por qué `otraVariable` no es una copia?

```csharp
public class ServicioBiblioteca
{
    private readonly IGenericRepository<Libro> _repositorio;

    public event Action<Libro> LibroRegistrado;

    public int CantidadRegistrados { get; private set; }

    public ServicioBiblioteca(IGenericRepository<Libro> repositorio)
    {
        _repositorio = repositorio;
    }

    // Handlers con nombre: así se pueden dar de baja con -=
    public void Handler1(Libro l) => Console.WriteLine("Handler 1: " + l.Titulo);
    public void Handler2(Libro l) => Console.WriteLine("Handler 2: " + l.Titulo);
    public void Handler3(Libro l) => Console.WriteLine("Handler 3: " + l.Titulo);

    public void DarDeAlta(Libro libro)
    {
        _repositorio.Agregar(libro);
        CantidadRegistrados++;
        LibroRegistrado?.Invoke(libro);
    }

    // Legal SOLO porque está dentro de la clase que declara el evento
    public void LimpiarSuscriptores() => LibroRegistrado = null;
}

var repo = new GenericRepository<Libro>();
var servicio = new ServicioBiblioteca(repo);

servicio.LibroRegistrado += servicio.Handler1;
servicio.LibroRegistrado += servicio.Handler2;

var otraVariable = servicio;                      // misma instancia, otro nombre
otraVariable.LibroRegistrado += servicio.Handler3;

servicio.DarDeAlta(new Libro { Titulo = "Rayuela", AnioPublicacion = 1963 });

Console.WriteLine("---");

servicio.LimpiarSuscriptores();
servicio.DarDeAlta(new Libro { Titulo = "Ficciones", AnioPublicacion = 1944 });
Console.WriteLine(servicio.CantidadRegistrados);
```

**// --- RESPUESTA ---**

**Salida:**
```text
Handler 1: Rayuela
Handler 2: Rayuela
Handler 3: Rayuela
---
2
```

**Justificación:**

- `otraVariable = servicio` **no copia** nada: en C# las clases son **tipos referencia**, así que ambas variables apuntan a **la misma instancia** en el heap. `otraVariable.LibroRegistrado += ...` se suscribe al **mismo evento**. Por eso los tres handlers ven el mismo `Libro`.
- Un evento es un **delegate multicast**: por dentro guarda una **lista** de delegates, en el orden en que se fueron sumando. Al invocar, los recorre **en ese orden**: 1, 2, 3. No hay ninguna prioridad, ni "el último gana", ni orden alfabético.
- `LibroRegistrado?.Invoke(libro)` usa el **operador de nulidad condicional** `?.`: si la lista de suscriptores es `null` (evento recién creado, sin nadie suscrito), no invoca nada y **no** lanza `NullReferenceException`. Es **obligatorio** en eventos.
- `LimpiarSuscriptores()` vacía la lista. La segunda llamada a `DarDeAlta` **igual persiste el libro** y suma al contador: sacar los suscriptores no afecta la operación. Por eso al final se imprime `2` y ningún handler.
- Los handlers son métodos con **firma `void HandlerN(Libro)`**, compatibles con `Action<Libro>`. `+=` con un método de grupo convierte el método en delegate; `+=` con una lambda (`l => ...`) la convierte en un delegate anónimo.
- `CantidadRegistrados { get; private set; }`: propiedad **pública** para leer (todos ven el contador) y de **escritura privada** (solo la clase lo modifica). Es el patrón de encapsulamiento para propiedades que no deben cambiar desde afuera.

**La trampa de este ejercicio: no se puede limpiar un evento desde afuera.**

Si en lugar de llamar a `LimpiarSuscriptores()` se escribe directamente esto, **no compila**:

```csharp
servicio.LibroRegistrado = null;      // error CS0070
```

```text
error CS0070: The event 'ServicioBiblioteca.LibroRegistrado' can only appear on the
left hand side of += or -= (except when used from within the type 'ServicioBiblioteca')
```

Esa es toda la razón de ser de la palabra `event`: **encapsular el delegate**. Desde afuera se puede **sumar** (`+=`) y **restar** (`-=`), nada más. Si el campo fuera `public Action<Libro> LibroRegistrado;` (sin `event`), cualquiera podría hacer `= null`, y hasta **invocarlo**: `servicio.LibroRegistrado(l libro)`, que es un bug de seguridad clásico.

**Y la trampa hermana: `-=` con una lambda no desuscribe nada.**

```csharp
servicio.LibroRegistrado += l => Console.WriteLine("Handler 3: " + l.Titulo);   // se suscribe
...
servicio.LibroRegistrado -= l => Console.WriteLine("Handler 3: " + l.Titulo);   // NO lo desuscribe
```

Las dos lambdas son **objetos distintos**, aunque se vean idénticas. La igualdad de delegates (que es la que usa `-=`) compara el **método de destino** y la **lista de valores de sus argumentos**, no el texto escrito. Como son dos lambdas distintas, el compilador genera dos métodos distintos: los delegates no se consideran iguales y **nada se desuscribe**, así que el Handler 3 seguiría apareciendo.

Por eso los handlers tienen que tener **nombre** (o guardarse en una variable) para poder darse de baja:

```csharp
Action<Libro> h3 = l => Console.WriteLine("Handler 3: " + l.Titulo);
servicio.LibroRegistrado += h3;
servicio.LibroRegistrado -= h3;      // ahora sí funciona
```

**Conceptos evaluables:**

| Concepto | Dónde |
|---|---|
| `delegate` / `Action<T>` | La variable que guarda "a quién llamar" |
| Multicast | Varios suscriptores en la misma variable |
| `event` | Encapsula el `delegate`: nadie puede invocarlo ni vaciarlo desde afuera, solo suscribirse (`+=`) o darse de baja (`-=`) |
| `?.Invoke(...)` | Evita `NullReferenceException` si no hay suscriptores |
| Tipo referencia | `otraVariable` y `servicio` son el mismo objeto |
| Orden de invocación | Orden de suscripción |

**Aplicado al proyecto:** el `ServicioBiblioteca` recibe `IGenericRepository<Libro>` **por constructor** (inyección de dependencias: la clase no crea el repositorio con `new`, lo recibe). Eso permite probarlo pasando un repositorio falso, y es la diferencia entre código testeable y código acoplado.

---

## Ejercicio 16 — Nivel 3 — `null`, `??`, `?.`, `TryParse` y `Dictionary<string, T>`

**Enunciado:** ¿Qué imprime? Explicar cada caso donde el comportamiento "obvio" no es el que ocurre. **El bloque (C) es un bug real del `Program.cs` actual.**

```csharp
// A) Igualdad entre objetos y entre strings
var l1 = new Libro { Id = 1, Titulo = "Rayuela" };
var l2 = new Libro { Id = 1, Titulo = "Rayuela" };
Console.WriteLine(l1 == l2);
Console.WriteLine(l1.Titulo == l2.Titulo);

// B) Concatenación
Console.WriteLine("Libro: " + 1 + 2);
Console.WriteLine("Libro: " + (1 + 2));

// C) Console.ReadLine() puede devolver ""  (Program.cs:156)
string entrada = "";                        // simula: el usuario solo apretó Enter
try
{
    int anio = int.Parse(entrada);
    Console.WriteLine(anio);
}
catch (FormatException ex)
{
    Console.WriteLine("FormatException controlada");
}

// D) Operadores de nulidad
string titulo = null;
Console.WriteLine(titulo ?? "(sin título)");
Console.WriteLine(titulo?.Length);
Console.WriteLine(titulo?.Length ?? -1);
Console.WriteLine(string.IsNullOrWhiteSpace(titulo) ? "vacío" : "con datos");

// E) Dictionary de strings
var porCategoria = new Dictionary<string, int>();
porCategoria["Novela"] = 3;
porCategoria["novela"] = 5;
Console.WriteLine(porCategoria.Count);
Console.WriteLine(porCategoria["Novela"]);
Console.WriteLine(porCategoria.ContainsKey("Novela"));
Console.WriteLine(porCategoria.TryGetValue("Ensayo", out var cantidad));
Console.WriteLine(cantidad);

// F) Primera búsqueda sin resultados
var libros = new List<Libro>();
Console.WriteLine(libros.FirstOrDefault() == null);
Console.WriteLine(libros.FirstOrDefault()?.Titulo ?? "(no hay)");
Console.WriteLine(libros.Count == 0 ? "vacía" : "con datos");
```

**// --- RESPUESTA ---**

**Salida:**
```text
False
True
Libro: 12
Libro: 3
FormatException controlada
(sin título)

-1
vacío
2
3
True
False
0
True
(no hay)
vacía
```

**Justificación de las trampas:**

**A) Igualdad.**
- `l1 == l2` → **False**. El operador `==` de una clase sin sobrecarga compara **referencias**: dos objetos distintos en memoria.
- `l1.Titulo == l2.Titulo` → **True**, porque el `==` de `string` compara **valor** (contenido). **Esta es la clave del ejercicio: en C# las clases comparan por referencia y los `string` por valor.** Consecuencia directa: `if (libro1 == libro2)` sobre entidades de EF Core compara objetos, **no** si son el mismo libro. Para eso, `libro1.Id == libro2.Id`.

**B) Concatenación.** `"Libro: " + 1 + 2` se evalúa **de izquierda a derecha**: `"Libro: " + 1` = `"Libro: 1"`, y `"Libro: 1" + 2` = `"Libro: 12"`. Los paréntesis cambian el resultado: `"Libro: " + (1 + 2)` = `"Libro: 3"`.

**C) El bug real.** Hay que distinguir dos entradas distintas:

| Qué pasó | `Console.ReadLine()` devuelve | `int.Parse` lanza |
|---|---|---|
| El usuario apretó Enter | `""` | **`FormatException`** |
| Fin de flujo (archivo vacío, `Ctrl+Z`) | `null` | **`ArgumentNullException`** |

Las dos cortan el programa si no se controlan, y `catch (FormatException)` **solo** cubre la primera: con `entrada = null` la `ArgumentNullException` se escapa y la aplicación se cae igual. En el `Program.cs` real el caso habitual es el primero (el usuario apretó Enter sin escribir nada), y el patrón `int.Parse(Console.ReadLine())` aparece en las líneas **156**, **168**, **180**, **260**, **286**, **321** y **379**.

La corrección que cubre **los dos** casos de una sola vez es `int.TryParse`, que no lanza nada: devuelve `false` y deja `0` en el `out`. Y si se quiere mantener el `Parse`, hacen falta **los dos** `catch` (o uno solo con un filtro):

```csharp
try
{
    int anio = int.Parse(Console.ReadLine());
}
catch (FormatException)          // la entrada no es un número (incluye "")
{
    Console.WriteLine("Año inválido.");
}
catch (ArgumentNullException)    // no hubo entrada (fin de flujo)
{
    Console.WriteLine("No se recibió ninguna entrada.");
}
```

O, en una sola cláusula con un filtro de patrón:

```csharp
catch (Exception ex) when (ex is FormatException or ArgumentNullException)
{
    Console.WriteLine("Entrada inválida.");
}
```

Nunca `catch (Exception)` a secas: ahí entra todo, y el bugs se esconden.

**D) Operadores de nulidad.**
- `titulo ?? "(sin título)"` → el **operador de coalescencia**: si la izquierda es `null`, devuelve la derecha.
- `titulo?.Length` → **línea vacía**: el operador de acceso seguro `?.` devuelve `null` de tipo `int?`, y `WriteLine` imprime cadena vacía. **No** lanza `NullReferenceException`.
- `titulo?.Length ?? -1` → **-1**: el `?.` corta **toda** la cadena, así que el `??` se aplica al `int?` completo.
- El **operador ternario** `cond ? a : b` elige entre dos expresiones. `string.IsNullOrWhiteSpace` es la forma correcta de validar entrada de usuario: detecta `null`, `""` y `"   "` (solo espacios).

**E) `Dictionary<string, int>`.**
- `"Novela"` y `"novela"` son **claves distintas** (1) porque el comparador por defecto de `string` es **ordinal y sensible a mayúsculas**.
- `porCategoria["Novela"]` → **3**, y no 5: son claves diferentes.
- `ContainsKey("Novela")` → **True**.
- `TryGetValue("Ensayo", out var cantidad)` → devuelve **`False`** y deja `cantidad = 0` (el valor por defecto), **sin lanzar excepción**. En cambio `porCategoria["Ensayo"]` lanzaría `KeyNotFoundException`. **Por eso `TryGetValue` es el método seguro.**

**F) Búsquedas sin resultados.**
- `libros.FirstOrDefault() == null` → **True**: sobre una lista de clases devuelve `null`.
- `libros.FirstOrDefault()?.Titulo ?? "(no hay)"` → **`(no hay)`**: el `?.` corta la cadena y el `??` pone el texto.
- Sobre una lista de `int`, `FirstOrDefault()` devolvería `0` (el valor por defecto), **no** `null`. Esa es la diferencia entre `List<int>` y `List<Libro>`.
- El **operador ternario** `cond ? a : b` evalúa solo una de las dos ramas.

**Corrección definitiva para el `Program.cs` real:**

```csharp
int anio;
while (!int.TryParse(Console.ReadLine(), out anio) || anio < 1450)
{
    Console.Write("Año inválido. Ingrese un número: ");
}
```

```csharp
string titulo;
while (string.IsNullOrWhiteSpace(titulo = Console.ReadLine()))
{
    Console.Write("El título no puede estar vacío: ");
}
```

---

# CATEGORÍA C — INTERPRETACIÓN DE CÓDIGO

## Ejercicio 17 — Nivel 1 — Ejecución diferida: cuándo se consulta realmente la base

**Enunciado:** Explicar, sin ejecutar nada, qué hace cada método de `LibroRepository`, **cuándo** se ejecuta la consulta SQL y qué problema tiene cada uno.

```csharp
public class LibroRepository : GenericRepository<Libro>
{
    // Método 1 — el que está en LibroRepository.cs:10
    public List<Libro> ObtenerLibrosPorMasRecientes()
    {
        return _context.Libro
                       .OrderByDescending(l => l.AnioPublicacion)
                       .ToList();
    }

    // Método 2 — versión SIN ToList()
    public IEnumerable<Libro> ObtenerActivos()
    {
        return _context.Libro
                       .Where(l => l.Activo);
    }

    // Método 3 — versión CON ToList()
    public List<Libro> ObtenerActivosMaterializado()
    {
        return _context.Libro
                       .Where(l => l.Activo)
                       .ToList();
    }

    // Método 4 — el que está en LibroRepository.cs:32
    public Libro? ObtenerLibroPorId(int id)
    {
        return _context.Libro
                       .FirstOrDefault(l => l.Id == id);
    }

    // Método 5 — consulta de solo lectura
    public Libro? ObtenerLibroConAutor(int id)
    {
        return _context.Libro
                       .Include(l => l.Autor)
                       .AsNoTracking()
                       .FirstOrDefault(l => l.Id == id);
    }
}
```

**// --- RESPUESTA ---**

**Método 1 — `ObtenerLibrosPorMasRecientes()`: ejecuta siempre.**

- `_context.Libro` es un `DbSet<Libro>` que implementa **`IQueryable<Libro>`**. La cadena `OrderByDescending(...)` sigue siendo un `IQueryable`: todavía **no** se consultó nada.
- El `ToList()` es el **punto de corte de la ejecución diferida**: ahí EF Core traduce a SQL, manda la consulta a SQLite, lee las filas y arma los objetos `Libro` en memoria.
- Devuelve un `List<Libro>` **independiente** del contexto: se puede recorrer varias veces, después de cerrar el contexto, sin coste extra.
- **Ojo:** este método **no** tiene `AsNoTracking()`. Cada `Libro` queda **rastreado** en el Change Tracker. Con 5.000 libros, son 5.000 entidades vigiladas en memoria que nadie va a modificar.

**Método 2 — `ObtenerActivos()`: NO se ejecuta todavía.**

- Devuelve un `IEnumerable<Libro>` que en realidad **guarda la receta** (`WHERE Activo = 1`), no los datos.
- El SQL se genera y se manda **cuando alguien recorre la colección**: `foreach`, `ToList()`, `Count()`, `Any()`, etc.
- **Consecuencias:** (a) si el contexto ya fue cerrado antes del recorrido → **excepción**; (b) recorrerlo dos veces = **dos consultas**; (c) sin `ToList()`, el Change Tracker se llena en cada recorrido.
- Cuándo conviene: cuando se va a filtrar en memoria, para encadenar más operadores, o para exponer un `IQueryable` a un consumidor avanzado.

**Método 3 — `ObtenerActivosMaterializado()`: ejecuta una vez.**

- Igual que el método 2, pero con `ToList()`: una sola consulta, resultados en memoria, reutilizables.

**Método 4 — `ObtenerLibroPorId()`: `FirstOrDefault`, consulta única, 0 o 1 filas.**

- Traduce a `SELECT ... FROM Libro WHERE Id = @id LIMIT 1`. Devuelve `null` si no encuentra nada (de ahí el `Libro?` con anotaciones de nulabilidad: el compilador obliga a chequear el `null` antes de usar el resultado).
- Lanza la consulta **en el momento de la llamada** (no hay ejecución diferida: devuelve un objeto, no una colección).
- **Va rastreado**: el `Libro` queda vigilado. Si después el `Program.cs` hace `libro.Titulo = "x"; repo.Modificar(libro);`, el resultado es el esperado.

**Método 5 — `ObtenerLibroConAutor()`: consulta única con `JOIN`, sin rastreo.**

- `Include(l => l.Autor)` agrega el `JOIN` con la tabla `Autor`, así que `libro.Autor.Nombre` es seguro.
- `AsNoTracking()` evita el Change Tracker: es la opción correcta para **consultar** un libro y mostrarlo.
- `FirstOrDefault` sigue garantizando 0 o 1 fila.

**Los tres mensajes clave del ejercicio:**

1. **`IQueryable` = la receta; `IEnumerable`/`List` = la comida.** Solo el primero sabe traducirse a SQL.
2. **`ToList()` es donde se paga el costo.** Antes de esa llamada no se envió ni un solo comando a la base de datos.
3. **`AsNoTracking()` en las consultas, rastreo en las de modificación.** `ModificarLibro` y `EliminarLibro` del `Program.cs` **necesitan** el rastreo; `MostrarLibros` y `BuscarLibroPorId` **no** lo necesitan. Por eso `GenericRepository.ObtenerTodos()` y `ObtenerTodosCon()` sí lo tienen, y los seis métodos de `LibroRepository` no: es una inconsistencia real del proyecto.

**Consecuencia práctica:** si un consumidor recorre `ObtenerActivos()` y accede a `libro.Autor.Nombre`, sin `Include` y sin Lazy Loading, revienta con `NullReferenceException`. Con Lazy Loading serían 1 + N consultas.

---

## Ejercicio 18 — Nivel 2 — Patrón Strategy aplicado a la salida de la aplicación

**Enunciado:** Explicar qué hace este fragmento, qué patrón de diseño implementa y por qué es preferible a un `switch`.

```csharp
public interface IFormatoInforme
{
    string Nombre { get; }
    void Imprimir(IEnumerable<Libro> libros);
}

public class InformeConsola : IFormatoInforme
{
    public string Nombre => "Consola";

    public void Imprimir(IEnumerable<Libro> libros)
    {
        foreach (var libro in libros.OrderBy(l => l.Titulo))
            Console.WriteLine($"[{libro.Id}] {libro.Titulo} ({libro.AnioPublicacion})");
    }
}

public class InformeDetallado : IFormatoInforme
{
    private readonly bool _incluirInactivos;

    public InformeDetallado(bool incluirInactivos)
    {
        _incluirInactivos = incluirInactivos;
    }

    public string Nombre => "Detallado";

    public void Imprimir(IEnumerable<Libro> libros)
    {
        var filtrados = _incluirInactivos ? libros : libros.Where(l => l.Activo);

        foreach (var libro in filtrados.OrderByDescending(l => l.AnioPublicacion))
            Console.WriteLine($"#{libro.Id} | {libro.Titulo} | {libro.AnioPublicacion} | Activo={libro.Activo} | AutorId={libro.AutorId}");
    }
}

public class InformeCsv : IFormatoInforme
{
    public string Nombre => "CSV";

    public void Imprimir(IEnumerable<Libro> libros)
    {
        Console.WriteLine("Id;Titulo;Anio");

        foreach (var libro in libros)
            Console.WriteLine($"{libro.Id};{libro.Titulo};{libro.AnioPublicacion}");
    }
}

public class GeneradorInformes
{
    private readonly List<IFormatoInforme> _formatos = new List<IFormatoInforme>();

    public void AgregarFormato(IFormatoInforme formato) => _formatos.Add(formato);

    public void Generar(IEnumerable<Libro> libros)
    {
        foreach (var formato in _formatos)
        {
            Console.WriteLine($"=== Informe {formato.Nombre} ===");
            formato.Imprimir(libros);
        }
    }
}

// --- Uso ---
var repo = new LibroRepository();
var libros = repo.ObtenerTodosCon("Autor");

var generador = new GeneradorInformes();
generador.AgregarFormato(new InformeConsola());
generador.AgregarFormato(new InformeDetallado(false));
generador.AgregarFormato(new InformeCsv());
generador.Generar(libros);
```

**// --- RESPUESTA ---**

**Qué hace, paso a paso:**

1. `IFormatoInforme` define el **contrato mínimo** de un formato: un nombre y la acción de imprimir. La interfaz **declara** los miembros, **no los implementa**.
2. `InformeConsola`, `InformeDetallado` e `InformeCsv` son **implementaciones concretas**, cada una con su propia regla: una ordena alfabéticamente; otra ordena por año descendente y filtra los inactivos según un parámetro; otra genera un CSV con `;` como separador.
3. `InformeDetallado` recibe `bool incluirInactivos` **por constructor**: es un parámetro de configuración, y el objeto queda listo para usarse sin reconfigurarlo después.
4. `GeneradorInformes` **depende de la abstracción** (`List<IFormatoInforme>`), nunca de las implementaciones. Al llamar a `formato.Imprimir(libros)`, el compilador **no sabe** cuál de las tres clases hay en cada posición: en **tiempo de ejecución** se ejecuta la versión correcta. Eso es **polimorfismo** + **desacoplamiento**.
5. `libros.Where(l => l.Activo)` y `libros.OrderBy(...)` se ejecutan **dentro de cada formato**, en memoria, sobre el `List<Libro>` que ya fue materializado. Ojo: si `libros` fuera un `IQueryable`, el `Where` viajaría a la base de datos; como es un `List`, se filtra en memoria.

**Patrón implementado: Strategy (Estrategia).** Tres algoritmos intercambiables para el mismo problema ("presentar la lista de libros").

**¿Por qué es mejor que un `switch`?**

| Aspecto | Con `switch` | Con Strategy |
|---|---|---|
| Agregar un formato nuevo | Hay que **modificar** el `switch` y recompilar | Se escribe `class InformeJson : IFormatoInforme` y se pasa a `AgregarFormato` |
| Principio Open/Closed | **Roto**: abierto a modificación | **Respetado**: abierto a extensión, cerrado a modificación |
| Riesgo | Olvidarse de un `case` = error en runtime | Imposible: no hay un `switch` que actualizar |
| Cantidad de condicionales | Un `if`/`switch` por formato | **Cero** |
| Ubicación de la lógica | Mezclada en el `switch` | Cada formato en su propia clase |

**Los tres principios de POO que hay que citar:**

- **Responsabilidad Única:** cada formato tiene una sola razón para cambiar.
- **Abierto/Cerrado (OCP):** se extiende sin modificar el `GeneradorInformes`.
- **Inversión de Dependencias (DIP):** el generador depende de la **interfaz** `IFormatoInforme`, no de `InformeConsola`.

**Frase evaluable:** "El `GeneradorInformes` **no necesita saber** si el formato es consola, CSV o detallado. Eso es el patrón Strategy: la interfaz define el punto de extensión."

**Variante que separa responsabilidades aún más:** que cada formato exponga `IEnumerable<Libro> Aplicar(IEnumerable<Libro> libros)` en vez de `Imprimir`: el generador solo llamaría a `Aplicar` y después a `Mostrar`, separando **qué** se filtra de **cómo** se muestra.

---

## Ejercicio 19 — Nivel 2 — Lectura de relaciones: `Include`, `ThenInclude` y el límite del `Include` por `string`

**Enunciado:** Explicar qué devuelve este código, cuántas consultas SQL se ejecutan, y qué pasa si además se pide la categoría.

```csharp
// AccesoDatos/Repositories/GenericRepository.cs:30 (real)
public List<T> ObtenerTodosCon(string propiedadRelacionada)
{
    return _context.Set<T>()
                   .Include(propiedadRelacionada)
                   .AsNoTracking()
                   .ToList();
}

// Program.cs:232 (real)
var libros = libroRepository.ObtenerTodosCon("Autor");

foreach (var libro in libros)
    Console.WriteLine($"{libro.Titulo} - {libro.Autor.Nombre}");     // OK

foreach (var libro in libros)
    Console.WriteLine($"{libro.Titulo} - {libro.Categoria.Nombre}");  // <-- ¿y esto?
```

**// --- RESPUESTA ---**

**Qué devuelve `ObtenerTodosCon("Autor")`:** una lista de todos los libros con **`Autor` cargado** (mediante un `LEFT JOIN` con la tabla `Autor`) y **sin rastrear** (`AsNoTracking`), porque la idea es mostrar, no modificar.

**Cuántas consultas SQL:** **una**. `Include("Autor")` agrega el `JOIN` a la consulta original, así que SQLite recibe un solo `SELECT` con las dos tablas.

**¿Y la línea de `Categoria`?** **Revienta:**

```text
System.NullReferenceException: Object reference not set to an instance of an object.
```

`ObtenerTodosCon("Autor")` cargó **solo** el autor. La propiedad `Libro.Categoria` sigue siendo `null` porque nadie pidió información de la tabla `Categoria`. `Include` no es "cargar todas las relaciones": hay que **nombrar** cada una.

**Las tres formas de corregirlo:**

```csharp
// Opción 1 — la más simple: dos Include en la misma consulta
var libros = _context.Libro
                      .Include(l => l.Autor)
                      .Include(l => l.Categoria)
                      .AsNoTracking()
                      .ToList();

// Opción 2 — dos llamadas al repositorio genérico
var conAutor = repo.ObtenerTodosCon("Autor");
var conCategoria = repo.ObtenerTodosCon("Categoria");

// Opción 3 — el modelo invertido: partir de Autor, que SÍ tiene la colección
var autores = _context.Autor
                        .Include(a => a.Libros)              // colección uno-a-muchos
                            .ThenInclude(l => l.Categoria)   // divesca un nivel más
                        .AsNoTracking()
                        .ToList();
```

**Análisis de la opción 3, cláusula por cláusula:**

| Cláusula | Función |
|---|---|
| `_context.Autor` | Entidad raíz de la consulta. |
| `AsNoTracking()` | Solo lectura: sin Change Tracker, menos memoria, sin riesgo de guardar cambios por accidente. |
| `Include(a => a.Libros)` | Carga la relación **de colección** (uno-a-muchos) con `LEFT JOIN`. |
| `.ThenInclude(l => l.Categoria)` | Se **encadena** al `Include` anterior: divesca un nivel (cada libro, con su categoría). Sin esto, `l.Categoria` sería `null`. |
| `ToList()` | Ejecuta y materializa. |

**La diferencia entre `Include` y `ThenInclude`:**

- `Include` carga el **primer** nivel de relación desde la entidad raíz. Devuelve un `IIncludable<T, TProperty>`.
- `ThenInclude` **continúa** un `Include` anterior, un nivel más adentro. Funciona **tanto con colecciones como con referencias**: no está reservado a las relaciones uno-a-muchos.
  - Tras una **colección**: `.Include(a => a.Libros).ThenInclude(l => l.Categoria)`.
  - Tras una **referencia**: `.Include(p => p.Pais).ThenInclude(pa => pa.Continente)`.
- La única **regla** es de encadenamiento: `ThenInclude` tiene que ir pegado al `Include` (o al `ThenInclude`) anterior, porque se aplica sobre el `IIncludable` que ese devuelve. Encerrarlo en un `Where` o en un `Select` rompe la cadena, y el compilador avisa con `CS1061`.
- **En este modelo concreto** (que es pobre en relaciones), el único camino posible es a través de la colección: `Autor` → `Libros` → `Categoria`, porque `Libro` solo tiene dos navegaciones de referencia y `Autor` no tiene ninguna de referencia hacia la que seguir. Por eso el ejemplo del ejercicio funciona. Si el modelo tuviera `Libro → Pedido → Cliente → Pais`, la cadena sería de referencias y también valdría.

**Lo que `ThenInclude` **no** hace:** no resuelve solo el nivel siguiente. Es un error común creer que `.Include(a => a.Libros)` ya trae la categoría de cada libro, porque "los libros vienen con su autor". No: el nivel cargado es exactamente el que se nombra, un nivel por `Include`/`ThenInclude`.

**Sobre la sobrecarga con `string` (`ObtenerTodosCon(string)`) — un análisis que vale el parcial:**

| | `Include("Autor")` (string) | `Include(l => l.Autor)` (lambda) |
|---|---|---|
| Validación en compilación | **No**: si se escribe `"Autorr"`, compila y falla en runtime | **Sí**: el compilador verifica el nombre de la propiedad |
| Renombrar la propiedad | Rompe en silencio (error en runtime) | Lo detecta el compilador |
| Navegar dos niveles | `Include("Libros.Categoria")` (con puntos, **desde la entidad raíz**) | `.Include(...).ThenInclude(...)` |
| Extensibilidad | Un método genérico sirve para cualquier relación, sin conocer el modelo | Cada relación necesita su método |

En un proyecto real, la versión con lambda es la recomendada. El método genérico por `string` es cómodo justamente porque `GenericRepository<T>` no conoce el modelo, pero paga el precio de no tener comprobación de tipos.

**El problema N+1, que es lo que más se evalúa:**

```csharp
// MAL: con Lazy Loading activado, 1 consulta por cada autor
foreach (var libro in repo.ObtenerTodos())
    Console.WriteLine(libro.Autor.Nombre);

// BIEN: 1 sola consulta
foreach (var libro in repo.ObtenerTodosCon("Autor"))
    Console.WriteLine(libro.Autor.Nombre);
```

Con 500 libros, la versión mala hace **501** consultas. En una base de datos real, con 10 ms de latencia por consulta, son 5 segundos. En una PC de escritorio con SQLite local "funciona", y por eso el error pasa desapercibido hasta que el proyecto crece.

**Sobre la navegación inversa:** `Autor.Libros` es la lista de navegación inversa (`List<Libro> Libros { get; set; } = new();` en `Models/Autor.cs`). Si se pide `_context.Autor` **sin** `Include(a => a.Libros)`, la lista queda **vacía** (no `null`, gracias al `= new()`), y el `foreach` no recorre nada: cero resultados, **sin ningún error visible**. Esa es la falla más silenciosa del proyecto.

---

## Ejercicio 20 — Nivel 2 — Por qué una clase abstracta puede tener una lista, e invariantes

**Enunciado:** En la clase abstracta `Vehiculo` se declara la lista de objetos `viajes`. ¿Por qué es posible declarar y utilizar esa lista en una clase abstracta? Analizá además el resto de decisiones de diseño.

```csharp
public class Viaje
{
    public int Id { get; set; }
    public string Destino { get; set; }
    public DateTime Fecha { get; set; }
    public int VehiculoId { get; set; }
    public Vehiculo Vehiculo { get; set; }
}

public abstract class Vehiculo
{
    private readonly List<Viaje> _viajes = new List<Viaje>();

    public string Patente { get; }
    public int Capacidad { get; }
    public IReadOnlyList<Viaje> Viajes => _viajes;

    protected Vehiculo(string patente, int capacidad)
    {
        Patente = patente;
        Capacidad = capacidad;
    }

    public void RegistrarViaje(Viaje viaje)
    {
        if (_viajes.Count >= Capacidad)
            throw new InvalidOperationException("No hay lugares disponibles.");

        _viajes.Add(viaje);
    }

    public bool CancelarViaje(int idViaje)
    {
        var viaje = _viajes.FirstOrDefault(v => v.Id == idViaje);

        if (viaje == null)
            return false;

        _viajes.Remove(viaje);
        return true;
    }

    public abstract decimal CalcularPeaje();
}

public class Camion : Vehiculo
{
    public Camion(string patente, int capacidad, double kilometraje)
        : base(patente, capacidad)
    {
        Kilometraje = kilometraje;
    }

    public double Kilometraje { get; }
    public bool TieneCabina { get; init; } = true;

    public override decimal CalcularPeaje()
    {
        return (decimal)Kilometraje * 0.05m * 0.9m;
    }
}
```

**// --- RESPUESTA ---**

**¿Por qué se puede declarar una lista en una clase abstracta?**

Porque **`abstract` no significa "incompleto"**. Significa exactamente dos cosas:

1. **No se puede instanciar directamente** (`new Vehiculo()` → error CS0144).
2. **Puede declarar miembros abstractos** que las clases derivadas **están obligadas** a implementar (`CalcularPeaje()`).

Todo lo demás —campos, propiedades, listas, constructores, métodos con cuerpo, `static`, constantes— es **código común totalmente válido** en una clase abstracta. Y el recorrido de `_viajes` es código común, porque **todos** los vehículos (camiones, ómnibus, colectivos) tienen viajes.

**El beneficio concreto:** la lista se declara **una sola vez** y se hereda. Si se declarara en cada subclase, habría que duplicarla N veces, y desde un `List<Vehiculo>` sería imposible recorrer los viajes de cualquiera de ellos, porque el compilador solo vería la lista de la subclase concreta.

**Análisis de cada decisión de diseño:**

| Elemento | Justificación |
|---|---|
| `private readonly List<Viaje> _viajes` | `private`: detalle interno, las derivadas no lo tocan. `readonly`: la **referencia** no se puede reasignar (`_viajes = new List<Viaje>()` daría CS0191), pero el contenido sí se puede agregar. |
| `public IReadOnlyList<Viaje> Viajes => _viajes;` | **Encapsulamiento**: se puede **leer** la lista, no **modificarla**. Nadie desde afuera puede hacer `vehiculo.Viajes.Add(...)` ni `.Clear()`. La única puerta de entrada es `RegistrarViaje`. |
| Devolver `IReadOnlyList<T>` y no `List<T>` | Devolver `List<T>` expondría el objeto real y permitiría `Add`, `RemoveAt`, `Sort` y `Clear` desde afuera, rompiendo la invariante. |
| `RegistrarViaje` con `if` + `throw` | **Invariante de la clase**: la lista nunca puede tener más elementos que `Capacidad`. Se protege en **un único punto**, que es el diseño defensivo correcto. |
| `CancelarViaje` con `FirstOrDefault` y retorno `bool` | No lanza excepción si el viaje no existe: informa el resultado. Robusto frente a entrada del usuario. |
| `abstract decimal CalcularPeaje()` | Obliga a que cada vehículo defina su tarifa. Si fuera `virtual`, un transporte podría no implementarla y usar una tarifa por defecto incorrecta. |
| `protected Vehiculo(...)` | Solo las derivadas pueden invocar el constructor base. Y como `Patente` y `Capacidad` son `{ get; }`, **nadie** puede cambiarlas después: son inmutables. |
| `: base(patente, capacidad)` | Obligatorio llamar al constructor base **antes** de usar los miembros de instancia del propio objeto. |
| `init` en `TieneCabina` | `init` deja asignar el valor **solo durante la inicialización**: en el inicializador de objetos (`new Camion { TieneCabina = false }`) o dentro del propio constructor de la clase. **Después** es de solo lectura, y asignarlo da **error CS8852**. Es la forma moderna de propiedades inmutables. |
| `(decimal)Kilometraje * 0.05m * 0.9m` | `double * decimal` **no** es una operación válida (error CS0019). Se convierte primero a `decimal` para calcular el peaje con precisión decimal. |

**Consecuencia del `throw`:** si se llama a `RegistrarViaje` sin control, el programa se corta con `InvalidOperationException`. La versión robusta es capturarla:

```csharp
try
{
    vehiculo.RegistrarViaje(viaje);
    Console.WriteLine("Viaje registrado.");
}
catch (InvalidOperationException ex)
{
    Console.WriteLine(ex.Message);      // "No hay lugares disponibles."
}
```

**Aplicado al proyecto:** el mismo razonamiento aplica a `Models/Autor.cs:11`:

```csharp
public List<Libro> Libros { get; set; } = new();
```

- El `= new()` evita el `NullReferenceException` en el `foreach` antes de que EF Core cargue nada.
- Pero deja expuesta la lista: cualquiera puede hacer `autor.Libros = null;` y romper el `foreach`. La versión encapsulada sería:

```csharp
private readonly List<Libro> _libros = new();
public IReadOnlyList<Libro> Libros => _libros;
```

**Pregunta habitual del parcial:** ¿qué pasaría si `Vehiculo` **no** fuera abstracta y no implementara `CalcularPeaje()`? → **error CS0534**: "no implementa el miembro abstracto heredado". Ese es el mecanismo por el cual el compilador garantiza que todo vehículo sepa calcular su peaje.

---

## Ejercicio 21 — Nivel 3 (integrador) — Un servicio de aplicación con validación, DI y excepciones

**Enunciado:** Este es el `AltaLibro` del `Program.cs` (líneas 150-196), refactorizado a una clase de servicio. Explicar qué hace, qué garantiza y qué defectos de diseño tiene.

```csharp
public record Resultado(bool Ok, string Mensaje, int? Id = null);

public class LibroService
{
    private readonly IGenericRepository<Libro> _libros;
    private readonly IGenericRepository<Autor> _autores;
    private readonly IGenericRepository<Categoria> _categorias;

    public LibroService(
        IGenericRepository<Libro> libros,
        IGenericRepository<Autor> autores,
        IGenericRepository<Categoria> categorias)
    {
        _libros = libros;
        _autores = autores;
        _categorias = categorias;
    }

    public Resultado Registrar(string titulo, int anio, int autorId, int categoriaId)
    {
        try
        {
            if (string.IsNullOrWhiteSpace(titulo))
                return new Resultado(false, "El título no puede estar vacío.");

            if (anio < 1450 || anio > DateTime.Now.Year)
                return new Resultado(false, "El año está fuera de rango.");

            if (_autores.ObtenerPorId(autorId) == null)
                return new Resultado(false, "El autor no existe.");

            if (_categorias.ObtenerPorId(categoriaId) == null)
                return new Resultado(false, "La categoría no existe.");

            var duplicado = _libros
                .ObtenerTodos()
                .FirstOrDefault(l => l.Titulo == titulo && l.AnioPublicacion == anio);

            if (duplicado != null)
                return new Resultado(false, "Ya existe un libro con ese título y año.");

            var libro = new Libro
            {
                Titulo = titulo.Trim(),
                AnioPublicacion = anio,
                AutorId = autorId,
                CategoriaId = categoriaId,
                Activo = true
            };

            _libros.Agregar(libro);

            return new Resultado(true, "Libro registrado correctamente.", libro.Id);
        }
        catch (DbUpdateException ex)
        {
            return new Resultado(false, "Error al guardar en la base de datos.");
        }
        catch (Exception ex)
        {
            return new Resultado(false, "Error inesperado.");
        }
        finally
        {
            Console.WriteLine("Fin del intento de alta.");
        }
    }
}
```

**// --- RESPUESTA ---**

**Qué hace, en orden:**

1. **Recibe sus dependencias por el constructor** (`IGenericRepository<Libro>`, `<Autor>`, `<Categoria>`) y las guarda en campos de solo lectura. **No crea ninguna con `new`**: espera que se las inyecten. Eso es **Inyección de Dependencias**, y permite probarlo con dobles de prueba sin tocar la base real.
2. **Valida antes de tocar la base:** título no vacío, año en rango (1450 → año actual, porque la imprenta de Gutenberg es de 1440), y que el autor y la categoría **existan** (consulta con `ObtenerPorId`, que usa `Find`).
3. **Detecta duplicados** con `ObtenerTodos()` + `FirstOrDefault` (filtro en memoria: aceptable con pocos datos, no con 100.000 libros).
4. **Construye la entidad** con un **inicializador de objeto** (`new Libro { ... }`), que es la forma idiomática en C#.
5. **Persiste** con `_libros.Agregar(libro)`. Recién después de esa llamada, `libro.Id` tiene el valor generado por la base de datos (clave autoincremental), y por eso el `Resultado` lo incluye.
6. **`catch` ordenados de más específico a más genérico:** `DbUpdateException` (clave foránea inexistente, restricción de unicidad violada, tabla inexistente) se captura **antes** que `Exception`. Si el orden fuera invertido, el `catch (Exception)` se comería todo y el específico nunca se ejecutaría.
7. **`finally`** se ejecuta **siempre**, haya éxito o excepción. Es el lugar correcto para liberar recursos, escribir en el log o limpiar estado. Si `finally` tuviera un `return`, descartaría el valor de retorno de los `catch`: por eso no debe.

**Los defectos que tiene que identificar el alumno:**

- **El `AltaLibro` real no captura ninguna excepción.** Las líneas 150-196 del `Program.cs` no tienen un solo `try`: si el usuario aprieta Enter en la línea 156, el `FormatException` sube hasta `Main`, no hay nadie que la capture y **la aplicación termina**. Lo mismo con un `DbUpdateException` si la clave foránea no existe. El manejo de errores no es opcional en un método que lee entrada del usuario.
- **Fuga de información (el error a evitar al agregar el `try`):** la tentación es `catch (Exception ex) { Console.WriteLine(ex.Message); }`, y eso está mal. `ex.Message` puede filtrar rutas de archivo, nombres de tabla o detalles del motor. Lo correcto: **registrar** el detalle en un log y **devolver** un mensaje genérico.
- **`catch (Exception)` es demasiado amplio:** captura desde `NullReferenceException` hasta `OutOfMemoryException`, incluidas las que **no tiene sentido recuperar**. Se debe capturar cada tipo concreto.
- **El `finally` escribe en consola:** un servicio de aplicación no debería imprimir. Eso es responsabilidad de la capa de presentación (`Program.cs`).
- **Rendimiento:** `ObtenerTodos().FirstOrDefault(...)` trae **todos** los libros de la base para buscar uno. Con 100.000 libros es lentísimo. Debería existir un método de consulta en el repositorio: `_libros.BuscarPorTituloYAnio(titulo, anio)`.
- **Sin transacción:** si `_libros.Agregar` funcionara y algo posterior fallara, no hay forma de deshacer. Aquí es tolerable (una sola operación), pero con un `Update` + `Insert` haría falta `BeginTransaction()` / `Commit()` / `Rollback()`.
- **Normalización posterior a la validación:** se valida `titulo` y recién ahí se hace `titulo.Trim()`. Mejor normalizar primero y validar después.
- **Los `catch` no registran nada:** el detalle se pierde. Con un `ILogger` inyectado, se escribiría el `ex` completo en el log y el usuario vería un mensaje genérico.
- **El servicio devuelve `int? Id` y la capa de presentación no lo usa:** el `Program.cs` real imprime `"Libro registrado correctamente."` sin decir qué ID quedó. Con el `Resultado` se puede informar "Libro registrado con ID 42".

**La respuesta que resume el parcial:**

> El **servicio de aplicación** valida la entrada, arma el agregado, **delega** la persistencia al repositorio y devuelve un resultado. El **repositorio** sabe *cómo* se guarda (EF Core, SQL, transacciones) pero no *qué* es un libro válido. La **entidad** (`Libro`) solo tiene datos. Eso es **separación de responsabilidades** y **el patrón Repositorio**; y como el servicio depende de `IGenericRepository<T>` y no de `GenericRepository<T>`, el código queda **desacoplado** y **testeable**: se puede pasar un repositorio falso que no toque la base.

**Versión mejorada del manejo de errores (para mostrar el criterio del docente):**

```csharp
public class LibroService
{
    private readonly IGenericRepository<Libro> _libros;
    private readonly ILogger<LibroService> _logger;      // inyectado, no creado con new

    // ... constructor y validaciones ...

    public Resultado Registrar(string titulo, int anio, int autorId, int categoriaId)
    {
        try
        {
            // ...Validar, armar el Libro y llamar a _libros.Agregar...

            return new Resultado(true, "Libro registrado correctamente.", libro.Id);
        }
        catch (DbUpdateException ex)
        {
            // El detalle va al log; el usuario recibe un mensaje genérico.
            _logger.LogError(ex, "Error de base de datos al registrar {Titulo}", titulo);
            return new Resultado(false, "No se pudo guardar el libro. Intentá de nuevo.");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error inesperado al registrar {Titulo}", titulo);
            return new Resultado(false, "Ocurrió un error inesperado.");
        }
    }
}
```

**Por qué `ILogger` y no `Console.WriteLine`:** permite escribir en archivo, en la consola, en un sistema remoto o en un visor de eventos **sin tocar el código del servicio**. Es el mismo principio de inversión de dependencias que se viene viendo en los ejercicios 18 y 24: el servicio depende de la abstracción, no de la tecnología concreta.

---

# CATEGORÍA D — OPCIÓN MÚLTIPLE

## Ejercicio 22 — Nivel 1 — Instanciación de los repositorios del proyecto

**Enunciado:** Estos son los tipos reales de `AccesoDatos/Repositories/`. ¿Cuáles de las líneas **compilan**? Elegí la opción correcta y justificá.

```csharp
// AccesoDatos/Repositories/ (real)
public interface IGenericRepository<T> where T : class { ... }
public class GenericRepository<T> : IGenericRepository<T> where T : class { ... }
public class LibroRepository : GenericRepository<Libro> { ... }

// Program.cs (real, líneas 4-6)
IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();
IGenericRepository<Categoria> categoriaRepository = new GenericRepository<Categoria>();
LibroRepository libroRepository = new LibroRepository();
```

```csharp
// Líneas a evaluar
// (1)
IGenericRepository<Libro> a = new GenericRepository<Libro>();
// (2)
IGenericRepository<Libro> b = new LibroRepository();
// (3)
GenericRepository<Libro> c = new LibroRepository();
// (4)
LibroRepository d = new LibroRepository();
// (5)
IGenericRepository<Libro> e = new GenericRepository<Categoria>();
// (6)
IGenericRepository<Libro> f = new IGenericRepository<Libro>();
// (7)
Console.WriteLine(b.ObtenerCantidadLibrosActivos());
```

**Opciones:**

**a)** Compilan (1), (2), (3) y (4); (5), (6) y (7) **no** compilan.
**b)** Compilan todas: la interfaz y la clase concreta son intercambiables.
**c)** Compilan solamente (1) y (4): una variable de tipo interfaz solo puede inicializarse con la implementación **exacta**.
**d)** (5) y (7) sí compilan, porque el tipo genérico `T` se resuelve en tiempo de ejecución.

**// --- RESPUESTA ---**

**Respuesta correcta: la opción a).**

| Línea | Estado | Razón |
|---|---|---|
| (1) | Compila | Se crea un `GenericRepository<Libro>` (que implementa `IGenericRepository<Libro>`) y se guarda en una variable de interfaz. Es lo que hace el `Program.cs` real con `Autor` y `Categoria`. |
| (2) | Compila | `LibroRepository` hereda de `GenericRepository<Libro>`, que a su vez implementa `IGenericRepository<Libro>`. La **interfaz se hereda dos niveles**: el compilador la encuentra en toda la cadena. |
| (3) | Compila | `LibroRepository` **es** un `GenericRepository<Libro>` (herencia directa): conversión implícita de clase derivada a clase base. |
| (4) | Compila | Se crea y se guarda en una variable de su propio tipo. Es la línea 6 real del `Program.cs`. |
| (5) | **No compila** | **error CS0029**: `GenericRepository<Categoria>` implementa `IGenericRepository<Categoria>`, **no** `IGenericRepository<Libro>`. Las interfaces genéricas son **invariantes**: `IGenericRepository<Categoria>` no es convertible a `IGenericRepository<Libro>`, aunque ambas usen `where T : class`. |
| (6) | **No compila** | **error CS0144**: no se puede crear una instancia de un **tipo de interfaz**. `new IGenericRepository<Libro>()` no tiene implementación detrás. Hay que escribir `new GenericRepository<Libro>()`. |
| (7) | **No compila** | **error CS1061**: `b` está declarada como `IGenericRepository<Libro>`, y esa interfaz **no** declara `ObtenerCantidadLibrosActivos()`: el método existe solo en `LibroRepository` (el **repositorio específico**). El compilador solo conoce los miembros de la interfaz. |

**Los cuatro conceptos que hay que nombrar:**

1. **Conversión implícita derivada → base** (regla estándar de asignación): toda clase se puede guardar en una variable de su base, y toda clase que implementa una interfaz se puede guardar en una variable de esa interfaz.
2. **Interfaz no instanciable** (`CS0144`).
3. **Invarianza de los genéricos:** `IGenericRepository<Libro>` y `IGenericRepository<Categoria>` son tipos **distintos** e incompatibles, aunque los dos usen el constraint `where T : class`.
4. **El tipo de la variable manda sobre los métodos disponibles:** por eso el `Program.cs` real declara `LibroRepository libroRepository = ...` y no `IGenericRepository<Libro>`: necesita los 6 métodos de LINQ específicos.

**Reflexión de diseño (si la pide el docente):** la buena práctica es depender de la abstracción (`IGenericRepository<Libro>`), pero entonces se pierden los métodos específicos. La solución profesional es **inyectar la interfaz del repositorio específico**:

```csharp
public interface ILibroRepository
{
    int ObtenerCantidadLibrosActivos();
    List<Libro> ObtenerLibrosPorMasRecientes();
    // ...los 6 métodos de LibroRepository
}

public class LibroRepository : GenericRepository<Libro>, ILibroRepository
{
    ...
}

// En Program.cs
ILibroRepository libroRepository = new LibroRepository();
Console.WriteLine(libroRepository.ObtenerCantidadLibrosActivos());   // compila
```

Se conserva la abstracción **y** se accede a los métodos propios: la interfaz propia actúa como contrato.

---

## Ejercicio 23 — Nivel 1 — Modificadores de acceso entre los dos proyectos

**Enunciado:** `AccesoDatos` y `AppConsola` son **dos ensamblados distintos** (dos `.csproj`, dos `.dll`). ¿Qué afirmaciones son correctas? Elegí la opción correcta.

```csharp
// ===== Proyecto AccesoDatos (AccesoDatos.dll) =====
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

        // Se agregan solo para este ejercicio:
        internal int CantidadEjemplares { get; set; }
        protected string CodigoInterno { get; set; }
        private decimal _costoInterno;
        private protected string Notas { get; set; }
        protected internal string Etiqueta { get; set; }

        public void ActualizarEjemplares(int cantidad)
        {
            CantidadEjemplares += cantidad;      // (1)
            Etiqueta = "ACT-" + Id;              // (2)
        }
    }
}
```

```csharp
// En AppConsola (AppConsola.dll): sin herencia, otro namespace
var libro = new Libro { Titulo = "Rayuela", AnioPublicacion = 1963, Activo = true };  // (3)
Console.WriteLine(libro.Titulo);                 // (4)
Console.WriteLine(libro.CantidadEjemplares);     // (5)
Console.WriteLine(libro.Etiqueta);               // (6)
Console.WriteLine(libro.CodigoInterno);          // (7)
Console.WriteLine(libro.Notas);                  // (8)
libro.Id = 99;                                  // (9)
```

**Opciones:**

**a)** Solo los miembros `public` son accesibles desde `AppConsola`. Las líneas (5), (6), (7) y (8) **no compilan**; (9) **sí**.
**b)** Todos los miembros son accesibles desde `AppConsola`, porque la clase `Libro` es `public`.
**c)** `internal`, `protected` y `private protected` también son accesibles, porque ambos proyectos están en la **misma solución** de Visual Studio.
**d)** La línea (7) es accesible, porque `CodigoInterno` tiene `get` y `set`, y eso lo hace "menos restrictivo que private".

**// --- RESPUESTA ---**

**Respuesta correcta: la opción a).**

| Línea | Estado | Razón |
|---|---|---|
| (3) | Compila | Se usan solo propiedades `public` (`Titulo`, `AnioPublicacion`, `Activo`). |
| (4) | Compila | `Titulo` es `public`. |
| (5) | **No compila** | `CantidadEjemplares` es `internal`: solo se ve **dentro del ensamblado `AccesoDatos`**. `AppConsola` es otro `.dll`. |
| (6) | **No compila** | `Etiqueta` es `protected internal`: desde otro ensamblado solo sería accesible desde una clase **derivada** de `Libro`, y en `AppConsola` no hay ninguna. |
| (7) | **No compila** | `CodigoInterno` es `protected`: solo la clase `Libro` y sus derivadas. Que tenga `get` y `set` no cambia nada: el modificador de acceso del **miembro** manda. |
| (8) | **No compila** | `Notas` es `private protected`: hacen falta **las dos** condiciones (ser derivada **y** estar en el mismo ensamblado). |
| (9) | **Compila** | `Id` es `public` con `set` público. Permite cambiar la clave primaria desde afuera (con EF Core eso es un error de diseño: la PK no debería tocarse a mano). |
| (1) y (2) | Compilan | Dentro de `Libro` **mismo** se accede a `internal`, a `protected internal` y a `private`. Un método de la clase ve todos sus miembros, incluidos los privados. |

**Matriz de visibilidad (la tabla que hay que saber de memoria):**

| Modificador | Se ve en… | ¿Otro ensamblado, sin herencia? | ¿Otro ensamblado, con herencia? |
|---|---|---|---|
| `private` | La misma clase | No | No |
| `private protected` | Derivadas **del mismo** ensamblado | No | No |
| `protected` | La misma clase y sus derivadas | No | **Sí** |
| `internal` | Cualquiera **del mismo** ensamblado | No | No |
| `protected internal` | La misma, derivadas, o cualquiera del mismo ensamblado | No | **Sí** |
| `public` | Cualquiera, en cualquier parte | **Sí** | **Sí** |

**Consecuencias prácticas para el proyecto (muy evaluables):**

- `Program.cs:4` declara `IGenericRepository<Autor> autorRepository = new GenericRepository<Autor>();`. Todo eso compila porque `IGenericRepository<T>`, `GenericRepository<T>`, `Libro`, `Autor` y `Categoria` son **`public`**. **Si alguna de esas clases fuera `internal`**, el `Program.cs` entero dejaría de compilar con **CS0122** ("inaccessible due to its protection level") en cada línea de uso.
- Al revés: `_context` es `protected` en `GenericRepository<T>`, y desde `AppConsola` **no** se puede tocar, aunque la clase sea `public`. Que un miembro sea `public` no da acceso a los `protected` o `private` de la misma clase.
- Consecuencia de diseño: el proyecto **no** expone el `DbContext` a la capa de presentación. `ApplicationDbContext` es `public` (lo necesita `GenericRepository`), pero `_context` es `protected` y `OnConfiguring` es `protected override`. La app de consola habla con la base **solo a través de los repositorios**, que es justamente lo que busca el patrón Repositorio.

---

## Ejercicio 24 — Nivel 2 — Interfaz versus clase abstracta en el patrón Repositorio

**Enunciado:** Se quiere unificar los repositorios del proyecto bajo una clase base abstracta. ¿Cuáles líneas **compilan**? Elegí la opción correcta.

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

public abstract class RepositorioBase<T> : IGenericRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;

    protected RepositorioBase(ApplicationDbContext context)
    {
        _context = context;
    }

    public abstract void Agregar(T entidad);          // cada repo define su alta

    public List<T> ObtenerTodos() => _context.Set<T>().AsNoTracking().ToList();

    public List<T> ObtenerTodosCon(string propiedad)
        => _context.Set<T>().Include(propiedad).AsNoTracking().ToList();

    public T ObtenerPorId(int id) => _context.Set<T>().Find(id);

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

public class LibroRepository : RepositorioBase<Libro>
{
    public LibroRepository(ApplicationDbContext context) : base(context) { }

    public override void Agregar(Libro entidad)
    {
        _context.Libro.Add(entidad);
        _context.SaveChanges();
    }

    public bool ExistenLibrosActivos() => _context.Libro.Any(l => l.Activo);
}

// Líneas a evaluar
// (A)
IGenericRepository<Libro> a = new LibroRepository(context);
// (B)
RepositorioBase<Libro> b = new LibroRepository(context);
// (C)
RepositorioBase<Libro> c = new RepositorioBase<Libro>(context);
// (D)
IGenericRepository<Libro> d = new RepositorioBase<Libro>(context);
// (E)
LibroRepository e = a;
// (F)
var f = new LibroRepository(context).ExistenLibrosActivos();
```

**Opciones:**

**a)** Compilan (A), (B) y (F); (C), (D) y (E) **no** compilan.
**b)** Compilan todas: una clase abstracta se puede instanciar si implementa la interfaz completa.
**c)** Compilan (A), (C) y (F); (B), (D) y (E) no compilan.
**d)** Compilan (A), (B) y (E); (C) y (D) sí, pero (F) no.

**// --- RESPUESTA ---**

**Respuesta correcta: la opción a).**

| Línea | Estado | Razón |
|---|---|---|
| (A) | Compila | `LibroRepository` implementa `IGenericRepository<Libro>` **por herencia**: `LibroRepository` → `RepositorioBase<T>` → `IGenericRepository<T>`. La interfaz se hereda **dos niveles** y el compilador la encuentra. |
| (B) | Compila | `LibroRepository` hereda directamente de `RepositorioBase<Libro>`: conversión implícita derivada → base. |
| (C) | **No compila** | **error CS0144**: `RepositorioBase<T>` es `abstract`, así que no se puede instanciar. Además, aunque no lo fuera, declara `Agregar` como `abstract` sin implementarlo. |
| (D) | **No compila** | **error CS0144**: misma causa. Que implemente la interfaz no alcanza: una clase abstracta **nunca** se instancia, ni por su propio tipo ni por la interfaz. |
| (E) | **No compila** | **error CS0266**: `a` está declarada como `IGenericRepository<Libro>`, así que el compilador solo conoce los 6 métodos de la interfaz, no el tipo `LibroRepository`. Necesita un cast: `var e = (LibroRepository)a;` o `LibroRepository e = a as LibroRepository;` (que devuelve `null` si no es del tipo). |
| (F) | Compila | Se crea el objeto y se invoca un método **concreto** de `LibroRepository` directamente sobre el resultado. Encadenar así es legal para métodos de instancia. |

**La diferencia clave entre interfaz y clase abstracta (lo que el parcial realmente pregunta):**

| | Interfaz | Clase abstracta |
|---|---|---|
| ¿Puede tener estado (campos, propiedades)? | No | Sí |
| ¿Puede tener implementación concreta? | Moderno: sí, con métodos `default` (C# 8) | Sí, con métodos `virtual` |
| ¿Herencia múltiple? | Sí: N interfaces | No: una sola clase base |
| Relación semántica | "puede hacer esto" (una **capacidad**) | "es un tipo de este, pero **incompleto**" |
| ¿Se puede instanciar? | Las implementaciones concretas sí | **Nunca** directamente |
| Acceso a `_context` | No puede (no tiene miembros) | Sí, si es `protected` |

**Por qué `IGenericRepository<T>` + `RepositorioBase<T>` y no solo `RepositorioBase<T>`:**

- La **interfaz** define el **contrato** (qué se puede hacer). Sirve para que el `LibroService` (ejercicio 21) dependa de la abstracción y pueda recibir cualquier implementación, incluso un **mock** en los tests.
- La **clase abstracta** define el **comportamiento común** (`Modificar`, `Eliminar`, `ObtenerTodos`, `ObtenerTodosCon`, `ObtenerPorId`), que **no** hay que reescribir en cada repositorio, y puede tener **estado** (`_context`, que es `protected` justamente para eso).
- `Agregar` queda `abstract` porque cada entidad puede necesitar una validación o una carga previa distinta al alta.

**Y lo que aporta el repositorio específico:** `LibroRepository` **hereda** el CRUD y **agrega** lo que le es propio (`ExistenLibrosActivos`, `ObtenerLibrosPorMasRecientes`, `ObtenerCantidadLibros`, `ObtenerLibroPorId`, `ObtenerLibrosOrdenadosPorTitulo`). Esa es la razón de ser del patrón Repositorio: **genérico para lo repetitivo, específico para lo que aporta valor**.

---

## Ejercicio 25 — Nivel 3 — `virtual`, `override`, `sealed`, `new` y accesibilidad inconsistente

**Enunciado:** Se quiere modelar la validación de las entidades del proyecto. ¿Cuál afirmación es correcta?

```csharp
// ===== Proyecto AccesoDatos =====
namespace AccesoDatos.Models
{
    public abstract class Entidad
    {
        public int Id { get; set; }

        public abstract void Validar();                            // obligatorio
        public virtual string Resumen() => $"#{Id}";               // opcional
        public override string ToString() => $"{GetType().Name}: {Resumen()}";
    }

    // --- Clase 1 ---
    public class Libro : Entidad
    {
        public override void Validar() { }
        public sealed override string Resumen() => $"Libro {Id}";  // (1)
    }

    // --- Clase 2 ---
    public class Novela : Libro
    {
        public override string Resumen() => $"Novela {Id}";        // (2)
    }

    // --- Clase 3 ---
    public class Ensayo : Entidad
    {
        public override void Validar() { }
        public string Resumen() => "Ensayo";                        // (3)
        public abstract void Aprobar();                             // (4)
    }

    // --- Clase 4 (otra clase en el mismo archivo) ---
    internal abstract class EntidadInterna
    {
        public abstract void Validar();
    }

    public class Categoria : EntidadInterna                        // (5)
    {
        public override void Validar() { }
    }
}
```

```csharp
// --- La línea que sí se puede ejecutar ---
var entidades = new List<Entidad> { new Libro { Id = 7 } };
foreach (var e in entidades)
    Console.WriteLine(e);
```

**Opciones:**

**a)** El código **no** compila y el primer error está en `Novela`.
**b)** Solo `Libro` y `Categoria` compilan; `Ensayo` y `Novela` producen errores.
**c)** Solo `Libro` y `Ensayo` compilan; `Novela` y `Categoria` producen errores.
**d)** Todas las clases compilan, porque todas heredan de una clase abstracta.

**// --- RESPUESTA ---**

**Respuesta correcta: la opción b).**

| Punto | Estado | Diagnóstico |
|---|---|---|
| `Libro` | **Compila** | Implementa el único miembro abstracto (`Validar`). `sealed override` es legal: **sella** la implementación de `Resumen()`, así que ninguna subclase puede cambiarla. |
| (2) `Novela` | **No compila** | **error CS0239**: no se puede sobrescribir el miembro heredado porque no está marcado `virtual`, `abstract` ni `override`. Como `Libro.Resumen()` es `sealed`, `Novela` está **obligada** a usar `"Libro 7"`. Si se quisiera permitir el cambio, `Libro` no debería declarar `sealed`. |
| (3) `Ensayo.Resumen()` | Compila, con **advertencia CS0114** | Se está **ocultando** (`hiding`), no sobrescribiendo: el `ToString()` heredado **seguirá llamando** a `Entidad.Resumen()` y devolverá `"Ensayo: #3"`, no lo que el alumno espera. La forma correcta es `public override string Resumen() => "Ensayo";`, o `public new string Resumen() => "Ensayo";` si de verdad quieren las dos versiones conviviendo. |
| (4) `Aprobar()` abstracto en `Ensayo` | **No compila** | **error CS0513**: `'Ensayo.Aprobar()' is abstract but it is contained in non-abstract type 'Ensayo'`. Debería ser `public abstract class Ensayo : Entidad`. |
| (5) `Categoria : EntidadInterna` | **No compila** | **error CS0060** (accesibilidad inconsistente): una clase `public` no puede heredar de una clase `internal`, menos accesible. Cualquier consumidor del ensamblado podría ver `Categoria` pero no podría nombrar a su clase base. Debe ser `public abstract class EntidadInterna` o `internal class Categoria`. |
| línea ejecutable | Compila | `new Libro { Id = 7 }` cumple el contrato de `Entidad`. |

**Salida de la línea ejecutable:**
```text
Libro: Libro 7
```

Desglose del `ToString()`: `GetType().Name` devuelve el **tipo real** del objeto (`Libro`), y `Resumen()` se resuelve en **tiempo de ejecución** contra la implementación más derivada. Por eso imprime `"Libro: Libro 7"` y no `"Entidad: #7"`: dentro de `ToString()` hay una llamada **virtual** disfrazada de llamada normal.

**Tabla resumen de las palabras clave (esto es lo que se pide):**

| Palabra | Significado | Qué pasa si falta |
|---|---|---|
| `abstract` | Sin cuerpo; las derivadas **están obligadas** a implementarlo | CS0534 en la derivada |
| `virtual` | Tiene cuerpo, pero **se puede** sobrescribir | — |
| `override` | Sobrescribe un `virtual`/`abstract` del padre. **Obligatorio** para que haya polimorfismo | CS0114 (advertencia): el método se oculta y se pierde el polimorfismo |
| `new` (en un método) | **Oculta** el miembro del padre sin sobrescribirlo. Los dos coexisten | Sin polimorfismo: el `ToString()` del padre sigue llamando al de la base |
| `sealed` (clase) | Impide la herencia | — |
| `sealed override` (método) | Permite sobrescribir, pero **ninguna** subclase puede seguir sobrescribiendo | CS0239 en la clase nieta |
| `override` de un `override` | Es implícito: sigue siendo virtual **salvo** que se marque `sealed` | — |
| `internal` base + `public` derivada | Accesibilidad inconsistente | CS0060 |

**Conclusión de diseño (la frase que suma puntos):** el `Program.cs` real no usa `ToString()` en ningún lado, así que `Console.WriteLine(libro)` imprimiría `AccesoDatos.Models.Libro`. Agregar un `override string ToString()` a `Libro` con el título y el año es la forma más barata de mejorar la salida de consola **de todo el proyecto de una sola vez**.

---

# SECCIÓN DE RESPUESTAS RÁPIDAS — CLAVE DE CORRECCIÓN

> Para imprimir en el parcial: ocultar esta tabla y usarla para la corrección oral o la corrección rápida.

| # | Cat. | Nivel | Clave |
|---|---|---|---|
| 1 | Corrección | 1 | **Compila**, pero `Libro.Autor` queda `null` sin `Include` → **`NullReferenceException`** (o N+1 con Lazy Loading). Corregir con `.Include(l => l.Autor)`. `ObtenerTodosCon("Autor")` es el que funciona; `ObtenerTodos()` no. |
| 2 | Corrección | 1 | **No compila**: **CS0122** (`private` no se ve en la clase hija). `private` → **`protected`**. Es el error más caro de un parcial de POO. |
| 3 | Corrección | 1 | **No compila**: **CS0120** (miembro de instancia en contexto `static`). Agregar los paréntesis: `contador.Consultas()`. Imprime `1`, `1`, `2`. `static` = compartido por toda la clase. |
| 4 | Corrección | 2 | **No compila**: **CS0535** × 4 — faltan `ObtenerTodosCon`, `ObtenerPorId`, `Modificar` y `Eliminar`. La interfaz se implementa **completa** o no se implementa. |
| 5 | Corrección | 2 | **No compila**: **CS1656** (reasignar la variable del `foreach`). Y además `struct` = **copia**: `Mover()` no persiste. Corregir reasignando el resultado (`posiciones[i] = p.Mover(5, 2)`), con `record struct` inmutable, o usando un arreglo con `ref` (el indexer de `List<T>` da **CS0206**). |
| 6 | Corrección | 2 | **No compilan** (B) **CS0266**, (D) **CS0266**, (E) **CS0030**, (F) **CS0019**. (A) `int/int` = división entera: `1957`. (C) `int.Parse("")` lanza `FormatException`; con `null` lanza `ArgumentNullException` y el `catch (FormatException)` no la cubre → `int.TryParse`. |
| 7 | Corrección | 3 | **No compila**: **CS0266** (`List<string>` → `List<Libro>`, porque `Select` devuelve `IEnumerable<string>` y `List<T>` es invariante). Además: falta `Include` (NRE/N+1), la proyección no va en el repositorio y `new ApplicationDbContext()` por método es antipatrón. Solución: `record LibroActivo` + `Select` al servidor. |
| 8 | Corrección | 3 | Compila: imprime `2` y `1`. El `Dictionary` compara **por referencia** (`Libro` no redefine `Equals`/`GetHashCode`); con `string` compara **por valor**. Usar `Id` como clave, o `record`, o sobreescribir `Equals` + `GetHashCode` (regla: si son iguales, mismo `GetHashCode`). |
| 9 | ¿Qué imprime? | 1 | `1. Constructor estático` / `2. Nueva conexión: Autores` / `2. Nueva conexión: Libros` / `[Autores] total=102` / `[Libros] total=102` / `102`. El constructor `static` corre una sola vez, antes de la primera instancia. |
| 10 | ¿Qué imprime? | 1 | `1957`, `1957.25`, `1`, `20825.4250`, `20825.42`, `0.30000000000000004`, `False`, `True`, `19`, `19.63`. `int/int` es división entera; `double` no es exacto; el `decimal` guarda además la **escala** (4 decimales) y `Math.Round` usa el **redondeo al par** (`ToEven`), por eso `20825.4250` → `20825.42` y no `20825.43`. |
| 11 | ¿Qué imprime? | 1 | Con la entrada declarada `3, 11, 99, 0`: `Seleccione una opción: Ejecutando opción 3` / `  item 1` / `  item 3` / `Seleccione una opción: Ejecutando opción 11` / `Seleccione una opción: Seleccione una opción: Fin` / `R` / `a` / `u` / `e`. `continue` salta la vuelta (el `i++` igual se ejecuta), `break` corta el ciclo **antes** del `WriteLine`, así que la letra que lo dispara no se imprime. Sin `ReadLine` el `while (true)` es **infinito**. |
| 12 | ¿Qué imprime? | 2 | `Revista 'Time' (2020) - Agotada` / `Libro 'Rayuela' (1963) - En catálogo` / `Libro 'Ficciones' (1944) - En catálogo`. Despacho virtual: manda el **tipo del objeto**, no el de la variable. `base(...)` ejecuta primero el constructor de la base. |
| 13 | ¿Qué imprime? | 2 | `1 \| Cien años de soledad \| 1960`, `4 \| Ficciones \| 1940`, `2 \| Rayuela \| 1960`, `Cien años de soledad`, `Ficciones`, `3`, `True`, `7829`, `23`, `Activo=True: 3 libro(s)`, `Activo=False: 1 libro(s)`. |
| 14 | ¿Qué imprime? | 2 | `4`, `4`, `False`, `True`, `True`, `False`, `19`, `120`, `-4249290049419214848` (overflow en contexto *unchecked*). "Anita lava la tina" **no** es palíndromo si no se limpian los espacios. Toda recursión necesita **caso base**; `Substring(1)` es O(n²). |
| 15 | ¿Qué imprime? | 2 | `Handler 1: Rayuela`, `Handler 2: Rayuela`, `Handler 3: Rayuela`, `---`, `2`. Las clases son tipos **referencia**: `otraVariable` y `servicio` son el mismo objeto. Orden = orden de suscripción. `?.Invoke` evita la NRE. Vaciar el evento desde afuera es **CS0070**; `-=` con una lambda nueva **no** desuscribe. |
| 16 | ¿Qué imprime? | 3 | `False`, `True`, `Libro: 12`, `Libro: 3`, `FormatException controlada`, `(sin título)`, (línea vacía), `-1`, `vacío`, `2`, `3`, `True`, `False`, `0`, `True`, `(no hay)`, `vacía`. **Clases comparan por referencia, `string` por valor**; `int.Parse("")` lanza `FormatException` y `int.Parse(null)` lanza `ArgumentNullException` (que el `catch (FormatException)` **no** cubre) → `TryParse`. |
| 17 | Interpretación | 1 | `IQueryable` = la **receta** (no hay SQL hasta que se recorre). `ToList()` = donde se paga el costo. `Find`/`FirstOrDefault` = consulta única. `AsNoTracking` solo en lectura: los 6 métodos de `LibroRepository` no lo tienen (inconsistencia real). |
| 18 | Interpretación | 2 | Patrón **Strategy**: el `GeneradorInformes` depende de `IFormatoInforme`, no de las implementaciones. Permite agregar formatos sin modificar nada (OCP + DIP + Responsabilidad Única). |
| 19 | Interpretación | 2 | `ObtenerTodosCon("Autor")` = **1 consulta** con `LEFT JOIN`. `Categoria` queda `null` → **NRE**. `ThenInclude` divesca un nivel más y funciona **tanto con colecciones como con referencias**; solo debe encadenarse sobre el `Include` anterior. El `Include(string)` no valida en compilación. |
| 20 | Interpretación | 2 | `abstract` no impide tener listas ni código: impide **instanciar**. `List<Viaje>` es código común a todos los vehículos. `private readonly` + `IReadOnlyList` encapsulan; `RegistrarViaje` protege la invariante de `Capacidad`. |
| 21 | Interpretación | 3 | Servicio con **DI** por constructor: valida → arma el `Libro` → persiste → devuelve `Resultado`. `catch` de más específico a más genérico; `finally` siempre corre. Fallas: el `AltaLibro` real **no captura nada** (el `FormatException` de la línea 156 mata la app), `catch (Exception)` demasiado amplio, `ObtenerTodos()` para buscar duplicados, sin transacción, escribe en consola, `ex.Message` filtraría información. |
| 22 | Opción múltiple | 1 | **Opción a)**. (1)-(4) compilan; (5) **CS0029** (genéricos invariantes), (6) **CS0144** (interfaz no instanciable), (7) **CS1061** (el método no está en la interfaz). |
| 23 | Opción múltiple | 1 | **Opción a)**. Desde otro ensamblado sin herencia solo se ve lo `public`. (5), (6), (7) y (8) fallan con **CS0122**; (9) compila. Las líneas (1) y (2) compilan: dentro de la clase se ve todo. |
| 24 | Opción múltiple | 2 | **Opción a)**. (A), (B) y (F) compilan; (C) y (D) **CS0144** (clase abstracta), (E) **CS0266** (de interfaz a clase concreta, requiere cast). |
| 25 | Opción múltiple | 3 | **Opción b)**. `Libro` OK (`sealed override` válido). (2) **CS0239**, (3) advertencia **CS0114**, (4) **CS0513**, (5) **CS0060**. Imprime `Libro: Libro 7`. |

---

## Criterios de corrección sugeridos

- **Cada ítem:** 2 puntos.
- **Justificación:** 1 punto por la explicación (no alcanza con decir "no compila": hay que nombrar la causa y, si se puede, el código de error).
- **Corrección propuesta:** 1 punto solo si el código corregido **compila de verdad**.
- **En "¿Qué imprime?":** 1 punto por la salida exacta y 1 punto por el mecanismo (polimorfismo, ejecución diferida, división entera, tipo valor vs referencia, etc.).
- **En los ítems integradores (nivel 3):** exigir que se identifiquen **al menos dos** defectos de diseño, no solo errores de compilación.
- **Ítem 16 (el bug del `int.Parse`):** es el error real del entregable. Si el alumno lo detecta sin ayuda, vale bonificación.

## Cómo probar los ejercicios

1. Descomprimir `U9EjercicioEntregable-main.zip` (el proyecto es una solución `.slnx` de .NET 10).
2. En `AppConsola/AppConsola/Program.cs`, reemplazar el contenido por el fragmento del ejercicio (los que se pueden ejecutar) y correr con `F5`.
3. Los ejercicios con `ApplicationDbContext` necesitan la base `C:\databases\BaseDatosEjercicios.db` y las migraciones aplicadas (`dotnet ef database update` desde la carpeta `AccesoDatos`).
4. Para los errores de compilación (1, 2, 4, 5, 6, 7, 22-25), no hace falta ejecutarlos: el objetivo es leer el diagnóstico que muestra el compilador en la lista de errores.
5. Verificar el estado del proyecto con `dotnet build` y leer la lista de errores. **Todos** los códigos citados en las respuestas se comprobaron contra el compilador de .NET 10 (Roslyn) y son los que emite:

| Código | Significado | Dónde aparece |
|---|---|---|
| CS0019 | Operador `*` entre `double` y `decimal` | Ej. 6 (F) |
| CS0029 | Sin conversión implícita de tipos | Ej. 22 (5) |
| CS0030 | No se puede convertir `string` a `int` con un cast | Ej. 6 (E) |
| CS0060 | Accesibilidad inconsistente: base `internal`, clase `public` | Ej. 25 (5) |
| CS0070 | El evento solo admite `+=` y `-=` desde afuera | Ej. 15 |
| CS0114 | El miembro **oculta** al heredado; falta `override` o `new` | Ej. 25 (3) |
| CS0120 | Se llama a un miembro de instancia en un contexto `static` | Ej. 3 |
| CS0122 | El miembro es inaccesible por su nivel de protección | Ej. 2 y 23 |
| CS0144 | No se puede instanciar una interfaz ni una clase abstracta | Ej. 22 (6) y 24 (C, D) |
| CS0206 | Un indexer que no devuelve `ref` no se puede usar con `ref` | Ej. 5 |
| CS0239 | No se puede sobrescribir un miembro `sealed` | Ej. 25 (2) |
| CS0266 | Falta conversión implícita (tipos distintos o sin cast) | Ej. 6 (B, D), 7 y 24 (E) |
| CS0513 | Miembro abstracto en una clase **no** abstracta | Ej. 25 (4) |
| CS0534 | No se implementa un miembro abstracto heredado | Ej. 12, 20 y 25 |
| CS0535 | No se implementa un miembro de la interfaz | Ej. 4 |
| CS1061 | El tipo no tiene ese miembro (no está en la interfaz) | Ej. 22 (7) |
| CS1656 | No se puede asignar a la variable de iteración del `foreach` | Ej. 5 |
