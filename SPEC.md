# Control de inventario V1

## Producto

- Aplicación web responsive en español, con interfaz en Tailwind y shadcn/ui.
- La usa una pequeña empresa, con dos roles:
  - **Administrador:** gestiona usuarios, productos, mínimos, ajustes y costos.
  - **Operador:** registra entradas y salidas y consulta el stock. No ve el costo.
- Hay 2 almacenes fijos: **San Carlos** y **Game Palace**.
  - Se crean en el seed.
  - El administrador puede renombrarlos, pero no crear ni borrar.
- El catálogo de productos es único y compartido; el stock se lleva por separado en cada almacén.
  - Campos del producto: SKU, nombre, categoría, unidad de medida y costo unitario opcional.
  - El SKU es único, se ingresa a mano, se normaliza a mayúsculas y no se puede cambiar una vez que hay movimientos.
  - Los productos y los usuarios se desactivan, nunca se borran. Un producto inactivo no admite movimientos nuevos pero conserva su historial.
- Las cantidades son decimales, `numeric(14,3)`.

## Reglas de negocio

- Tres tipos de movimiento: entrada, salida y ajuste.
  - El stock inicial se carga como entrada con la nota "stock inicial".
  - Cada movimiento puede llevar una nota o referencia opcional.
- Los movimientos no se editan ni se borran, y cada uno guarda usuario y fecha.
- Un ajuste lo hace solo el administrador, con cantidad con signo (+/-) y motivo obligatorio.
- Nunca se permite stock negativo, ni en salidas ni en ajustes. El sistema rechaza la operación con un mensaje claro.
- No hay transferencias entre almacenes. Se registran una salida y una entrada a mano.
- El stock mínimo se define por producto y almacén.
  - La alerta salta cuando el stock es menor o igual al mínimo.
  - Si el mínimo es 0 o no está definido, no hay alerta.
- Las fechas se guardan en UTC y se muestran en America/Lima.

## Pantallas y reportes

- Panel de inicio: productos bajo el mínimo, por almacén.
- Formulario de movimiento corto: recuerda el último almacén y busca el producto por SKU o nombre. Sin escáner de código de barras en la V1.
- Dos vistas con filtros: stock actual e historial de movimientos (fecha, producto, almacén, tipo). Ambas se exportan a CSV, que sirve de respaldo manual.

## Arquitectura

- Next.js desplegado en Vercel (plan gratuito).
- Postgres en Supabase (plan gratuito), usado solo como base de datos.
  - `DATABASE_URL` apunta al pooler con `pgbouncer=true`.
  - `DIRECT_URL` es la conexión directa, para `prisma migrate`.
- ORM: Prisma.
- Autenticación: Better Auth, con nombre de usuario y contraseña.
  - Sin registro público y sin recuperación por correo.
  - El administrador crea usuarios y restablece contraseñas.
  - El primer administrador sale de un script de seed, con credenciales en variables de entorno.
- El saldo se guarda por producto y almacén y se actualiza en la misma transacción que inserta el movimiento.
  - Protección contra stock negativo: `UPDATE` condicional dentro de `prisma.$transaction`.
  - Segunda barrera: restricción `CHECK (cantidad >= 0)` en la base de datos.
- Pruebas automatizadas solo de la lógica de movimientos: salida sin stock suficiente, ajuste y concurrencia sobre el mismo producto. Las pantallas se prueban a mano.

## Criterios de aceptación de la V1

1. El administrador crea usuarios y productos, y define el mínimo por almacén.
2. El operador registra entradas y salidas, y una salida mayor al stock se rechaza con un mensaje claro.
3. El administrador registra ajustes con motivo, y no puede dejar stock negativo.
4. Dos salidas simultáneas sobre el mismo producto nunca dejan el stock negativo.
5. El panel de inicio muestra los productos con stock menor o igual al mínimo, por almacén.
6. El stock actual y el historial se filtran y se exportan a CSV.
7. Ningún movimiento se edita ni se borra, y todo tiene usuario y fecha en hora de Lima.
8. Funciona en celular, con login y sin registro público, desplegado en Vercel.

## Fuera de la V1

Transferencias entre almacenes, lotes y vencimientos, proveedores y órdenes de compra, valorización del inventario, importación masiva de Excel/CSV, escáner de códigos, alertas por correo o WhatsApp, recuperación de contraseña por correo y respaldos automáticos.

## Orden de construcción

1. Proyecto, base de datos y esquema Prisma.
2. Login, roles y seed.
3. Productos y almacenes.
4. Movimientos, con sus pruebas.
5. Panel de alertas.
6. Stock, historial y CSV.
7. Despliegue.

## Riesgo conocido

El plan gratuito de Supabase pausa los proyectos que pasan una semana sin uso. Para una empresa que usa el sistema a diario no debería ser un problema.
