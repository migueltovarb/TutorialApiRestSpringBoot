# Tutorial: Creación de una API REST con Spring Boot
### Sistema de Gestión de Biblioteca — Diseño de Software · 4° Semestre

---

## Índice

1. [El Modelo Conceptual](#1-el-modelo-conceptual)
2. [Del Modelo Conceptual al Modelo por Capas](#2-del-modelo-conceptual-al-modelo-por-capas)
3. [El Modelo de Desarrollo](#3-el-modelo-de-desarrollo)
4. [Base de Datos: MongoDB Atlas](#4-base-de-datos-mongodb-atlas)
5. [Creando el Proyecto Spring Boot en IntelliJ IDEA](#5-creando-el-proyecto-spring-boot-en-intellij-idea)
6. [Implementando la Capa Model](#6-implementando-la-capa-model)
7. [Implementando la Capa Repository](#7-implementando-la-capa-repository)
8. [Implementando los DTOs](#8-implementando-los-dtos)
9. [Implementando la Capa Service](#9-implementando-la-capa-service)
10. [Implementando la Capa Controller](#10-implementando-la-capa-controller)
11. [Configurando la Conexión a MongoDB](#11-configurando-la-conexión-a-mongodb)
12. [Probando la API con Postman](#12-probando-la-api-con-postman)

---

## 1. El Modelo Conceptual

### ¿Qué es un modelo conceptual?

Un **modelo conceptual** es una representación abstracta del dominio del problema que estamos resolviendo. Imagínalo como un plano de arquitectura: muestra las "piezas" importantes del sistema, sus características y cómo se relacionan entre sí, **sin importar aún qué tecnología se usará para construirlo**.

En el diseño de software, el modelo conceptual se expresa habitualmente como un **Diagrama de Clases UML**, donde:

- Cada **clase** representa una entidad del mundo real (un libro, un usuario, un préstamo).
- Los **atributos** de cada clase describen sus características (un libro tiene ISBN, título, autor...).
- Las **relaciones** entre clases representan cómo se asocian esas entidades (un préstamo involucra a un usuario y a un ejemplar).

### El Modelo Conceptual del Sistema de Biblioteca

Nuestro sistema de biblioteca gestiona los siguientes conceptos del mundo real:

- **Usuario**: Persona que interactúa con el sistema. Puede ser Estudiante, Profesor o Bibliotecario.
- **Libro**: Obra intelectual identificada por su ISBN, título, autor, año y categoría.
- **Ejemplar**: Copia física de un libro que puede prestarse.
- **Préstamo**: Registro de que un usuario tomó prestado un ejemplar en una fecha determinada.

Observa el modelo conceptual del sistema:

![Modelo Conceptual del Sistema de Biblioteca](modelo_conceptual.png)

### ¿Qué nos dice el modelo conceptual?

Analicemos las entidades y sus relaciones:

| Entidad | Atributos Principales | Qué representa |
|---|---|---|
| `Usuario` | id, nombre, correo | Base para todos los tipos de usuario |
| `Estudiante` | codigoEstudiante, programa | Usuario que puede solicitar préstamos |
| `Profesor` | codigoProfesor, facultad | Usuario que puede solicitar préstamos |
| `Bibliotecario` | codigoEmpleado, turno | Usuario que registra préstamos y devoluciones |
| `Libro` | isbn, titulo, autor, anioPublicacion, categoria | La obra intelectual |
| `Ejemplar` | codigoEjemplar, estado, ubicacion | La copia física prestable |
| `Prestamo` | fechaPrestamo, fechaDevolucion, estado | El acto de prestar |

> 💡 **Punto clave**: El modelo conceptual responde al **"¿qué existe en el negocio?"**. No habla de tablas, bases de datos, endpoints ni código. Es el lenguaje común entre el desarrollador y el cliente.

---

## 2. Del Modelo Conceptual al Modelo por Capas

### ¿Por qué organizar el código en capas?

Imagina que estás construyendo una empresa. ¿Dejarías que el mensajero de correspondencia tomara decisiones financieras? ¿O que el contador atendiera a los clientes en la recepción? Claramente no: cada persona tiene un rol específico y se comunica solo con quien corresponde.

En el software pasa exactamente lo mismo. La **Arquitectura por Capas** organiza el código en niveles con responsabilidades claramente separadas. Cada capa:

- Tiene una **única responsabilidad**
- Se comunica **solo con la capa adyacente**
- Puede **modificarse independientemente** sin romper el resto del sistema


---

### ¿Qué es HTTP?

Antes de hablar de capas, necesitamos entender el "idioma" en que se comunica nuestra API con el mundo: **HTTP** (*HyperText Transfer Protocol*).

HTTP es el protocolo de comunicación que usa la web. Cada vez que abres una página, tu navegador envía una **petición HTTP** al servidor, y el servidor responde con datos. Nuestra API funciona exactamente igual, solo que en lugar de páginas HTML devuelve datos en formato **JSON**.

#### Los métodos HTTP

Una petición HTTP siempre tiene un **método** que indica la *intención* de quien la hace:

| Método | Intención | Analogía cotidiana |
|---|---|---|
| `GET` | Obtener información (solo lectura) | Consultar el catálogo de la biblioteca |
| `POST` | Crear algo nuevo | Registrar un libro nuevo |
| `PUT` | Reemplazar/actualizar algo existente | Corregir los datos de un libro |
| `DELETE` | Eliminar algo | Dar de baja un libro del sistema |

#### URLs como identificadores de recursos

En HTTP, cada "cosa" que maneja el sistema se identifica con una **URL**:

```
http://localhost:8080/api/libros          ← la colección de libros
http://localhost:8080/api/libros/64a8f2  ← un libro específico (por su ID)
http://localhost:8080/api/usuarios        ← la colección de usuarios
```

La combinación de **método + URL** define exactamente qué acción se realiza:

```
GET    /api/libros        → listar todos los libros
POST   /api/libros        → crear un libro nuevo
GET    /api/libros/64a8f2 → consultar el libro con ese ID
PUT    /api/libros/64a8f2 → actualizar ese libro
DELETE /api/libros/64a8f2 → eliminar ese libro
```

#### El cuerpo de la petición (body)

Cuando se crea o actualiza algo, los datos viajan en el **body** de la petición, en formato JSON:

```json
{
  "isbn": "978-0-13-468599-1",
  "titulo": "Clean Code",
  "autor": "Robert C. Martin",
  "anioPublicacion": 2008,
  "categoria": "Programación"
}
```

#### Códigos de respuesta HTTP

El servidor siempre responde con un **código numérico** que indica si la operación fue exitosa o no:

| Código | Nombre | Cuándo se usa |
|---|---|---|
| `200 OK` | Éxito | Consulta o actualización exitosa |
| `201 Created` | Creado | Recurso creado correctamente |
| `204 No Content` | Sin contenido | Eliminación exitosa (no hay nada que devolver) |
| `404 Not Found` | No encontrado | El recurso solicitado no existe |
| `500 Internal Server Error` | Error del servidor | Algo falló inesperadamente en el backend |

---

### ¿Qué es REST?

**REST** (*Representational State Transfer*) es un estilo de arquitectura para diseñar APIs sobre HTTP. No es un protocolo ni una librería: es un conjunto de principios que, si se siguen, producen APIs predecibles, simples y fáciles de consumir.

Los principios clave de REST son:

| Principio | Qué significa |
|---|---|
| **Recursos por URL** | Cada entidad del sistema (libro, usuario) tiene su propia URL |
| **Acciones con verbos HTTP** | No se inventa URLs como `/crearLibro`; se usa `POST /libros` |
| **Sin estado (stateless)** | Cada petición es independiente; el servidor no recuerda peticiones anteriores |
| **Respuestas en JSON** | El formato estándar para intercambiar datos en APIs modernas |

Una API que sigue estos principios se llama **API REST** o **API RESTful**.

---

### ¿Qué es una API?

**API** significa *Application Programming Interface* (Interfaz de Programación de Aplicaciones). Es un **contrato** que define cómo interactuar con un sistema desde afuera, sin necesidad de conocer su implementación interna.

Piénsalo como el menú de un restaurante: tú ves las opciones disponibles (los endpoints), haces un pedido (la petición), y recibes lo que pediste (la respuesta JSON). No necesitas saber cómo funciona la cocina (el código interno).

En nuestro caso, la API REST de la biblioteca le dirá al mundo:
> *"Si me mandas un `POST /api/libros` con estos datos, te guardaré el libro y te devolveré el ID generado."*

---

### ¿Por qué construimos este proyecto como una API REST?

Podríamos haber construido una aplicación web tradicional que devuelva páginas HTML. Pero usamos una API REST porque:

| Razón | Descripción |
|---|---|
| **Separación cliente/servidor** | El frontend (web, móvil) y el backend son independientes; cada uno puede cambiar sin afectar al otro |
| **Estándar universal** | Cualquier tecnología (JavaScript, Python, Kotlin, Swift) puede consumir una API REST |
| **Escalabilidad** | El backend puede desplegarse en múltiples servidores sin cambiar el contrato de la API |
| **Base del desarrollo moderno** | El 90 % de los sistemas actuales exponen sus datos como APIs REST |
| **Aprendizaje transferible** | Lo que aprendes aquí aplica directamente a proyectos reales en la industria |

El flujo completo de una petición en nuestro sistema será:

```
┌─────────────┐   HTTP Request (JSON)   ┌──────────────────────┐   Consulta   ┌───────────────────┐
│   Cliente   │ ─────────────────────▶  │  API REST            │ ───────────▶ │  MongoDB Atlas    │
│  (Postman,  │                         │  (Spring Boot)       │              │  (base de datos   │
│  frontend)  │ ◀─────────────────────  │  localhost:8080      │ ◀─────────── │   en la nube)     │
└─────────────┘   HTTP Response (JSON)  └──────────────────────┘   Resultado  └───────────────────┘
```

Ahora que entiendes el "idioma" HTTP y el estilo REST, tiene mucho más sentido hablar de cómo organizamos el código internamente.



### Las 5 capas de nuestra API REST

```
┌─────────────────────────────────────────────────┐
│              CAPA CONTROLLER                    │
│  Recibe peticiones HTTP · Define los endpoints  │
│  GET /api/libros · POST /api/libros · etc.      │
├─────────────────────────────────────────────────┤
│                CAPA DTO                         │
│  Define la "forma" de los datos que            │
│  entran y salen del API (sin exponer BD)       │
├─────────────────────────────────────────────────┤
│               CAPA SERVICE                      │
│  Contiene la lógica de negocio                 │
│  Las "reglas" del sistema viven aquí           │
├─────────────────────────────────────────────────┤
│             CAPA REPOSITORY                     │
│  Habla con la base de datos                    │
│  Guarda, busca, actualiza, elimina documentos  │
├─────────────────────────────────────────────────┤
│               CAPA MODEL                        │
│  Representa los documentos de MongoDB          │
│  Mapeo directo Clase Java ↔ Documento MongoDB  │
└─────────────────────────────────────────────────┘
                       ↕
              [ Base de Datos MongoDB ]
```

### Descripción detallada de cada capa

---

#### 🟦 Capa Model — Las entidades

**¿Qué es?**
Representa las entidades del sistema como objetos Java que mapean directamente a documentos en MongoDB.

**¿Qué contiene?**
Clases Java anotadas con `@Document` (para MongoDB) que tienen los atributos del modelo conceptual.

**Ejemplo del mundo real:**
Es como la ficha de catalogación de un libro en la biblioteca: tiene campos definidos (ISBN, título, autor) y se guarda físicamente en el sistema de archivos de la biblioteca.

**Relación con el modelo conceptual:**
La clase `Libro` del modelo conceptual se convierte en la clase `Libro.java` con `@Document`, y sus atributos (isbn, titulo, autor) se convierten en campos del documento MongoDB.

---

#### 🟩 Capa Repository — El acceso a datos

**¿Qué es?**
La capa que gestiona toda la comunicación con la base de datos. Es el único lugar del código que "sabe" cómo guardar y recuperar datos.

**¿Qué contiene?**
Interfaces que extienden `MongoRepository`. Spring Boot implementa automáticamente métodos como `save()`, `findById()`, `findAll()`, `deleteById()`.

**Ejemplo del mundo real:**
Es como el archivero de la biblioteca: conoce exactamente dónde está guardado cada libro, puede buscarlos por título, guardar nuevos registros o retirar los obsoletos.

**Ventaja clave:**
Gracias a Spring Data MongoDB, no necesitas escribir el código de acceso a la base de datos: ¡se genera automáticamente!

---

#### 🟨 Capa Service — La lógica de negocio

**¿Qué es?**
Aquí viven las reglas del negocio. Esta capa decide *qué pasa* cuando llega una petición.

**¿Qué contiene?**
Una **interfaz** que define el contrato de operaciones y una **implementación** que las lleva a cabo usando el Repository.

**¿Por qué interfaz + implementación?**
Permite cambiar la implementación (por ejemplo, si mañana cambias MongoDB por PostgreSQL) sin que el Controller tenga que cambiar nada. Esto aplica el **Principio de Inversión de Dependencias**.

**Ejemplo del mundo real:**
Es como el jefe de préstamos: sabe las reglas (un ejemplar ya prestado no puede prestarse de nuevo, un préstamo tiene fecha límite de devolución, etc.).

---

#### 🟧 Capa DTO — Los objetos de transferencia

**¿Qué es?**
DTO significa *Data Transfer Object*. Son clases que definen exactamente la forma de los datos que *entran* al API y los que *salen* de él.

**¿Qué contiene?**
- `LibroRequest`: lo que el cliente *envía* al crear o actualizar un libro (sin ID, porque MongoDB lo genera)
- `LibroResponse`: lo que el API *devuelve* al cliente (con ID y todos los campos)

**¿Por qué son importantes?**
- **Seguridad**: no expones campos internos de la base de datos
- **Flexibilidad**: puedes cambiar la estructura de la BD sin cambiar lo que ve el cliente
- **Validación**: puedes validar los datos de entrada antes de procesarlos

**Ejemplo del mundo real:**
El formulario que llena un usuario para solicitar un libro es diferente a la ficha interna que maneja el sistema. El usuario no ve (ni debe ver) el código interno del ejemplar o la fecha de última revisión.

---

#### 🟥 Capa Controller — El punto de entrada

**¿Qué es?**
Es la "puerta de entrada" del API. Recibe las peticiones HTTP, extrae los datos y delega el trabajo al Service.

**¿Qué contiene?**
Clases anotadas con `@RestController` con métodos que responden a URLs específicas (`GET /api/libros`, `POST /api/libros`, etc.).

**Ejemplo del mundo real:**
Es como la recepción de la biblioteca: recibe a los usuarios, entiende qué necesitan ("quiero devolver este libro", "quiero consultar si hay ejemplares de X") y los dirige al departamento correcto.

---

### Cómo se transforma el Modelo Conceptual en capas

| Modelo Conceptual | Se convierte en... | Capa |
|---|---|---|
| Clase `Libro` con atributos | `Libro.java` con `@Document` | **Model** |
| "Los libros se guardan en algún lugar" | `LibroRepository.java` | **Repository** |
| Operación "registrar libro" / reglas de negocio | `LibroService.java` + `LibroServiceImpl.java` | **Service** |
| "Qué datos recibe/devuelve el API" | `LibroRequest.java` / `LibroResponse.java` | **DTO** |
| "Cómo accede el cliente al sistema" | `LibroController.java` | **Controller** |

> 🎯 **Para reflexionar**: Una sola clase del modelo conceptual (`Libro`) se convierte en **5 archivos** en el modelo de desarrollo. Esto no es complejidad innecesaria: cada archivo tiene una responsabilidad específica y hace el sistema más fácil de mantener, modificar y probar.

---

## 3. El Modelo de Desarrollo

El modelo de desarrollo es la traducción técnica del modelo conceptual. Muestra cómo quedan organizadas **todas** las clases del sistema usando la arquitectura por capas, ya pensando en Java y Spring Boot.

![Modelo de Desarrollo del Sistema de Biblioteca](Modelo_desarrollo.png)

### ¿Qué hay de nuevo en este diagrama?

Comparado con el modelo conceptual, el modelo de desarrollo introduce conceptos técnicos:

**1. Paquetes (namespaces)**
El diagrama muestra los paquetes `controller`, `service`, `repository`, `dto` y `model`. Estos corresponden directamente a las carpetas que crearemos en nuestro proyecto Java.

**2. Interfaces vs. Implementaciones**
Para cada servicio hay dos elementos:
- `LibroService` → la interfaz (el "contrato")
- `LibroServiceImpl` → la implementación (el "código real")

Esta separación aplica el **Principio SOLID de Inversión de Dependencias**: el Controller depende de la interfaz, no de la implementación concreta.

**3. Inyección de dependencias**
Observa las flechas en el diagrama:
- `LibroController` tiene una referencia a `LibroService`
- `LibroServiceImpl` tiene una referencia a `LibroRepository`

Spring Boot gestiona estas referencias automáticamente. Tú no creas los objetos con `new`; Spring Boot los crea e inyecta donde se necesitan.

**4. DTOs diferenciados**
`LibroRequest` (entrada) y la respuesta con `id` (salida) son estructuras separadas, aunque compartan muchos campos.

> 📌 **Flujo completo de una petición**:
> ```
> Cliente HTTP
>     → LibroController (recibe y valida la petición HTTP)
>         → LibroService (aplica reglas de negocio)
>             → LibroRepository (consulta/modifica MongoDB)
>                 → MongoDB Atlas (base de datos)
>             ← Libro (documento de MongoDB)
>         ← LibroResponse (DTO de salida)
>     ← Respuesta JSON al cliente
> ```

---

## 4. Base de Datos: MongoDB Atlas

### ¿Por qué necesitamos una base de datos?

Sin base de datos, cuando el servidor de Spring Boot se reinicia, todos los datos desaparecen. La base de datos nos permite:

| Necesidad | Cómo lo resuelve la BD |
|---|---|
| **Persistencia** | Los datos sobreviven al reinicio del servidor |
| **Consultas eficientes** | Buscar entre millones de registros en milisegundos |
| **Integridad** | Evitar datos duplicados o inconsistentes |
| **Concurrencia** | Múltiples usuarios accediendo a los mismos datos simultáneamente |

### Tipos de bases de datos

#### Bases de datos Relacionales (SQL)

Organizan los datos en **tablas** con filas y columnas, con un esquema rígido definido previamente.

```
Tabla: libros
┌────┬─────────────────┬────────────────────┬────────────────┐
│ id │ isbn            │ titulo             │ autor          │
├────┼─────────────────┼────────────────────┼────────────────┤
│  1 │ 978-0-13-46859  │ Clean Code         │ Robert Martin  │
│  2 │ 978-0-13-23481  │ The Pragmatic Prog │ David Thomas   │
└────┴─────────────────┴────────────────────┴────────────────┘
```

**Características:**
- Usan **SQL** como lenguaje de consulta
- Las relaciones entre tablas se definen con claves foráneas
- Ideales para datos con estructuras fijas y relaciones complejas
- Ejemplos: **MySQL, PostgreSQL, Oracle, SQL Server**

#### Bases de datos No Relacionales (NoSQL)

Almacenan datos en formatos flexibles: documentos JSON, grafos, clave-valor, etc. No requieren un esquema fijo.

```json
// Documento en MongoDB (colección: libros)
{
  "_id": "64a8f2b3c4d5e6f7a8b9c0d1",
  "isbn": "978-0-13-468599-1",
  "titulo": "Clean Code",
  "autor": "Robert C. Martin",
  "anioPublicacion": 2008,
  "categoria": "Programación"
}
```

**Características:**
- Esquema flexible: cada documento puede tener campos diferentes
- Escalan horizontalmente con facilidad
- Ideales para datos cambiantes, desarrollo ágil y grandes volúmenes
- Ejemplos: **MongoDB, Redis, Cassandra, Firebase**

### ¿Por qué MongoDB para este proyecto?

| Ventaja | Descripción |
|---|---|
| **Documentos JSON** | Los datos se guardan como documentos JSON, igual que los objetos Java |
| **Sin migraciones** | Puedes agregar campos sin alterar toda la estructura |
| **Spring Data MongoDB** | Integración nativa con Spring Boot: el mapeo es automático |
| **Atlas gratuito** | Tier M0 sin costo para desarrollo y aprendizaje |
| **Adoptado en la industria** | Lo usan eBay, Forbes, Toyota, Bosch, entre muchos otros |

---

### Creando tu base de datos en MongoDB Atlas (Free Tier)

MongoDB Atlas es la versión en la nube de MongoDB. El **tier M0** es completamente gratuito y no requiere tarjeta de crédito. Ofrece 512 MB de almacenamiento, suficiente para cualquier proyecto educativo.

---

#### Paso 1: Crear una cuenta

1. Abre tu navegador y ve a **[https://www.mongodb.com/atlas](https://www.mongodb.com/atlas)**
2. Haz clic en el botón verde **"Try Free"**
3. Puedes registrarte con:
   - **Correo electrónico**: llena el formulario con nombre, correo y contraseña
   - **Google**: recomendado, es más rápido
4. Si te registras con correo, recibirás un email de verificación. Ábrelo y haz clic en el enlace de confirmación.
5. Al ingresar por primera vez, Atlas puede preguntarte sobre tu uso. Selecciona **"Learn MongoDB"** o **"I'm exploring"** y continúa.

---

#### Paso 2: Crear un Cluster gratuito (M0)

Después de iniciar sesión, Atlas te mostrará un asistente de inicio. Si no aparece automáticamente, busca el botón **"Build a Database"** o **"Create"** en el panel principal.

1. Verás tres opciones de plan. Selecciona la columna de la izquierda: **M0 — Free**

   ```
   ┌─────────────┬─────────────┬─────────────┐
   │  M0 FREE ✅  │    M10      │    M30      │
   │    $0/mes   │  $0.08/hr   │  $0.20/hr   │
   │  512 MB     │  2 GB RAM   │  4 GB RAM   │
   └─────────────┴─────────────┴─────────────┘
   ```

2. En la sección **Cloud Provider & Region**:
   - **Proveedor**: AWS (Amazon Web Services) — predeterminado, déjalo así
   - **Región**: Elige la más cercana a tu ubicación. Para Latinoamérica: **São Paulo (sa-east-1)** o **N. Virginia (us-east-1)**

3. En el campo **Cluster Name**, cambia el nombre a: `biblioteca-cluster`

4. Haz clic en el botón verde **"Create"** al final de la página

> ⏳ **Atlas tardará entre 1 y 3 minutos en crear el cluster.** Verás una barra de progreso. No cierres la página.

---

#### Paso 3: Crear un usuario de base de datos

Mientras el cluster se crea (o inmediatamente después), Atlas te mostrará el paso de **Security Quickstart**. Si no lo muestra automáticamente, ve a **Security > Database Access** en el menú izquierdo.

1. Haz clic en **"Add New Database User"**
2. En **Authentication Method** selecciona: **Password**
3. Configura las credenciales:
   - **Username**: `adminBiblioteca`
   - **Password**: Haz clic en **"Autogenerate Secure Password"** y copia la contraseña generada, o escribe una que recuerdes fácilmente (ejemplo: `Biblio2024!`)
4. En **Database User Privileges**, selecciona: **"Atlas admin"**
5. Haz clic en **"Add User"**

> 🔐 **IMPORTANTE**: Guarda el usuario y la contraseña en un lugar seguro (un bloc de notas, por ejemplo). Los necesitarás más adelante para configurar Spring Boot.

---

#### Paso 4: Configurar el acceso de red

MongoDB Atlas bloquea todas las conexiones por defecto. Solo las IPs que autorices explícitamente pueden conectarse a tu base de datos. Esto es una medida de seguridad fundamental.

1. En el menú izquierdo, ve a **Security > Network Access**
2. Haz clic en **"Add IP Address"**
3. En el cuadro de diálogo verás dos opciones principales:

   ```
   ┌─────────────────────────────────────────┐
   │  ADD CURRENT IP ADDRESS                 │
   │  Agrega tu IP actual automáticamente   │
   │  ✅ Úsala si trabajas siempre          │
   │     desde el mismo lugar               │
   ├─────────────────────────────────────────┤
   │  ALLOW ACCESS FROM ANYWHERE             │
   │  Agrega 0.0.0.0/0                      │
   │  Permite cualquier IP                  │
   │  Útil si tu IP cambia (WiFi,           │
   │  universidad, casa...)                 │
   └─────────────────────────────────────────┘
   ```

4. Para este tutorial, haz clic en **"Allow Access from Anywhere"** (esto facilita que trabajes desde la universidad, tu casa o cualquier red)
5. En el campo **Comment** escribe: `desarrollo-tutorial`
6. Haz clic en **"Confirm"**

> ⚠️ **Nota de seguridad**: "Allow from anywhere" (0.0.0.0/0) es conveniente para desarrollo pero **nunca debe usarse en producción**. En un entorno real siempre especificarías rangos de IP específicos.

---

#### Paso 5: Obtener la cadena de conexión

La cadena de conexión (*connection string*) es la URL que Spring Boot usará para conectarse a tu base de datos en Atlas.

1. Ve a **Database** en el menú izquierdo
2. Junto a `biblioteca-cluster` haz clic en el botón **"Connect"**
3. En el asistente, selecciona **"Drivers"**
4. Configura:
   - **Driver**: Java
   - **Version**: 4.3 or later
5. Atlas mostrará un string similar a este:
   ```
   mongodb+srv://adminBiblioteca:<password>@biblioteca-cluster.ab1cd.mongodb.net/?retryWrites=true&w=majority
   ```
6. Copia este string y **reemplaza `<password>`** con la contraseña que creaste en el Paso 3
7. El string final quedará algo así:
   ```
   mongodb+srv://adminBiblioteca:Biblio2024!@biblioteca-cluster.ab1cd.mongodb.net/biblioteca_db?retryWrites=true&w=majority
   ```
   > Nota: agrega `/biblioteca_db` antes del `?` para especificar el nombre de la base de datos.

8. Guarda este string. Lo usarás en el archivo `application.properties` de Spring Boot.

---

## 5. Creando el Proyecto Spring Boot en IntelliJ IDEA

### Prerrequisitos

Antes de comenzar, verifica que tengas instalado:

| Herramienta | Verificación | Cómo instalar si no lo tienes |
|---|---|---|
| **IntelliJ IDEA** | Abre la app | [jetbrains.com/idea](https://www.jetbrains.com/idea/) (Community es gratis) |
| **JDK 17** | Abre una terminal y escribe `java -version` | [adoptium.net](https://adoptium.net/) |
| **Postman** | Abre la app | [postman.com/downloads](https://www.postman.com/downloads/) |

---

### Paso 1: Crear el proyecto con Spring Initializr

IntelliJ IDEA tiene Spring Initializr integrado, lo que permite configurar y crear el proyecto desde la interfaz gráfica sin salir del IDE.

1. Abre IntelliJ IDEA
2. En la pantalla de bienvenida, haz clic en **"New Project"**
   - Si ya tienes un proyecto abierto: ve al menú **File > New > Project...**

3. En el **panel izquierdo** del asistente de nuevo proyecto, selecciona **"Spring Boot"**
   > Si no aparece "Spring Boot", busca "Spring Initializr" en la misma lista

4. Configura los campos del formulario de la siguiente manera:

   | Campo | Valor que debes escribir |
   |---|---|
   | **Name** | `biblioteca-api` |
   | **Location** | La carpeta donde quieres guardar el proyecto |
   | **Language** | `Java` |
   | **Type** | `Maven` |
   | **Group** | `com.biblioteca` |
   | **Artifact** | `biblioteca-api` |
   | **Package name** | `com.biblioteca` |
   | **Project SDK / JDK** | `17` |
   | **Java** | `17` |
   | **Packaging** | `Jar` |

5. Haz clic en **"Next"**

---

### Paso 2: Seleccionar las dependencias

En la siguiente pantalla del asistente verás un buscador de dependencias. Aquí agregas las librerías que tu proyecto necesita.

1. Verifica que la versión de **Spring Boot** sea **3.3.x** (en el selector de la parte superior)

2. Busca y selecciona cada una de las siguientes dependencias:

   **Spring Web**
   - Escribe `Spring Web` en el buscador
   - Selecciónala ✅
   - Permite crear endpoints HTTP REST

   **Spring Data MongoDB**
   - Escribe `MongoDB` en el buscador
   - Selecciona **"Spring Data MongoDB"** ✅
   - Provee la integración con MongoDB y las anotaciones `@Document`, `@Id`, etc.

   **Lombok**
   - Escribe `Lombok` en el buscador
   - Selecciónala ✅
   - Genera automáticamente getters, setters, constructores y más, reduciendo el código repetitivo

3. Verifica que en el panel derecho aparezcan las tres dependencias seleccionadas

4. Haz clic en **"Create"**

> ⏳ **IntelliJ descargará automáticamente todas las dependencias de Maven.** Observa la barra de progreso en la esquina inferior derecha del IDE. Espera hasta que desaparezca (puede tomar 1-3 minutos según tu conexión a internet).

---

### Paso 3: Verificar el archivo pom.xml

El archivo `pom.xml` es el corazón del proyecto Maven. Abre este archivo y verifica que tenga estas dependencias:

**Archivo: `pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
        <relativePath/>
    </parent>

    <groupId>com.biblioteca</groupId>
    <artifactId>biblioteca-api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>biblioteca-api</name>
    <description>API REST para el Sistema de Biblioteca</description>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Web: para crear endpoints REST -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Data MongoDB: integración con MongoDB -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>

        <!-- Lombok: reduce código repetitivo -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Para pruebas -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

### Paso 4: Crear los paquetes (packages)

Antes de escribir código, vamos a organizar el proyecto creando los paquetes que corresponden a las capas de nuestra arquitectura.

1. En el panel **Project** de IntelliJ (lado izquierdo), expande la ruta:
   `src > main > java > com > biblioteca`

2. **Clic derecho** sobre la carpeta `biblioteca` → **New > Package**

3. Crea los siguientes paquetes uno por uno:
   - `com.biblioteca.model`
   - `com.biblioteca.repository`
   - `com.biblioteca.service`
   - `com.biblioteca.service.impl`
   - `com.biblioteca.dto`
   - `com.biblioteca.controller`

   > Para cada uno: clic derecho sobre `com.biblioteca` → New > Package → escribe el nombre completo (ej: `com.biblioteca.model`) → Enter

Después de crear todos los paquetes, la estructura del proyecto en IntelliJ se verá así:

```
📁 biblioteca-api
└── 📁 src
    └── 📁 main
        ├── 📁 java
        │   └── 📁 com
        │       └── 📁 biblioteca
        │           ├── 📁 controller          ← Capa Controller
        │           ├── 📁 dto                 ← DTOs (entrada/salida)
        │           ├── 📁 model               ← Capa Model
        │           ├── 📁 repository          ← Capa Repository
        │           ├── 📁 service             ← Interfaz Service
        │           │   └── 📁 impl            ← Implementación Service
        │           └── 📄 BibliotecaApiApplication.java
        └── 📁 resources
            └── 📄 application.properties
```

---

## 6. Implementando la Capa Model

La capa Model contiene las clases que representan los documentos de MongoDB. Una clase de modelo en Spring Data MongoDB es simplemente una clase Java con anotaciones especiales que le indican a Spring cómo guardarla en la base de datos.

### Crear la clase Libro

1. **Clic derecho** sobre el paquete `com.biblioteca.model`
2. Selecciona **New > Java Class**
3. Escribe el nombre: `Libro`
4. Asegúrate de que el tipo sea **Class** (no Interface ni Enum)
5. Haz clic en **OK**

Ahora reemplaza todo el contenido del archivo con el siguiente código:

**Archivo: `src/main/java/com/biblioteca/model/Libro.java`**

```java
package com.biblioteca.model;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "libros")
public class Libro {

    @Id
    private String id;

    private String isbn;
    private String titulo;
    private String autor;
    private int anioPublicacion;
    private String categoria;
}
```

### ¿Qué hace cada anotación?

| Anotación | Qué hace |
|---|---|
| `@Document(collection = "libros")` | Le dice a Spring que esta clase es un documento MongoDB y se almacena en la colección llamada `"libros"` |
| `@Id` | Marca el campo `id` como el identificador único. MongoDB lo asigna automáticamente como un ObjectId |
| `@Data` | Lombok genera automáticamente: `getters`, `setters`, `toString()`, `equals()` y `hashCode()` |
| `@NoArgsConstructor` | Lombok genera el constructor sin argumentos (requerido por Spring) |
| `@AllArgsConstructor` | Lombok genera el constructor con todos los argumentos |

> 💡 **¿Por qué `String id` y no `Long id`?**
> MongoDB utiliza **ObjectId** como identificador, que es un valor hexadecimal de 24 caracteres generado automáticamente (ejemplo: `"64a8f2b3c4d5e6f7a8b9c0d1"`). Spring Data MongoDB lo convierte automáticamente entre `ObjectId` y `String`.

---

## 7. Implementando la Capa Repository

El Repository es la interfaz que le permite al Service comunicarse con MongoDB. Lo mejor es que **no necesitas escribir el código de las consultas**: Spring Data MongoDB las implementa automáticamente al extender `MongoRepository`.

### Crear la interfaz LibroRepository

1. **Clic derecho** sobre `com.biblioteca.repository`
2. Selecciona **New > Java Class**
3. Nombre: `LibroRepository`
4. **Muy importante**: en el selector de tipo, elige **Interface** (no Class)
5. Haz clic en **OK**

**Archivo: `src/main/java/com/biblioteca/repository/LibroRepository.java`**

```java
package com.biblioteca.repository;

import com.biblioteca.model.Libro;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface LibroRepository extends MongoRepository<Libro, String> {

    // Al extender MongoRepository, Spring genera automáticamente estos métodos:
    //
    // Libro save(Libro libro)          → Crear o actualizar un documento
    // Optional<Libro> findById(String id) → Buscar un documento por su ID
    // List<Libro> findAll()            → Obtener todos los documentos
    // void deleteById(String id)       → Eliminar un documento por su ID
    // boolean existsById(String id)   → Verificar si un documento existe
    // long count()                     → Contar el total de documentos

}
```

### Entendiendo `MongoRepository<Libro, String>`

La interfaz `MongoRepository` recibe dos parámetros de tipo:
- **`Libro`**: el tipo de documento que maneja este repositorio
- **`String`**: el tipo del campo ID (en MongoDB usamos `String` para el ObjectId)

Spring Boot crea automáticamente una implementación de esta interfaz cuando arranca la aplicación. Es decir, **el repositorio "funciona solo"** para las operaciones básicas.

---

## 8. Implementando los DTOs

Los DTOs definen la "forma" de los datos que viajan entre el cliente y el API. Vamos a crear dos: uno para lo que *recibe* el API y otro para lo que *devuelve*.

### ¿Por qué no usar directamente la clase Libro?

Considera este escenario: cuando el cliente quiere **crear** un libro, no debe enviar el campo `id` (lo genera MongoDB automáticamente). Pero cuando el API **devuelve** un libro, sí debe incluir el `id` para que el cliente sepa cómo identificarlo.

Con DTOs queda claro y separado:
- `LibroRequest` → lo que el cliente **envía** (sin `id`)
- `LibroResponse` → lo que el API **devuelve** (con `id`)

---

### Crear LibroRequest

1. **Clic derecho** sobre `com.biblioteca.dto`
2. **New > Java Class** → nombre: `LibroRequest` → tipo: **Class**

**Archivo: `src/main/java/com/biblioteca/dto/LibroRequest.java`**

```java
package com.biblioteca.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class LibroRequest {

    private String isbn;
    private String titulo;
    private String autor;
    private int anioPublicacion;
    private String categoria;
}
```

---

### Crear LibroResponse

1. **Clic derecho** sobre `com.biblioteca.dto`
2. **New > Java Class** → nombre: `LibroResponse` → tipo: **Class**

**Archivo: `src/main/java/com/biblioteca/dto/LibroResponse.java`**

```java
package com.biblioteca.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class LibroResponse {

    private String id;         // ← LibroResponse SÍ incluye el id
    private String isbn;
    private String titulo;
    private String autor;
    private int anioPublicacion;
    private String categoria;
}
```

> 📌 **La diferencia clave**: `LibroResponse` incluye el campo `id` (que MongoDB genera), mientras que `LibroRequest` no lo tiene porque el cliente no lo conoce aún al momento de crear el libro.

---

## 9. Implementando la Capa Service

La capa Service contiene la lógica de negocio. Seguimos el patrón **Interfaz + Implementación** que vimos en el modelo de desarrollo.

### Crear la interfaz LibroService

1. **Clic derecho** sobre `com.biblioteca.service`
2. **New > Java Class** → nombre: `LibroService` → tipo: **Interface**

**Archivo: `src/main/java/com/biblioteca/service/LibroService.java`**

```java
package com.biblioteca.service;

import com.biblioteca.dto.LibroRequest;
import com.biblioteca.dto.LibroResponse;

import java.util.List;

public interface LibroService {

    LibroResponse crearLibro(LibroRequest request);

    LibroResponse actualizarLibro(String id, LibroRequest request);

    void eliminarLibro(String id);

    LibroResponse consultarLibro(String id);

    List<LibroResponse> listarLibros();
}
```

Esta interfaz define el **contrato**: cualquier clase que implemente `LibroService` debe tener exactamente estos cinco métodos. El Controller solo conocerá esta interfaz, no la implementación concreta.

---

### Crear la implementación LibroServiceImpl

1. **Clic derecho** sobre `com.biblioteca.service.impl`
2. **New > Java Class** → nombre: `LibroServiceImpl` → tipo: **Class**

**Archivo: `src/main/java/com/biblioteca/service/impl/LibroServiceImpl.java`**

```java
package com.biblioteca.service.impl;

import com.biblioteca.dto.LibroRequest;
import com.biblioteca.dto.LibroResponse;
import com.biblioteca.model.Libro;
import com.biblioteca.repository.LibroRepository;
import com.biblioteca.service.LibroService;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class LibroServiceImpl implements LibroService {

    private final LibroRepository libroRepository;

    // Spring Boot inyecta automáticamente el repositorio aquí (constructor injection)
    public LibroServiceImpl(LibroRepository libroRepository) {
        this.libroRepository = libroRepository;
    }

    @Override
    public LibroResponse crearLibro(LibroRequest request) {
        // 1. Crear un nuevo objeto Libro con los datos del request
        Libro libro = new Libro();
        libro.setIsbn(request.getIsbn());
        libro.setTitulo(request.getTitulo());
        libro.setAutor(request.getAutor());
        libro.setAnioPublicacion(request.getAnioPublicacion());
        libro.setCategoria(request.getCategoria());

        // 2. Guardar en MongoDB (save() retorna el objeto con el ID generado)
        Libro libroGuardado = libroRepository.save(libro);

        // 3. Convertir el Libro guardado a LibroResponse y retornarlo
        return mapToResponse(libroGuardado);
    }

    @Override
    public LibroResponse actualizarLibro(String id, LibroRequest request) {
        // 1. Buscar el libro existente. Si no existe, lanzar excepción
        Libro libro = libroRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Libro no encontrado con id: " + id));

        // 2. Actualizar los campos con los nuevos datos
        libro.setIsbn(request.getIsbn());
        libro.setTitulo(request.getTitulo());
        libro.setAutor(request.getAutor());
        libro.setAnioPublicacion(request.getAnioPublicacion());
        libro.setCategoria(request.getCategoria());

        // 3. Guardar los cambios en MongoDB
        Libro libroActualizado = libroRepository.save(libro);

        // 4. Retornar la respuesta
        return mapToResponse(libroActualizado);
    }

    @Override
    public void eliminarLibro(String id) {
        // 1. Verificar que el libro exista antes de eliminar
        if (!libroRepository.existsById(id)) {
            throw new RuntimeException("Libro no encontrado con id: " + id);
        }

        // 2. Eliminar el documento de MongoDB
        libroRepository.deleteById(id);
    }

    @Override
    public LibroResponse consultarLibro(String id) {
        // Buscar el libro. Si no existe, lanzar excepción
        Libro libro = libroRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Libro no encontrado con id: " + id));

        return mapToResponse(libro);
    }

    @Override
    public List<LibroResponse> listarLibros() {
        // Obtener todos los libros y convertirlos a LibroResponse usando Java Streams
        return libroRepository.findAll()
                .stream()
                .map(this::mapToResponse)
                .collect(Collectors.toList());
    }

    // ─────────────────────────────────────────────
    // Método auxiliar: convierte Libro → LibroResponse
    // ─────────────────────────────────────────────
    private LibroResponse mapToResponse(Libro libro) {
        return new LibroResponse(
                libro.getId(),
                libro.getIsbn(),
                libro.getTitulo(),
                libro.getAutor(),
                libro.getAnioPublicacion(),
                libro.getCategoria()
        );
    }
}
```

### Entendiendo el código

| Elemento | Explicación |
|---|---|
| `@Service` | Le indica a Spring que esta clase es un componente de servicio. Spring la administra y puede inyectarla donde se necesite |
| Constructor con `LibroRepository` | Patrón de **Inyección por Constructor**: Spring detecta que necesita un `LibroRepository` y lo provee automáticamente |
| `orElseThrow(...)` | Si `findById()` no encuentra el libro, `Optional` está vacío y se lanza una `RuntimeException` con un mensaje descriptivo |
| `.stream().map(this::mapToResponse)` | Java Streams: transforma cada objeto `Libro` de la lista en un `LibroResponse` |
| `mapToResponse(libro)` | Método privado auxiliar que convierte un `Libro` (modelo de BD) en un `LibroResponse` (DTO de salida). Mantiene la separación entre capas |

---

## 10. Implementando la Capa Controller

El Controller es la interfaz HTTP de tu API. Define qué URLs existen y qué hace el sistema cuando alguien las invoca.

### Crear LibroController

1. **Clic derecho** sobre `com.biblioteca.controller`
2. **New > Java Class** → nombre: `LibroController` → tipo: **Class**

**Archivo: `src/main/java/com/biblioteca/controller/LibroController.java`**

```java
package com.biblioteca.controller;

import com.biblioteca.dto.LibroRequest;
import com.biblioteca.dto.LibroResponse;
import com.biblioteca.service.LibroService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/libros")
public class LibroController {

    private final LibroService libroService;

    public LibroController(LibroService libroService) {
        this.libroService = libroService;
    }

    // ─────────────────────────────────────────────
    // POST /api/libros
    // Crear un nuevo libro
    // ─────────────────────────────────────────────
    @PostMapping
    public ResponseEntity<LibroResponse> crearLibro(@RequestBody LibroRequest request) {
        LibroResponse response = libroService.crearLibro(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    // ─────────────────────────────────────────────
    // GET /api/libros
    // Listar todos los libros
    // ─────────────────────────────────────────────
    @GetMapping
    public ResponseEntity<List<LibroResponse>> listarLibros() {
        List<LibroResponse> libros = libroService.listarLibros();
        return ResponseEntity.ok(libros);
    }

    // ─────────────────────────────────────────────
    // GET /api/libros/{id}
    // Consultar un libro por su ID
    // ─────────────────────────────────────────────
    @GetMapping("/{id}")
    public ResponseEntity<LibroResponse> consultarLibro(@PathVariable String id) {
        LibroResponse response = libroService.consultarLibro(id);
        return ResponseEntity.ok(response);
    }

    // ─────────────────────────────────────────────
    // PUT /api/libros/{id}
    // Actualizar un libro existente
    // ─────────────────────────────────────────────
    @PutMapping("/{id}")
    public ResponseEntity<LibroResponse> actualizarLibro(
            @PathVariable String id,
            @RequestBody LibroRequest request) {
        LibroResponse response = libroService.actualizarLibro(id, request);
        return ResponseEntity.ok(response);
    }

    // ─────────────────────────────────────────────
    // DELETE /api/libros/{id}
    // Eliminar un libro
    // ─────────────────────────────────────────────
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> eliminarLibro(@PathVariable String id) {
        libroService.eliminarLibro(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Mapa completo de endpoints del API

| Método HTTP | URL | Descripción | Código de Respuesta |
|---|---|---|---|
| `POST` | `/api/libros` | Crear un nuevo libro | `201 Created` |
| `GET` | `/api/libros` | Obtener todos los libros | `200 OK` |
| `GET` | `/api/libros/{id}` | Obtener un libro por ID | `200 OK` |
| `PUT` | `/api/libros/{id}` | Actualizar un libro | `200 OK` |
| `DELETE` | `/api/libros/{id}` | Eliminar un libro | `204 No Content` |

### Entendiendo las anotaciones

| Anotación | Qué hace |
|---|---|
| `@RestController` | Marca la clase como controlador REST. Combina `@Controller` + `@ResponseBody`: todos los métodos retornan directamente JSON |
| `@RequestMapping("/api/libros")` | URL base para todos los endpoints de este controlador |
| `@PostMapping` | El método responde a peticiones `HTTP POST` |
| `@GetMapping` | El método responde a peticiones `HTTP GET` |
| `@PutMapping("/{id}")` | El método responde a `HTTP PUT` y el `{id}` es un parámetro de la URL |
| `@DeleteMapping("/{id}")` | El método responde a `HTTP DELETE` |
| `@RequestBody` | Spring convierte automáticamente el JSON del cuerpo de la petición al objeto Java (`LibroRequest`) |
| `@PathVariable String id` | Extrae el valor de `{id}` de la URL |
| `ResponseEntity<T>` | Permite controlar explícitamente el código HTTP de la respuesta (`201`, `200`, `204`, etc.) |

---

## 11. Configurando la Conexión a MongoDB

### Editar application.properties

Abre el archivo `src/main/resources/application.properties` y reemplaza todo su contenido con lo siguiente:

**Archivo: `src/main/resources/application.properties`**

```properties
# ─────────────────────────────────────────────
# Configuración de la aplicación
# ─────────────────────────────────────────────
spring.application.name=biblioteca-api

# ─────────────────────────────────────────────
# Conexión a MongoDB Atlas
# Reemplaza la URI con tu cadena de conexión real
# obtenida en el Paso 5 de la sección MongoDB Atlas
# ─────────────────────────────────────────────
spring.data.mongodb.uri=mongodb+srv://adminBiblioteca:TuPassword@biblioteca-cluster.ab1cd.mongodb.net/biblioteca_db?retryWrites=true&w=majority

# Nombre de la base de datos en MongoDB
spring.data.mongodb.database=biblioteca_db

# ─────────────────────────────────────────────
# Servidor
# ─────────────────────────────────────────────
server.port=8080
```

> ⚠️ **IMPORTANTE**: Reemplaza el valor de `spring.data.mongodb.uri` con la cadena de conexión que copiaste de MongoDB Atlas. Asegúrate de:
> - Que `TuPassword` sea la contraseña real del usuario `adminBiblioteca`
> - Que `biblioteca-cluster.ab1cd.mongodb.net` sea tu URL real del cluster (la que te dio Atlas)
> - Que `/biblioteca_db` esté incluido en la URI (es el nombre de la base de datos que se creará automáticamente)

---

### Verificar la estructura final del proyecto

Antes de ejecutar la aplicación, confirma que tu proyecto tenga exactamente esta estructura:

```
biblioteca-api/
└── src/
    └── main/
        ├── java/com/biblioteca/
        │   ├── BibliotecaApiApplication.java    ← Clase principal (ya existía)
        │   ├── controller/
        │   │   └── LibroController.java          ← ✅ Capa Controller
        │   ├── dto/
        │   │   ├── LibroRequest.java             ← ✅ DTO de entrada
        │   │   └── LibroResponse.java            ← ✅ DTO de salida
        │   ├── model/
        │   │   └── Libro.java                   ← ✅ Capa Model
        │   ├── repository/
        │   │   └── LibroRepository.java          ← ✅ Capa Repository
        │   └── service/
        │       ├── LibroService.java             ← ✅ Interfaz Service
        │       └── impl/
        │           └── LibroServiceImpl.java     ← ✅ Implementación Service
        └── resources/
            └── application.properties           ← ✅ Configuración MongoDB
```

---

### Ejecutar la aplicación

1. En IntelliJ IDEA, localiza el archivo `BibliotecaApiApplication.java` y ábrelo

2. Verás la clase principal de Spring Boot:

```java
package com.biblioteca;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BibliotecaApiApplication {

    public static void main(String[] args) {
        SpringApplication.run(BibliotecaApiApplication.class, args);
    }
}
```

3. Para ejecutar la aplicación, tienes dos opciones:
   - Haz clic en el ícono ▶️ verde que aparece a la izquierda del método `main`
   - Usa el atajo de teclado: **Ctrl + Shift + F10** (Windows/Linux) o **Ctrl + R** (Mac)

4. Observa la consola inferior de IntelliJ. Si todo está bien configurado, verás:

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.3.x)

... Started BibliotecaApiApplication in 4.2 seconds (process running for 4.9)
```

> ✅ Si ves la línea **"Started BibliotecaApiApplication in X seconds"**, ¡tu API está en ejecución y conectada a MongoDB Atlas!

> ❌ Si ves un error como `Could not connect to MongoDB`, verifica:
> - Que la URI en `application.properties` sea correcta
> - Que la contraseña no tenga caracteres especiales sin codificar
> - Que la red de tu computador permita conexiones salientes (firewall, VPN)
> - Que hayas configurado "Allow Access from Anywhere" en MongoDB Atlas

---

## 12. Probando la API con Postman

### ¿Qué es Postman?

Postman es la herramienta estándar de la industria para probar y documentar APIs. Permite enviar peticiones HTTP (GET, POST, PUT, DELETE) de forma visual, ver las respuestas y guardar las peticiones como colecciones reutilizables.

### Configuración inicial

1. Abre Postman
2. Haz clic en **"New"** → **"Collection"**
3. Nómbrala: `Biblioteca API`
4. Dentro de la colección, crea cada una de las siguientes peticiones

La URL base del API es: **`http://localhost:8080`**

---

### Prueba 1: Crear un libro

| Campo | Valor |
|---|---|
| **Método** | `POST` |
| **URL** | `http://localhost:8080/api/libros` |
| **Body** | raw → JSON |

**JSON a enviar en el Body:**
```json
{
  "isbn": "978-0-13-468599-1",
  "titulo": "Clean Code",
  "autor": "Robert C. Martin",
  "anioPublicacion": 2008,
  "categoria": "Programación"
}
```

Haz clic en **Send**.

**Respuesta esperada — `201 Created`:**
```json
{
  "id": "64a8f2b3c4d5e6f7a8b9c0d1",
  "isbn": "978-0-13-468599-1",
  "titulo": "Clean Code",
  "autor": "Robert C. Martin",
  "anioPublicacion": 2008,
  "categoria": "Programación"
}
```

> 📋 **Copia el valor del campo `id`** de la respuesta. Lo necesitarás para las pruebas 3, 4 y 5.

---

### Prueba 2: Listar todos los libros

| Campo | Valor |
|---|---|
| **Método** | `GET` |
| **URL** | `http://localhost:8080/api/libros` |
| **Body** | Ninguno |

**Respuesta esperada — `200 OK`:**
```json
[
  {
    "id": "64a8f2b3c4d5e6f7a8b9c0d1",
    "isbn": "978-0-13-468599-1",
    "titulo": "Clean Code",
    "autor": "Robert C. Martin",
    "anioPublicacion": 2008,
    "categoria": "Programación"
  }
]
```

---

### Prueba 3: Consultar un libro por ID

| Campo | Valor |
|---|---|
| **Método** | `GET` |
| **URL** | `http://localhost:8080/api/libros/64a8f2b3c4d5e6f7a8b9c0d1` |
| **Body** | Ninguno |

> ⚠️ Reemplaza `64a8f2b3c4d5e6f7a8b9c0d1` con el ID real que obtuviste en la Prueba 1.

**Respuesta esperada — `200 OK`:**
```json
{
  "id": "64a8f2b3c4d5e6f7a8b9c0d1",
  "isbn": "978-0-13-468599-1",
  "titulo": "Clean Code",
  "autor": "Robert C. Martin",
  "anioPublicacion": 2008,
  "categoria": "Programación"
}
```

---

### Prueba 4: Actualizar un libro

| Campo | Valor |
|---|---|
| **Método** | `PUT` |
| **URL** | `http://localhost:8080/api/libros/64a8f2b3c4d5e6f7a8b9c0d1` |
| **Body** | raw → JSON |

**JSON a enviar en el Body:**
```json
{
  "isbn": "978-0-13-468599-1",
  "titulo": "Clean Code: A Handbook of Agile Software Craftsmanship",
  "autor": "Robert C. Martin",
  "anioPublicacion": 2008,
  "categoria": "Ingeniería de Software"
}
```

**Respuesta esperada — `200 OK`:** El libro con el título y la categoría actualizados.

---

### Prueba 5: Eliminar un libro

| Campo | Valor |
|---|---|
| **Método** | `DELETE` |
| **URL** | `http://localhost:8080/api/libros/64a8f2b3c4d5e6f7a8b9c0d1` |
| **Body** | Ninguno |

**Respuesta esperada — `204 No Content`:** La respuesta no tiene cuerpo. El código 204 confirma que el libro fue eliminado correctamente.

Para verificar la eliminación, vuelve a ejecutar la **Prueba 2** (listar todos) y confirma que el libro ya no aparece.

---

### Resumen de todas las pruebas

| # | Operación | Método HTTP | URL | Requiere Body |
|---|---|---|---|---|
| 1 | Crear libro | `POST` | `/api/libros` | ✅ JSON con datos del libro |
| 2 | Listar libros | `GET` | `/api/libros` | ❌ |
| 3 | Consultar libro | `GET` | `/api/libros/{id}` | ❌ |
| 4 | Actualizar libro | `PUT` | `/api/libros/{id}` | ✅ JSON con datos actualizados |
| 5 | Eliminar libro | `DELETE` | `/api/libros/{id}` | ❌ |

---

## ¡Felicitaciones! 🎉

Has construido exitosamente una API REST completa con:

- ✅ **Arquitectura por capas** (Model, Repository, Service, DTO, Controller)
- ✅ **Spring Boot 3.3** como framework
- ✅ **MongoDB Atlas** como base de datos en la nube
- ✅ **CRUD completo** para gestión de libros
- ✅ **Postman** para probar todos los endpoints

---

## Flujos pendientes para implementar

Este tutorial cubrió el flujo completo de **gestión de libros**. Siguiendo exactamente el mismo patrón de capas, ahora te corresponde implementar:

### 📋 Flujo de Usuarios
Repite el mismo proceso creando:
- `model/Usuario.java` (con campos: nombre, correo, tipoUsuario)
- `repository/UsuarioRepository.java`
- `dto/UsuarioRequest.java` y `dto/UsuarioResponse.java`
- `service/UsuarioService.java` y `service/impl/UsuarioServiceImpl.java`
- `controller/UsuarioController.java` con endpoints en `/api/usuarios`

### 📚 Flujo de Préstamos
Es el flujo más complejo. Considera:
- Al **crear un préstamo**, verifica que el ejemplar esté en estado `"DISPONIBLE"`
- Al **registrar una devolución**, cambia el estado del ejemplar a `"DISPONIBLE"`
- Un préstamo debe guardar: `usuarioId`, `ejemplarId`, `fechaPrestamo`, `fechaDevolucionEsperada`, `estado`

---

*Tutorial creado para el curso de Diseño de Software — 4° Semestre de Ingeniería de Software*
*Fecha: Mayo 2026*
