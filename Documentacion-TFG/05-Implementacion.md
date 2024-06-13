# Implementación

## Back-End

Comenzando con el Back-End, partimos de unas bases de datos ya creadas, por lo que nuestro punto de partida es Database first, siendo así necesario generar el contexto y los modelos de la base de datos para hacer uso de ella mediante Entity Framework:

```{.bash .numberLines}
#Consola
dotnet ef dbcontext scaffold "server=(local);database=AppEF;Trusted_Connection=True" Microsoft.EntityFrameworkCore.SqlServer --context-dir Contexto --output-dir Modelos
```

Este es el comando usado para generar el contexto en nuestro proyecto, por lo que deberemos sustituir los valores de server y database por nuestra cadena de conexión y la base de datos que queremos usar.

### Uso del contexto y los modelos

Una vez generado el contexto y los modelos proseguiremos haciendo la estructura que va a tener nuestra api para poder conectarse con la base de datos.

La idea es añadir capas de abstracción entre nuestro controlador y nuestro contexto para mantener nuestro código ordenado y fácilmente accesible, además de facilitarnos la lectura del mismo, evitando tener un solo archivo con miles de lineas de código (aunque mientras más complejo es el proyecto, más difícil es evitar esto).

\pagebreak

### Patrón a utilizar

Haremos uso de un patrón repositorio-servicio, lo que significa que contaremos con nuestro repositorio, el cual se comunicará con el contexto y manejará los datos para devolver aquello que solicitamos, enviando esos datos al servicio y este los devolverá al Front-End.

La estructura será simple, contaremos una interfaz en nuestro repositorio, donde indicaremos las funciones que este incluirá.

```{.csharp .numberLines}
//IVehiculoRepository.cs
public interface IVehiculoRepository {
    Task<Vehiculo> GetById(int id);
    Task Insert(Vehiculo entity, int idCreador, string conn);
    void Update(Vehiculo entity, int idEditor);
    Task DeleteById(int id, int idBorrador);
    IQueryable<Vehiculo> GetAll(int idEmpresa, bool obtenerDesactivados = false);
    PagedList<Vehiculo> GetAllPaginated(VehiculoParams parameters, int idEmpresa, bool obtenerDesactivados = false);
    void SearchByName(ref IQueryable<Vehiculo> vehiculos, string matricula);
    void ApplySort(ref IQueryable<Vehiculo> query, string orderByQueryString);
    int NextIdSequence(string conn);
}
```

E interfaces en cada uno de nuestros servicios, donde también se mostrarán las funciones que estos incluirán.

```{.csharp .numberLines}
//IVehiculoServicio.cs*
public interface IVehiculoService {
    Task<VehiculoDTO> GetById(int Id, string cadenaConexion);
    Task<Vehiculo> GetByIdNoMap(int Id, string cadenaConexion);
    Task<List<VehiculoDTO>> GetVehiculos(int idEmpresa, string cadenaConexion);
    Task<(List<VehiculoDTO> listadoVehiculosDto, MetadataDto metadataDto)> GetVehiculosPaginated(VehiculoParams parameters, string cadenaConexion, int idEmpresa);
    Task AddVehiculo(VehiculoDTO vehiculo, string cadenaConexion, int idCreador);
    Task UpdateVehiculo(VehiculoDTO vehiculo, string cadenaConexion, int idEditor);
    Task DeleteById(int id, string cadenaConexion, int idBorrador);
}
```

Con solo ver esto, os podréis imaginar que este proceso tiene que repetirse tantas veces como modelos tengamos, lo que a primera vista se siente como algo innecesario ya que .NET cuenta con algo llamado genéricos, y esto significa que podemos crear un repositorio con todas las funciones que sea de tipo genérico (hace referencia a una clase indefinida, es decir, sirve para todo) y una vez hecho eso, simplemente deberíamos crear instancias del repositorio genérico pero asignándole un tipo, de esa forma  solo tenemos un repositorio en vez de decenas de ellos.

Esto es totalmente válido ya que es una buena forma de trabajar, sin embargo, anteriormente nombramos que vamos a trabajar con campos de auditoría, estos concretamente indican qué persona hizo que cosa, ya sea quién insertó el registro, quien lo modificó y quien lo borró junto a sus respectivas fechas y horas.

No suena como algo negativo, pero como al trabajar con tipos genéricos, realmente no tiene un tipo definido, no es posible acceder a sus propiedades.

Esto claramente tiene solución, y es buscar la forma de indicar que ese repositorio genérico va a manejar objetos cuyas clases contengan los campos de auditoría que necesitamos, y realmente no es muy complejo de hacer, pero no siempre podemos usar este método así que inevitablemente en algunos casos vamos a tener que crear un repositorio por cada modelo que tengamos, sin embargo, el repositorio genérico y los específicos se suelen usar de forma conjunta ya que hay casos en los que vas a necesitar uno u otro, más adelante lo veremos.

```{.csharp .numberLines}
//IGenericRepository 
public interface IGenericRepository<TEntity> where TEntity : class {
    Task<TEntity> GetById(int id);
    Task Insert(TEntity entity);
    Task Update(TEntity entity);
    Task DeleteById(int id);
    IQueryable<TEntity> GetAll();
    PagedList<TEntity> GetAllPaginated(QueryStringParameters parameters);
}
```

\pagebreak

Ahora necesitamos un área de trabajo, ahí vamos a almacenar tanto nuestros repositorios específicos como las instancias de nuestro repositorio general y funciones que queramos darle.

```{.csharp .numberLines}
//IUnitOfWork.cs
public interface IUnitOfWork : IDisposable {
    IVehiculoRepository VehiculoRepositorio { get; }
    IMarcaRepository MarcaRepositorio { get; }
    IModeloRepository ModeloRepositorio { get; }
    IGenericRepository<Usuario> UsuariosRepositorio { get; }
    IGenericRepository<EmpresasActiva> EmpresasActivaRepositorio { get; }
    IEmpresaRepository EmpresaRepositorio { get; }
    ICombustibleRepository CombustibleRepositorio { get; }
    IGenericRepository<EmpresasUsuario> EmpresasUsuarioRepositorio { get; }
    ICambioEmpresaRepository RepositorioCambioEmpresa { get; }
    public int SaveChanges();
    public Task<int> SaveChangesAsync();
}
```

Con ver esta interfaz ya os podréis imaginar lo que está pasando, y la realidad es que como dije anteriormente, el repositorio genérico y los específicos pueden trabajar de forma conjunta, siendo así que en los casos en los que vaya a necesitar tocar campos de auditoría, voy a crear un repositorio nuevo con todo lo que conlleva, es decir, la duplicación de código, pero en los casos en los que no lo necesite como los usuarios, ya que en este caso no vamos a añadir usuarios, ni modificarlos o borrarlos, lo mismo pasa con nuestras empresas activas y empresas usuarios, no vamos a tocar auditoría, por lo que podemos hacer uso de nuestro repositorio genérico y simplemente le establecemos el tipo (la clase) a la que van a pertenecer.

\pagebreak

De esta forma, desde nuestra unidad de trabajo vamos a tener acceso a todos los repositorios, sin importar que sean repositorios específicos o instancias del genérico.

```{.csharp .numberLines }
//UnitOfWork.cs 
public class UnitOfWork : IUnitOfWork, IDisposable {
    private readonly RegistroGeneralContext _context;
    private string _cadenaConexion;

    public UnitOfWork(RegistroGeneralContext context) {
        _context = context;
    }

    public UnitOfWork(string? cadenaConexion) {
        _cadenaConexion = cadenaConexion;
        _context = new RegistroGeneralContext (_cadenaConexion);
    }

    private IVehiculoRepository? _repositorioVehiculo;
    private IModeloRepository? _repositorioModelo;
    private IMarcaRepository? _repositorioMarca;
    private IGenericRepository<Usuario>? _repositorioUsuario;
    private IGenericRepository<EmpresasActiva>? _repositorioEmpresasActiva;
    private IEmpresaRepository? _repositorioEmpresa;
    private ICombustibleRepository? _repositorioCombustible;
    private IGenericRepository<EmpresasUsuario>? _repositorioEmpresasUsuario;
    private ICambioEmpresaRepository? _repositorioCambioEmpresa;

    IGenericRepository<Usuario> IUnitOfWork.UsuariosRepositorio => _repositorioUsuario ??= new GenericRepository<Usuario>(_context);
    IGenericRepository<EmpresasActiva> IUnitOfWork.EmpresasActivaRepositorio => _repositorioEmpresasActiva ??= new GenericRepository<EmpresasActiva>(_context);
    IGenericRepository<EmpresasUsuario> IUnitOfWork.EmpresasUsuarioRepositorio => _repositorioEmpresasUsuario ??= new GenericRepository<EmpresasUsuario>(_context);
    public ICombustibleRepository CombustibleRepositorio => _repositorioCombustible ??= new CombustibleRepository(_context);
    public IModeloRepository ModeloRepositorio => _repositorioModelo ??= new ModeloRepository(_context);
    public IMarcaRepository MarcaRepositorio => _repositorioMarca ??= new MarcaRepository(_context);
    public ICambioEmpresaRepository RepositorioCambioEmpresa => _repositorioCambioEmpresa ??= new CambioEmpresaRepository(_context);
    public IEmpresaRepository EmpresaRepositorio => _repositorioEmpresa ??= new EmpresaRepository(_context);
    public IVehiculoRepository VehiculoRepositorio => _repositorioVehiculo ??= new VehiculoRepository(_context);


    public void Dispose() {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing) {
        _context?.Dispose();
    }

    public int SaveChanges() {
        return _context.SaveChanges();
    }

    public Task<int> SaveChangesAsync() {
        return _context.SaveChangesAsync();
    }
}
```

