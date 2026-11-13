# Especificación de la prueba técnica

Fecha: 2018/05/17

## Consigna

La prueba consiste en el desarrollo de dos aplicaciones que expongan servicios REST simples y un proceso Batch con la especificación detallada a continuación.

## Entregable

Se espera entregar:

- Un ZIP con ambos proyectos.
- Un script para levantar las dos aplicaciones.
- Un documento breve explicando la solución implementada.

## Tecnologías para todas las aplicaciones

- Java 8
- Spring 5
- MySQL 5.x
- Maven
- Jetty embebido
- Si es necesario, se pueden utilizar otras tecnologías adicionales.

## Aplicaciones

### 1. Aplicación Merchant

#### Servicio: CHECK

Dado un `merchant_id`, el servicio debe devolver:

- `200 OK` si el merchant existe.
- `404 Not Found` si no existe.

---

### 2. Aplicación PM (Payment Manager)

#### Servicio: SALE

Debe registrar una venta en la base de datos.

El `amount` debe persistirse siempre en USD, utilizando el servicio de fixer.io para obtener la cotización de la moneda.

##### Request

```json
{
  "currency": "String",
  "amount": "Decimal",
  "transaction_id": "String",
  "merchant_id": "String"
}
```

##### Response

```json
{
  "id": "String"
}
```

##### Restricciones

- El servicio debe ser idempotente para la tupla `(merchant_id, transaction_id)`.
- Debe devolver el mismo `id` si se invoca más de una vez con el mismo `merchant_id` y `transaction_id`.
- El `id` debe ser autogenerado y único en la plataforma.
- El servicio de fixer.io no debe ser consultado más de una vez cada 30 segundos.
- Todas las transacciones deben iniciar en estado `PENDING`.
- El `merchant_id` debe validarse contra el servicio `CHECK` de la aplicación Merchant.

#### Servicio: STATUS

Permite consultar el estado de un cobro a partir del `id` devuelto por el servicio `SALE`.

##### Request

```text
GET /status/{id}
```

##### Response

```json
{
  "id": "String",
  "date": "String",
  "transaction_id": "String",
  "merchant_id": "String",
  "amount_usd": "Decimal",
  "status": "String"
}
```

`status` puede tomar los siguientes valores:

- `PAID`
- `PENDING`
- `REJECTED`

> `date` representa la fecha de creación.

#### Servicio: BALANCE

Debe devolver el monto acumulado en USD de las transacciones en estado `PENDING`, `PAID` y `REJECTED`, para un merchant y un rango de fechas determinado.

##### Request

```text
GET /balance?merchant_id={merchant_id}&from={fecha}&to={fecha}
```

##### Response

```json
{
  "total_paid": "Decimal",
  "total_pending": "Decimal",
  "total_rejected": "Decimal"
}
```

---

## Proceso Batch

Debe existir un job que:

- tome todas las transacciones con estado `PENDING`,
- con probabilidad del 70% pase la transacción a `PAID`,
- y en el resto de los casos la marque como `REJECTED`.

Debe ejecutarse cada 30 segundos.

---

## Tests

Se deben implementar tests para ambas aplicaciones.

---

## Nota de implementación

Asumiendo una solución en Java, se recomienda utilizar Spring Framework y Maven para estructurar ambos servicios REST y el proceso Batch, manteniendo una arquitectura simple y clara.
