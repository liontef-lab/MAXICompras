Proceso de Compras MAXI (Bizagi 11.2.5)

Implementación del proceso de compras con verificación de stock, generación de orden de compra, validación de carpeta, evaluación de propuestas y cancelación automática si el proveedor no responde en 3 días.
Incluye integración SOAP con OfficeService.asmx para generar el número de orden.

Índice

Arquitectura y alcance

Requisitos

Estructura del repositorio

Modelo de datos

Diseño del proceso (BPMN)

Formularios

Reglas y ruteo (gateways)

Estados del caso

Integración SOAP: GeneratePurchaseOrder

Notificaciones

Pruebas (Work Portal)

Exportables (BEX/BDEX) y entrega

Solución de errores frecuentes

Arquitectura y alcance

Bizagi Modeler: documentación del flujo (archivo .bpm).

Bizagi Studio (On-Premises .NET): automatización, formularios, reglas y conectores.

Web Service: https://legacy.bizagi.com/OfficeSupplyWS/OfficeService.asmx?WSDL

Método usado: GeneratePurchaseOrder

Entrada: sFileContent (string)

Salida: GeneratePurchaseOrderResult (string con el número de orden)

Requisitos

Bizagi Studio v11.2.5 (o 11.2.x)

SQL Server (Express funciona para desarrollo)

Conectividad a legacy.bizagi.com (para consumir el WSDL)

Usuario con permisos de deployment local

Estructura del repositorio
/docs
   README.md              ← este archivo
/model
    ComprasMAXI.bpm ← diagrama BPMN (Modeler)
/export
   MAXICompras.bex ← automatización (Bizagi Studio)
   MAXICompras.bdex← datos maestros/paramétricos


Modelo de datos

Entidad de Proceso: MAXIProcesoCompras

Atributo	Tipo	Comentario
Sucursal	String	Sucursal solicitante
FechaSolicitud	Date	Fecha de la solicitud
Productos	String	Descripción/Lista
Cantidad	Integer	Cantidad
HayStock	Boolean	Calculado/entrada
CarpetaCompleta	Boolean	Validación de soporte
PropuestaAprobada	Boolean	Decisión del supervisor
ProveedorRespondio	Boolean	Marca de respuesta
PropuestaAdjunta	File	Soporte de propuesta (File uploads)
Soportes	File	Soportes de carpeta (File uploads ECM si aplica)
NumerodeOrden	String	Número de orden (WS)
ResultadoOrden	String	Campo de auditoría 

Entidad parámetro: Estado (Parameter entity)

Atributos (string): Nueva, EnCurso, Cancelada, Atendida

Relación en MAXIProcesoCompras → Estado (uno a uno)

Diseño del proceso (BPMN)

Lanes: Compras, Inventario, Adquisiciones, Supervisor.
Hitos: “Milestone 1”.

Flujo (resumen):

Registrar solicitud (Compras – User Task)

Validar stock disponible (Inventario – Service/Script Task automática)

Gateway XOR ¿Hay stock en bodega?

Sí → Entregar productos → Notificar a sucursal (Intermediate throw message) → Solicitud atendida (End)

No → Generar orden de compra (Adquisiciones – Service Task con WS) → Preparar carpeta con solicitud → Validar carpeta completa → Gateway XOR ¿Carpeta completa?

No → Completar carpeta (User Task) y reintenta validación

Sí → Enviar orden de compra al proveedor (User/Service Task) → Timer (<=3 días) → Gateway XOR ¿Proveedor respondió en 3 días?

No → Cancelar solicitud (automática) → Notificar a Compras Orden cancelada → End

Sí → Registrar respuesta del proveedor → Revisar propuestas proveedores (Supervisor) → Gateway XOR ¿Propuesta aprobada?

No → Notificar rechazo

Sí → Seleccionar proveedor → Realizar la compra (Compras – User Task) → End

Usa timer intermediate event de 3 días antes de la evaluación de respuesta del proveedor.

Formularios
1) Registrar solicitud

Campos: Sucursal, FechaSolicitud, Productos, Cantidad, PropuestaAdjunta (File uploads)
Validaciones básicas (requeridos, formatos).

2) Validar carpeta completa / Completar carpeta

CarpetaCompleta (Yes/No)

Soportes (File uploads o File uploads ECM)

Muy importante: en el panel Properties del control File uploads selecciona Data source → MAXIProcesoCompras.Soportes. Si no lo asignas, verás “Data source property is required”.

3) Revisar propuestas proveedores

PropuestaAprobada (Yes/No)

Visualizar PropuestaAdjunta, ProveedorRespondio, comentarios.

Reglas y ruteo (gateways)
¿Hay stock en bodega?

Flujo “Sí” (Always): cuando MAXIProcesoCompras.HayStock = true
Flujo “No”: Based on expression → condición:

if (<MAXIProcesoCompras.HayStock> = false) return true;
return false;


Expression manager → Behaviour → Based on the result of an expression → pestaña New → agrega regla.

¿Carpeta completa?

Sí: <MAXIProcesoCompras.CarpetaCompleta> = true
No: <MAXIProcesoCompras.CarpetaCompleta> = false

¿Proveedor respondió en 3 días?

Timer: Intermediate event (3 días)

Sí: <MAXIProcesoCompras.ProveedorRespondio> = true

No: el flujo “NO” será el default o Based on expression con false.

¿Propuesta aprobada?

Sí: <MAXIProcesoCompras.PropuestaAprobada> = true

No: <MAXIProcesoCompras.PropuestaAprobada> = false

Estados del caso

Usa reglas On Exit de cada actividad para setear el parámetro Estado.

Expresiones (Expression → On Exit):

