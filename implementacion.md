# Implementación de OpenTelemetry en .NET 9 API

## 📋 Resumen
Esta guía te ayudará a integrar tu API d# 📝 Configurar Logging con OpenTelemetry
builder.Logging.AddOpenTelemetry(loggingBuilder =>
{
    loggingBuilder
        .SetResourceBuilder(resourceBuilder)
        .AddConsoleExporter() // Para desarrollo local
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(otlpEndpoint);
            options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
        });
});

var app = builder.Build();u stack de observabilidad que incluye:
- **OpenTelemetry Collector** (puertos 4317 gRPC / 4318 HTTP)
- **Prometheus** (métricas)
- **Tempo** (trazas distribuidas)
- **Grafana** (visualización)

---

## 🚀 1. Instalación de Paquetes NuGet

Agrega estos paquetes a tu proyecto .NET 9:

```bash
# Paquetes principales de OpenTelemetry
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Extensions.Hosting

# Instrumentación automática
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore  # Si usas EF Core
dotnet add package OpenTelemetry.Instrumentation.SqlClient  # Si usas SQL Server

# Exportadores
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore

# Para logging
dotnet add package OpenTelemetry.Exporter.Console  # Para debugging local
```

---

## ⚙️ 2. Configuración en Program.cs

```csharp
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Logs;

var builder = WebApplication.CreateBuilder(args);

// Configuración de servicios
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// 🔧 Configuración de OpenTelemetry
var serviceName = "mi-api-dotnet";
var serviceVersion = "1.0.0";
var otlpEndpoint = "http://tu-vps:4318"; // Cambia por la IP de tu VPS

// Configurar el recurso (información del servicio)
var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService(serviceName: serviceName, serviceVersion: serviceVersion)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName,
        ["service.instance.id"] = Environment.MachineName
    });

// 📊 Configurar Métricas
builder.Services.AddOpenTelemetry()
    .WithMetrics(metricsBuilder =>
    {
        metricsBuilder
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddProcessInstrumentation()
            // Exportar a OpenTelemetry Collector
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri(otlpEndpoint);
                options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
            })
            // Exportar métricas de Prometheus
            .AddPrometheusExporter();
    })
    // 🔍 Configurar Trazas
    .WithTracing(tracingBuilder =>
    {
        tracingBuilder
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation(options =>
            {
                options.RecordException = true;
                options.Filter = (context) => !context.Request.Path.StartsWithSegments("/health");
            })
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation() // Si usas EF Core
            .AddSqlClientInstrumentation() // Si usas SQL Server
            // Exportar a OpenTelemetry Collector
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri(otlpEndpoint);
                options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
            });
    });

// 📝 Configurar Logging con OpenTelemetry
builder.Logging.AddOpenTelemetry(loggingBuilder =>
{
    loggingBuilder
        .SetResourceBuilder(resourceBuilder)
        .AddConsoleExporter() // Para desarrollo local
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(otlpEndpoint);
            options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
        });
});

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();

// 📊 Exponer métricas de Prometheus
app.MapPrometheusScrapingEndpoint();

app.MapControllers();

app.Run();
```

---

## 🎯 3. Ejemplo de Controller con Telemetría Personalizada

