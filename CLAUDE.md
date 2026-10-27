# Idioma

Responde siempre en español en este proyecto.

# Especificación

La única fuente de verdad es `SPEC-2.md`. Ignora `SPEC.md`, que es una versión anterior y desactualizada.

# Resumen del proyecto (según SPEC-2.md)

Control de inventario V1: aplicación web responsive en español para una pequeña empresa, con dos almacenes fijos (San Carlos y Game Palace) y dos roles (administrador y operador).

## Stack

- Next.js en Vercel (plan Hobby), Tailwind y shadcn/ui.
- Postgres en Supabase (plan gratuito), solo como base de datos.
  - `DATABASE_URL`: pooler con `pgbouncer=true`.
  - `DIRECT_URL`: conexión directa, para `prisma migrate`.
- Prisma como ORM.
- Better Auth con nombre de usuario y contraseña. Sin registro público ni recuperación por correo. Si el plugin de username exige correo, generar uno ficticio por usuario.

## Reglas que no se deben romper

- Los permisos se aplican en el servidor (API y acciones), no solo en la interfaz. El costo nunca se envía al operador: ni en pantallas, ni en respuestas de API, ni en CSV.
- Nunca hay stock negativo. Salidas y ajustes negativos usan `UPDATE` condicional dentro de `prisma.$transaction`; las entradas usan `upsert` del saldo. Segunda barrera: `CHECK (cantidad >= 0)` como SQL manual en la migración (Prisma no lo declara).
- El saldo se actualiza en la misma transacción que inserta el movimiento.
- Los movimientos no se editan ni se borran. Los errores se corrigen con un ajuste (solo administrador, cantidad con signo distinta de 0 y motivo obligatorio). No hay movimiento de reversa en la V1.
- Entradas y salidas: cantidad mayor que 0. Cantidades en `numeric(14,3)`.
- Productos y usuarios se desactivan, nunca se borran. Al desactivar un usuario se cierran sus sesiones activas de inmediato.
- SKU único, a mano, en mayúsculas e inmutable una vez que hay movimientos.
- Los almacenes salen del seed: se pueden renombrar, no crear ni borrar. No hay transferencias.
- Alerta de stock mínimo (por producto y almacén): stock menor o igual al mínimo. Mínimo 0 o sin definir no alerta; sin saldo cuenta como 0; los inactivos no alertan.
- Fechas en UTC en la base de datos, mostradas en America/Lima. Los filtros por fecha usan los límites del día en America/Lima.
- El historial exige rango de fechas. Los CSV son de consulta, no de respaldo.

## Pruebas

Solo automatizadas para la lógica de movimientos: salida sin stock, ajuste, entrada sobre saldo inexistente y concurrencia sobre el mismo producto. Corren contra una base real local o un esquema aparte, nunca contra producción. Las pantallas se prueban a mano.

## Orden de construcción

1. Proyecto, base de datos, esquema Prisma y seed (almacenes y primer administrador).
2. Login y roles.
3. Productos y almacenes.
4. Movimientos, con sus pruebas.
5. Panel de alertas.
6. Stock, historial y CSV.
7. Despliegue.

## Fuera de la V1

Transferencias, reversas, lotes y vencimientos, proveedores y órdenes de compra, valorización, importación masiva, escáner, alertas por correo o WhatsApp, recuperación de contraseña por correo y respaldos automáticos.

## Respaldo

Mientras no haya respaldos automáticos, se hace un `pg_dump` manual periódico y se guarda fuera de Supabase.
