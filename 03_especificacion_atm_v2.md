# Especificación de Cajero (ATM) — v2 (simplificado)
**BCP Simulado — Proyecto de Sistemas Distribuidos**

## 1. Idea general

Cada banco puede implementar su propio cajero (interfaz simulada), pero
ese cajero debe poder operar con cuentas de **cualquiera de los 4
bancos**, no solo las propias. Para lograrlo, el cajero nunca accede
directamente a la base de datos de otro banco: siempre pide la
autorización al banco emisor de la tarjeta/cuenta mediante el contrato
técnico (documento 02).

## 2. Flujo de retiro

1. El usuario se identifica en el cajero con: `banco_id` de su cuenta,
   número de cuenta (o número de tarjeta simulado) y PIN.
2. El cajero envía la solicitud al **banco emisor** de esa cuenta
   (puede ser el mismo banco dueño del cajero, o uno distinto):

```
POST /cajero/autorizaciones
X-Bank-Id: <banco dueño del cajero>
X-Timestamp: <fecha/hora actual, UTC>
X-Idempotency-Key: <uuid generado por el cajero>
X-Signature: <firma calculada con la clave privada del banco dueño del cajero, ver documento 02>

{
  "operacion_id": "uuid",
  "banco_cuenta": "BCO2",
  "numero_cuenta": "...",
  "pin": "....",
  "importe": 300000,
  "moneda": "PYG"
}
```

3. El banco emisor valida, en este orden:
   - Que la cuenta exista y pertenezca a ese banco.
   - Que el PIN sea correcto (si no lo es, responde `RECHAZADO` con
     `PIN_INCORRECTO`, ver documento 02, sección 6).
   - Que la cuenta esté activa.
   - Que el importe no supere el límite por retiro (G. 1.500.000).
   - Que no se supere el límite diario combinado de la cuenta
     (G. 10.000.000).
   - Que haya saldo suficiente.

4. Si todo es correcto, el banco emisor **debita el importe de forma
   atómica** y responde `200 OK` con estado `AUTORIZADO`. Esta misma
   respuesta representa tanto la autorización como el débito: no hay un
   paso separado de "dispensación" a nivel de protocolo (la dispensación
   física del cajero es un detalle de interfaz, no de interoperabilidad).
5. Si alguna validación falla, el banco emisor responde con estado
   `RECHAZADO` y el motivo correspondiente (ver catálogo de errores del
   documento 02), sin debitar nada.
6. El cajero, al recibir `AUTORIZADO`, simula la entrega del dinero y
   muestra el resultado al usuario. Al recibir `RECHAZADO`, muestra el
   motivo (fondos insuficientes, PIN incorrecto, etc.) sin simular
   entrega.

Nota: la firma del header `X-Signature` identifica **al banco que llama**
(el dueño del cajero), igual que en cualquier otra solicitud entre
bancos (documento 02, sección 1). Es independiente del PIN, que
identifica al usuario frente a su banco emisor.

## 3. Idempotencia

- `X-Idempotency-Key` (o el campo `operacion_id`) es obligatorio.
- Si el cajero no recibe respuesta a tiempo y reintenta con el mismo
  `operacion_id`, el banco emisor debe devolver el mismo resultado de la
  primera vez, sin debitar dos veces.
- Igual que en las transferencias: si no hay respuesta, el cajero no debe
  asumir que el retiro falló ni que fue exitoso. Debe poder consultar el
  estado antes de reintentar o de informar un error definitivo al
  usuario. Se recomienda usar el mismo endpoint de consulta por
  `transfer_id`/`operacion_id` que el documento 02 define para
  transferencias, adaptado a operaciones de cajero.

## 4. PIN incorrecto

- Cada intento fallido se responde con `motivo_rechazo: PIN_INCORRECTO`
  (documento 02, sección 6). El cajero puede mostrarle al usuario que el
  PIN es incorrecto y permitir un nuevo intento, hasta el límite de la
  siguiente regla.
- Se simula un máximo de 3 intentos consecutivos de PIN incorrecto por
  operación de cajero (no es necesario implementar bloqueo persistente de
  tarjeta entre sesiones, salvo que un grupo quiera sumarlo como mejora).
- Al tercer intento fallido, el cajero cancela la operación y así se
  informa al usuario.

## 5. Qué queda fuera del alcance obligatorio

- Estados de "resultado desconocido" o reconciliación manual del
  cajero con el banco emisor.
- Bloqueo persistente de tarjeta entre distintas sesiones de uso.
- Retiro en efectivo con denominaciones de billetes (alcanza con simular
  el resultado "entregado" o "no entregado").

## 6. Casos que debe cubrir cada implementación de cajero

- Retiro exitoso con cuenta del mismo banco que el cajero.
- Retiro exitoso con cuenta de otro banco.
- Rechazo por fondos insuficientes.
- Rechazo por PIN incorrecto.
- Rechazo por importe mayor al límite de retiro.
- Rechazo por límite diario combinado excedido.
- Reintento con el mismo `operacion_id` tras un timeout simulado, sin
  doble débito.
