# Plan de Pruebas Mínimo — v2 (simplificado)
**BCP Simulado — Proyecto de Sistemas Distribuidos**

Estos son los casos que cada banco debería poder demostrar, como mínimo,
antes de la integración final con los otros 3 grupos. No reemplazan las
pruebas internas de cada equipo; son el piso común de conformidad con el
reglamento (documento 01) y el contrato técnico (documento 02).

## A. Cuentas y transferencias internas

1. Transferencia interna exitosa (mismo banco): el saldo de origen baja y
   el de destino sube exactamente el importe transferido.
2. Transferencia interna rechazada por fondos insuficientes: ningún saldo
   cambia.
3. Transferencia interna rechazada por cuenta destino inexistente.
4. Transferencia interna rechazada por cuenta origen bloqueada o cerrada.
5. Transferencia interna rechazada por superar el límite por operación
   (G. 10.000.000).

## B. Transferencias interbancarias

6. Transferencia interbancaria exitosa: el banco origen debita, el banco
   destino acredita, ambos quedan con estado `COMPLETADA`/equivalente.
7. Transferencia interbancaria rechazada porque la cuenta destino no
   existe en el banco destino: el banco origen libera la reserva y el
   saldo de origen queda sin cambios.
8. Transferencia interbancaria rechazada por fondos insuficientes en
   origen (no debería llegar a contactar al destino).
9. Transferencia interbancaria rechazada por superar el límite diario
   combinado de la cuenta origen.
10. Reintento de una transferencia interbancaria ya completada, usando la
    misma clave de idempotencia: el resultado es el mismo que la primera
    vez y **no** se acredita ni debita una segunda vez.
11. Simulación de timeout: el banco destino tarda en responder (o no
    responde) y el banco origen, en vez de reintentar a ciegas, consulta
    el estado del crédito antes de decidir.

## C. Autenticación e idempotencia

12. Solicitud entre bancos sin `X-Api-Key` válida: debe rechazarse con
    `401 Unauthorized`, sin aplicar ningún efecto económico.
13. Dos solicitudes con la misma clave de idempotencia pero datos
    distintos (por ejemplo, distinto importe): debe responder
    `CONFLICTO_IDEMPOTENCIA`, sin aplicar ningún efecto económico de la
    segunda solicitud.

## D. Concurrencia

14. Dos solicitudes simultáneas que intentan debitar la misma cuenta por
    un importe que, sumado, supera el saldo disponible: solo una debe
    tener éxito; la otra debe rechazarse por fondos insuficientes (no se
    permite que el saldo quede negativo).

## E. Cajero

15. Retiro exitoso desde el cajero de un banco, sobre una cuenta de otro
    banco.
16. Retiro rechazado por PIN incorrecto (respuesta con `motivo_rechazo:
    PIN_INCORRECTO`).
17. Retiro rechazado por importe mayor al límite de retiro
    (G. 1.500.000).
18. Retiro rechazado por límite diario combinado ya alcanzado (por
    ejemplo, tras una transferencia previa el mismo día).
19. Reintento de un retiro con el mismo `operacion_id` tras timeout
    simulado: no se debita dos veces.

## F. Auditoría

20. Verificar que, tras ejecutar los casos anteriores, cada banco tiene
    un registro consultable (log o tabla) de cada operación con efecto
    económico, con fecha/hora, cuentas, importe y resultado — y que en
    ningún registro aparece el PIN en texto plano.

## Cómo usar este plan

- Cada grupo ejecuta estos casos contra su propio banco antes de la
  integración.
- En la prueba de integración, se repiten los casos B, C y E pero contra
  los bancos de los otros 3 grupos, para confirmar que el contrato se
  cumple de extremo a extremo y no solo dentro de cada banco por
  separado.
- Si un grupo quiere sumar casos adicionales (más límites, más tipos de
  rechazo, pruebas de carga), es bienvenido, pero estos 20 son el mínimo
  exigido para considerar un banco conforme al reglamento.
