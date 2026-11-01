# Contrato de Interoperabilidad — v2 (simplificado)
**BCP Simulado — Proyecto de Sistemas Distribuidos**

Este documento define cómo se comunican los 4 bancos entre sí para
ejecutar transferencias interbancarias y para que los cajeros consulten
y debiten cuentas de cualquier banco. Es el contrato técnico que
complementa al reglamento general (documento 01).

## 1. Identificación entre bancos (firma digital)

- Cada banco expone una API HTTP (REST, JSON) en una URL conocida por los
  demás.
- Cada banco tiene un `banco_id` fijo (acordado en el documento 01).
- Antes de empezar el desarrollo, **cada grupo genera su propio par de
  claves Ed25519** (una privada y una pública):
  - La **clave privada nunca se comparte con nadie**, ni siquiera con el
    BCP simulado. Se guarda solo en el sistema del banco que la generó
    (variable de entorno o archivo de configuración fuera del
    repositorio).
  - La **clave pública sí se comparte**: cada grupo se la entrega al BCP
    simulado, que arma un registro con las 4 claves públicas y se lo
    distribuye a los 4 bancos. Compartir una clave pública no es un
    riesgo: solo sirve para verificar firmas, no para crearlas.
- Generar un par Ed25519 es una sola línea en casi cualquier lenguaje o
  con OpenSSL, por ejemplo:

```bash
openssl genpkey -algorithm Ed25519 -out privada.pem
openssl pkey -in privada.pem -pubout -out publica.pem
```

- **Formato obligatorio de la clave pública entregada al BCP simulado:**
  base64 de los 32 bytes crudos de la clave pública Ed25519, sin
  cabecera PEM (`-----BEGIN PUBLIC KEY-----`) ni saltos de línea. Esto
  es necesario porque distintas librerías exportan la clave en formatos
  distintos por defecto (PEM, DER, hex, base64 de bytes crudos); fijar
  un único formato evita que un banco no pueda leer la clave de otro
  aunque ambos usen Ed25519 correctamente. La mayoría de los lenguajes
  pueden extraer los 32 bytes crudos de un archivo PEM con una función
  estándar de su librería de criptografía (por ejemplo, el método para
  exportar la clave en formato "raw" en vez de "PEM"/"DER").
- El BCP simulado centraliza las 4 claves públicas, en este formato, en
  un único archivo de registro (`06_registro_bancos_v2.json`) que
  distribuye a los 4 bancos. Ese archivo también define el `banco_id` y
  la URL base de cada banco.
- Toda solicitud entre bancos debe incluir estos headers:

```
X-Bank-Id: BCO1
X-Timestamp: 2026-10-26T14:32:10Z
X-Idempotency-Key: <identificador único generado por quien llama>
X-Signature: <firma en base64, ver abajo cómo se calcula>
```

- **Cómo se firma una solicitud (lo hace quien llama):**
  1. Se arma una "cadena canónica" con los datos clave de la solicitud,
     separados por salto de línea:
     ```
     METODO
     RUTA
     X-Bank-Id
     X-Idempotency-Key
     X-Timestamp
     CUERPO_JSON_TAL_CUAL_SE_ENVIA
     ```
     Ejemplo real:
     ```
     POST
     /interbancario/creditos
     BCO1
     3fa85f64-5717-4562-b3fc-2c963f66afa6
     2026-10-26T14:32:10Z
     {"transfer_id":"3fa85f64-5717-4562-b3fc-2c963f66afa6","cuenta_destino":"1002",...}
     ```
  2. Se firma esa cadena (codificada en UTF-8) con la **clave privada**
     del banco que llama, usando Ed25519.
  3. La firma resultante se codifica en base64 y se envía en el header
     `X-Signature`.

