# Contexto técnico — Vulnerabilidades resueltas y patrones validados

## Propósito

Este documento contiene el contexto técnico acumulado durante la corrección de vulnerabilidades detectadas mediante **Veracode Static Analysis** en una aplicación ASP.NET Core / C#.

Debe utilizarse como complemento del prompt general de trabajo.

El objetivo es que una nueva conversación conozca:

* Qué vulnerabilidades ya fueron analizadas.
* Qué patrones fueron reportados por Veracode.
* Qué soluciones se intentaron.
* Qué soluciones finalmente eliminaron los hallazgos.
* Qué comportamientos particulares del análisis estático fueron observados.
* Qué decisiones de diseño deben conservarse para evitar regresiones.

---

# Estado de vulnerabilidades trabajadas

| CWE     | Vulnerabilidad                         | Estado                      |
| ------- | -------------------------------------- | --------------------------- |
| CWE-918 | Server-Side Request Forgery (SSRF)     | Resolved                    |
| CWE-201 | Information Exposure Through Sent Data | Resolved                    |
| CWE-404 | Improper Resource Shutdown or Release  | Addressed                   |
| CWE-295 | Improper Certificate Validation        | Pattern reviewed/documented |

---

# CWE-918 — Server-Side Request Forgery (SSRF)

## Hallazgo original

Veracode detectó información proveniente de un request externo utilizada para construir una URL que finalmente llegaba a:

```csharp
HttpClient.GetAsync(...)
```

El flujo original era conceptualmente:

```text
HTTP Request
      |
      v
SRedKeyProcessPaymentDto
      |
      v
GetSRedKeyUrlFromDto()
      |
      v
string / Uri
      |
      v
HttpClientUtility.GetAsync()
      |
      v
HttpClient.GetAsync()
```

Veracode interpretaba la URL como controlada, al menos parcialmente, por datos no confiables.

---

## Arquitectura inicial relevante

Existía un mapper responsable de construir una URL completa:

```csharp
var url =
    _sRedKeyMapper.GetSRedKeyUrlFromDto(request);
```

Posteriormente:

```csharp
var result =
    await _httpClient.GetAsync(url);
```

La URL contenía:

* Base URL desde configuración.
* `TrxType`
* `LocationID`
* `ComputerID`
* `TrxInfo`

Parte de la información provenía directamente del DTO del request.

---

## Primeras mitigaciones intentadas

Se implementaron controles como:

* `Uri.TryCreate`.
* Validación de scheme.
* Allowlist exacta de hosts.
* Validación del puerto.
* Bloqueo de `UserInfo`.
* Validación numérica de campos.
* `AllowAutoRedirect = false`.
* Validación manual antes de `GetAsync`.
* Cambio de `string` a `Uri`.
* Construcción de un `Uri` relativo.
* Separación entre `BaseAddress` y URI relativa.

Ejemplo de validación:

```csharp
if (!AllowedHosts.Contains(uri.IdnHost))
{
    throw new InvalidOperationException(
        "The destination host is not allowed.");
}
```

Estas medidas mejoraban la seguridad real, pero **Veracode continuó reportando CWE-918**.

---

## Observación importante sobre Veracode

El Data Path demostró que Veracode continuaba siguiendo los valores del DTO hasta el objeto `Uri`.

Conceptualmente:

```text
DTO value
    |
    v
query
    |
    v
Uri
    |
    v
GetAsync(Uri)
```

Las funciones personalizadas de validación no estaban siendo reconocidas como sanitizers suficientes.

Por tanto, continuar agregando validaciones no resolvía el problema de análisis estático.

---

## Diseño final que resolvió CWE-918

Se separó completamente el **destino HTTP** de los **datos de la solicitud**.

### Destino controlado por la aplicación

Se creó un `HttpClient` específico con:

```csharp
_sRedKeyHttpClient = new HttpClient(handler)
{
    BaseAddress = baseUri
};
```

El `BaseAddress` proviene de configuración controlada por la aplicación y se valida independientemente.

Las redirecciones automáticas están deshabilitadas:

```csharp
var handler = new HttpClientHandler
{
    // Automatic redirects are disabled to prevent requests to untrusted destinations.
    AllowAutoRedirect = false
};
```

---

## El mapper dejó de retornar URLs

El mapper dejó de producir:

```text
string URL
```

o:

```text
Uri
```

En su lugar retorna únicamente los datos requeridos por el servicio:

```csharp
public sealed class SRedKeyRequestParts
{
    public string TrxType { get; init; } = "";
    public string LocationId { get; init; } = "";
    public string ComputerId { get; init; } = "";
    public string TrxInfo { get; init; } = "";
}
```

Esto establece una frontera clara:

```text
Mapper
  |
  +--> Request data

HttpClientUtility
  |
  +--> Destination
```

---

## Patrón final de `GetSRedKeyAsync`

El patrón que finalmente hizo desaparecer el hallazgo fue:

```csharp
public async Task<string> GetSRedKeyAsync(
    string trxType,
    string locationId,
    string computerId,
    string trxInfo)
{
    string validatedTrxType =
        ValidateNumericValue(trxType, nameof(trxType));

    string validatedLocationId =
        ValidateNumericValue(locationId, nameof(locationId));

    string validatedComputerId =
        ValidateNumericValue(computerId, nameof(computerId));

    var query =
        HttpUtility.ParseQueryString(string.Empty);

    query["TrxType"] = validatedTrxType;
    query["LocationID"] = validatedLocationId;
    query["ComputerID"] = validatedComputerId;
    query["TrxInfo"] = trxInfo ?? string.Empty;

    string relativeUrl =
        $"WS/CMSWSGMF/default.aspx?{query}";

    // TODO: The URL may contain sensitive information and should be sanitized before logging.
    _loggingHelper.Log(
        $"Attempting to make a GET request to url: {relativeUrl}");

    using HttpResponseMessage response =
        await _sRedKeyHttpClient.GetAsync(relativeUrl);

    EnsureResponseIsNotRedirect(response);

    return await response.Content.ReadAsStringAsync();
}
```

---

## Cambio decisivo observado

Este patrón seguía siendo reportado:

```csharp
var requestUri = new Uri(
    $"{endpointPath}?{query}",
    UriKind.Relative);

await _sRedKeyHttpClient.GetAsync(requestUri);
```

Mientras que este patrón eliminó el hallazgo:

```csharp
string relativeUrl =
    $"{endpointPath}?{query}";

await _sRedKeyHttpClient.GetAsync(relativeUrl);
```

Esto es una **observación del comportamiento de Veracode en este proyecto**, no una regla general indicando que `GetAsync(string)` sea más seguro que `GetAsync(Uri)`.

La regla real de seguridad continúa siendo:

> Untrusted input must not control the destination of a server-side HTTP request.

---

## Regla arquitectónica que debe conservarse

Los siguientes elementos deben permanecer bajo control de la aplicación:

```text
Scheme
Host
Port
BaseAddress
Endpoint/path
```

Los datos externos únicamente pueden participar como:

```text
Query parameter values
Request body values
```

---

# CWE-201 — Information Exposure Through Sent Data

## Hallazgo original

Existía un método HTTP genérico con este contrato:

```csharp
Task<HttpResponseMessage> PostAsync(
    Uri uri,
    object? body = null);
```

Implementado aproximadamente como:

```csharp
var requestBody =
    JsonSerializer.Serialize(body ?? new { });

var data = new StringContent(
    requestBody,
    Encoding.UTF8,
    "application/json");

return await _httpClient.PostAsync(uri, data);
```

Veracode reportó varios CWE-201 asociados al flujo hacia `PostAsync`.

---

## Contexto adicional

La aplicación utiliza una clase `AppSettings` respaldada por:

```csharp
Dictionary<string, string?>
```

y una parte considerable de la configuración se carga mediante:

```csharp
builder.Configuration.AsEnumerable()
```

Inicialmente se consideró que esta arquitectura podía provocar que Veracode tratara valores de configuración como potencialmente sensibles.

Sin embargo, eliminar `AsEnumerable()` **no era viable**, ya que la aplicación necesita cargar múltiples parámetros de configuración.

Por tanto, no debe proponerse eliminar esta arquitectura automáticamente para resolver futuros hallazgos similares.

---

## Problema con `object`

El contrato:

```csharp
PostAsync(Uri, object)
```

era demasiado genérico.

Conceptualmente:

```text
object
   |
   v
Unknown runtime content
   |
   v
JsonSerializer.Serialize(object)
   |
   v
Network
```

Para el análisis estático era difícil determinar exactamente qué datos podían estar siendo enviados.

---

## Solución final

Se cambió el método a un contrato genérico fuertemente tipado:

```csharp
Task<HttpResponseMessage> PostAsync<T>(
    Uri uri,
    T body)
    where T : class;
```

Implementación:

```csharp
public async Task<HttpResponseMessage> PostAsync<T>(
    Uri uri,
    T body)
    where T : class
{
    ValidateUrl(uri);

    var requestBody =
        JsonSerializer.Serialize(body);

    // TODO: The URL and request body may contain sensitive information and should be sanitized before logging.
    _loggingHelper.Log(
        $"Attempting to make a POST request to url: {uri} with body {requestBody}");

    var data = new StringContent(
        requestBody,
        Encoding.UTF8,
        "application/json");

    return await _httpClient.PostAsync(
        uri,
        data);
}
```

