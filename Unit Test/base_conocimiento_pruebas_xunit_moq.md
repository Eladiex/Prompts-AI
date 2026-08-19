# Base de conocimiento específica del chat: pruebas unitarias C# con xUnit y Moq

Este documento contiene información específica aprendida durante el chat. Debe usarse como contexto adicional junto con el prompt general. Puede expandirse conforme se trabajen nuevos métodos, clases o proyectos.

---

## 1. Contexto técnico observado

Durante este chat se trabajó principalmente con:

1. .NET 8.
2. C# compatible con .NET 8.
3. xUnit.
4. Moq.
5. Servicios y controllers en C#.
6. DTOs en `Core.Models.Dtos`.
7. `ResponseData` desde `Core.Utility.Data.AppCommon`.
8. `QueriesToExecute` para llamadas a stored procedures.
9. Stored procedures definidos en clases como `OrderStoredProcedures`, `OrderDetailStoredProcedures`, `OrderStagingStoredProcedures` y `StoreOrderMgmtStoredProcedures`.

Import frecuente:

```csharp
using static Core.Utility.Data.AppCommon;
```

---

## 2. Preferencias de trabajo confirmadas por el usuario

1. Trabajar siempre prueba por prueba.
2. Aunque se elijan varias pruebas, entregar una sola y esperar confirmación.
3. Al implementar una prueba, indicar progreso: “Trabajando prueba X de Y…”.
4. Al terminar una prueba, listar las pruebas pendientes.
5. Si una clase de prueba ya existe, no volver a generar la clase completa; entregar solo el método.
6. Si el usuario comparte una versión modificada y dice que funciona, esa versión se toma como base.
7. Se acepta agrupar pruebas con `[Theory]` e `[InlineData]` cuando el flujo es el mismo.
8. Se prefiere evitar pruebas duplicadas o excesivamente fragmentadas si una prueba parametrizada cubre el mismo patrón.
9. Validar explícitamente `Times.Never` en escenarios donde una dependencia no debe ejecutarse.

---

## 3. Patrones técnicos reutilizables

### 3.1 Patrón para respuestas `DataTable` con status

Para métodos que validan:

```csharp
if (result.Rows.Count == 0 || result.Rows[0]["status"].ToString() != "success")
{
    throw new Exception("...");
}
```

Se usó este patrón:

```csharp
[Theory]
[InlineData("failure", 1)]
[InlineData("error", 1)]
[InlineData("", 0)]
public void Metodo_ResponseWithoutSuccessStatus_ThrowsException(string status, int rows)
{
    // Arrange
    var dataTable = new DataTable();
    dataTable.Columns.Add("status", typeof(string));

    if (rows != 0)
        dataTable.Rows.Add(status);

    // Act
    var exception = Assert.Throws<Exception>(() => service.Metodo(request));

    // Assert
    Assert.Equal("Mensaje esperado", exception.Message);
}
```

### 3.2 Patrón para validar stored procedure y parámetros

```csharp
_queries.Verify(
    x => x.GetDBData(It.Is<QueriesToExecute.QueryInfo>(query =>
        query.Query == StoredProcedureEsperado &&
        query.Parameters.ContainsKey("Parametro") &&
        query.Parameters["Parametro"].ToString() == valorEsperado.ToString())),
    Times.Once);
```

### 3.3 Patrón para validar que no se llama una dependencia

```csharp
_mock.Verify(
    x => x.Metodo(
        It.IsAny<string>(),
        It.IsAny<decimal>(),
        It.IsAny<int>()),
    Times.Never);
```

---

## 4. `SharedOrderOperationsService`

Se trabajaron pruebas para:

1. `DeleteOrderAssignment`
2. `PrependOrderNotes`
3. `InsertOrderAttribute`
4. `InsertOrderPayment`
5. `InsertOrderPaymentCCOffline`
6. `InsertOrderPaymentLoyaltyOffline`
7. `CommitStagingTables`
8. `InsertTransLogRecord`
9. `AssignTrxActionAndRequestIdsToObjects(OrderDto, string)`
10. `AssignTrxActionAndRequestIdsToObjects(StageTableData, string)`

Patrones importantes:

1. Validar stored procedure usado.
2. Validar parámetros enviados.
3. Validar retorno del mismo `DataTable` o `ResponseData`.
4. Validar excepción cuando `status` no es `success`.
5. Para métodos de pago y atributos, usar el patrón `[Theory]` con `failure`, `error` y tabla sin filas.
6. Para asignación de acciones, validar `TrxActionID` esperado según acción: insert, update, delete.
7. Para acciones no soportadas, validar excepción `"Unsupported action"`.