- **Cómo se verifica (lo hace quien recibe):**
  1. Busca la clave **pública** de `X-Bank-Id` en su copia del registro.
     Si ese `banco_id` no está registrado, responde `BANCO_NO_RECONOCIDO`.
  2. Reconstruye la misma cadena canónica a partir del método, la ruta,
     los headers y el cuerpo recibidos (tal cual llegaron, sin
     reformatear el JSON).
  3. Verifica la firma de `X-Signature` contra esa cadena, usando la
     clave pública del banco. Si no verifica, responde `401
     Unauthorized` con `FIRMA_INVALIDA`.
  4. Verifica que `X-Timestamp` no difiera del reloj del receptor en más
     de 5 minutos (en cualquier sentido). Si está fuera de rango,
     responde `401 Unauthorized` con `TIMESTAMP_FUERA_DE_RANGO`. Esto
     evita que alguien capture una solicitud válida y la reenvíe sin
     cambios mucho después.
- Con este esquema, **ningún banco puede suplantar a otro**: aunque todos
  conozcan la clave pública de los demás, solo el dueño de la clave
  privada correspondiente puede producir una firma válida en su nombre.
- Se recomienda HTTPS si el entorno de la demo lo permite; no es
  obligatorio usar certificados propios ni mTLS (la firma ya resuelve la
  identificación; HTTPS acá solo agrega confidencialidad en tránsito).

## 2. Transferencias internas (mismo banco)

Cuando origen y destino son cuentas del **mismo banco**, no se usa nada
de este contrato de interoperabilidad: no hay llamada HTTP a otro banco,
no hay firma, no hay estados `RESERVADA`/`PENDIENTE`. Todo se resuelve
con una sola transacción local de base de datos, porque no hay dos
sistemas independientes que coordinar.

1. **Revisar idempotencia primero.** Antes de tocar saldos, verificar si
   ya existe un registro con esa `X-Idempotency-Key` (por ejemplo, con
   una columna `idempotency_key` con restricción `UNIQUE` en la tabla de
   operaciones). Si ya existe, se devuelve el resultado guardado sin
   repetir el efecto económico.
2. **Abrir una transacción de base de datos** (`BEGIN`). Todo lo que
   sigue debe aplicarse como un bloque único: o se confirma entero
   (`COMMIT`) o se revierte entero (`ROLLBACK`); nunca queda a medias.
3. **Bloquear ambas cuentas en un orden consistente**, por ejemplo por
   número de cuenta ascendente, sin importar cuál es origen y cuál
   destino (`SELECT ... FOR UPDATE` o el equivalente del motor usado).
   Este orden fijo es importante: si dos transferencias cruzadas se
   ejecutan al mismo tiempo (A→B y B→A), bloquear siempre en el mismo
   orden evita que ambas transacciones se bloqueen entre sí
   indefinidamente (deadlock).
4. **Validar con los datos ya bloqueados**: ambas cuentas activas, saldo
   suficiente en origen, límite por operación y límite diario no
   excedidos (documento 01, sección 4). Si algo falla, `ROLLBACK` y se
   responde el código de error correspondiente (mismo catálogo de la
   sección 6 de este documento).
5. **Debitar y acreditar** dentro de la misma transacción, y registrar la
   operación (para auditoría e idempotencia) también dentro de esa
   transacción.
6. **Confirmar** (`COMMIT`). Recién ahí el cambio es visible y
   definitivo.

Ejemplo ilustrativo en SQL (cada grupo lo adapta a su motor y lenguaje):

```sql
BEGIN;

INSERT INTO operaciones (idempotency_key, tipo, estado)
VALUES ('3fa85f64-...', 'TRANSFERENCIA_INTERNA', 'EN_PROCESO');
-- Si la clave ya existe, esto falla por la restricción UNIQUE; el
-- código de la aplicación debe capturarlo y devolver el resultado
-- guardado anteriormente para esa misma clave.

SELECT saldo, estado FROM cuentas WHERE numero_cuenta = '1001' FOR UPDATE;
SELECT saldo, estado FROM cuentas WHERE numero_cuenta = '1002' FOR UPDATE;
-- (bloqueo en orden consistente: menor número de cuenta primero)

-- (validaciones en código: cuentas activas, saldo suficiente, límites)

UPDATE cuentas SET saldo = saldo - 150000 WHERE numero_cuenta = '1001';
UPDATE cuentas SET saldo = saldo + 150000 WHERE numero_cuenta = '1002';

UPDATE operaciones SET estado = 'COMPLETADA'
WHERE idempotency_key = '3fa85f64-...';

COMMIT;
```

