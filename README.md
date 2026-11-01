# Sistema Bancario Distribuido Simulado — Introducción General
**Proyecto final — Sistemas Distribuidos**

## De qué trata el proyecto

El trabajo consiste en simular el sistema bancario de Paraguay. Se forman
4 grupos, cada uno construye su propio banco (frontend + backend + base
de datos, con la tecnología que elija cada grupo), y los 4 bancos deben
poder:

- Hacer transferencias entre cuentas propias y entre cuentas de bancos
  distintos.
- Operar con un cajero automático simulado que funcione con cuentas de
  **cualquiera** de los 4 bancos, no solo las propias.

Hay un líder de proyecto general, que actúa como **BCP simulado**: define
las reglas comunes que los 4 bancos deben cumplir para que sus sistemas,
construidos de forma independiente, puedan comunicarse entre sí sin
coordinarse caso por caso. Esos reglamentos son los documentos de esta
carpeta.

## Por qué hace falta un reglamento común

Cada grupo puede elegir su propia tecnología, pero las 4 implementaciones
tienen que "hablar el mismo idioma" para que una transferencia iniciada
en un banco se pueda completar en otro. El reglamento define exactamente
ese idioma común: qué datos se intercambian, en qué formato, con qué
reglas de negocio, y cómo manejar los casos en que algo falla o tarda.

## Documentos de este reglamento

1. **`01_reglamento_general_v2.md`** — Reglas de negocio: cómo se
   identifican las cuentas, moneda, límites de importe, estados de
   cuenta, los tipos de operación que existen, y los principios
   obligatorios (idempotencia, trazabilidad, etc.) que aplican a todo el
   sistema.
2. **`02_contrato_interoperabilidad_v2.md`** — El contrato técnico: cómo
   se autentican los bancos entre sí, cómo funciona una transferencia
   dentro del mismo banco, cómo funciona una transferencia entre bancos
   distintos (paso a paso), qué endpoints debe exponer cada banco, y el
   catálogo de errores común.
3. **`03_especificacion_atm_v2.md`** — Cómo debe funcionar el cajero
   simulado: el flujo de retiro, cómo se comunica con el banco emisor de
   la cuenta, e idempotencia aplicada a retiros.
4. **`04_plan_pruebas_v2.md`** — Los casos de prueba mínimos que cada
   banco debe poder demostrar antes de la integración final con los
   otros 3 grupos.
5. **`05_openapi_v2.yaml`** — La especificación técnica formal (OpenAPI)
   de los endpoints que define el documento 02. Se puede importar
   directamente en Swagger, Postman o Insomnia para generar la base de
   la API sin transcribir el contrato a mano.
6. **`06_registro_bancos_v2.json`** — El registro con el `banco_id`, URL
   base y clave pública de cada uno de los 4 bancos. Lo distribuye y
   mantiene actualizado el BCP simulado (ver documento 02, sección 1).

## Cómo usar estos documentos

- Si sos parte de un grupo y vas a implementar un banco, **empezá por el
  01 y el 02**: ahí están las reglas de negocio y el contrato técnico
  completo. El 03 solo hace falta si tu grupo va a construir la parte de
  cajero.
- Usá el **05 (OpenAPI)** como referencia técnica directa mientras
  programás: ahí están los endpoints, los campos exactos y los tipos de
  dato, sin tener que releer la prosa del documento 02 cada vez.
- Antes de la integración con los otros 3 bancos, revisá el **04** y
  verificá que tu implementación pasa esos casos.
- Si algo no está claro o genera dudas de interpretación, consultalo con
  el líder de proyecto (BCP simulado) antes de asumir una solución por
  tu cuenta — la idea del reglamento es evitar justamente que cada grupo
  interprete algo distinto.

## Qué es obligatorio y qué es opcional

Todo lo que está en estos 5 documentos es el **mínimo obligatorio** para
que los 4 bancos sean interoperables entre sí. Un grupo puede sumar
funcionalidad extra sobre su propia implementación (comisiones, tipos de
cuenta adicionales, mejoras de seguridad, manejo más sofisticado de
fallos, etc.), siempre que:

- Sin romper el contrato común (documento 02): los otros 3 bancos tienen
  que poder seguir comunicándose con el tuyo sin cambios de su lado.
  Lo extra se puede sumar, pero no se puede sacar ni cambiar el
  comportamiento de lo que ya es obligatorio.
- No sea un requisito para que el proyecto grupal funcione: lo mínimo
  exigido es justamente lo que documentan estos 5 archivos.