---

## 5. `StoreOrderMgmtController`

Método trabajado:

```csharp
ExecuteOrderMgmtActions(StoreOrderMgmtActionRequestModel request)
```

Acciones válidas:

1. `moveOrder`
2. `voidOrder`
3. `reOpenOrder`

Se acordó que estas tres acciones podían cubrirse con una sola prueba parametrizada porque comparten el mismo patrón.

Pruebas trabajadas:

```csharp
ExecuteOrderMgmtActions_ValidAction_CallsExpectedDomainMethodAndReturnsResponse
ExecuteOrderMgmtActions_UnsupportedAction_ThrowsArgumentException
```

Idea clave:

1. Para acciones válidas, validar que se llame el método correcto del dominio.
2. Validar que retorne el mismo `ResponseData`.
3. Validar que acciones inválidas lancen `ArgumentException`.
4. Mantener separado el caso de excepción de los casos válidos.

---

## 6. `StoreOrderActions`

Métodos de interfaz:

1. `StoreOrderMgmtMoveOrder`
2. `StoreOrderMgmtVoidOrder`
3. `StoreOrderMgmtReOpenOrder`

Ya existían pruebas para los dos primeros. Se agregó cobertura directa para:

```csharp
StoreOrderMgmtReOpenOrder_GivenData_CallsExecute
```

Validaciones usadas:

1. Stored procedure `StoreOrderMgmtStoredProcedures.spStoreOrderMgmtReOpenOrder`.
2. Parámetro `OrdersID`.
3. Llamada a `_queries.Execute`.
4. Retorno del mismo `ResponseData`.

---

## 7. `SharedOrderDetailOperationsService`

Métodos públicos identificados:

1. `InsertOrderDetailAndGetData`
2. `InsertOrderDetailAutoSchedulingD2C`
3. `UpdateOrderDetailAndGetData`
4. `OrderSchedulingErrorUpdate`
5. `DeleteOrderDetail`
6. `IssueAutoSchedulerRequests`
7. `InsertOrderDetailKitRecords`
8. `InsertOrderDetailAttribute`
9. `OrderDetailIsFundraiser`
10. `UpdateOrderDetailD2CStage`

Métodos sin cobertura directa detectados en ese momento:

1. `InsertOrderDetailAttribute`
2. `OrderDetailIsFundraiser`
3. `UpdateOrderDetailD2CStage`

Pruebas agregadas:

```csharp
InsertOrderDetailAttribute_ValidRequest_UsesExpectedStoredProcedureAndReturnsResult
InsertOrderDetailAttribute_ResponseWithoutSuccessStatus_ThrowsException
OrderDetailIsFundraiser_LocationTypeID_ReturnsExpectedResult
UpdateOrderDetailD2CStage_ValidRequest_UsesStagingStoredProcedureAndReturnsResult
UpdateOrderDetailD2CStage_ResponseWithoutSuccessStatus_ThrowsException
```

Detalles importantes:

1. `OrderDetailIsFundraiser` retorna `true` cuando `LocationTypeID == 14`.
2. `InsertOrderDetailAttribute` usa:
   - `OrderStoredProcedures.InsertOrderDetailAttributeSproc`
   - `OrderStagingStoredProcedures.OrderDetailAttributeSvcIntStageInsert` cuando `useStagingTables = true`
3. `UpdateOrderDetailD2CStage` usa:
   - `OrderStagingStoredProcedures.OrderDetailSvcIntStageUpdateD2C`
4. Ambos métodos lanzan excepción cuando el resultado viene vacío o con `status` distinto de `success`.

---

## 8. `OrderSaveController`

Método trabajado:

```csharp
saveOrder(OrderSaveRequestModel request)
```

Foco del usuario:

Probar que se llama `SendDataBackToStore` si el objeto `response.data` contiene una propiedad `order` y ese `order` tiene `orderStatusId == 17`.

Lógica relevante:

1. Para `request.action == "saveOrder"`, se ejecutan acciones internas como `insertOrder`, `updateOrder`, pagos, etc.
2. Si `tempResponse.status == success`, se llama:
   - `CommitStagingTables()`
   - `IssuePendingAutoScheduler()`
3. Luego se arma una lista de respuestas.
4. Se busca si algún `value.data` contiene propiedad `order`.
5. Si existe, se intenta castear a `Dictionary<string, object>`.
6. Si `orderData["orderStatusId"] == 17`, se llama:

```csharp
_orderActions.SendDataBackToStore(
    "Orders",
    decimal.Parse(orderData["ordersId"].ToString()),
    int.Parse(orderData["locationId"].ToString()),
    ISharedOrderOperationsService.TransLogAction.Update);
```