Registrar solicitud → a “Nueva”

<MAXIProcesoCompras.Estado> = <Estado.Nueva>;


Cualquier actividad “en curso” (por ejemplo, al entrar a Preparar carpeta…):

<MAXIProcesoCompras.Estado> = <Estado.EnCurso>;


Cancelar solicitud:

<MAXIProcesoCompras.Estado> = <Estado.Cancelada>;


Solicitud atendida (rama de stock sí):

<MAXIProcesoCompras.Estado> = <Estado.Atendida>;

Integración SOAP: GeneratePurchaseOrder

Objetivo: invocar GeneratePurchaseOrder y guardar el número de orden en MAXIProcesoCompras.NumerodeOrden.

0) Preparar dato de entrada (evitar “Value cannot be null: s”)

Antes de llamar al WS (por ejemplo, On Exit de Preparar carpeta con solicitud o On Enter de Generar orden de compra) inicializa un valor en NumerodeOrden para garantizar que sFileContent no sea nulo:

if (IsNullOrEmpty(<MAXIProcesoCompras.NumerodeOrden>)) {
    <MAXIProcesoCompras.NumerodeOrden> = "REQ-" & CStr(Me.Case.Id) & "-" & FormatDate(Now(), "yyyyMMddHHmmss");
}

1) Abrir el asistente del conector

Selecciona la tarea automática Generar orden de compra

Propiedades → Define Integration Interfaces (Optional) → Web service connector

2) Resolver URL

Service Type: SOAP

URL: https://legacy.bizagi.com/OfficeSupplyWS/OfficeService.asmx?WSDL → Go

Service port: OfficeServiceSoap12(SOAP 1.2)

Interface methods: selecciona GeneratePurchaseOrder

Interface name: OfficeService (o el que prefieras) → Next

3) Input parameters (Data to send)

Param. requerido: sFileContent (string)

Mapping recomendado:

Simple mapping: arrastra MAXIProcesoCompras.NumerodeOrden → sFileContent

(Alternativa): Advanced mapping (XSLT) si quieres construir un payload. Ejemplo mínimo válido:

<?xml version="1.0" encoding="utf-8"?>
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform" version="1.0">
  <xsl:output method="xml" encoding="utf-8" indent="yes"/>
  <xsl:template match="/">
    <GeneratePurchaseOrder xmlns="http://tempuri.org/">
      <sFileContent>
        <xsl:value-of select="/App/MAXIProcesoCompras/NumerodeOrden"/>
      </sFileContent>
    </GeneratePurchaseOrder>
  </xsl:template>
</xsl:stylesheet>


Si solo envías el string, el simple mapping es suficiente.

4) Response data (Output)

GeneratePurchaseOrderResult (string) → mapea a MAXIProcesoCompras.NumerodeOrden

(Opcional) guarda una copia en ResultadoOrden.

5) Error handling

Action: Throw Exception (o Continue si prefieres no romper el caso).

(Opcional) Add error validation para interpretar mensajes del servicio.

6) Guardar y publicar

Finaliza el asistente → Save

En el Wizard: continúa hasta Execute para probar.

Notificaciones

Notificar a sucursal: Intermediate Throw Message modelado como Service Task o Script Task con envío de correo (Template).

Notificar orden cancelada / rechazo: Email Template simple con campos del caso.

Pruebas (Work Portal)

Run → abrir Work Portal.

Crear nuevo caso: completa Sucursal, FechaSolicitud, Productos, Cantidad.

En Validar stock: pon HayStock = true/false para cubrir ambas ramas.

Si NO hay stock:

Completa Preparar carpeta y Validar carpeta completa = Sí.

La tarea Generar orden de compra invocará el WS: verifica que MAXIProcesoCompras.NumerodeOrden quede con el valor devuelto.

Forzar la ruta sin respuesta para validar la cancelación tras 3 días (reduce el tiempo en pruebas si lo deseas).

Exportables (BEX/BDEX) y entrega

BEX: Studio → Export/Import → Export Process → selecciona el proceso → genera .bex.

BTEX: Studio → Data → Entities → exporta datos de la entidad parámetro (Estado) a .btex.

Entrega final:

model/MAXICompras.bpm

export/MAXICompras.bex

export/MAXIComprasMAXI.btex

docs/README.md (este)

Solución de errores frecuentes
“Value cannot be null. Parameter name: s”

El servicio espera sFileContent distinto de null.
Solución: Inicializa MAXIProcesoCompras.NumerodeOrden antes de invocar el WS (ver sección Preparar dato de entrada).

“No se puede transformar los datos con el mapeo avanzado / XSLT”

El XSLT no encuentra el nodo o el namespace no coincide.
Soluciones:

Prefiere simple mapping (arrastrar atributo → parámetro) si envías solo strings.

Si usas XSLT, respeta el namespace y la ruta del nodo (/App/MAXIProcesoCompras/...).

“Not supported event definition for an intermediate throw event” (Modeler)

Suele ocurrir si se usa un evento intermedio arrojado con una definición inválida.
Solución: cambia a Service Task de notificación o usa Message throw estándar sin correlaciones avanzadas.

Control File uploads marca “Data source property is required”

No asignaste el Data source.
Solución: Properties → Data source → selecciona MAXIProcesoCompras.Soportes (o el atributo File correcto).

Notas finales

El flujo propuesto cumple con lo solicitado y usa solo el método GeneratePurchaseOrder.

Si en el futuro deseas validar proveedor o obtener número de orden por otro método, repite el asistente con VerifyVendor / PurchaseOrderNumber.

Mantén el Estado del caso actualizado en cada transición para que los reportes sean consistentes.