No hay timeout ni consulta de estado para este caso: al no haber red
entre origen y destino, no existe el escenario de "no sé si se aplicó"
que motiva el protocolo interbancario de la sección 4.

## 3. Idempotencia

- `X-Idempotency-Key` es obligatorio en toda solicitud con efecto
  económico (crear transferencia, debitar por cajero). El mismo
  principio de idempotencia aplica a las transferencias internas de la
  sección 2, aunque ahí se resuelve con una restricción `UNIQUE` en la
  base de datos local en vez de coordinarse entre dos bancos.
- Cada banco debe guardar, por un tiempo razonable (al menos 24 horas),
  el resultado de cada clave de idempotencia que procesó.
- Si llega una solicitud repetida con la misma clave:
  - Si los datos de la solicitud son iguales a la primera vez → se
    responde con el mismo resultado que la primera vez, sin volver a
    aplicar el efecto económico.
  - Si los datos son distintos → se responde `409 Conflict` (la clave ya
    se usó con otro contenido).
- Formato sugerido de clave: UUID v4.

## 4. Protocolo de transferencia interbancaria

Participan dos bancos: el **origen** (de donde sale el dinero) y el
**destino** (donde se acredita). El origen siempre inicia el proceso.

### Paso 1 — Reserva en origen

El banco origen valida localmente: cuenta activa, saldo suficiente,
límite por operación y límite diario no excedidos. Si todo es correcto,
**reserva** el importe (lo descuenta del saldo disponible, pero no lo da
por transferido todavía) y genera un `transfer_id` único.

Si alguna validación falla, la operación termina ahí con estado
`RECHAZADA` y no se contacta al destino.

### Paso 2 — Solicitud de crédito al destino

El banco origen llama al banco destino:

```
POST /interbancario/creditos
X-Bank-Id: <origen>
X-Timestamp: <fecha/hora actual, UTC>
X-Idempotency-Key: <transfer_id>
X-Signature: <firma calculada como se describe en la sección 1>

{
  "transfer_id": "uuid",
  "cuenta_destino": "numero_cuenta",
  "banco_origen": "BCO1",
  "cuenta_origen": "numero_cuenta",
  "importe": 150000,
  "moneda": "PYG",
  "concepto": "texto libre, opcional"
}
```

El banco destino valida: cuenta existe, está activa, el `banco_origen`
es uno de los 4 bancos reconocidos. Si todo es correcto, **acredita el
importe de forma atómica** y responde `200 OK` con estado `ACREDITADO`.
Si alguna validación falla, responde con estado `RECHAZADO` y un motivo
(ver catálogo de errores).

Esta operación debe ser idempotente: si el banco destino recibe de nuevo
el mismo `transfer_id`, debe devolver el mismo resultado sin acreditar
dos veces.

### Paso 3 — Confirmación en origen

- Si el destino respondió `ACREDITADO`: el origen convierte la reserva en
  débito definitivo y marca la transferencia como `COMPLETADA`.
- Si el destino respondió `RECHAZADO`: el origen libera la reserva
  (el dinero vuelve a estar disponible) y marca la transferencia como
  `FALLIDA`, guardando el motivo.

### Paso 4 — Qué hacer si no hay respuesta (timeout)

Si el origen no recibe respuesta del destino en un tiempo razonable
(se sugiere 5 segundos, acordable entre los grupos), **no debe asumir
nada**. Debe consultar el estado antes de decidir:

```
GET /interbancario/creditos/{transfer_id}
X-Bank-Id: <origen>
X-Timestamp: <fecha/hora actual, UTC>
X-Idempotency-Key: <nuevo uuid para esta consulta>
X-Signature: <firma de esta solicitud>
```

- Si el destino responde que el crédito existe y fue aplicado → el
  origen confirma el débito (como en el paso 3, caso éxito).