---

## DTO específico para comunicación externa

Además del método genérico, se creó un DTO específico para representar únicamente los datos autorizados a salir de la aplicación.

Ejemplo:

```csharp
public sealed class ArCustomerRequestDto
{
    [JsonPropertyName("locationId")]
    public int LocationId { get; init; }

    [JsonPropertyName("customerAuthCode")]
    public int CustomerAuthCode { get; init; }

    [JsonPropertyName("posEventId")]
    public int PosEventId { get; init; }
}
```

Se reemplazó un objeto anónimo por:

```csharp
var requestObject = new ArCustomerRequestDto
{
    LocationId = locationId,
    CustomerAuthCode = customerAuthCode,
    PosEventId = posEventId
};

var response =
    await _httpClient.PostAsync(
        parsedUri,
        requestObject);
```

---

## Resultado

La combinación:

```text
Dedicated outbound DTO
        +
PostAsync<T>
```

eliminó los hallazgos CWE-201.

---

## Lección principal

El DTO de salida debe verse como una **allowlist explícita de información autorizada a cruzar el límite de confianza**.

Flujo recomendado:

```text
Incoming DTO
      |
      v
Validation
      |
      v
Mapping
      |
      v
Outbound DTO
      |
      v
PostAsync<T>()
      |
      v
Serialize<T>()
      |
      v
External Service
```

No utilizar automáticamente el DTO recibido por el API como DTO de una integración externa.

---

## Importante

El uso de:

```csharp
PostAsync<T>
```

no convierte datos sensibles en seguros.

Un DTO podría seguir conteniendo:

```csharp
Password
AccessToken
Secret
CVV
```

y esos datos seguirían requiriendo controles específicos.

La ventaja del patrón es hacer explícito qué información está saliendo de la aplicación.

---

# CWE-404 — Improper Resource Shutdown or Release

## Hallazgo

Veracode identificó objetos `StreamReader` creados sin liberación determinística.

Ejemplos similares a:

```csharp
new StreamReader(
    httpContext.Request.Body,
    Encoding.UTF8);
```

y:

```csharp
new StreamReader(memStream);
```

---

## Corrección aplicada

Utilizar `using` para liberar el wrapper:

```csharp
using var requestReader =
    new StreamReader(
        httpContext.Request.Body,
        Encoding.UTF8,
        detectEncodingFromByteOrderMarks: true,
        bufferSize: 1024,
        leaveOpen: true);
```

Cuando el stream subyacente pertenece al framework:

```csharp
leaveOpen: true
```

es importante.

---

## Ownership de recursos

Debe mantenerse esta distinción:

```text
Application creates resource
        |
        v
Application owns resource
        |
        v
Application disposes resource
```

frente a:

```text
Framework creates resource
        |
        v
Application temporarily uses it
        |
        v
Application must not close it
```

En ASP.NET Core:

```csharp
httpContext.Request.Body
httpContext.Response.Body
```

pertenecen al framework.

No deben cerrarse directamente.

---

## Problema funcional descubierto

Después del cambio apareció un error similar a:

```text
The input does not contain any JSON tokens.
Expected the input to start with a valid JSON token...
```

Esto ocurrió porque leer `Request.Body` avanzaba la posición del stream.

Si no se restauraba, el model binder o middleware posterior encontraba un body vacío.

---

## Patrón correcto

Cuando el body debe leerse múltiples veces:

```csharp
httpContext.Request.EnableBuffering();
```

y posteriormente:

```csharp
try
{
    using var requestReader =
        new StreamReader(
            httpContext.Request.Body,
            Encoding.UTF8,
            detectEncodingFromByteOrderMarks: true,
            bufferSize: 1024,
            leaveOpen: true);

    string requestBody =
        await requestReader.ReadToEndAsync();

    // Process request body.
}
finally
{
    // Always restore the request body for the next middleware.
    httpContext.Request.Body.Position = 0;
}
```

---

## Regla importante

`using` y `finally` cumplen responsabilidades diferentes:

```text
using
  |
  +--> Dispose locally owned wrapper/resource

finally
  |
  +--> Restore shared state
```

No confundir resource disposal con stream state restoration.

---

## `Response.Body`

Cuando middleware reemplaza temporalmente:

```csharp
httpContext.Response.Body
```

debe conservar el original:

```csharp
Stream originalBody =
    httpContext.Response.Body;
```

y restaurarlo en `finally`.

---

# CWE-295 — Improper Certificate Validation

## Patrón vulnerable

Se revisó código similar a:

```csharp
ServicePointManager.ServerCertificateValidationCallback +=
    (sender, certificate, chain, errors) =>
    {
        return true;
    };
```