```csharp
using System.Diagnostics;
using System.Diagnostics.Metrics;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ProductosController : ControllerBase
{
    private readonly ILogger<ProductosController> _logger;
    private static readonly ActivitySource ActivitySource = new("MiAPI.Productos");
    private static readonly Meter Meter = new("MiAPI.Productos");
    
    // Métricas personalizadas
    private static readonly Counter<int> ProductosConsultados = 
        Meter.CreateCounter<int>("productos_consultados_total", "Número total de productos consultados");
    
    private static readonly Histogram<double> TiempoRespuesta = 
        Meter.CreateHistogram<double>("productos_tiempo_respuesta_ms", "Tiempo de respuesta en milisegundos");

    public ProductosController(ILogger<ProductosController> logger)
    {
        _logger = logger;
    }

    [HttpGet]
    public async Task<IActionResult> GetProductos()
    {
        var stopwatch = Stopwatch.StartNew();
        
        // 🔍 Crear un span personalizado
        using var activity = ActivitySource.StartActivity("GetProductos");
        activity?.SetTag("operation.name", "get_productos");
        activity?.SetTag("user.id", "usuario123"); // Agregar contexto
        
        try
        {
            _logger.LogInformation("Iniciando consulta de productos");
            
            // Simular trabajo
            await Task.Delay(Random.Shared.Next(100, 500));
            
            var productos = new[]
            {
                new { Id = 1, Nombre = "Producto 1", Precio = 100.50 },
                new { Id = 2, Nombre = "Producto 2", Precio = 200.75 }
            };
            
            // 📊 Registrar métricas
            ProductosConsultados.Add(productos.Length, 
                new KeyValuePair<string, object?>("endpoint", "get_productos"));
            
            stopwatch.Stop();
            TiempoRespuesta.Record(stopwatch.ElapsedMilliseconds, 
                new KeyValuePair<string, object?>("endpoint", "get_productos"),
                new KeyValuePair<string, object?>("status", "success"));
            
            // 🔍 Agregar información al span
            activity?.SetTag("productos.count", productos.Length);
            activity?.SetStatus(ActivityStatusCode.Ok);
            
            _logger.LogInformation("Consulta de productos completada. Total: {Count}", productos.Length);
            
            return Ok(productos);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            TiempoRespuesta.Record(stopwatch.ElapsedMilliseconds,
                new KeyValuePair<string, object?>("endpoint", "get_productos"),
                new KeyValuePair<string, object?>("status", "error"));
            
            // 🔍 Registrar error en el span
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.SetTag("error", true);
            
            _logger.LogError(ex, "Error al consultar productos");
            
            return StatusCode(500, "Error interno del servidor");
        }
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetProducto(int id)
    {
        using var activity = ActivitySource.StartActivity("GetProducto");
        activity?.SetTag("producto.id", id);
        
        _logger.LogInformation("Consultando producto con ID: {ProductoId}", id);
        
        if (id <= 0)
        {
            activity?.SetStatus(ActivityStatusCode.Error, "ID inválido");
            _logger.LogWarning("Intento de consulta con ID inválido: {ProductoId}", id);
            return BadRequest("ID debe ser mayor a 0");
        }
        
        // Simular trabajo
        await Task.Delay(Random.Shared.Next(50, 200));
        
        var producto = new { Id = id, Nombre = $"Producto {id}", Precio = id * 10.5 };
        
        ProductosConsultados.Add(1, 
            new KeyValuePair<string, object?>("endpoint", "get_producto_by_id"));
        
        activity?.SetStatus(ActivityStatusCode.Ok);
        _logger.LogInformation("Producto encontrado: {ProductoId}", id);
        
        return Ok(producto);
    }
}
```

---

## 🔧 4. Configuración en appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "OpenTelemetry": "Information"
    }
  },
  "OpenTelemetry": {
    "ServiceName": "mi-api-dotnet",
    "ServiceVersion": "1.0.0",
    "OtlpEndpoint": "http://tu-vps:4318"
  },
  "AllowedHosts": "*"
}
```

---

## 📊 5. Métricas Disponibles en Prometheus

Tu API enviará automáticamente estas métricas:

### Métricas de ASP.NET Core:
- `http_request_duration_ms` - Duración de requests HTTP
- `http_requests_total` - Total de requests HTTP
- `aspnetcore_requests_per_second` - Requests por segundo

### Métricas del Sistema:
- `process_cpu_usage` - Uso de CPU
- `dotnet_gc_collections_total` - Recolecciones de basura
- `process_memory_usage` - Uso de memoria

### Métricas Personalizadas:
- `productos_consultados_total` - Contador de productos consultados
- `productos_tiempo_respuesta_ms` - Tiempo de respuesta

---

## 🎯 6. Configuración en Grafana

### 6.1 Agregar Prometheus como Data Source:
1. Ve a **Configuration > Data Sources**
2. Agrega **Prometheus** con URL: `http://prometheus:9090`

