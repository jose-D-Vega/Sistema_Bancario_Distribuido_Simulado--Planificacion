# Reglamento General — v2 (simplificado)
**BCP Simulado — Proyecto de Sistemas Distribuidos**

## 1. Alcance

Este reglamento aplica a los 4 bancos participantes del proyecto. Define
las reglas de negocio mínimas que cada banco debe respetar para que las
transferencias y los retiros por cajero funcionen de forma consistente
entre todos los sistemas. El funcionamiento interno de cada banco
(tecnología, frontend, base de datos) es libre, siempre que cumpla este
reglamento y el contrato técnico (documento 02).

## 2. Moneda e importes

- Moneda única: guaraníes (PYG).
- Los importes se representan como números enteros (sin decimales).
- No se permiten importes negativos ni iguales a cero en una operación.

## 3. Cuentas

- Cada cuenta se identifica de forma única mediante: `banco_id` +
  `numero_cuenta`.
- `banco_id`: código corto y fijo asignado a cada banco (por ejemplo,
  `BCO1`, `BCO2`, `BCO3`, `BCO4`). Se acuerda entre los 4 grupos antes de
  empezar a implementar.
- Cada cuenta tiene un estado: **Activa**, **Bloqueada** o **Cerrada**.
  - Activa: opera con normalidad.
  - Bloqueada: no puede debitarse ni acreditarse (ej. sospecha de fraude
    simulada). Se usa solo para poder probar el caso de rechazo.
  - Cerrada: no admite ninguna operación.
- Un banco es la única autoridad sobre el saldo de sus propias cuentas.
  Ningún otro banco puede modificar un saldo que no es suyo directamente;
  solo puede pedírselo mediante el contrato técnico (documento 02).

## 4. Límites

| Límite | Valor |
|---|---|
| Monto máximo por transferencia | G. 10.000.000 |
| Monto máximo por retiro en cajero | G. 1.500.000 |
| Límite diario combinado por cuenta (transferencias salientes + retiros) | G. 10.000.000 |

- El límite diario se reinicia a las 00:00, hora de Paraguay
  (`America/Asuncion`).
- Una operación que se inicia antes de medianoche y se confirma después no
  necesita manejo especial: alcanza con registrar la operación con la
  fecha/hora en que fue aceptada.
- Cada banco es responsable de validar estos límites sobre sus propias
  cuentas antes de autorizar un débito.

## 5. Tipos de operación

1. **Transferencia interna**: origen y destino son cuentas del mismo
   banco. Se resuelve con una transacción local; no involucra a otros
   bancos. El detalle técnico de implementación (bloqueo de cuentas,
   orden para evitar deadlocks, idempotencia local) está en el documento
   02, sección 2.
2. **Transferencia interbancaria**: origen y destino son cuentas de bancos
   distintos. Sigue el protocolo descrito en el documento 02.
3. **Retiro por cajero**: una cuenta de cualquier banco retira dinero desde
   el cajero de cualquiera de los 4 bancos. Sigue la especificación del
   documento 03.

## 6. Principios obligatorios

Estos principios son el corazón de lo que se evalúa en la materia y son
de cumplimiento obligatorio, independientemente de la tecnología que use
cada grupo:

- **Idempotencia**: repetir la misma operación con la misma clave de
  idempotencia nunca debe producir un doble efecto (doble débito, doble
  crédito).
- **Nada de dinero creado ni destruido**: toda transferencia exitosa debe
  reflejarse como un débito en origen y un crédito equivalente en destino.
  Si una parte falla, la otra debe revertirse o no debe ejecutarse.
- **Timeout no es resultado**: si un banco no recibe respuesta a tiempo de
  otro banco, no debe asumir éxito ni fracaso. Debe consultar el estado
  antes de reintentar, revertir o informar al usuario.
- **Trazabilidad mínima**: toda operación con efecto económico debe quedar
  registrada con fecha/hora, cuentas involucradas, importe, resultado y un
  identificador único de operación.
- **Nunca registrar el PIN** ni ningún dato sensible equivalente en texto
  plano en ningún registro, log o base de datos.

## 7. Rol del BCP simulado

El BCP simulado (coordinación general del proyecto):

- Asigna y mantiene la lista de `banco_id` de los 4 grupos.
- Resuelve dudas sobre la interpretación de este reglamento y del
  contrato técnico.
- Puede pedir a cada grupo evidencia de que cumple los casos de prueba del
  documento 04 antes de la integración final.
- No participa en ninguna transferencia ni retiro: toda operación es
  directamente entre bancos, o entre un cajero y el banco emisor de la
  cuenta.

## 8. Qué queda fuera del alcance obligatorio

Para que el proyecto sea viable en el tiempo disponible, **no** son
requisito para aprobar (aunque un grupo puede sumarlos como mejora
opcional):

- Cifrado mTLS o certificados propios entre bancos.
- Recuperación automática ante caída de un banco en medio de una
  operación (alcanza con dejar la operación en un estado consultable).
- Conciliación o reconciliación automática de operaciones con resultado
  desconocido.
- Comisiones, múltiples monedas, o tipos de cuenta adicionales (caja de
  ahorro, cuenta corriente, etc.) salvo que un grupo quiera sumarlo por su
  cuenta sin romper el contrato común.

## 9. Decisiones pendientes de acordar entre los 4 grupos

Antes de empezar a programar, los 4 grupos deben ponerse de acuerdo sobre:

- Los 4 valores de `banco_id`.
- La URL/puerto donde cada banco expondrá su API (puede ser localhost con
  puertos distintos para la demo, o IPs de red local).
- Que cada grupo genere su propio par de claves Ed25519 y entregue al
  BCP simulado únicamente su clave **pública**, para armar el registro
  compartido (ver documento 02, sección 1).
- Fecha de una prueba de integración intermedia, antes de la entrega
  final.