- Si el destino responde que no tiene registro de ese `transfer_id` → el
  origen puede reintentar el paso 2 con el mismo `transfer_id`
  (es seguro por ser idempotente), o liberar la reserva y marcar la
  transferencia como `FALLIDA` tras un número razonable de reintentos.
- Si el destino tampoco responde a la consulta → la transferencia queda
  en estado `PENDIENTE` y debe poder consultarse más tarde; **no se
  revierte automáticamente sin confirmar antes que el crédito no fue
  aplicado.**

### Estados posibles de una transferencia

`RECIBIDA` → `RESERVADA` → (`COMPLETADA` | `FALLIDA` | `PENDIENTE`)

- `RECIBIDA`: la solicitud llegó y pasó validación de formato.
- `RESERVADA`: el importe fue reservado en origen, en proceso.
- `COMPLETADA`: débito y crédito confirmados en ambos bancos.
- `FALLIDA`: no se acreditó en destino; reserva liberada en origen.
- `PENDIENTE`: esperando poder confirmar el resultado real con el
  destino (caso de timeout sin resolver). Debe poder consultarse y
  resolverse manualmente o por reintento posterior.

## 5. Endpoints mínimos que cada banco debe exponer

| Método y ruta | Uso |
|---|---|
| `POST /interbancario/creditos` | Recibe una solicitud de crédito de otro banco (paso 2) |
| `GET /interbancario/creditos/{transfer_id}` | Consulta el estado de un crédito ya solicitado (paso 4) |
| `GET /cuentas/{numero_cuenta}` | Consulta datos básicos de una cuenta propia (existe, activa, titular) — usado por cajeros y por otros bancos para validaciones simples |
| `POST /cajero/autorizaciones` | Autoriza y debita un retiro de cajero sobre una cuenta propia (ver documento 03) |
| `GET /cajero/autorizaciones/{operacion_id}` | Consulta el estado de una autorización de cajero ya solicitada — mismo uso que la consulta de créditos interbancarios, pero para retiros (ver documento 03) |

Cada grupo puede exponer endpoints adicionales para su propio frontend;
estos cuatro son los que deben responder exactamente según este contrato
para garantizar la interoperabilidad.

## 6. Formato de errores

Toda respuesta de error debe tener este formato:

```json
{
  "error": "CODIGO_ERROR",
  "mensaje": "Descripción legible del error"
}
```

### Catálogo mínimo de códigos de error

| Código | HTTP | Significado |
|---|---|---|
| `CUENTA_NO_EXISTE` | 404 | La cuenta indicada no existe en este banco |
| `CUENTA_INACTIVA` | 409 | La cuenta está bloqueada o cerrada |
| `PIN_INCORRECTO` | 403 | El PIN enviado no coincide con el de la cuenta (solo aplica a operaciones de cajero, ver documento 03) |
| `FONDOS_INSUFICIENTES` | 409 | El saldo disponible no alcanza |
| `LIMITE_EXCEDIDO` | 409 | Se supera el límite por operación o el diario |
| `BANCO_NO_RECONOCIDO` | 400 | El `banco_id` de origen no está en el registro de los 4 bancos |
| `SOLICITUD_INVALIDA` | 400 | Falta un campo requerido o el formato es incorrecto |
| `FIRMA_INVALIDA` | 401 | La firma no verifica contra la clave pública del `banco_id` declarado |
| `TIMESTAMP_FUERA_DE_RANGO` | 401 | `X-Timestamp` difiere más de 5 minutos del reloj del receptor |
| `CONFLICTO_IDEMPOTENCIA` | 409 | La clave de idempotencia ya se usó con datos distintos |
| `NO_ENCONTRADO` | 404 | El `transfer_id` consultado no existe |
| `ERROR_INTERNO` | 500 | Error no controlado del banco que responde |

## 7. Registro mínimo de auditoría

Cada banco debe guardar, por cada operación con efecto económico, al
menos: `transfer_id` o identificador equivalente, fecha/hora, cuentas
origen y destino (banco + número), importe, estado final, y el código de
error si aplica. Nunca se guarda el PIN ni la clave privada en estos
registros.