---

## 9. Clase base confirmada para `OrderSaveControllerTests`

El usuario compartió esta clase como base funcional y confirmada.

Mocks:

```csharp
private readonly Mock<IOrderActions> _orderActions;
private readonly Mock<IOrderManagementActions> _manageOrderActions;
```

Constructor:

```csharp
public OrderSaveControllerTests()
{
  _orderActions = new Mock<IOrderActions>();
  _manageOrderActions = new Mock<IOrderManagementActions>();
}
```

Helper confirmado:

```csharp
private OrderSaveRequestModel GetComplexOrderActionRequestModel(string action)
{
  return new OrderSaveRequestModel
  {
    action = "saveOrder",
    user = new BaseRequestModel.UserModel
    {
      firstName = "test",
      lastName = "test",
      employeeId = "111"
    },
    data = new List<OrderSaveDataModel>
    {
      new OrderSaveDataModel
      {
        action = action,
        actionId = "update-order-1",
        sequence = 1,
        actionData = new
        {
          order = new
          {
            ordersId = 12345m,
            locationId = 5999,
            orderStatusId = 17,
            orderDate = DateTime.Now,
            drawerDate = DateTime.Now,
            locationTypeId = 1,
            b2BOrder = 0
          }
        }
      }
    }
  };
}
```

---

## 10. Pruebas trabajadas para `OrderSaveController`

Prueba base confirmada:

```csharp
SaveOrder_UpdateOrderResponseWithOrderStatusId17_CallsSendDataBackToStore
```

Pruebas adicionales generadas:

```csharp
SaveOrder_UpdateOrderResponseWithOrderStatusIdDifferentThan17_DoesNotCallSendDataBackToStore
SaveOrder_ResponseDataWithoutOrder_DoesNotCallSendDataBackToStore
SaveOrder_FailedAction_DoesNotCommitOrSendDataBackToStore
```

La prueba inicialmente propuesta sobre validar el retorno agregado se descartó para este contexto.

### 10.1 Setup común

```csharp
var request = GetComplexOrderActionRequestModel("updateOrder");

_orderActions
    .Setup(x => x.UpdateOrder(It.IsAny<Core.Models.Dtos.OrderDto>()))
    .Returns(responseData);

var controller = new OrderSaveController(_orderActions.Object, _manageOrderActions.Object);
```

### 10.2 Verificaciones comunes para success

```csharp
_orderActions.Verify(
    x => x.UpdateOrder(It.IsAny<Core.Models.Dtos.OrderDto>()),
    Times.Once);

_orderActions.Verify(
    x => x.CommitStagingTables(),
    Times.Once);

_orderActions.Verify(
    x => x.IssuePendingAutoScheduler(),
    Times.Once);
```

### 10.3 Verificación cuando `orderStatusId == 17`

```csharp
_orderActions.Verify(
    x => x.SendDataBackToStore(
        "Orders",
        ordersId,
        locationId,
        ISharedOrderOperationsService.TransLogAction.Update),
    Times.Once);
```

### 10.4 Verificación cuando no debe enviar a tienda

```csharp
_orderActions.Verify(
    x => x.SendDataBackToStore(
        It.IsAny<string>(),
        It.IsAny<decimal>(),
        It.IsAny<int>(),
        It.IsAny<ISharedOrderOperationsService.TransLogAction>()),
    Times.Never);
```

### 10.5 Verificación cuando la acción falla

```csharp
_orderActions.Verify(x => x.CommitStagingTables(), Times.Never);
_orderActions.Verify(x => x.IssuePendingAutoScheduler(), Times.Never);
```

---

## 11. Convenciones de estilo observadas

1. El usuario usa indentación de dos espacios en algunas clases recientes.
2. Se aceptan nombres largos de pruebas si son claros.
3. El usuario prefiere pruebas enfocadas en comportamiento observable.
4. La validación de `ResponseData.status` suele ser útil, pero no debe desplazar el foco principal de interacción.
5. Cuando un helper ya existe y funciona, se debe reutilizar.
6. Si el usuario dice “ya está probada que funciona”, no hay que reescribir la base salvo solicitud expresa.

---

## 12. Pendientes o ideas futuras

Este documento puede expandirse con:

1. Nuevas clases de prueba.
2. Nuevos helpers confirmados.
3. Patrones por proyecto.
4. DTOs con propiedades críticas.
5. Stored procedures frecuentes.
6. Mensajes exactos de excepción por método.
7. Casos donde conviene agrupar pruebas.
8. Casos donde se decidió no probar cierto comportamiento.