### 6.2 Agregar Tempo como Data Source:
1. Ve a **Configuration > Data Sources**
2. Agrega **Tempo** con URL: `http://tempo:3200`

### 6.3 Dashboard Sugerido:
```json
{
  "dashboard": {
    "title": "Mi API .NET 9 - Observabilidad",
    "panels": [
      {
        "title": "Requests por Segundo",
        "query": "rate(http_requests_total[5m])"
      },
      {
        "title": "Tiempo de Respuesta P95",
        "query": "histogram_quantile(0.95, rate(http_request_duration_ms_bucket[5m]))"
      },
      {
        "title": "Productos Consultados",
        "query": "rate(productos_consultados_total[5m])"
      }
    ]
  }
}
```

---

## 🚀 7. Comandos para Probar

### Generar tráfico de prueba:
```bash
# Hacer requests a tu API
curl -X GET "https://tu-api.com/api/productos"
curl -X GET "https://tu-api.com/api/productos/1"
curl -X GET "https://tu-api.com/api/productos/999"
```

### Verificar métricas:
```bash
# Ver métricas de Prometheus directamente
curl http://tu-vps:8889/metrics
```

---

## 🔍 8. Troubleshooting

### Verificar conectividad:
```bash
# Probar conectividad con OpenTelemetry Collector
curl -v http://tu-vps:4318/v1/traces
```

### Logs útiles:
```csharp
// Agregar en Program.cs para debugging
builder.Logging.AddConsole();
builder.Logging.SetMinimumLevel(LogLevel.Debug);
```

---

## ✅ Resultado Final

Con esta configuración tendrás:
- ✅ **Métricas automáticas** de ASP.NET Core en Prometheus
- ✅ **Trazas distribuidas** en Tempo
- ✅ **Logs estructurados** con contexto
- ✅ **Métricas personalizadas** de tu negocio
- ✅ **Dashboards** en Grafana para visualización

¡Tu API estará completamente instrumentada y lista para observabilidad completa! 🎉

---

## 📝 9. Configuración de Logs para Loki

### 9.1 Logs Estructurados en tu Controller:

```csharp
[HttpPost]
public async Task<IActionResult> CrearProducto([FromBody] ProductoDto producto)
{
    using var activity = ActivitySource.StartActivity("CrearProducto");
    
    // Log estructurado con contexto
    _logger.LogInformation("Iniciando creación de producto: {@Producto}", producto);
    
    try
    {
        // Validación
        if (string.IsNullOrEmpty(producto.Nombre))
        {
            _logger.LogWarning("Intento de crear producto sin nombre. UserId: {UserId}", "usuario123");
            return BadRequest("El nombre es requerido");
        }
        
        // Simular guardado
        await Task.Delay(200);
        
        var nuevoProducto = new { Id = Random.Shared.Next(1000, 9999), producto.Nombre, producto.Precio };
        
        // Log de éxito con datos del resultado
        _logger.LogInformation("Producto creado exitosamente. ProductoId: {ProductoId}, Nombre: {Nombre}", 
            nuevoProducto.Id, nuevoProducto.Nombre);
        
        // Métricas
        ProductosCreados.Add(1, 
            new KeyValuePair<string, object?>("status", "success"));
        
        activity?.SetTag("producto.id", nuevoProducto.Id);
        activity?.SetStatus(ActivityStatusCode.Ok);
        
        return CreatedAtAction(nameof(GetProducto), new { id = nuevoProducto.Id }, nuevoProducto);
    }
    catch (Exception ex)
    {
        // Log de error con contexto completo
        _logger.LogError(ex, "Error al crear producto. Producto: {@Producto}, Error: {ErrorMessage}", 
            producto, ex.Message);
        
        ProductosCreados.Add(1, 
            new KeyValuePair<string, object?>("status", "error"));
        
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        return StatusCode(500, "Error interno del servidor");
    }
}
```