Esto deshabilita efectivamente la validación del certificado.

---

## Riesgo

El patrón permite aceptar:

```text
Expired certificate
Untrusted certificate
Wrong hostname
Invalid certificate chain
Attacker-controlled certificate
```

y posibilita ataques Man-in-the-Middle.

---

## Patrón recomendado

Siempre que sea posible, no configurar ningún callback personalizado.

Permitir que .NET realice su validación TLS estándar.

Ejemplo:

```csharp
var httpClient =
    new HttpClient();
```

Sin:

```csharp
ServerCertificateCustomValidationCallback
```

---

## Si se requiere validación personalizada

Preferir:

```csharp
HttpClientHandler
```

sobre configuraciones globales como:

```csharp
ServicePointManager.ServerCertificateValidationCallback
```

Nunca utilizar:

```csharp
(_, _, _, _) => true
```

Un callback personalizado debe rechazar cualquier certificado que no cumpla con la política de confianza.

---

## Certificate Pinning

No aplicar certificate pinning automáticamente para resolver CWE-295.

Sólo utilizarlo cuando exista:

* requisito explícito;
* control del certificado;
* estrategia documentada de rotación;
* proceso operacional para renovaciones.

Para infraestructura interna, preferir una CA interna confiable y la validación TLS estándar.

---

# Comportamientos de Veracode aprendidos

## Revisar siempre el Data Path

Cuando un hallazgo continúa después de una corrección:

1. Revisar la nueva línea marcada.
2. Expandir el Data Path.
3. Identificar el source.
4. Identificar las variables de propagación.
5. Identificar el sink.
6. Determinar qué valor sigue siendo considerado tainted.

No continuar agregando validaciones arbitrariamente.

---

## Custom validators pueden no ser sanitizers

Métodos como:

```csharp
ValidateUrl(...)
ValidateHost(...)
ValidateNumericValue(...)
```

pueden tener valor real de seguridad, pero Veracode puede continuar propagando el taint a través de ellos.

Si ocurre esto, buscar una arquitectura donde la frontera de confianza sea evidente por construcción.

---

## Strong typing ayuda al análisis

El cambio:

```text
object
```

a:

```text
Dedicated DTO + generic method
```

permitió eliminar CWE-201.

Por tanto, para integraciones HTTP nuevas, preferir contratos explícitos.

---

## Separar destination de request data

Para evitar SSRF:

```text
Application
   |
   +--> BaseAddress
   +--> Scheme
   +--> Host
   +--> Port
   +--> Endpoint

External data
   |
   +--> Query values
   +--> Body values
```

No permitir que información externa reconstruya el destino completo.

---

# Convenciones que deben mantenerse

## Código

* Existing comments should be preserved.
* New code comments must be written in English.
* Existing logs should normally be preserved during security remediation.
* If a log may contain sensitive information, add a `TODO` rather than silently removing it unless removal has been explicitly approved.

Ejemplo:

```csharp
// TODO: The URL may contain sensitive information and should be sanitized before logging.
_loggingHelper.Log(
    $"Attempting to make a GET request to url: {url}");
```

---

## Cambios

Preferir:

```text
Small localized security changes
```

sobre:

```text
Large architectural refactors
```

salvo que el Data Path demuestre que la arquitectura actual impide expresar correctamente la frontera de confianza.

---

# Patrones que deben conservarse para evitar regresiones

## SSRF

Conservar:

```text
Controlled BaseAddress
+
Fixed/known endpoint
+
Framework query construction
+
External values only as data
+
No uncontrolled redirects
```

Evitar volver a:

```text
DTO
 |
 v
Full URL / Uri
 |
 v
GetAsync(...)
```

---

## Information Exposure

Conservar:

```text
Dedicated outbound DTO
+
PostAsync<T>
+
Typed serialization
```

Evitar volver a:

```csharp
PostAsync(Uri, object)
```

cuando existe un contrato de request conocido.

---

## Resource Handling

Conservar:

```text
using
+
correct ownership
+
leaveOpen when required
+
finally for state restoration
```

---

## Certificate Validation

Conservar la validación TLS estándar siempre que sea posible.

Nunca introducir callbacks que acepten certificados incondicionalmente.

---

# Estado para continuar en una nueva conversación

Si aparece un nuevo hallazgo:

```text
1. Identify CWE
2. Review Veracode message
3. Review source code
4. Inspect Data Path
5. Identify source / propagation / sink
6. Propose minimal remediation options
7. Apply one change at a time
8. Run Veracode again
9. Compare new Data Path
10. Preserve patterns already proven to work
```

No repetir automáticamente intentos que ya demostraron no ser reconocidos por Veracode cuando existe un patrón posterior que sí resolvió el hallazgo.
