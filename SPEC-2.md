# Control de inventario V1

## Producto

- Aplicación web responsive en español, con interfaz en Tailwind y shadcn/ui.
- La usa una pequeña empresa, con dos roles:
  - **Administrador:** gestiona usuarios, productos, mínimos, ajustes y costos. También puede registrar entradas y salidas.
  - **Operador:** registra entradas y salidas y consulta el stock. No ve el costo.
- Hay 2 almacenes fijos: **San Carlos** y **Game Palace**.
  - Se crean en el seed.
  - El administrador puede renombrarlos, pero no crear ni borrar.
- El catálogo de productos es único y compartido; el stock se lleva por separado en cada almacén.
  - Campos del producto: SKU, nombre, categoría, unidad de medida y costo unitario opcional.
  - El costo unitario es solo informativo: la valorización del inventario está fuera de la V1.
  - El SKU es único, se ingresa a mano, se normaliza a mayúsculas y no se puede cambiar una vez que hay movimientos.
  - Los productos y los usuarios se desactivan, nunca se borran. Un producto inactivo no admite movimientos nuevos pero conserva su historial.
  - Al desactivar un usuario se cierran de inmediato sus sesiones activas, no solo se bloquea el siguiente login.
- Las cantidades son decimales, `numeric(14,3)`.

## Reglas de negocio

- Tres tipos de movimiento: entrada, salida y ajuste.
  - El stock inicial se carga como entrada con la nota "stock inicial".
  - Cada movimiento puede llevar una nota o referencia opcional.
  - En entradas y salidas la cantidad debe ser mayor que 0.
  - En ajustes la cantidad con signo no puede ser 0.
- Los movimientos no se editan ni se borran, y cada uno guarda usuario y fecha.
- Un ajuste lo hace solo el administrador, con cantidad con signo (+/-) y motivo obligatorio.
- Si un operador se equivoca en un movimiento, el administrador lo corrige con un ajuste. En la V1 no hay movimiento de reversa.
- Nunca se permite stock negativo, ni en salidas ni en ajustes. El sistema rechaza la operación con un mensaje claro.
- No hay transferencias entre almacenes. Se registran una salida y una entrada a mano.
- El stock mínimo se define por producto y almacén.
  - La alerta salta cuando el stock es menor o igual al mínimo.
  - Si el mínimo es 0 o no está definido, no hay alerta.
  - Un producto con mínimo definido y sin saldo en ese almacén cuenta como stock 0 y sí alerta.
  - Los productos inactivos no generan alertas.
- Las fechas se guardan en UTC y se muestran en America/Lima.
  - Los filtros por fecha usan los límites del día en America/Lima, no en UTC.

## Pantallas y reportes

- Panel de inicio: productos bajo el mínimo, por almacén.
- Formulario de movimiento corto: recuerda el último almacén y busca el producto por SKU o nombre. Sin escáner de código de barras en la V1.
- Dos vistas con filtros: stock actual e historial de movimientos (fecha, producto, almacén, tipo). Ambas se exportan a CSV.
  - El stock actual incluye los productos inactivos que tengan saldo, marcados como inactivos.
  - Los CSV del operador no incluyen el costo.
  - El historial exige un rango de fechas, para no superar los límites de tiempo de las funciones de Vercel. Si el volumen crece, el CSV se genera en streaming.
- El CSV sirve de consulta, no de respaldo completo: no restaura usuarios, productos ni mínimos (ver "Respaldo").

## Arquitectura

- Next.js desplegado en Vercel, plan Hobby (gratuito), por ser de uso interno.
  - Decisión asumida: los términos de Hobby limitan el uso a proyectos personales y no comerciales, y una herramienta interna de una empresa podría considerarse uso comercial. Si Vercel lo exigiera, la salida es pasar a Pro o mover el despliegue a otro hosting; el código no cambia.
- Postgres en Supabase (plan gratuito), usado solo como base de datos.
  - `DATABASE_URL` apunta al pooler con `pgbouncer=true`.
  - `DIRECT_URL` es la conexión directa, para `prisma migrate`.
- ORM: Prisma.
- Autenticación: Better Auth, con nombre de usuario y contraseña.
  - Sin registro público y sin recuperación por correo.
  - El administrador crea usuarios y restablece contraseñas.
  - El primer administrador sale de un script de seed, con credenciales en variables de entorno.
  - El plugin de username de Better Auth puede seguir pidiendo un correo interno. Hay que verificarlo al inicio y, si es así, generar un correo ficticio por usuario.
- Los permisos se aplican en el servidor (API y acciones), no solo en la interfaz. En particular, el costo nunca se envía al operador.
- El saldo se guarda por producto y almacén y se actualiza en la misma transacción que inserta el movimiento.
  - Salidas y ajustes negativos: `UPDATE` condicional dentro de `prisma.$transaction`.
  - Entradas: `upsert` del saldo, porque la fila puede no existir todavía.
  - Segunda barrera: restricción `CHECK (cantidad >= 0)` en la base de datos. Prisma no la declara en el esquema, así que va como SQL manual dentro de la migración.
- Pruebas automatizadas solo de la lógica de movimientos: salida sin stock suficiente, ajuste, entrada sobre saldo inexistente y concurrencia sobre el mismo producto. Las pantallas se prueban a mano.
  - Las pruebas corren contra una base real: Postgres local o un esquema aparte, nunca la base de producción.

## Criterios de aceptación de la V1

1. El administrador crea usuarios y productos, y define el mínimo por almacén.
2. El operador registra entradas y salidas, y una salida mayor al stock se rechaza con un mensaje claro.
3. El administrador registra ajustes con motivo, y no puede dejar stock negativo.
4. Dos salidas simultáneas sobre el mismo producto nunca dejan el stock negativo.
5. El panel de inicio muestra los productos con stock menor o igual al mínimo, por almacén.
6. El stock actual y el historial se filtran y se exportan a CSV.
7. Ningún movimiento se edita ni se borra, y todo tiene usuario y fecha en hora de Lima.
8. Funciona en celular, con login y sin registro público, desplegado en Vercel.
9. El operador no ve el costo en ninguna pantalla, respuesta de API ni CSV.
10. Un usuario desactivado pierde el acceso de inmediato.

## Fuera de la V1

Transferencias entre almacenes, movimientos de reversa, lotes y vencimientos, proveedores y órdenes de compra, valorización del inventario, importación masiva de Excel/CSV, escáner de códigos, alertas por correo o WhatsApp, recuperación de contraseña por correo y respaldos automáticos.

## Orden de construcción

1. Proyecto, base de datos, esquema Prisma y seed (almacenes y primer administrador).
2. Login y roles.
3. Productos y almacenes.
4. Movimientos, con sus pruebas.
5. Panel de alertas.
6. Stock, historial y CSV.
7. Despliegue.

## Respaldo

Supabase gratis no ofrece respaldos útiles y el CSV no restaura el sistema. Mientras no haya respaldos automáticos, se hace un `pg_dump` manual periódico (por ejemplo, semanal) y se guarda fuera de Supabase.

## Riesgos conocidos

- El plan gratuito de Supabase pausa los proyectos que pasan una semana sin uso. Para una empresa que usa el sistema a diario no debería ser un problema.
- Uso de Vercel Hobby en una empresa: riesgo asumido (ver "Arquitectura").
