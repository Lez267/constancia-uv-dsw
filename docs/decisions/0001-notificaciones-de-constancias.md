---
status: "proposed"
date: 2026-11-03
---

# Notificaciones de constancias mediante registros pendientes persistidos y procesamiento asíncrono

## Context and Problem Statement

ConstanciaUV permite a estudiantes autorizados solicitar y obtener constancias digitales. El sistema valida la identidad institucional del solicitante (R1), genera y almacena la constancia (R2), mantiene un historial de constancias generadas (R3) y notifica al estudiante cuando su constancia está disponible (R4).

La notificación se envía por correo electrónico mediante un servicio externo a ConstanciaUV (RT1). Esto obliga a decidir cómo se comunican la generación de constancias y ese proveedor, considerando que:

* si el proveedor no está disponible, la generación y el almacenamiento de constancias deben continuar (RT2);
* las notificaciones pendientes deben conservarse para intentar su envío posteriormente (RT3);
* el sistema debe prevenir solicitudes de notificación duplicadas para una misma constancia, sin suponer que el proveedor garantiza por sí mismo una entrega única (RT4).

La pregunta es: ¿conviene intentar el envío de inmediato desde el flujo que genera la constancia, o registrar la notificación como pendiente y que otro componente se comunique después con el proveedor?

Este ADR documenta únicamente esta decisión; no describe toda la arquitectura de ConstanciaUV.

## Decision Drivers

* **Continuidad del servicio y no pérdida de notificaciones ante fallos del proveedor (RT2 y RT3).** La generación y el almacenamiento de constancias no pueden depender de que el proveedor responda, y ninguna notificación debe perderse por una falla del proveedor. Es verificable: con el proveedor caído, las constancias se siguen generando y cada notificación queda registrada como pendiente.
* **Prevención de duplicados controlada por ConstanciaUV (RT4).** Como no se presupone que el proveedor garantice entrega única, la responsabilidad de evitar solicitudes duplicadas por constancia debe estar en un punto del sistema que ConstanciaUV controle. Es verificable: para una misma constancia no puede existir más de una solicitud de notificación.

## Considered Options

* Alternativa A. Comunicación directa con recuperación de fallos
* Alternativa B. Procesamiento asíncrono mediante eventos persistidos

## Decision Outcome

Chosen option: "Alternativa B. Procesamiento asíncrono mediante eventos persistidos", porque es la que mejor responde a los dos drivers:

* **RT2:** la generación y el almacenamiento terminan sin invocar al proveedor. El único trabajo adicional del flujo de generación es registrar la notificación pendiente, por lo que una caída del proveedor no lo afecta.
* **RT3:** el registro de la notificación pendiente se guarda de forma persistente junto con la constancia (en la misma transacción), de modo que no puede existir una constancia almacenada sin su notificación pendiente. Un componente independiente retoma esos registros y los reintenta hasta que se envíen.
* **RT4:** al existir un registro único por constancia (por ejemplo, con una restricción de unicidad sobre el identificador de la constancia) y un único componente que lo procesa, el control de duplicados queda concentrado en un solo lugar bajo control de ConstanciaUV, sin depender de garantías del proveedor.

En la Alternativa A también es posible cumplir estos requisitos, pero el control de fallos, pendientes y duplicados queda mezclado con el flujo de generación, y ese flujo se mantiene expuesto a la lentitud o indisponibilidad del proveedor.

### Consequences

* Good, porque la generación y el almacenamiento de constancias quedan aislados del proveedor externo, lo que cumple RT2 sin lógica especial en el flujo principal.
* Good, porque los pendientes se conservan de forma persistente y atómica con la constancia (RT3), y los reintentos y el control de duplicados se concentran en un solo componente (RT4).
* Bad, porque el estudiante recibe el aviso con cierta demora, ya que el envío ocurre cuando el componente independiente procesa el registro y no en el instante en que se almacena la constancia.
* Bad, porque se agrega un componente que construir, operar y monitorear (por ejemplo, vigilar registros que se acumulan o que fallan repetidamente), lo que aumenta la complejidad frente a la Alternativa A.
* Riesgo: si el componente independiente envía la notificación y falla antes de marcarla como enviada, podría reintentar el envío y producir un duplicado. La restricción de unicidad evita registros duplicados, pero no elimina por completo esta ventana de fallo, porque no se supone que el proveedor garantice entrega única. Debe vigilarse y mitigarse en la implementación (por ejemplo, reduciendo esa ventana y registrando el estado de cada intento).
* Trade-off: se acepta una mayor complejidad operativa y un envío diferido a cambio de independencia frente al proveedor, pendientes que no se pierden y un control de duplicados centralizado.

### Confirmation

Las siguientes comprobaciones podrían realizarse en una implementación futura; no se han ejecutado.

* **Continuidad con el correo caído (RT2 y RT3):** usar un proveedor simulado que no responda o devuelva error. Solicitar una constancia y verificar que se genera, se almacena y aparece en el historial (R2 y R3), y que existe un registro de notificación en estado pendiente. Al restablecer el proveedor simulado, verificar que el componente independiente envía la notificación y la marca como enviada.
* **Prevención de duplicados (RT4):** revisar el esquema de datos para confirmar que existe una restricción de unicidad por constancia, e intentar registrar dos veces la notificación de la misma constancia (incluso de forma concurrente) para comprobar que se rechaza el segundo registro. Probar también con más de una instancia del componente independiente que un mismo registro pendiente no se envíe dos veces.
* **Separación de responsabilidades:** en una revisión de código, confirmar que el flujo de generación de constancias no invoca directamente al proveedor externo.

## Pros and Cons of the Options

### Alternativa A. Comunicación directa con recuperación de fallos

Después de almacenar una constancia, la aplicación intenta solicitar directamente el envío de la notificación. Si el proveedor no responde, registra el envío pendiente para reintentarlo posteriormente.

* Good, porque es más simple de construir: no requiere un componente independiente de procesamiento para el primer intento.
* Good, porque el estudiante puede recibir la notificación casi de inmediato cuando el proveedor funciona.
* Bad, porque el flujo de generación depende del tiempo de respuesta del proveedor: una respuesta lenta o un tiempo de espera agotado afectan la operación que debería continuar (RT2).
* Bad, porque la notificación pendiente solo se registra si la aplicación detecta la falla; si la aplicación falla entre el almacenamiento y el registro del pendiente, la notificación se pierde (RT3).
* Bad, porque un tiempo de espera agotado es ambiguo (el proveedor pudo haber enviado el correo sin responder) y el reintento posterior puede duplicar la notificación (RT4).

### Alternativa B. Procesamiento asíncrono mediante eventos persistidos

Después de almacenar una constancia, la aplicación registra una notificación pendiente. Un componente independiente procesa posteriormente esos registros y se comunica con el proveedor externo.

* Good, porque desacopla la generación de constancias del proveedor, por lo que una caída del proveedor no afecta a RT2.
* Good, porque la notificación pendiente se registra junto con la constancia, lo que conserva los pendientes para reintentarlos (RT3).
* Good, porque concentra reintentos y control de duplicados en un solo componente y en un registro único por constancia (RT4).
* Bad, porque agrega un componente que operar y monitorear.
* Bad, porque el envío es diferido y el estudiante puede tardar más en recibir el aviso.

## More Information

Este documento es una propuesta académica elaborada el 2026-11-03; no representa una decisión institucional aprobada, por eso su estado es `proposed`. La decisión debe revisarse si cambian las condiciones originales, por ejemplo, si el proveedor modifica su interfaz, si cambia el volumen de notificaciones o si el aviso inmediato se vuelve un requisito.