### 9.2 Middleware para Logging de Requests:

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var startTime = DateTime.UtcNow;
        var stopwatch = Stopwatch.StartNew();
        
        // Log del request entrante
        _logger.LogInformation("HTTP Request iniciado: {Method} {Path} {QueryString}", 
            context.Request.Method, 
            context.Request.Path, 
            context.Request.QueryString);

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            
            // Log del response
            _logger.LogInformation("HTTP Request completado: {Method} {Path} {StatusCode} {Duration}ms", 
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
    }
}

// Agregar en Program.cs
app.UseMiddleware<RequestLoggingMiddleware>();
```

### 9.3 Configuración avanzada de Logging:

```csharp
// En Program.cs, configuración más detallada
builder.Logging.ClearProviders();
builder.Logging.AddOpenTelemetry(loggingBuilder =>
{
    loggingBuilder
        .SetResourceBuilder(resourceBuilder)
        .AddConsoleExporter()
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(otlpEndpoint);
            options.Protocol = OpenTelemetry.Exporter.OtlpExportProtocol.HttpProtobuf;
        });
});

// Configurar niveles de log
builder.Logging.SetMinimumLevel(LogLevel.Information);
builder.Logging.AddFilter("Microsoft.AspNetCore", LogLevel.Warning);
builder.Logging.AddFilter("Microsoft.EntityFrameworkCore", LogLevel.Warning);
```

---

## 🔍 10. Queries útiles en Grafana para Loki

### 10.1 Configurar Loki como Data Source:
1. Ve a **Configuration > Data Sources**
2. Agrega **Loki** con URL: `http://loki:3100`

### 10.2 Queries de ejemplo:

```logql
# Todos los logs de tu API
{service_name="mi-api-dotnet"}

# Solo errores
{service_name="mi-api-dotnet"} |= "ERROR"

# Logs de un endpoint específico
{service_name="mi-api-dotnet"} |= "GetProductos"

# Logs con contexto de usuario
{service_name="mi-api-dotnet"} |= "usuario123"

# Rate de errores por minuto
rate({service_name="mi-api-dotnet"} |= "ERROR" [5m])

# Logs agrupados por nivel
sum by (level) (rate({service_name="mi-api-dotnet"} [5m]))
```

### 10.3 Dashboard con Logs + Métricas + Trazas:

```json
{
  "dashboard": {
    "title": "Mi API .NET 9 - Observabilidad Completa",
    "panels": [
      {
        "title": "Requests por Segundo",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{service_name=\"mi-api-dotnet\"}[5m])"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat", 
        "targets": [
          {
            "expr": "rate({service_name=\"mi-api-dotnet\"} |= \"ERROR\" [5m])"
          }
        ]
      },
      {
        "title": "Logs en Tiempo Real",
        "type": "logs",
        "targets": [
          {
            "expr": "{service_name=\"mi-api-dotnet\"}"
          }
        ]
      },
      {
        "title": "Traces",
        "type": "traces",
        "datasource": "tempo"
      }
    ]
  }
}
```

---

## 🚀 11. Comandos para Probar Logs

### Verificar que Loki recibe logs:
```bash
# Ver logs en Loki directamente
curl "http://tu-vps:3100/loki/api/v1/query_range?query={service_name=\"mi-api-dotnet\"}"
```

### Generar logs de prueba en tu API:
```csharp
// Endpoint para generar logs de prueba
[HttpGet("test-logs")]
public IActionResult TestLogs()
{
    _logger.LogDebug("Log de Debug - Test");
    _logger.LogInformation("Log de Information - Test con datos: {Data}", new { Test = true, Time = DateTime.Now });
    _logger.LogWarning("Log de Warning - Test");
    _logger.LogError("Log de Error - Test de error simulado");
    
    return Ok("Logs generados");
}
```