Y en nuestra unidad de trabajo, implementamos la interfaz.

\pagebreak

### Repositorio General

```{.csharp .numberLines}
//GenericRepository.cs
public class GenericRepository<TEntity> : IGenericRepository<TEntity> where TEntity : class {

    private readonly DbSet<TEntity> _entity;
    public GenericRepository(RegistroGeneralContext context) {
        _entity = context.Set<TEntity>();
    }

    public async Task<TEntity> GetById(int id) {
        return await _entity.FindAsync(id);
    }

    public async Task Insert(TEntity entity) {
        _entity.Add(entity);
    }

    public async Task Update(TEntity entity) {
        _entity.Update(entity);
    }

    public async Task DeleteById(int id) {
        var elem = await _entity.FindAsync(id);
        if (elem != null) {
            _entity.Remove(elem);
        }
    }
}

```

Ahora solo queda implementar interfaz en nuestro repositorio genérico y en nuestros repositorios específicos, la única diferencia será que nuestro genérico trabajará con TEntity haciendo referencia a una clase cualquiera, y en nuestros específicos podremos tratar con los modelos concretos y tendremos acceso a sus propiedades.

\pagebreak

```{.csharp .numberLines}
//VehiculoRepository
public class VehiculoRepository : IVehiculoRepository {
    private readonly DbSet<Vehiculo> _vehiculos;
    private readonly DbSet<Modelo> _modelos;
    private readonly DbSet<Marca> _marcas;
    private readonly RegistroGeneralContext _context;
    public VehiculoRepository(RegistroGeneralContext context) {
        _vehiculos = context.Set<Vehiculo>();
        _modelos = context.Set<Modelo>();
        _marcas = context.Set<Marca>();
        _context = context;
    }

    public async Task<Vehiculo> GetById(int id) {
        var query = _vehiculos.Include(e => e.Modelo).Include(e => e.Marca).Include(e => e.TipoCombustible).Where(x => (x.BorradoLogico == false || x.BorradoLogico == null) && x.FechaBorradoLogico == null).ToList();
        return query.Where(x => x.Id == id).FirstOrDefault();

    }

    public async Task Insert(Vehiculo entity, int idCreador, string conn) {
        entity.IdCreador = idCreador;
        entity.FechaCreacion = DateTime.Now;
        entity.GuidRegistro = Guid.NewGuid().ToString().ToUpper();
        entity.Id = NextIdSequence(conn);
        entity.Activo = true;
        _vehiculos.Add(entity);
    }

    public void Update(Vehiculo entity, int idEditor) {
        entity.IdEditor = idEditor;
        entity.FechaModificacion = DateTime.Now;
        _vehiculos.Update(entity);
    }

    public async Task DeleteById(int id, int idBorrador) {
        var elem = await _vehiculos.FindAsync(id);
        if (elem != null) {
            elem.IdBorrador = idBorrador;
            elem.FechaBorradoLogico = DateTime.Now;
            elem.BorradoLogico = true;

            _vehiculos.Update(elem);

        }
    }
}
```

Como se puede apreciar, hay bastante diferencia entre uno y otro y no vamos a mostrar el archivo entero ya que ahora mismo no es necesario.

Sin embargo, si hay una función importante que mostrar y esa es el GetAll.

```{.csharp .numberLines}
//VehiculoRepository.cs
public PagedList<Vehiculo> GetAllPaginated(VehiculoParams parameters, int idEmpresa, bool obtenerDesactivados = false) {
    var query = _vehiculos.AsNoTracking().Include(e => e.Modelo).Include(e => e.Marca).Include(e => e.TipoCombustible).Where(e => e.IdEmpresa == idEmpresa);

    if (parameters.Marca != null) {
        query = query.Where(e => e.MarcaId == parameters.Marca);

    }

    if (parameters.Modelo != null) {
        query = query.Where(e => e.ModeloId == parameters.Modelo);
    }

    if (parameters.Combustible != null) {
        query = query.Where(e => e.TipoCombustibleId == parameters.Combustible);
    }

    if (!obtenerDesactivados) {
        query = query.Where(e => e.BorradoLogico == null || e.BorradoLogico == false);
    }

    SearchByName(ref query, parameters.Matricula);
    ApplySort(ref query, parameters.OrderBy);

    return PagedList<Vehiculo>.ToPagedList(query, parameters.PageNumber, parameters.PageSize);
}
```

Primero que todo, estamos devolviendo un Pagedlist de tipo vehículo, pagedlist no es más que una clase de utilidad creada para poder hacer que nuestros getAll no devuelvan todos los registros de nuestras tablas, sino que se devuelvan paginados en tandas de 5, 10 o 20 registros para no saturar el front con datos, también tenemos un objeto de la clase VehiculoParams, que cuenta con la información del tamaño de página, numero de página y otros datos útiles.

Por otro lado, estaremos pasando un idEmpresa, el cual nos sirve para devolver solo los vehiculos que pertenezcan a la empresa que queremos, y un boolean preestablecido a falso, esto se debe a que no realizaremos borrados físicos sino lógicos, es decir, el registro no se borra de la base de datos, sino que rellenamos campos como borradoLogico, idBorrador y fechaBorradoLogico para filtrar por aquellos elementos los cuales no estén marcados como borrados, y por ultimo tenemos una llamada hacia search by name y applySort.

Como indiqué anteriormente, toda la paginación, filtrado, búsqueda y ordenación se obtuvieron siguiendo el tutorial de [CodeMaze](https://code-maze.com/net-core-series/) &rarr; [https://code-maze.com/net-core-series/](https://code-maze.com/net-core-series/) .

### Servicios

Anteriormente mostramos la interfaz de nuestro servicio de vehículos, donde vemos aquellas funciones que queremos darle al servicio en concreto, esto lo que va a realizar es una llamada hacia la función correspondiente en nuestro repositorio.

```{.csharp .numberLines}
//ServicioVehiculo.cs
public class VehiculoService : IVehiculoService {
    private readonly IUnitOfWork _repositorioEspecifico;
    private readonly RegistroGeneralContext _context;
    private readonly IMapper _mapper;

    public VehiculoService(IUnitOfWork repositorioEspecifico, RegistroGeneralContext context, IMapper mapper) {
        _repositorioEspecifico = repositorioEspecifico;
        _context = context;
        _mapper = mapper;
    }

    public async Task AddVehiculo(VehiculoDTO vehiculoDTO, string cadenaConexion, int idCreador) {
        using (IUnitOfWork repositorioEspecifico = new UnitOfWork(cadenaConexion)) {
            Vehiculo vehiculo = _mapper.Map<Vehiculo>(vehiculoDTO);
            await repositorioEspecifico.VehiculoRepositorio.Insert(vehiculo, idCreador, cadenaConexion);
            repositorioEspecifico.SaveChanges();
        }
    }

    public async Task DeleteById(int id, string cadenaConexion, int idBorrador) {
        using (IUnitOfWork repositorioEspecifico = new UnitOfWork(cadenaConexion)) {
            await repositorioEspecifico.VehiculoRepositorio.DeleteById(id, idBorrador);
            repositorioEspecifico.SaveChanges();
        }
    }
}
```

\pagebreak

Aquí mostramos un par de funciones de nuestro servicio de vehículos, donde crearemos una nueva instancia de nuestro repositorio para cambiar hacia que base de datos hacemos la petición (esto ahora mismo no es importante, será explicado más adelante) y lo único que hacemos es una llamada a nuestro repositorio, o mapeamos el objeto recibido y lo enviamos al repositorio, esto se debe a que es bueno seguir una gestión de que parte trata con que tipos, pongamos un ejemplo.

En mi caso, quiero que mis controladores trabajen solo con dto's, que mis servicios realicen los mapeos y que mis repositorios solo trabajen con el modelo.

\pagebreak

### Controlador Vehículo

Una vez tenemos todos estos pasos, ya solo nos queda crear nuestro controlador, pongamos un ejemplo con nuestro controlador de vehículos.

```{.csharp .numberLines}
//VehiculoController.cs
[ApiController]
[Route("[controller]")]
public class VehiculoController : ControllerBase {

    private readonly IVehiculoService _vehiculoServicio;
    private readonly IEmpresaService _empresaServicio;
    private readonly ICombustibleService _combustibleServicio;
    private readonly IMarcaService _marcaServicio;
    private readonly IModeloService _modeloServicio;
    private readonly IConfiguration _configuration;
    private readonly JwtSettings _jwtSettings;

    public VehiculoController(IVehiculoService vehiculoServicio, ICombustibleService combustibleServicio, IConfiguration configuration, IOptions<JwtSettings> options, IEmpresaService empresaServicio, IMarcaService marcaServicio, IModeloService modeloServicio) {
        _vehiculoServicio = vehiculoServicio;
        _combustibleServicio = combustibleServicio;
        _configuration = configuration;
        _jwtSettings = options.Value;
        _empresaServicio = empresaServicio;
        _marcaServicio = marcaServicio;
        _modeloServicio = modeloServicio;
    }

    //GET
    [HttpGet("getVehiculo")]
    public async Task<ActionResult<VehiculoDTO>> GetVehicle(int id) {

        try {
            string conn = await _empresaServicio.GenerarCadenaDeConexionAsync((int)JwtUtil.ObtenerDatosEmpresaPeticion(Request.Headers, _configuration, _jwtSettings));
            VehiculoDTO vehiculo = await _vehiculoServicio.GetById(id, conn);

            if (vehiculo == null) {
                return StatusCode(404, "No se encuentra el vehículo buscado");
            } else {
                return StatusCode(200, vehiculo);
            }

        } catch (Exception ex) {
            return StatusCode(401, "Token no valido o inexistente");
        }
    }
}
```

Hagamos un repaso sobre este código:

Obtenemos un vehículo de nuestro servicio, el cual puede ser null, es decir, devolver un dato vacío, por lo que siempre debemos comprobar los errores y devolver el código de error para un correcto manejo desde el Front-End.

En caso de que el dato devuelto no sea vacío, haremos uso de una DTO para devolver los datos necesarios, ya que al hacer peticiones a bases de datos, estaríamos devolviendo el objeto entero, el cual contiene información que para el usuario es totalmente irrelevante.

### Inyección

Cabe destacar que todo esto te dará un error si no inyectamos nuestro contexto ni nuestros repositorios y servicios, por lo que necesitaremos añadir unas cuantas líneas a nuestro program.cs.

```{.csharp .numberLines}
//program.cs
builder.Services.AddDbContext<ContextoDB>(item => item.UseSqlServer(configuration.GetConnectionString("cadena")));
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
builder.Services.AddScoped<IVehiculoService, VehiculoService>();
```

Por cada servicio que añadamos, debemos inyectarlo también.

"cadena" es un string que tenemos guardado en nuestro appsettings.json y nos sirve precisamente para poder asignar nuestra cadena de conexión, ya que un contexto sin lugar al que apuntar no nos sirve de nada.

```{.json .numberLines}
//appSettings.json
"ConnectionStrings": {
  "cadena": "server=user;database=database;Trusted_Connection=True;TrustServerCertificate=True"
},
```

Trusted_Connection hace referencia a que estamos usando el certificado de windows, pero será necesario cambiar esto y añadir nuestro usuario y contraseña para subir el proyecto y que funcione desde un hosting

### Control de CORS

A la hora de intentar comunicar nuestro Front-End con la API, lo más probable que es que recibamos errores de CORS, por lo que también deberemos añadir en nuestro program.cs las siguientes líneas.

```{.csharp .numberLines}
//program.cs
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(
        policy => {
            policy.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod();
        });
});
```

Por defecto, no nos interesa añadir restricciones a la aplicación, queremos aceptar a todos los usuarios sin importar el origen, cabeceras ni peticiones que realicen.

\pagebreak

### Manejo de Swagger

El swagger es una herramienta realmente útil a la hora de probar nuestra API cuando no tenemos Front-End creado, nos permite acceder a cada uno de nuestros EndPoints(Funciones por cada uno de nuestros controladores) y así comprobar su funcionamiento, el uso es similar a PostMan.

Sin embargo, Swagger por defecto hay cosas que no implementa, y nosotros deberemos añadirlas, y la principal es el uso de token.

En nuestra aplicación, cada peticion realizada consultará las cabeceras en busca de un token, el cual pasará por un proceso de validación, y si desde nuestras peticiones por Swagger no incluimos token, siempre recibiremos errores, por lo que para activar esta opción, deberemos escribir las siguientes líneas.

```{.csharp .numberLines}
//program.cs
builder.Services.AddSwaggerGen(opt =>
{
    opt.SwaggerDoc("v1", new OpenApiInfo { Title = "Titulo", Version = "v1" });
    opt.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        In = ParameterLocation.Header,
        Description = "Please enter token",
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        BearerFormat = "JWT",
        Scheme = "bearer"
    });

    opt.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type=ReferenceType.SecurityScheme,
                    Id="Bearer"
                }
            },
            new string[]{}
        }
    });
});
```

Y también necesitaremos establecer el MiddleWare, que será el que compruebe la existencia del token en nuestras peticiones.

```{.csharp .numberLines}
//Program.cs 
//Validación del token
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
     options.TokenValidationParameters = new TokenValidationParameters {
         ValidateIssuer = true,
         ValidateAudience = false,
         ValidateLifetime = true,
         ValidateIssuerSigningKey = true,
         ValidIssuers = new[] { configuration["JwtSettings:Issuer"], configuration["JwtSettings:Audience"] },
         ValidAudiences = new[] { configuration["JwtSettings:Issuer"], configuration["JwtSettings:Audience"] },
         IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(configuration["JwtSettings:KeySecret"])),
         ClockSkew = TimeSpan.Zero
     });

builder.Services.AddAuthorization(options => {
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

Esto hará que las proximas veces que se abra el Swagger, contemos con una opción para añadir token en las peticiones que realicemos.

Sin embargo, actualmente no contamos con nuestra creación de token, por lo que lo dejaremos para más adelante.

### Personalización de Swagger

Otro dato a destacar, es que el Swagger acepta hojas de estilo, por lo que podemos ponerlo en modo oscuro si nos resulta más cómodo consultando esta página &rarr; [Swagger en modo oscuro](https://dev.to/amoenus/turn-swagger-theme-to-the-dark-mode-4l5f) &rarr; [https://dev.to/amoenus/turn-swagger-theme-to-the-dark-mode-4l5f](https://dev.to/amoenus/turn-swagger-theme-to-the-dark-mode-4l5f).

\pagebreak

### DTO'S

A la hora de hacer consultas y devolver datos, anteriormente dijimos que no es rentable el devolver el objeto con todas sus propiedades, por lo que debemos crear DTO's para cada objeto que queramos devolver, de hecho, en algunas ocasiones, nos interesará devolver el mismo objeto pero con diferentes propiedades dependiendo de la petición que se realice, esto se hace estableciendo las propiedades que variarán como nullables (permiten registrar el valor null), ya que en ocasiones, simplemente no nos es útil un valor del objeto y no queremos que de un error por no ponerlo, aunque otra opción es crear otra dto para el mismo objeto, cada persona decide su forma de trabajar.

```{.csharp .numberLines}
//VehiculoDTO.cs
public class VehiculoDTO
{
    public int? Id { get; set; }
    public string? Matricula { get; set; }
    public int? IdMarca { get; set; }
    public string? Marca { get; set; } 
    public int? IdModelo { get; set; }
    public string? Modelo { get; set; }
    public int? Id_TipoCombustible { get; set; }
    public string? Combustible { get; set; }
}
//MarcaDTO.cs
public class MarcaDTO
{
    public int Id { get; set; }
    public string? Nombre {  get; set; }
    public string? Observaciones { get; set; }
    public string? Codigo { get; set; }
    public int? Id_Empresa { get; set; }
}
//ModeloDTO.cs
public class ModeloDTO
{
    public int? Id { get; set; }
    public string? Nombre { get; set; }
    public string? Observaciones { get; set; }
    public string? ReferenciaExterna {  get; set; }
    public int? Id_Empresa { get; set; }
    public int? Id_Marca { get; set; }
}
//CombustibleDTO.cs
public class CombustibleDTO
{
    public int? Id { get; set; }
    public string? Nombre { get; set; }
    public int IdEmpresa { get; set; }
}
```

### Login

Pasaremos a nuestro EndPoint de login, el cual hace uso de un servicio y un repositorio, sin embargo, una vez enseñada la estructura básica, es más que evidente que para hacer el login, vamos a hacer uso de un GetAll en busca del listado de usuarios, el cual nos devolverá un IQueryable que consultaremos con un where, haciendo búsqueda por su Email.

Una vez tengamos el usuario, debemos comprobar la contraseña, pero en la base de datos no guardamos contraseñas en texto plano sino que están encriptadas, por lo tanto, será necesario encriptar la contraseña que el usuario pase en el inicio de sesión y compararla con la contraseña asociada al Email en la tabla de usuarios, en caso de ser correcto, daremos acceso, generaremos el token y lo devolveremos.

```{.csharp .numberLines}

//LoginController.cs
[HttpGet("login")]
public async Task<ActionResult<UsuarioEmpresaDTO>> CheckUser(string email, string pass)
{
    try
    {
        Usuario user = await _sesionServicio.CheckUser(email, pass);
        EmpresasActiva empresaActiva = await _empresaActivaServicio.CheckEmpresa(user.Id);
        Empresa empresa = await _empresaServicio.CheckEmpresa(empresaActiva.IdEmpresa);
        if (user == null || empresaActiva == null)
        {
            return StatusCode(404, "Error al iniciar sesión");
        }
        else
        {
            JwtSettings settings = new JwtSettings(
            keySecret: _configuration.GetSection("JwtSettings").GetSection("KeySecret").Value,
            audience: _configuration.GetSection("JwtSettings").GetSection("Audience").Value,
            issuer: _configuration.GetSection("JwtSettings").GetSection("Issuer").Value);

            (string token, DateTime fechaExpiracion) = JwtUtil.GenerateJwtEmpresaToken(email, user.GuidRegistro, user.Id, empresa.GuidRegistro, empresaActiva.IdEmpresa, settings);

            UsuarioEmpresaDTO userDTO = new UsuarioEmpresaDTO(user.Nombre, user.Email, empresaActiva.IdUsuario, empresaActiva.IdEmpresa, token, fechaExpiracion, user.GuidRegistro, empresa.GuidRegistro);
            return userDTO;
        }
    }
    catch (Exception ex)
    {
        return StatusCode(400, ex.Message);
    }
    {
    }
}
```

En este controlador hay más cosas de las explicadas, y esto se debe a que necesitarmeos datos como la empresa activa del usuario (ya que se implementará un sistema para cambiar la empresa activa, y con ello, los datos mostrados), y vemos que para la generación del token, hacemos uso de JwtSettings, cuyas 3 propiedades son consultadas en nuestro appSettings.json, el issuer es el emisor, el audience es el destinatario y el keySecret es el código que se utilizará para firmar ese token, muy importante mantenerlo en secreto.

```{.json .numberLines}
//appSettings.json
"JwtSettings": {
  "KeySecret": "example",
  "Issuer": "example",
  "Audience": "example"
},
```

\pagebreak

En cuanto a la función que necesitamos para generar el token, es esta.

```{.csharp .numberLines}
//JwtUtil.cs
    public static (string, DateTime) GenerateJwtEmpresaToken(string email, string guidUsuario, int idUsuario, string guidEmpresa, int idEmpresa, /*string urlComunidad,*/
        JwtSettings jwtSettings, bool? esAdministador = false) {


        // Generamos claims en funcion de los parámetros
        List<Claim> claims = new() {
            new Claim(JwtRegisteredClaimNames.Email, email),
            new Claim(Constantes.GUID_USUARIO, guidUsuario),
            new Claim(Constantes.ID_USUARIO, idUsuario.ToString()),
            new Claim(Constantes.GUID_EMPRESA, guidEmpresa),
            new Claim(Constantes.ID_EMPRESA, idEmpresa.ToString()),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            //new Claim(., urlComunidad)
        };

        if (esAdministador == true) { 
            claims.Add(new Claim(Constantes.ESADMINISTRADOR, "ESADMINISTRADOR"));
        } else {
            claims.Add(new Claim(Constantes.ESUSUARIO, "ESUSUARIO"));
        }


        SymmetricSecurityKey keySecret = new(Encoding.UTF8.GetBytes(jwtSettings.KeySecret));
        SigningCredentials credentials = new(keySecret, SecurityAlgorithms.HmacSha256);

        DateTime expiration = DateTime.Now.AddHours(10);   

        JwtSecurityToken token = new(
           issuer: jwtSettings.Issuer,
           audience: jwtSettings.Audience,
           claims: claims,
           expires: expiration,
           signingCredentials: credentials);

        return (new JwtSecurityTokenHandler().WriteToken(token), expiration);
    }
    

```

Mediante claims estamos añadiendo aquello que buscamos que nuestro token contenga, de esa forma podemos hacer que nuestro token indique si el usuario logeado es o no administrador.

### Cambio dinámico de nuestra cadena de conexión

Al tratar con varias bases de datos en entity framework, es muy importante saber como funcionan el contexto y los modelos, ya que, cuando un contexto apunta a una base de datos, sus estructuras deben ser idénticas, es decir, no puede haber ninguna diferencia ni en nombres de tablas, ni en tipos de datos...

Esto, a pesar de verse como algo complicado de mantener, permite otra cosa, y es la utilización del mismo contexto con múltiples bases de datos cuya estructura interna sea idéntica (lo cual vamos a aprovechar para este proyecto).

Lo complicado será manejar las instancias del contexto, ya que nuestra idea es crear instancias de contextos que apunten a diferentes bases de datos en tiempo de ejecución, y para ello, necesitaremos hacer lo siguiente.

```{.csharp .numberLines}
//context.cs
public partial class ContextoDB : DbContext
{

public ContextoDB(string cadenaConexion) {
    _cadenaConexion = cadenaConexion;
}

private DbContextOptionsBuilder _optionBuilder;

private string? _cadenaConexion = string.Empty;



protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
  {
    
    
    if (!optionsBuilder.IsConfigured)
    {
        optionsBuilder.UseSqlServer(_cadenaConexion);
    }

    _optionBuilder = optionsBuilder;
  }
}
```

Es decir, además de los constructores que se generan al hacer el scaffold (creación del contexto a través del comando &rarr; Database-First), debemos crear un constructor que acepte una cadena de conexión, necesitaremos el optionsBuilder para establecer configuracion, y definiremos un string vacío, el cual nos será útil para almacenar nuestra cadena de conexión.

Tras esto, sobreescribiremos el método onConfiguring (este se encarga de establecer la forma en la que la base de datos se relaciona con el contexto), por lo que en caso de no haber configuración previa, simplemente conectaremos el contexto con la base de datos a la que apunte la cadena de conexión, pero ya veremos como generaremos la cadena más adelante.

Ahora deberemos volver a nuestra unidad de trabajo,  y recordemos que teníamos un constructor que usaba el contexto, el cual vamos a mantener debido a que no siempre vamos a querer cambiar la cadena de conexión, tenemos una cadena predefinida y esa seguirá siendo usada.

```{.csharp .numberLines}
//UnitOfWork.cs
public class UnitOfWork : IUnitOfWork, IDisposable {
    private readonly RegistroGeneralContext _context;
    private string _cadenaConexion;

    public UnitOfWork(RegistroGeneralContext context) {
        _context = context;
    }

    public UnitOfWork(string? cadenaConexion) {
        _cadenaConexion = cadenaConexion;
        _context = new RegistroGeneralContext (_cadenaConexion);
    }
}
```

Crearemos otro constructor el cual acepte la cadena de conexión, y genere un nuevo contexto el cual apunte a una base de datos distinta.

Y ahora simplemente, en nuestros servicios necesitaremos recibir la cadena de conexión en nuestras funciones para poder hacer que estas se realicen en x base de datos.

```{.csharp .numberLines}
//VehiculoService.cs
public async Task AddVehiculo(VehiculoDTO vehiculoDTO, string cadenaConexion, int idCreador) {
    using (IUnitOfWork repositorioEspecifico = new UnitOfWork(cadenaConexion)) {
        Vehiculo vehiculo = _mapper.Map<Vehiculo>(vehiculoDTO);
        await repositorioEspecifico.VehiculoRepositorio.Insert(vehiculo, idCreador, cadenaConexion);
        repositorioEspecifico.SaveChanges();
    }
}
```

Se crea una nueva instancia de la unidad de trabajo la cual apunta, como ya vimos con el constructor, crea una nueva instancia del contexto apuntando a x base de datos, por lo que usaremos esa instancia para hacer las proximas operaciones (este proceso debe ser repetido en todas y cada una de nuestras funciones en los servicios excepto los servicios relacionados con el manejo de las empresas, recordemos que los datos de las empresas ya figuran en la base de datos a la que apuntamos por defecto).

Sin embargo, estamos usando interfaces, por lo que las interfaces también deben ser cambiadas indicando que reciben un string llamado cadenaConexion.

Una vez hecho esto, solo queda generar ese string.

Para ello, necesitamos añadir una función a nuestro servicio de empresas con nombre GenerarCadenaDeConexionAsync, obteniendo este un id.

```{.csharp .numberLines}
//EmpresasService.cs
public async Task<string> GenerarCadenaDeConexionAsync(int id)
{

    string cadenaDeConexion = "";

    var empresa = await GetById(id);

    string baseActiva = empresa.BaseActiva;
    string conn = _configuration.GetConnectionString("cadena");

    int igual1 = conn.IndexOf('=');
    int igual2 = conn.IndexOf('=', igual1 + 1) + 1;

    int punto1 = conn.IndexOf(';');
    int punto2 = conn.IndexOf(';', punto1 + 1);

    int length = punto2 - igual2;

    string oldConn = conn.Substring(igual2, length);
    cadenaDeConexion = conn.Replace(oldConn, baseActiva);

    return cadenaDeConexion;
}
```

Como veremos, estamos realizando una llamada a una función que se encuentra dentro del mismo servicio en que definimos esta, por lo que nos resulta bastante útil.

Buscaremos nuestra empresa por id y accederemos a su campo de base activa, esto es importante hacerlo por que el nombre de nuestras bases de datos corresponde con los valores de base activa, por lo que una vez hecho esto, solo debemos jugar con la cadena de conexión para conseguir reemplazar el nombre de la base de datos por el nombre de la base de datos a la que queremos apuntar.

Una vez tenemos esto hecho, tan solo queda llamar a esta funcion desde el constructor, y pasar este valor en las peticiones que hagamos.

\pagebreak

```{.csharp .numberLines}
//VehiculoController.cs
[HttpGet("getVehiculo")]
public async Task<ActionResult<VehiculoDTO>> GetVehicle(int id) {

    try {
        string conn = await _empresaServicio.GenerarCadenaDeConexionAsync((int)JwtUtil.ObtenerDatosEmpresaPeticion(Request.Headers, _configuration, _jwtSettings));
        VehiculoDTO vehiculo = await _vehiculoServicio.GetById(id, conn);

        if (vehiculo == null) {
            return StatusCode(404, "No se encuentra el vehículo buscado");
        } else {
            return StatusCode(200, vehiculo);
        }

    } catch (Exception ex) {
        return StatusCode(401, "Token no valido o inexistente");
    }
}
```

Probablemente os preguntéis que es esto

```{.csharp .numberLines}
//VehiculoController.cs
(int)JwtUtil.ObtenerDatosEmpresaPeticion(Request.Headers, _configuration, _jwtSettings)
```

\pagebreak

Esto es una función que tenemos en una clase de utilidad la cual nos permite obtener el id de empresa en base a un token que le pasemos, por lo que mirará entre sus claims hasta que encuentre el id de empresa y simplemente lo devolverá

```{.csharp .numberLines}
//JwtUtil.cs 
public static int? ObtenerDatosEmpresaPeticion(IHeaderDictionary headers, IConfiguration configuration, JwtSettings jwtSettings, bool validarToken = true) {
    int? idEmpresa = null;
    try {
        if (headers.ContainsKey("Authorization")) {
            JwtSecurityToken? validatedToken = CheckToken(headers["Authorization"].ToString()[7..], configuration, jwtSettings, validarToken);
            if (validatedToken != null) {
                idEmpresa = int.Parse(validatedToken.Claims.FirstOrDefault(x => x.Type == Constantes.ID_EMPRESA)?.Value ?? "-1");
            }
        }
    } catch (Exception ex) {
        return null;
    }
    return idEmpresa;
}
```

Esta función la he obtenido buscando información sobre los tokens jwt por internet, por lo que lo único que he tenido que hacer ha sido adaptarla a mis necesidades.

\pagebreak

## Front-End

Una vez tenemos el Back-End Terminado, podemos comenzar a realizar el apartado visual, recordemos que estamos haciendo uso de Vuejs con vuetify, por lo que el sistema que usaremos será el de componentes y vistas, sin embargo, aquí lo más destacable no será el código html ni css, sino el código JavaScript, es decir, como realizamos nuestras peticiones y como manejamos los datos obtenidos.

### Peticiones axios

Haremos uso de las peticiones que componen el patron CRUD, es decir, usaremos GET, POST, PUT y DELETE

La estructura de una petición es realmente simple, contamos con la cadena de conexión, los parámetros que enviaremos, los cuales son opcionales, un then en caso de éxito, y un catch en caso de fracaso.

```{.javascript .numberLines}
//Login.vue
 axios.get('https://localhost:7030/Login/login', {
          params: { email: email, pass: pass },
        })
        .then(function (response) {
          localStorage.setItem("token", JSON.stringify(response.data));

          $router.push("/");
        })
        .catch(function (error) {
          console.log("Error al tratar de hacer login");
        });
```

Sin embargo, a la hora de realizar peticiones, dijimos que era importante enviar siempre el token en las cabeceras, ya que en caso contrario, recibiríamos un error &rarr; (401), Unauthorized.

En ese caso, será necesario añadir las siguiente línea en nuestro código antes de la petición.

```{.javascript .numberLines}
const token = JSON.Parse(localstorage.GetItem('token'));
axios.defaults.headers.common["Authorization"] = 'Bearer ${token.token}';
```

Como en nuestro token vamos a guardar muchos datos que nos van a ser útiles, token.token hace referencia al código generado que debe ser validado, pero también guardaremos datos como token.fechaExpiracion, aunque esta viene dentro del propio token.

Hemos enseñado como mandar un token en las cabeceras de nuestra petición, sin embargo, ¿es realmente buena idea hacer esto?.

Bueno, dado a que nuestro token tiene un tiempo de vida limitado, eso significa que en cuanto caduque, dejará de ser válido y nuestra sesión debe expirar, por lo tanto, no es recomendable controlar en cada petición una por una si el token es o no válido, para esto, hacemos uso de los interceptores de axios.

Un interceptor establece el patrón de funcionamiento para todas y cada una de nuestras peticiones y respuestas, por lo que contaremos con un interceptor para peticiones, y otro para respuestas.

```{.javascript .numberLines}
//Axios.ts
axios.interceptors.request.use(
    function (config) {

      const token = localStorage.getItem("token");

      if (token) {
        const tokenParse = JSON.parse(token);
        config.headers.Authorization = 'Bearer ${tokenParse.token}';
      }
      return config;
    },
    function (error) {
      return Promise.reject(error);
    }
);
```

En todas nuestras peticiones, vamos a comprobar si tenemos token en el localstorage y vamos a enviarlo con las cabeceras.

```{.javascript .numberLines}
//Axios.ts 
import axios from "axios";
import router from "../router";

export default function axiosSetUp() {
  axios.defaults.baseURL = "https://localhost:7030/";
axios.interceptors.response.use(
    function (response) {
     
      return response;
    },
    async function (error) {
     
      const originalRequest = error.config;

      const tokenNoParse = localStorage.getItem("token");
      if (tokenNoParse) {
        const token = JSON.parse(tokenNoParse);
        const fecha = new Date(token.fechaExpiracion);

        if (error.response.status === 401 && (Date.now() - fecha.getTime() >= 300000)) {
          localStorage.clear();
          router.push("/login");
          return Promise.reject(error);
        } else if (error.response.status === 401 && (Date.now() - fecha.getTime() < 300000)) {
          renovarToken();
          originalRequest._retry = true;
          return axios(originalRequest);
        }
        return Promise.reject(error);
      }
    }
);
}
```

Y en todas nuestras respuestas vamos a comprobar si hay errores, por lo que en caso de obtener un error con http status 401, es decir, unauthorized, y el tiempo que nuestro token ha estado caducado sea mayor a 5 minutos, no vamos a permitir renovarlo, nos mandará al login.

Sin embargo, si el token caducó hace menos de 5 minutos, trataremos de renovarlo, siendo así que en caso de conseguirlo, repetiremos de nuevo la petición que nos dio el error unauthorized instantáneamente.

```{.javascript .numberLines}
//Axios.ts 
const renovarToken = () => {
    const tokenSinParse = localStorage.getItem("token");
    if (tokenSinParse) {
      const token = JSON.parse(tokenSinParse);
      const params = {
        params: {
          nombre: token.nombre,
          email: token.email,
          userId: token.userId,
          idEmpresa: token.idEmpresa,
          guidEmpresa: token.guiD_Empresa,
          guidUsuario: token.guiD_Usuario,
        },
      };
      axios
        .get("https://localhost:7030/Login/RestoreToken", params)
        .then((response) => {
          localStorage.setItem("token", JSON.stringify(response.data));
          localStorage.setItem("refresh", JSON.stringify(response.data));
        })
        .catch((error) => {
          localStorage.clear();
          router.push('/login')
        });
    }
  }
  
```

\pagebreak

Para nuestra función de renovar token, simplemente usaremos el token caducado con los datos válidos, y generaremos uno nuevo, es decir, tan solo cambiará el token en concreto (la cadena de caracteres serializada), y la fecha de expiración, por lo que al verificar el token de nuevo, volvemos a tener el token funcional durante el tiempo de vida del mismo.

Finalmente, para añadir el interceptor, entramos en nuestro main.ts, importamos el archivo y hacemos una llamada a la funcion.

```{.javascript .numberLines}
//Main.ts
import axiosSetup from '../src/helpers/axios';

axiosSetup();
```

## Apartados destacados

### Cambio de empresa activa

Como se dijo en un principio, contamos con varias empresas en nuestra base de datos principal, las cuales contienen sus datos en otras bases de datos, referenciado mediante el campo base activa.

Ahora solo nos queda añadir la funcionalidad del cambio de empresa, y la idea es simple.

Necesitaremos un repositorio exclusivo para este proceso, por lo que necesitaremos interfaz, repositorio, otra interfaz, servicio y un controlador, aunque como anteriormente ya tenía un controlador de empresas creado, no será necesario añadir uno nuevo.

Las funciones necesarias serán las de GetById y GetAll, ya que la idea es recibir 2 id's, uno de la empresa activa actual, y otro de la empresa a la que queremos cambiar.

```{.csharp .numberLines}
//IRepositorioCambioEmpresa.cs
public interface IRepositorioCambioEmpresa
{
    Task<Empresa> GetById(int id);
    Task CambioEmpresaActiva(int empresaAnterior, int empresaSiguiente);
    Task<IQueryable<EmpresasActiva>> GetAllEmpresasActivas();
    public int saveChanges();
}

//ICambioEmpresaActivaServicio.cs
public interface ICambioEmpresaActivaServicio
{
    Task<Empresa> GetById(int id);
    Task CambioEmpresaActiva(int empAnterior, int empSiguiente);
    Task<IQueryable> GetAllEmpresasActivas();
}


```

Haremos un GetById de ambas para verificar la exitencia en la base de datos y tras eso, un GetAll de nuestra tabla de Empresas activas, ya que ahi aparece la empresa activa actual, cuando la tengamos, haremos un update de su campo idEmpresa y guardaremos los cambios.

Esto hará que la proxima vez que, como para conseguir la empresa activa se hace referencia a este id concretamente, al hacer un cambio del id estaremos referenciando a otro registro de la tabla de empresas.

Una vez terminado eso, queremos que se nos renueve el token para no tener que reiniciar la sesión, pues simplemente crearemos en nuestro controlador de empresas un EndPoint de cambio de empresa activa en el cual recibamos los datos para la renovación del token y los id's de las empresas, con esos id's llamaremos a los servicios, que se encargarán de conectar con el repositorio y realizar lo que nombramos anteriormente, y una vez hecho eso, hacemos una llamada a nuestra función de generar token pero esta vez con los datos actualizados.

```{.csharp .numberLines}
//EmpresaController.cs
public async Task<ActionResult<UsuarioEmpresaDTO>> CambiarEmpresaActiva(int empAnterior, int empSiguiente, string nombre, string email, string guidUsuario, int userId)
{
    try
    {
        string conn = await _empresaServicio.GenerarCadenaDeConexionAsync((int)JwtUtil.ObtenerDatosEmpresaPeticion(Request.Headers, _configuration, _jwtSettings));

    }
    catch (Exception ex)
    {
        return StatusCode(401, "Token no valido o inexistente");
    }

    try
    {
        Empresa empresaAnterior = await _cambioEmpresaActivaServicio.GetById(empAnterior);
        Empresa empresaSiguiente = await _cambioEmpresaActivaServicio.GetById(empSiguiente);

        if (empresaAnterior is null || empresaSiguiente is null)
        {
            return StatusCode(404, "Empresa o empresas no encontradas");
        }
        else
        {
            try
            {
                await _cambioEmpresaActivaServicio.CambioEmpresaActiva(empresaAnterior.Id, empresaSiguiente.Id);

                JwtSettings settings = new JwtSettings(
                    keySecret: _configuration.GetSection("JwtSettings").GetSection("KeySecret").Value,
                    audience: _configuration.GetSection("JwtSettings").GetSection("Audience").Value,
                    issuer: _configuration.GetSection("JwtSettings").GetSection("Issuer").Value);

                (string token, DateTime fechaExpiracion) = JwtUtil.GenerateJwtEmpresaToken(email, guidUsuario, userId, empresaSiguiente.GuidRegistro, empSiguiente, settings);

                UsuarioEmpresaDTO userDTO = new UsuarioEmpresaDTO(nombre, email, userId, empSiguiente, token, fechaExpiracion, guidUsuario, empresaSiguiente.GuidRegistro);
                return userDTO;
            }
            catch (Exception ex)
            {
                return StatusCode(400, ex.Message);
            }
        }
    }
    catch (Exception ex)
    {
        return StatusCode(400, ex.Message);
    }
}
```

\pagebreak

Desde nuestro FrontEnd, solo debemos hacer una llamada al EndPoint pasando los datos, y en el .then guardaremos en el localStorage un token del response.data.

```{.javascript .numberLines}
//ProfileView.vue
Confirm() {
      const token = JSON.parse(localStorage.getItem('token'));
      const url = 'https://localhost:7030/Empresa/CambiarEmpresaActiva';
      const params = {
        params: {
            empAnterior:token.idEmpresa,
            empSiguiente:this.empresaSelect.id_Empresa,
            nombre:token.nombre,
            email:token.email,
            guidUsuario:token.guiD_Usuario,
            userId:token.userId
        }

      }      
    
      axios.put(url)
      .then((response) => {
        localStorage.setItem('token', JSON.stringify(response.data));
        $router.go(0);
      })
      .catch((error) => {
        console.log(error.response);
    })
  }
```

De esta forma, tras realizar la petición, en caso de éxito refrescaremos la página.

### Paginación, filtrado, búsqueda y ordenación

Si bien con un datatable (Vuetify) y con un datagrid (DevExtreme) podemos conseguir esto de forma automática y sin necesidad de preocupaciones, hay ciertos casos en los que necesitamos realizar estos procesos desde el Back-End, pongamos un ejemplo.

En un caso en el cual nuestras consultas vayan a devolver pocos registros de nuestras tablas, o mejor dicho, en el caso en que nuestras tablas contengan pocos registros, nos es totalmente indiferente que hayamos una petición que devuelva 100, 200 o incluso 500 registros, el datatable o el datagrid se encargará de hacer los procesos nombrados sin ningún problema.

Sin embargo, hay casos en los que una consulta no te va a devolver pocos registros, sino que puede llegar a devolver decenas de miles de registros, y trabajar con esos datos reduciría el rendimiento en gran cantidad.

Por eso, si dejamos esta tarea al Back-End podemos no devolver 10000 registros, sino devolver los 20 primeros, posteriormente otros 20 y así constántemente.

Toda la información sobre esto, ha sido sacada de [CodeMaze](https://code-maze.com/net-core-series/) &rarr; [https://code-maze.com/net-core-series/](https://code-maze.com/net-core-series/) en el apartado "Advanced ASP.NET Core Web API Concepts".

### Control de la auditoría

Para llevar un correcto seguimiento sobre lo que sucede con los datos almacenados en nuestras bases de datos, es recomendable hacer uso del control de auditoría, y esto no es más que indicar creación, actualización y borrado de registros, mostrando la fecha en la que se realizó y quién lo realizó (recordemos que no usamos borrado físico de datos, por lo que al borrar un registro, este simplemente no será devuelto en las consultas pero permanecerá en nuestra base de datos).

Para esto será necesario enviar en las peticiones post, put y delete, el id del usuario que realizó la petición, y la fecha simplemente podemos cambiarla desde nuestro Back-End con un Datetime.Now.

Sin embargo, es importante destacar que para realizar el control de la auditoría, necesitaremos hacer que nuestros repositorios específicos (que recordemos que eran instancias de un repositorio general al cual le indicabamos un tipo) pasen a convertirse en repositorios independientes, los cuales agruparemos en nuestra unidad de trabajo, esto se debe a que a la hora de trabajar con genéricos, no podemos acceder a las propiedades de una clase que no conocemos, y aunque existe la posibilidad de hacer uso de un repositorio genérico que nos permita realizar control de auditoría, es un proceso más complejo y decidí hacerlo de esta otra forma, cabe recalcar que ambas formas de trabajar son correctas, sin embargo, la forma usada en este caso implica el uso de más código.

```{.csharp .numberLines}
//VehiculoRepository.cs
public async Task DeleteById(int id, int idBorrador)
{
    var elem = await _entity.FindAsync(id);
    if (elem != null)
    {
        elem.IdBorrador = idBorrador;
        elem.FechaBorradoLogico = DateTime.Now;
        elem.BorradoLogico = true;
        _entity.Update(elem);
    }
}
```

De esta forma, como _entity apunta a la clase vehículo, podemos acceder a las propiedades idBorrador, FechaBorradoLógico y BorradoLógico.

Algo que puede parecer contradictorio, es que realmente estamos actualizando el elemento, sin embargo, tanto los nombres de las funciones, como la petición en nuestro controlador y en la llamada de axios, van a ser un put, es decir, un update, esto se debe a que lo que nos interesa es que la "intención" sea la de borrar el elemento, por ello nuestro proceso va a ser el de un borrado normal, excepto en el repositorio que como hemos visto, acabaremos haciendo un update.

### Mapeo de objetos

A la hora de devolver un objeto, hay muchas propiedades que no nos interesan devolver como por ejemplo la información de auditoría, y para ello, hicimos uso de las DTO's.

Si recibimos 10 datos de nuestra base de datos y los pasamos a DTO, no nos supondrá un problema, sin embargo, si pasamos 1000, la estrategia de un foreach que va accediendo una por una a las propiedades y creando un objeto al que se las asigna, deja de ser útil, por lo que necesitaremos de algo que optimice este proceso.

Aquí es donde Automapper entra en juego, el cual se encarga de transformar un objeto (source) a otro objeto (destination), suena a que nosotros estabamos haciendo lo mismo, sin embargo, este paquete NuGet está preparado para realizar todo este proceso de una forma mucho más eficiente.

```{.csharp .numberLines}
//Program.cs
builder.Services.AddAutoMapper(typeof(AutoMapperProfile));
```

\pagebreak

### Cómo lleva a cabo el mapeo?

La explicación es bastante simple, para poder llevar a cabo el mapeo, es necesario que en nuestra dto, las propiedades se llamen igual que en nuestro objeto base.

Sin embargo, esto no siempre es posible y automapper viene preparado para esto, ya que tenemos la posibilidad de crear perfiles "Funciones de mapeo" en las cuales definiremos como se llevará el mapeo entre dos clases en específico, es decir, una función de mapeo es el conjunto de reglas que se aplican al realizar un mapeo entre 2 clases en concreto, veamos un ejemplo.

```{.csharp .numberLines}
//FuncionesMapeos.cs
public class MapeoVehiculos_VehiculosDtoTypeConverter : ITypeConverter<Vehiculo, VehiculoDTO> {
    public VehiculoDTO Convert(Vehiculo source, VehiculoDTO destination, ResolutionContext context) {
        destination = new();

        if(source is not null) {
            destination.Id = source.Id;
            destination.Matricula = source.Matricula;
            if(source.Marca is not null) {
                destination.IdMarca = source.Marca.Id;
                destination.Marca = source.Marca.Nombre;
            }

            if(source.Modelo is not null) {
                destination.IdModelo = source.Modelo.Id;
                destination.Modelo = source.Modelo.Nombre;
            }

            if(source.TipoCombustible is not null) {
                destination.Id_TipoCombustible = source.TipoCombustible.Id;
                destination.Combustible = source.TipoCombustible.Nombre;
            }
        }

        return destination;
    }
}
```

Estas reglas solo se aplicarán para los mapeos en los que enviemos un vehiculo y querramos obtener un vehiculoDTO, y en cuanto a su funcionamiento, simplemente comprobamos que las propiedades que pueden ser null no lo sean, y establecemos la relación entre ambas clases para que a la hora de hacer el mapeo, no haya ningun error y las propiedades no puedan quedar marcadas como null.

Una vez hecho esto, debemos establecer los perfiles, es decir, cada una de las relaciones que vamos a llevar a cabo.

```{.csharp .numberLines}
//AutoMapperProfile.cs
public class AutoMapperProfile : Profile {
    public AutoMapperProfile() {
        #region MapeoVehiculos
        CreateMap<Vehiculo, VehiculoDTO>().ConvertUsing<MapeoVehiculos_VehiculosDtoTypeConverter>();
        #endregion
    }
}
```

"#Region" no es más que una forma de agrupar código en bloques, si bien en archivos pequeños no es necesario, cuando tienes mucho código es una buena forma de organizarlo para localizar rápidamente aquello a lo que quieres acceder, se pueden contraer y expandir las regiones.

Y como estamos trabajando con inyección de dependencias, en nuestro program.cs debemos poner el servicio de nuestro automapper.

```{.csharp .numberLines}
//Program.cs
builder.Services.AddAutoMapper(typeof(AutoMapperProfile));
```

Ahora, en nuestro controlador de vehiculos no va a ser necesario realizar la creación y asignación de valores en la DTO.

```{.csharp .numberLines}
//VehiculoController.cs
[HttpGet("getAllVehiculos")]
public async Task<ActionResult<List<VehiculoDTO>>> GetAllVehiculos([FromQuery] VehiculoParams parameters, int idEmpresa) {

    try {
        string conn = await _empresaServicio.GenerarCadenaDeConexionAsync((int)JwtUtil.ObtenerDatosEmpresaPeticion(Request.Headers, _configuration, _jwtSettings));


        (List<VehiculoDTO> listadoVehiculosDto, MetadataDto metadataDto) vehiculos = await _vehiculoServicio.GetVehiculosPaginated(parameters, conn, idEmpresa);

        Response.Headers.Append("X-Pagination", JsonSerializer.Serialize(vehiculos.metadataDto));

        return StatusCode(200, vehiculos.listadoVehiculosDto);
    } catch (Exception ex) {
        return StatusCode(401, "Token no valido o inexistente");
    }
}
```

Ahora simplemente haremos la llamada a nuestro servicio, que es donde realizaremos el mapeo de los objetos (no es obligatorio realizarlo en el servicio, puede ser desde el controlador, o incluso desde el repositorio).

```{.csharp .numberLines}
//VehiculoService.cs
public async Task<(List<VehiculoDTO> listadoVehiculosDto, MetadataDto metadataDto)> GetVehiculosPaginated(VehiculoParams parameters, string cadenaConexion, int idEmpresa) {
    PagedList<Vehiculo> arrayVehiculos = new PagedList<Vehiculo>();

    using (IUnitOfWork repositorioEspecifico = new UnitOfWork(cadenaConexion)) {
        arrayVehiculos = repositorioEspecifico.VehiculoRepositorio.GetAllPaginated(parameters, idEmpresa);

    }

    List<VehiculoDTO> vehiculoDTOs = _mapper.Map<List<VehiculoDTO>>(arrayVehiculos);
    MetadataDto metadataDto = new MetadataDto(arrayVehiculos.TotalCount, arrayVehiculos.PageSize, arrayVehiculos.CurrentPage, arrayVehiculos.TotalPages, arrayVehiculos.HasNext, arrayVehiculos.HasPrevious);

    return (vehiculoDTOs, metadataDto);
}
```

Esto es todo lo que necesitamos, y hemos conseguido optimizar nuestras peticiones gracias al uso de automapper.

También funciona en el caso contrario, es decir, nosotros insertamos un objeto vehiculoDTO, y ese objeto lo mapeamos al tipo vehiculo.

### Inclusión de clases mediante Entity Framework

Si visteis el anterior apartado, os pudo haber generado curiosidad el mapeo de vehiculo a vehiculoDTO, ya que estabamos accediendo a propiedades dentro de las propiedades, como en este ejemplo.

```{.csharp .numberLines}
//FuncionesMapeos.cs
destination.Marca = source.Marca.Nombre;
```

Sin embargo, en nuestro modelo de vehiculo no contamos con una propiedad que se llame Marca, es como si tuviesemos una clase dentro de la propia clase de vehiculo.

Esto que habéis visto, no es más que un método que tiene Entity Framewok de enviar un objeto que contenga propiedades que no le pertenecen.

Lo explicaremos más adelante, pero para comprender un poco a lo que nos estamos refiriendo, esto nos permitiría enviar a nuestra tabla en el front, objetos vehiculoDTO que contengan tanto el id como el nombre de la marca, modelo y combustible, pero ahorrándonos las llamadas extra a la base de datos.

Para ello, necesitaremos modificar nuestro modelo de vehiculos y hacer lo siguiente.

```{.csharp .numberLines}
//Vehiculo.cs
public int? ModeloId { get; set; }
public int? MarcaId { get; set; }
public int? TipoCombustibleId { get; set; }


public virtual Modelo? Modelo { get; set; }
public virtual Marca? Marca { get; set; }
public virtual TiposCombustible? TipoCombustible { get; set; }
```

Entre sus propiedades vamos a añadir modelo, marca y combustible, que harán referencia a los modelos que se encuentran en nuestra base de datos.

Y en cuanto a los id's, anteriormente sus nombres eran IdMarca, IdModelo e IdTipoCombustible, sin embargo, la regla es que para que Entity sepa que estamos incluyendo modelos dentro de otro modelo, su clave foránea debe ser Nombre del modelo + identificador, por lo que deberemos cambiarlos.

Cabe recalcar que estableceremos las propiedades como nullables para que solo nos devuelva los modelos cuando lo necesitemos, aunque lo explicaremos en el siguiente ejemplo.

#### Comprobación

Ahora, para hacer que esto funcione, solo debemos hacer una cosa a la hora de hacer peticiones a nuestra base de datos.

```{.csharp .numberLines}
//VehiculoRepository
var query = _vehiculos.Include(e => e.Modelo).Include(e => e.Marca).Include(e => e.TipoCombustible).Where(e => e.IdEmpresa == idEmpresa);
```

¿Simple, no?, solo necesitamos indicar en la peticion que queremos incluir modelo, marca y tipocombustible, siendo así que cuando hagamos nuestra petición, nos devolverá el objeto o la lista de objetos con todas las propiedades tanto suyas, como las de los modelos incluidos, esto se resume en que es un join que nos permite almacenar en nuestro objeto propiedades pertenecientes a otros modelos, y si lo juntamos con el mapeo del punto anterior, no tendremos dificultad alguna para trabajar con un objeto que tenga tantas propiedades, simplemente nuestra función de mapeo nos lo hace automáticamente sin que tengamos que preocuparnos.

### Más funcionalidades de los mapeos

Una vez tenemos implementados nuestros mapeos, podremos usarlos para cualquier acción, ya sea la inserción, actualización y obtención de elementos.

Como nuestro controlador siempre va a trabajar con DTO's, eso significa que los mapeos nos serían útiles para transformas esas DTO's a los modelos almacenados, por lo que necesitamos poder mapear de modelos a DTO's y de DTO's a modelos.

```{.csharp .numberLines}
//AutoMapperProfile.cs
#region MapeoVehiculos
CreateMap<Vehiculo, VehiculoDTO>().ConvertUsing<MapeoVehiculos_VehiculosDtoTypeConverter>();
//Mapeo inverso
CreateMap<VehiculoDTO, Vehiculo>().ConvertUsing<MapeoVehiculosDTO_VehiculosTypeConverter>();
#endregion
```

Simplemente implementamos el mapeo inverso, de tal forma que la función creada para mapear de modelo a DTO, la volveremos a hacer pero al revés.

Esto no se aplica a todos los casos, ya que como dijimos anteriormente, si las propiedades de nuestro modelo y DTO coinciden en nombre, no es necesario implementar función de mapeo, por lo que nuestro código se reduciría a esto.

```{.csharp .numberLines}
// AutoMapperProfile.cs
#region MapeoMarcas
CreateMap<Marca, MarcaDTO>().ReverseMap();
#endregion

#region MapeoModelos
CreateMap<Modelo, ModeloDTO>().ReverseMap();
#endregion

#region MapeoEmpresas
CreateMap<Empresa, EmpresaDTO>().ReverseMap();
#endregion
```

En este caso, como los nombres de las propiedades coinciden, para nuestras marcas, modelos  y empresas, no necesitaremos crear función de mapeo.

\pagebreak

### Uso de secuencias para la generación de los id's

Como las tablas que usaremos no cuentan con id autoincremental y no es muy buena idea ir poniendo id's al azar al insertar valores, haremos uso de una secuencia.

```{.sql .numberLines}
--SQLServer Query
CREATE SEQUENCE [dbo].[GENERACIONID] 
 AS [int]
 START WITH 100
 INCREMENT BY 1
 MINVALUE -2147483648
 MAXVALUE 2147483647
 CACHE  250 
```

En nuestro caso, estamos haciendo uso de SQLServer, por lo que desde el Management Studio, accederemos a la base de datos en la que queramos generar la secuencia (En nuestro caso es en todas) y simplemente ejecutamos la consulta, por lo que una vez hecho esto, podremos acceder a ella desde la ventana de consultas.

Sin embargo, a nosotros no nos interesa el acceder a la secuencia desde el Management Studio, sino que buscamos generar nuestro id en tiempo de ejecución al realizar el insert.

En este punto, desconozco si desde Entity Framework se puede acceder a las secuencias almacenadas en nuestra base de datos (Doy por hecho que si ya que aparece reflejado en nuestro contexto), sin embargo, lo haremos de forma manual haciendo uso de SQLClient y SQLCommands.

```{.csharp .numberLines}
//VehiculoRepository.cs
public int NextIdSequence(string conn) {
    int nextId = 0;
    using (SqlConnection connection = new SqlConnection(conn)) {
        connection.Open();
        using (SqlCommand cmd = new SqlCommand("SELECT NEXT VALUE FOR GENERACIONID", connection)) {
            nextId = (int)cmd.ExecuteScalar();
        }
        connection.Close();
    }
    return nextId;
}
```

Necesitaremos una cadena de conexión para acceder a la base de datos, por lo que si recordamos, anteriormente implementamos la forma de realizar consultas a una base de datos u otra dependiendo de nuestra empresa activa, es decir, ya contamos con la cadena de conexión necesaria.

\pagebreak

El funcionamiento es simple, tan solo abrimos una conexión, creamos el comando de consulta y lo ejecutamos, para guardar el valor en una variable y devolverla.

```{.csharp .numberLines}
//VehiculoRepository.cs
public async Task Insert(Vehiculo entity, int idCreador, string conn) {
    entity.IdCreador = idCreador;
    entity.FechaCreacion = DateTime.Now;
    entity.GuidRegistro = Guid.NewGuid().ToString().ToUpper();
    entity.Id = NextIdSequence(conn);
    _vehiculos.Add(entity);
}
```

Y posteriormente, desde  nuestra función de insert, llamaremos a esta otra función pasando la cadena.

\pagebreak

## Funciones extra

### Selector de tema

Para implementar el apartado de preferencias de color, vamos a necesitar hacer uso de las variables en nuestro css, lo que nos permite asignar un valor a cada variable, estos valores serán en nuestro caso colores.

Por otro lado, necesitaremos un archivo donde almacenar los diferentes temas con los colores que contienen cada uno.

Y finalmente, implementar el cambio mediante nuestro código, comencemos enseñando el fichero en el que guardaremos los diferentes temas.

```{.typescript .numberLines}
//themes.ts
const predeterminado  = {
    primary: "#106c4c",
    secondary: "#ccc",
    accent: "#000",
    error: "#ff0000",
}
const morado  = {
    primary: "#B03BFF",
    secondary: "#000",
    accent: "#000",
    error: "#FF8B81",
}
const naranja = {
    primary: "#f59120",
    secondary: "#000",
    accent: "#000",
    error: "#ff613d",
}


export default {
    predeterminado: predeterminado,
    morado: morado,
    naranja: naranja
}
```

Tenemos un objeto por cada tema con cada uno de los colores que utilizaremos posteriormente.

\pagebreak

```{.typescript .numberLines}
//App.vue
<script lang="ts">
import Vue from 'vue';
import themes from "@/assets/themes"
type typesThemes = 'predeterminado' | 'morado' | 'naranja'

export default Vue.extend({
  name: 'App',
  computed:{
    cssProps(){
      const current = themes[localStorage.getItem("tema") as typesThemes]
      return{
        '--primary-color' : current ? current.primary : themes['predeterminado'].primary,
        '--secondary-color' : current ? current.secondary : themes['predeterminado'].secondary,
        '--accent' : current ? current.accent : themes['predeterminado'].accent,
        '--error' : current ? current.error : themes['predeterminado'].error,
      }
    }
  },
  data: () => ({ }),
});
</script>
```

En nuestro App.vue importaremos el fichero .ts con nuestro temas, de esta forma podremos llamarlos y acceder a sus propiedades.

El truco está en usar cssProps, lo que nos permitirá declarar variables globales (justo lo que queremos).

Primero seleccionamos el tema guardado en nuestro localStorage, y una vez hecho eso definimos una variable y le asignamos el valor de la propiedad deseada en base a nuestro tema almacenado, si no tenemos ningun tema almacenado en el localStorage, simplemente usamos el tema por defecto.

Si hemos hecho esto, ahora solo quedaría escribir nuestro css de esta forma.

```{.css .numberLines}
/*styles.css*/
.botonesModal button {
    height: 3rem;
    color: white;
    font-weight: 500;
    background-color: var(--primary-color);
    border-right: 1px solid white;
    border-left: 1px solid white;
    width: 50%;
  }
```

Vayamos un poco más lejos, si partimos de la idea de que tenemos una aplicación en la que el código css es normal que se repita, como a la hora de darle estilo a botones, tablas, dialogs...

Repetir el mismo código css por cada una de nuestras vistas no es una práctica muy eficiente por nuestra parte, aquí es donde entra en juego una simple hoja de estilos independiente.

Podemos crear en nuestra carpeta de assets  un fichero css y ahí escribir los estilos que se van a repetirse en nuestra vistas, luego simplemente importamos el fichero en nuestro main.ts y somos libres de asignar esas clases a nuestras etiquetas sin importar en que fichero se encuentren, al tenerlo definido e importado, estaríamos reduciendo el código repetido en cada una de nuestras vistas o componentes.

```{.typescript .numberLines}
//Main.ts
import './assets/estilos.css';
```

### Gestión de errores

Si bien estamos haciendo uso de try catch en nuestros controladores, hecho que nos permite atrapar los errores que surjan y controlarlos a nuestra voluntad, no es una buena práctica llenar nuestros controladores con código de gestión de errores, para esto solemos hacer uso de un middleware, el cual hará (como su nombre indica) de capa intermedia entre las peticiones y podrémos establecer la lógica del manejo de errores ahí, veamos que es lo que necesitamos para implementarlo.

#### Logger

Logger nos permitirá escribir mensajes en consola o guardar dichos registros en ficheros de registro (logs) para poder llevar seguimiento de fallos y en general, del funcionamiento de nuestra api.

```{.csharp .numberLines}
//ILoggerManager.cs
public interface ILoggerManager {
    void LogInfo(string message);
    void LogWarn(string message);
    void LogDebug(string message);
    void LogError(string message);
}
```

Comenzaremos con la interfaz de nuestro Logger, la cual simplemente contará con una función por cada mensaje de log que vamos a enviar, en este caso será info, debug, warning y error.

\pagebreak

```{.csharp .numberLines}
//LoggerManager.cs
public class LoggerManager : ILoggerManager {
    private static ILogger logger = LogManager.GetCurrentClassLogger();

    public LoggerManager() {
        LogManager.Setup().LoadConfigurationFromFile(String.Concat(Directory.GetCurrentDirectory(), "/nlog.config"));
    }

    public void LogInfo(string message) {
        logger.Info(message);
    }

    public void LogWarn(string message) {
        logger.Warn(message);
    }

    public void LogDebug(string message) {
        logger.Debug(message);
    }

    public void LogError(string message) {
        logger.Error(message);
    }
}
```

Seguimos con nuestra clase que implementará dicha interfaz, y en su constructor debemos indicarle el archivo de donde obtendrá la configuración, el cual debemos crear.

\pagebreak

```{.xml .numberLines}
<!-- nlog.config -->
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <targets>
        <target name="console" xsi:type="ColoredConsole" layout="${longdate} [${whenEmpty:whenEmpty=${threadid}:inner=${threadname}}] ${level} ${logger} ${message} ${exception:format=tostring}">
            <highlight-row condition="level == LogLevel.Error" foregroundColor="Red" />
            <highlight-row condition="level == LogLevel.Warn" foregroundColor="Yellow" />
            <highlight-row condition="level == LogLevel.Debug=" foregroundColor="Blue" />
            <highlight-row condition="level == LogLevel.Info" foregroundColor="White" />
        </target>
        <target xsi:type="File" name="file" layout="${longdate} ${level} ${logger} ${message} ${exception:format=tostring}" fileName="${basedir}/logfile.log" keepFileOpen="false" encoding="iso-8859-2" />
    </targets>
    <rules>
        <logger name="*" minlevel="Info" writeTo="console" />
    </rules>
</nlog>
```

Tras esto, solo queda implementar nuestro logger en el program.cs.

```{.csharp .numberLines}
//Program.cs
var logger = app.Services.GetRequiredService<ILoggerManager>();
```

Cabe recalca que para usar logger, debemos instalar la el paquete nuGet NLog.Web.AspNetCore.

Una vez tenemos nuestro logger configurado e implementado, podemos comenzar con el middleware (tutorial obtenido de [CodeMaze](https://code-maze.com/global-error-handling-aspnetcore/) &rarr; [https://code-maze.com/global-error-handling-aspnetcore/](https://code-maze.com/global-error-handling-aspnetcore/)).

Dado que la explicación aparece en la propia página y en la guía proporcionada por el chico (yo simplemente he seguido los pasos) dejaré el código a continuación.

Dentro de nuestro directorio de modelos, crearemos una clase llamada ErrorDetails.

```{.csharp .numberLines}
//ErrorDetails.cs
public class ErrorDetails
{
    public int StatusCode { get; set; }
    public string Message { get; set; }
    public override string ToString() {
        return JsonSerializer.Serialize(this);
    }
}
```

Crearemos un directorio llamado CustomExceptionMiddleware y dentro, un fichero llamado ExceptionMiddleware.

```{.csharp .numberLines}
//ExceptionMiddleware.cs
public class ExceptionMiddleware {
    private readonly RequestDelegate _next;
    private readonly ILoggerManager _logger;

    public ExceptionMiddleware(RequestDelegate next, ILoggerManager logger) {
        _logger = logger;
        _next = next;
    }

    public async Task InvokeAsync(HttpContext httpContext) {
        try {
            await _next(httpContext);
        } catch (AccessViolationException avEx) {
            _logger.LogError($"A new violation exception has been thrown: {avEx}");
            await HandleExceptionAsync(httpContext, avEx);
        } catch (Exception ex) {
            _logger.LogError($"Something went wrong: {ex}");
            await HandleExceptionAsync(httpContext, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception) {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;

        var message = exception switch {
            AccessViolationException => "Access violation error from the custom middleware",
            _ => "Internal Server Error from the custom middleware."
        };

        await context.Response.WriteAsync(new ErrorDetails() {
            StatusCode = context.Response.StatusCode,
            Message = message
        }.ToString());
    }
}
```

\pagebreak

Tras esto, nuevamente crearemos un directorio llamado Extensions y dentro, un fichero llamado ExceptionMiddlewareExtensions.

```{.csharp .numberLines}
//ExceptionMiddlewareExtensions
public static class ExceptionMiddlewareExtensions {
    public static void ConfigureCustomExceptionMiddleware(this WebApplication app) {
        app.UseMiddleware<ExceptionMiddleware>();
    }
}
```

Para finalmente volver a nuestro Program.cs y añadir la siguiente línea.

```{.csharp .numberLines}
//Program.cs
app.ConfigureCustomExceptionMiddleware();

```

Si hemos seguido todos los pasos del tutorial, podremos eliminar nuestro bloques de try catch de los controladores ya que el manejo de errores será llevado a cabo de forma automática por nuestro middleware.

Y por supuesto, podemos hacer uso de logger para mostrar por consola la información que se requiera, mostraré un ejemplo.

```{.csharp .numberLines}
//VehiculoController.cs
[HttpGet("getVehiculo")]
public async Task<ActionResult<VehiculoDTO>> GetVehicle(int id) {
    _logger.LogInfo("Fetching vehicle with ID: " + id);

    string conn = await _empresaServicio.GenerarCadenaDeConexionAsync((int)JwtUtil.ObtenerDatosEmpresaPeticion(Request.Headers, _configuration, _jwtSettings));
    VehiculoDTO vehiculo = await _vehiculoServicio.GetById(id, conn);

    if (vehiculo == null) {
        _logger.LogWarn("Vehicle with ID: " + id + " not found.");
        return NotFound("No se encuentra el vehículo buscado");
    } else {
        _logger.LogInfo($"Returning vehicle with ID: {id}");
        return Ok(vehiculo);
    }
}
```

Como se ha mostrado, nos olvidamos de la gestión de errores, simplemente devolvemos nuestro statusCode con el contenido que se requiera.

\pagebreak
