# Prompts para desarrollar el Sistema de Gestión Comercial

> [!IMPORTANT]
> **Reglas de uso (leelas antes de empezar)**
> 1. Usá **un prompt por vez**, en orden. No pases al siguiente hasta que el anterior funcione y lo entiendas.
> 2. Después de cada prompt: **probalo vos**, leé el diff y hacé **un commit** (idealmente desde la cuenta de GitHub de quien trabajó). Eso te da el historial de commits que piden.
> 3. Cuando el agente se equivoque, anotalo en `docs/BITACORA-IA.md` (qué hizo mal y cómo lo corregiste). Te lo van a preguntar en la defensa.
> 4. Si un prompt genera demasiado código de golpe, cortalo en partes: "hacé solo el servicio", "ahora solo la ruta", "ahora solo la pantalla".

---

## Prompt 0 — Contexto permanente para el agente

**Antes de este prompt:** creá la carpeta `docs/` y pegá el enunciado completo (secciones 1 a 12) en `docs/ENUNCIADO.md`.

```text
Vamos a construir el "2º parcial - Sistema de gestión comercial". El enunciado completo está en docs/ENUNCIADO.md: leelo entero antes de hacer nada.

Creá en la raíz un archivo AGENTS.md (y una copia CLAUDE.md si tu herramienta lo usa) con las reglas que vas a respetar durante TODO el proyecto. Contenido:

# Reglas del proyecto
## Stack (obligatorio, no reemplazar)
- TypeScript estricto, Next.js (última estable) con App Router, Route Handlers para la API REST bajo /api.
- Tailwind CSS + shadcn/ui. Gráficos con Chart.js + react-chartjs-2.
- Drizzle ORM + drizzle-kit (migraciones). Cliente @libsql/client: SQLite local (file:local.db) en desarrollo y Turso en producción, elegido SOLO por variables de entorno.
- Zod para validación, con esquemas compartidos cliente/servidor.
- Sesión propia en cookie httpOnly, SameSite=Lax, firmada con `jose` (HS256) usando SESSION_SECRET. Contraseñas con bcryptjs.
- pnpm como gestor de paquetes.

## Arquitectura en capas (sección 10 del enunciado)
- src/db: esquema, conexión y migraciones. Sin reglas de negocio.
- src/services: UNA función por operación. Orden fijo: (1) validar con Zod, (2) verificar rol, (3) aplicar reglas, (4) devolver la forma pública de la API. No conocen HTTP, cookies ni React.
- src/app/api/**/route.ts: adaptadores finos. Usan el wrapper común `apiHandler`; solo llaman a un servicio y responden.
- src/actions: Server Actions con wrapper común que devuelve { ok, data } | { ok: false, error }. Llaman a LOS MISMOS servicios.
- src/lib/schemas: esquemas Zod compartidos. Los de edición usan z.strictObject (rechazan campos no previstos).
- src/lib/errors.ts: única clase ApiError(status, code, message, details). Los códigos de error son los de la sección 11.8.
- src/lib/format.ts: ÚNICO lugar donde se formatea moneda ($ 12.345,50), fecha dd/mm/aaaa, hora de Argentina (America/Argentina/Buenos_Aires), documentos y numeración 0001-00000001.
- src/lib/constants.ts: ÚNICO lugar para punto de venta, roles, formas de pago, medios de pago, marcas de tarjeta, motivos de stock, límites (pageSize, etc.).
- Toda operación que toca más de una tabla va en db.transaction().
- La existencia de un producto SOLO cambia dentro de moverStock() (UPDATE condicional `WHERE existencia >= ?` + insert del movimiento en la misma transacción).
- El saldo del cliente se deriva SIEMPRE de movimientos_cuenta (débitos − créditos).
- Importes en centavos como enteros. Nunca float.
- Cada función de servicio lleva un comentario JSDoc citando el requerimiento o regla (ej: "RF-V6, reglas 1 y 2").
- PROHIBIDO usar la instancia global `db` dentro de funciones auxiliares que participen en una transacción (`moverStock`, `calcularSaldo`, `registrarMovimientoCuenta`, `siguienteNumero`). Deben recibir y ejecutar todo sobre el parámetro `tx` para no bloquear SQLite (SQLITE_BUSY).
- En Next.js 15+, `cookies()`, `headers()`, `params` y `searchParams` son Promesas: SIEMPRE usá `await` al leerlos en Server Components, Route Handlers y funciones de sesión.

## Interfaz
- Páginas de lectura = Server Components que llaman a servicios. Filtros, búsqueda y paginado en la URL (searchParams).
- Client Components chicos. Las páginas no llevan etiquetas HTML sueltas: usan shadcn/ui, primitivas propias (Stack, Inline, Grid, Text, Heading) y componentes de dominio.
- Componentes reutilizables: PageHeader, LoadingState, EmptyState, ErrorState, ConfirmDialog, FormField, MoneyInput, StatusBadge, Pagination.

## Forma de trabajo
- No avances a otro módulo sin que te lo pida. Al terminar cada tarea: listá archivos creados/modificados, explicá en 5-10 líneas qué hiciste y cómo probarlo, y proponé un mensaje de commit (Conventional Commits, en español).
- Código y nombres de dominio en español (como el enunciado). Sin comentarios obvios; sí comentarios que expliquen el porqué.
- Nunca pongas secretos en el repo. Usá .env (ignorado) y .env.example.

No escribas código todavía. Cuando termines, resumime en una lista los 4 hitos y qué vas a construir en cada uno.
```

---

## HITO 1 — Base, usuarios y productos

### Prompt 1 — Proyecto base

```text
Creá el proyecto Next.js en esta carpeta (TypeScript, App Router, Tailwind, ESLint, carpeta src/, alias @/*) usando pnpm. Inicializá shadcn/ui y agregá los componentes: button, input, label, card, table, dialog, alert-dialog, select, badge, dropdown-menu, sheet, sonner, tabs, skeleton, separator, form, popover, command, textarea, switch, avatar, tooltip.

Instalá: drizzle-orm @libsql/client zod jose bcryptjs chart.js react-chartjs-2, y como dev: drizzle-kit tsx @types/bcryptjs.

Configurá los scripts de package.json para que funcione EXACTAMENTE esto, sin pasos manuales:
pnpm install / cp .env.example .env / pnpm run db:migrate / pnpm run db:seed / pnpm run dev
Scripts: dev, build, start, lint, db:generate (drizzle-kit generate), db:migrate (drizzle-kit migrate), db:seed (tsx src/db/seed.ts), db:studio.

Creá .env.example con DATABASE_URL=file:local.db, DATABASE_AUTH_TOKEN= (vacío), SESSION_SECRET= con un valor de ejemplo claramente marcado como "cambiar". Verificá que .gitignore excluya .env, local.db*.
Inicializá git si no está. No construyas pantallas todavía.
```

### Prompt 2 — Esquema de base de datos

```text
Implementá src/db/schema.ts con Drizzle (sqlite-core) según la sección 7 del enunciado: usuarios, categorias, productos, movimientos_stock, clientes, facturas, factura_items, pagos, movimientos_cuenta. Agregá también una tabla `secuencias` (clave texto PK: 'factura' | 'recibo', punto_venta, ultimo_numero) para la numeración correlativa sin huecos (regla 4).

Requisitos:
- Todas las tablas con id entero autoincremental y created_at (timestamp en UTC).
- Importes en centavos como integer. Nunca real.
- Únicos: usuarios.email, categorias.nombre, productos.codigo, clientes (tipo_doc, nro_doc normalizado sin puntos ni guiones), facturas (punto_venta, numero), pagos (punto_venta, numero_recibo).
- productos.existencia es un campo guardado pero solo lo modifica moverStock(); agregá un CHECK (existencia >= 0).
- facturas: cliente_id nullable (null = consumidor final), forma_pago, tarjeta_tipo, tarjeta_marca, tarjeta_ultimos4 (NUNCA el número completo), monto_recibido, estado ('emitida'|'anulada'), motivo_anulacion, anulada_por, anulada_en.
- movimientos_stock: tipo ('venta'|'anulacion'|'ingreso'|'egreso'), cantidad con signo, existencia_resultante, motivo, observacion, producto_id, usuario_id, factura_id nullable.
- movimientos_cuenta: tipo ('debito'|'credito'), importe positivo, descripcion, cliente_id, factura_id o pago_id.
- Índices para las consultas frecuentes (facturas por fecha, movimientos por producto, movimientos_cuenta por cliente).
- Definí las relations de Drizzle.

Creá src/db/index.ts: un único cliente libsql que lee DATABASE_URL y DATABASE_AUTH_TOKEN (opcional). Creá drizzle.config.ts con dialect "turso" leyendo las mismas variables (así sirve para local y para Turso).
Generá la migración inicial con db:generate y aplicala con db:migrate. Explicame cómo sabe la app si usa la base local o Turso.
```

### Prompt 3 — Infraestructura común (errores, sesión, wrappers, formatos)

```text
Construí la infraestructura transversal (sección 10.3, 10.4, 10.5 y 11.1), sin pantallas:

1. src/lib/errors.ts: clase ApiError(status, code, message, details?) y helpers por cada código de la 11.8 (ej: errores.noEncontrado(), errores.sinPermiso()...). Función zodADetalles(error) que convierte un ZodError en [{ campo, mensaje }] con mensajes en español.
2. src/lib/constants.ts: PUNTO_VENTA = 1, ROLES, FORMAS_PAGO, TIPOS_TARJETA, MARCAS_TARJETA, MEDIOS_PAGO_CUENTA (efectivo, transferencia, tarjeta_debito), MOTIVOS_INGRESO, MOTIVOS_EGRESO, PAGE_SIZE_DEFAULT=20, PAGE_SIZE_MAX=100, TZ='America/Argentina/Buenos_Aires'.
3. src/lib/format.ts: formatearMoneda(centavos) -> "$ 12.345,50", formatearFecha, formatearFechaHora (hora Argentina), formatearNumeroComprobante(pv, n) -> "0001-00000001", formatearRecibo -> "R 0001-00000001", formatearDocumento, normalizarDocumento. Y parsearPesosACentavos para el campo de importe. Sin floats en la conversión (trabajá con strings/enteros).
4. src/lib/fechas.ts: rangoDiaArgentina('AAAA-MM-DD') y rangoMesActual() devolviendo límites en UTC.
5. src/lib/paginacion.ts: esquema Zod de page/pageSize y helper que arma { data, page, pageSize, total }.
6. src/lib/auth/session.ts: crearSesion(userId) firma un JWT con jose (exp 8 h) y lo guarda en cookie httpOnly (usando `await cookies()` de next/headers), SameSite=Lax, secure en producción; borrarSesion(); leerSesion(). obtenerUsuarioActual() verifica la firma y BUSCA EL USUARIO EN LA BASE en cada pedido: si no existe o está inactivo devuelve null (regla 10.5).
7. src/lib/auth/permisos.ts: exigirRol(usuario, ...roles) que lanza ApiError 403 SIN_PERMISO.
8. src/lib/api/handler.ts: apiHandler(fn, { publico?: boolean }) para Route Handlers: exige sesión (401 NO_AUTENTICADO), resuelve params (son Promise en Next reciente), parsea JSON (400 JSON_INVALIDO), lee searchParams, llama a fn y traduce ApiError al formato { error: { code, message, details } }. Errores inesperados: console.error en servidor y 500 ERROR_INTERNO sin detalle.
9. src/lib/actions/action.ts: wrapper equivalente para Server Actions que devuelve { ok: true, data } | { ok: false, error }.
10. Verificá en package.json qué versión exacta de Next se instaló: si es Next 15 usá `src/middleware.ts` (exportando `middleware`); si es Next 16 usá `src/proxy.ts` (exportando `proxy`). Si no hay cookie de sesión válida (firma verificada con jose), redirige a /login; deja pasar /login, /api/auth/login y estáticos. Las rutas /api sin sesión deben responder 401 JSON ({ error: { code: "NO_AUTENTICADO", message: "..." } }), nunca redirigir.

Mostrame un ejemplo de cómo quedaría una ruta de 3-5 líneas usando apiHandler.
```

### Prompt 4 — Autenticación y usuarios

```text
Implementá autenticación y usuarios (RF-U1 a RF-U5, sección 11.3):

- src/lib/schemas/auth.ts y usuarios.ts (Zod, compartidos).
- src/services/auth.ts: login (401 CREDENCIALES_INVALIDAS con el MISMO mensaje si no existe, contraseña mal o inactivo), cambiarPassword (422 CONTRASENA_ACTUAL_INCORRECTA, mínimo 8).
- src/services/usuarios.ts: listar (q, activo, paginado, nunca devolver el hash), crear (409 EMAIL_DUPLICADO), actualizar (422 OPERACION_SOBRE_SI_MISMO si un admin se desactiva o se quita el rol admin), resetearPassword (genera temporal aleatoria, la devuelve una sola vez). Todo solo admin.
- Rutas: /api/auth/login, logout, me, password; /api/usuarios, /api/usuarios/[id], /api/usuarios/[id]/reset-password. Cada una de pocas líneas con apiHandler.
- Pantalla /login (fuera del layout principal), con validación por campo y toast de error.

No hagas todavía el layout ni la pantalla de usuarios. Al final dame comandos curl (o PowerShell Invoke-RestMethod) para probar: login correcto, login incorrecto, /api/auth/me con y sin cookie.
```

### Prompt 5 — Layout, primitivas y componentes compartidos

```text
Construí la estructura visual (secciones 4, 8 y 10.6):

- Route group src/app/(app)/layout.tsx: verifica usuario con obtenerUsuarioActual() (si no hay, redirect /login). Panel lateral fijo con los módulos Inicio, Ventas, Productos, Clientes, Usuarios (este último solo visible para admin), resaltando el módulo activo. Barra superior con nombre y rol del usuario, link a /perfil y botón Salir.
- Responsive desde 360 px: en móvil el panel se colapsa en un Sheet con botón hamburguesa.
- Primitivas en src/components/layout: Stack, Inline, Grid, Text, Heading.
- Compartidos en src/components/shared: PageHeader (título, descripción, acciones), LoadingState (skeletons), EmptyState, ErrorState, ConfirmDialog (para anular/dar de baja/desactivar), FormField (label + control + error), MoneyInput (muestra pesos, maneja centavos), StatusBadge, Pagination (usa searchParams).
- Hook src/hooks/use-form-action.ts: errores por campo (primero Zod en cliente, después los del servidor), estado de envío y toast del resultado.
- Proponé una identidad visual propia y consistente (paleta, tipografía, radios) definida en las variables CSS del tema de shadcn. NO copies las pantallas de referencia.
- Páginas /perfil (cambiar contraseña) y /usuarios (listado con búsqueda, alta/edición en modal, desactivar con confirmación, reset de contraseña mostrando la temporal una sola vez). Usá loading.tsx y error.tsx de Next para los estados.
```

### Prompt 6 — Categorías, productos y stock

```text
Implementá productos y existencias (RF-P1 a RF-P7, sección 11.4):

1. src/services/stock.ts con la función moverStock(tx, { productoId, delta, tipo, motivo, observacion, usuarioId, facturaId? }):
   - IMPORTANTE: Toda consulta adentro de moverStock debe usar `tx` (nunca `db`).
   - Si delta < 0: UPDATE productos SET existencia = existencia + delta WHERE id = ? AND existencia >= abs(delta). Si no modificó filas -> ApiError 409 STOCK_INSUFICIENTE con details [{ productoId, solicitado: abs(delta), disponible }].
   - Si delta > 0: UPDATE sumando.
   - Lee la existencia resultante e inserta el movimiento en la MISMA transacción (`tx`).
   - Comentario: "Regla 1 y 2, sección 10.2. ÚNICO lugar donde cambia la existencia."
   Agregá registrarMovimientoManual (admin, ingreso/egreso con motivos válidos de constants, valida motivo según tipo) que abre `db.transaction()` y llama a moverStock(tx, ...). Y listarMovimientos que devuelva EXACTAMENTE la estructura JSON de la sección 11.4 de docs/ENUNCIADO.md (incluyendo objeto `producto: { id, codigo }`, `usuario: { id, nombre }` y `referencia: { facturaId, numero } | null`).
2. src/services/categorias.ts: listar activas, crear (409 NOMBRE_DUPLICADO), actualizar (409 CATEGORIA_EN_USO si se desactiva con productos activos). Admin para escribir.
3. src/services/productos.ts: listar (q por código/descripción, categoriaId, bajoMinimo, activo con 'todos' solo para admin, paginado, con bajoMinimo calculado), obtener (404), crear (409 CODIGO_DUPLICADO; si viene stockInicial, en la misma transacción llama a moverStock con motivo "Stock inicial"), actualizar (z.strictObject: si llega `existencia` -> 422 VALIDACION; baja lógica con activo:false).
4. Rutas: /api/categorias, /api/categorias/[id], /api/productos, /api/productos/[id], /api/productos/[id]/movimientos (GET y POST), /api/movimientos-stock.
5. Pantallas: /productos (catálogo paginado con filtros en la URL, insignia "Bajo mínimo", botones de edición solo para admin), /productos/[id] (ficha con pestañas Datos / Existencias-Movimientos, modal de ingreso y egreso), /productos/categorias.
Las mutaciones desde la interfaz usan Server Actions que llaman a los mismos servicios.
```

### Prompt 7 — Seed mínimo para el hito 1

```text
Creá src/db/seed.ts (idempotente: limpia las tablas en orden y vuelve a cargar). Por ahora: 1 admin (admin@comercio.test / Admin1234) y 1 vendedor (vendedor@comercio.test / Vendedor1234), 5 categorías y 30 productos realistas para una ferretería. El stock inicial se carga LLAMANDO a productos.crear con stockInicial (sección 10.7: el seed usa los mismos servicios), con un usuario admin como contexto. Dejá algunos productos por debajo del mínimo.
Después verificá: pnpm db:seed, pnpm dev, login como vendedor y probá que POST /api/productos responda 403.
```

> [!TIP]
> **Fin del hito 1.** Antes de seguir, hacé el primer deploy (Prompt 15) para que el hito "proyecto publicado" quede cumplido desde el principio.

---

## HITO 2 — Clientes y cuenta corriente

### Prompt 8 — Clientes, cuenta corriente y pagos

```text
Implementá clientes y cuenta corriente (RF-C1 a RF-C7, sección 11.5):

1. src/services/cuentas.ts: calcularSaldo(tx|db, clienteId) = SUM(débitos) − SUM(créditos) desde movimientos_cuenta (ÚNICA fuente de verdad del saldo, regla 7). registrarMovimientoCuenta(tx, {...}). obtenerCuenta(clienteId, desde?, hasta?) con saldoAnterior y saldo acumulado por renglón, en orden cronológico.
2. src/services/secuencias.ts: siguienteNumero(tx, tipo: 'factura'|'recibo') que dentro de la transacción `tx` se asegure de que exista la fila inicial (o usá INSERT ... ON CONFLICT DO UPDATE incrementando `ultimo_numero + 1` con RETURNING) para el `PUNTO_VENTA`. Nunca uses `db` adentro de esta función, solo `tx` (regla 4: sin huecos porque si la transacción falla, el incremento también se revierte). Revisá docs/ENUNCIADO.md sección 11.5 para que los JSON de clientes, cuenta y pagos tengan los nombres de campos exactos.
3. src/services/clientes.ts: listar (q por nombre o documento ignorando puntos y guiones, conDeuda, orden saldo|nombre, paginado, con saldo; con conDeuda=true agrega totalAdeudado), obtener (con disponible y las 6 últimas facturas), crear (409 DOCUMENTO_DUPLICADO; validación de CUIT/DNI en el esquema), actualizar (422 SALDO_PENDIENTE si se desactiva o se le quita la cuenta corriente con saldo ≠ 0).
4. src/services/pagos.ts: registrarPago(usuario, clienteId, datos): en UNA transacción verifica cuenta habilitada (422 CUENTA_CORRIENTE_NO_HABILITADA), importe <= saldo (422 PAGO_SUPERA_DEUDA, regla 8), asigna número de recibo, inserta pago y crédito. obtenerPago(id) con la misma forma de la respuesta.
5. Rutas: /api/clientes, /api/clientes/[id], /api/clientes/[id]/cuenta, /api/clientes/[id]/pagos, /api/pagos/[id].
6. Pantallas: /clientes (listado con saldo), /clientes/[id] (ficha con pestañas Datos / Cuenta corriente / Últimas facturas, botón Registrar pago en modal), /clientes/saldos (con deuda, mayor a menor, total adeudado), /clientes/pagos/[id]/recibo (vista imprimible con @media print).
7. Agregá 10 clientes al seed usando clientes.crear (algunos con cuenta corriente y límite).
```

---

## HITO 3 — Ventas

### Prompt 9 — Servicio de facturación (el corazón del parcial)

```text
Implementá src/services/facturas.ts. Es la parte más importante: hacelo con cuidado y comentá cada paso con la regla que cumple.

emitirFactura(usuario, datosSinValidar), siguiendo EXACTAMENTE los 9 pasos de la sección 11.6 dentro de UN db.transaction():
1. Validar con el esquema Zod (src/lib/schemas/facturas.ts): clienteId nullable, items >= 1 sin productos repetidos, cantidades enteras > 0, pago como discriminatedUnion por `forma`. Para tarjeta: z.strictObject con tipo, marca y ultimos4 /^\d{4}\$/; cualquier campo extra (ej. numero) -> 422 (regla 9).
2. Cliente activo; si es cuenta_corriente: cliente registrado y habilitado (422 CUENTA_CORRIENTE_NO_HABILITADA, regla 6).
3-4. Leer productos (activos, si no 422 PRODUCTO_INACTIVO), tomar el precio ACTUAL del catálogo, calcular subtotales y total en enteros. El cliente nunca manda precios.
5. Contado: montoRecibido (default total) >= total, si no 422 MONTO_INSUFICIENTE. Cuenta corriente: saldo actual + total <= límite, si no 422 LIMITE_CREDITO_EXCEDIDO con details { saldoActual, limite, excedente }.
6. siguienteNumero(tx, 'factura').
7. Insertar factura e items copiando descripción y precio unitario (regla 3).
8. Por cada renglón: moverStock(tx, { delta: -cantidad, tipo: 'venta', facturaId }). Si falla cualquiera, toda la transacción se revierte. Para STOCK_INSUFICIENTE, juntá en details todos los productos que fallan (podés hacer una pre-verificación para el mensaje, pero la garantía real es el UPDATE condicional).
9. Si es cuenta corriente: registrarMovimientoCuenta débito "Factura 0001-0000XXXX".
Devolver la forma pública de la 11.6 (con vuelto para contado, tarjeta para tarjeta, saldoCliente para cuenta corriente).

anularFactura(usuario, id, datos): solo admin, motivo obligatorio (422 VALIDACION), 409 FACTURA_ANULADA si ya lo está. En una transacción: marca anulada con usuario/fecha/motivo, moverStock positivo tipo 'anulacion' por renglón y crédito en cuenta si fue cuenta corriente (regla 5).

listarFacturas (q por número o nombre de cliente, desde, hasta en hora Argentina, clienteId, vendedorId, forma, estado, paginado, más nueva primero, totalEmitido), obtenerFactura (con anulacion si corresponde), proximoNumero (informativo).

Rutas: /api/facturas, /api/facturas/proximo-numero, /api/facturas/[id], /api/facturas/[id]/anulacion.

Después escribí un script scripts/prueba-concurrencia.ts que deje un producto con existencia 1 y lance dos emitirFactura en paralelo con Promise.allSettled: debe quedar exactamente una emitida y la otra con STOCK_INSUFICIENTE. Agregá el script "test:concurrencia" a package.json.

IMPORTANTE para evitar SQLITE_BUSY en la prueba de concurrencia y en emisión:
1. Verificá que dentro de `db.transaction(async (tx) => { ... })` en `emitirFactura` y `anularFactura`, TODAS las lecturas (cliente, productos) y llamadas a `calcularSaldo(tx, ...)`, `siguienteNumero(tx, ...)`, `moverStock(tx, ...)` y `registrarMovimientoCuenta(tx, ...)` pasen y usen `tx` y jamás `db`.
2. En `src/db/index.ts`, asegurate de configurar `PRAGMA busy_timeout = 5000;` (o equivalente en el cliente local) para que en SQLite local las dos ventas simultáneas de `prueba-concurrencia.ts` esperen el lock en vez de fallar con SQLITE_BUSY.
```

### Prompt 10 — Pantalla Nueva venta

```text
Construí /ventas/nueva (RF-V1 a RF-V5, sección 8 "Venta ágil"). Client Component dividido en componentes de dominio chicos (SelectorComprador, BuscadorProductos, TablaItems, PanelPago, ResumenTotal) y un hook useVenta con useReducer para el estado.

- Comprador: por defecto "Consumidor final". Buscador de clientes por nombre o documento (consume GET /api/clientes con debounce). Botón "Nuevo cliente" abre un modal que hace POST /api/clientes y lo selecciona SIN perder la venta.
- Ítems: buscador por código o descripción (GET /api/productos, solo activos). Navegación con flechas ↑↓ y Enter para agregar; foco vuelve al buscador. Si el producto ya está, suma cantidad. Cada renglón: descripción, cantidad editable, precio unitario, subtotal, quitar. Mostrar existencia disponible y no permitir superarla (RF-V4).
- Pago: contado (monto recibido con MoneyInput y vuelto en vivo), tarjeta (tipo, marca, últimos 4 dígitos; jamás pedir el número completo), cuenta corriente (deshabilitada si es consumidor final o cliente no habilitado; mostrar saldo, límite y disponible).
- Total recalculado en cada cambio (solo para mostrar; el real lo calcula el servidor).
- Atajos: F2 foco en buscador de productos, F4 ir al pago, Ctrl+Enter confirmar. Mostralos en un texto de ayuda.
- Mostrar el próximo número (GET /api/facturas/proximo-numero) como informativo.
- Confirmar: POST /api/facturas. Si hay error, mostrar el mensaje preciso según error.code (STOCK_INSUFICIENTE marcando los renglones con details). Si sale bien: toast y redirect al comprobante.
Validá en el cliente con el MISMO esquema Zod que usa el servidor.
```

### Prompt 11 — Listado, detalle, comprobante y anulación

```text
Construí:
- /ventas: listado (Server Component) con filtros en la URL: rango de fechas, cliente, forma de pago, vendedor, estado y búsqueda por número; paginado; total emitido del filtro; insignia Emitida/Anulada.
- /ventas/[id]: detalle de la factura. Botón "Anular" solo para admin, con ConfirmDialog que exige motivo (Server Action -> facturas.anularFactura). Si está anulada, mostrar fecha, usuario y motivo.
- /ventas/[id]/comprobante: vista imprimible (RF-V7) con datos del comercio (en constants), número, fecha y hora Argentina, vendedor, comprador, detalle, total y forma de pago. Botón Imprimir y estilos @media print que ocultan el layout. Leyenda "Comprobante interno sin validez fiscal".
```

---

## HITO 4 — Cierre

### Prompt 12 — Tablero

```text
Implementá src/services/tablero.ts y GET /api/tablero exactamente con la forma de la sección 11.7. Solo facturas emitidas; día y mes en hora Argentina. ventasDiarias siempre con 30 elementos (rellenar con 0). masVendidos: top 5 del mes por unidades.
Pantalla / (inicio): 4 tarjetas de indicadores (RF-T1), gráfico de barras de ventas diarias y gráfico de dona por forma de pago con Chart.js + react-chartjs-2 (registrá solo los elementos que usás), tabla de más vendidos y link a productos bajo mínimo (/productos?bajoMinimo=true). Montos formateados con format.ts, también en los tooltips del gráfico. Estados de carga y error.
```

### Prompt 13 — Seed completo

```text
Completá el seed: además de lo existente, generá 50 facturas repartidas en los últimos 30 días y algunos pagos, usando facturas.emitirFactura y pagos.registrarPago (sección 10.7). Mezclá las tres formas de pago, ambos vendedores, consumidor final y clientes. Anulá 2 o 3 facturas con facturas.anularFactura. Para fechar en el pasado, agregá a los servicios un parámetro opcional interno `ahora` (solo usado por el seed, nunca expuesto por la API).
Usá un generador pseudoaleatorio con semilla fija para que el resultado sea reproducible. Asegurate de que el stock alcance (cargá ingresos de reposición si hace falta).
Al final, que el seed ejecute la verificación del Prompt 14 e imprima OK.
```

### Prompt 14 — Verificación de consistencia y auditoría

```text
1. Creá scripts/verificar-consistencia.ts ("pnpm verificar"): para cada producto, existencia == SUM(movimientos_stock.cantidad); para cada cliente, saldo calculado == débitos − créditos y nunca negativo; numeración de facturas y recibos correlativa sin huecos. Imprime las diferencias o "OK".
2. Hacé una auditoría del proyecto contra la lista de control 10.8 y las reglas de AGENTS.md y presentame un informe (NO corrijas todavía):
   - ¿Alguna ruta o Server Action contiene lógica de negocio o consultas directas a la base?
   - ¿Algún lugar modifica productos.existencia fuera de moverStock()?
   - ¿Algún componente formatea moneda/fecha por su cuenta?
   - ¿Alguna operación multi-tabla está fuera de una transacción?
   - ¿Todas las rutas admin verifican rol en el SERVICIO?
   - ¿Alguna página usa etiquetas HTML sueltas o estilos fuera de Tailwind?
   - ¿Hay any, código muerto o duplicado?
   - ¿Los 31 endpoints de la 11.2 existen con el método correcto?
3. Generá scripts/probar-permisos.ps1 que haga login como vendedor y llame a 5 rutas de admin mostrando que todas devuelven 403.
```

### Prompt 15 — Publicación en Vercel + Turso

```text
Guiame paso a paso (yo ejecuto los comandos) para publicar:
1. Crear la base en Turso (CLI o dashboard), obtener URL libsql:// y token.
2. Aplicar las migraciones a Turso con drizzle-kit migrate apuntando a esas variables, y correr el seed contra Turso.
3. Subir el repo a GitHub, importar en Vercel y cargar DATABASE_URL, DATABASE_AUTH_TOKEN y SESSION_SECRET (generado con openssl rand -base64 32 o equivalente en PowerShell) en Project Settings > Environment Variables.
4. Verificar que el sitio publicado usa Turso: crear un producto en producción y verlo en el dashboard/shell de Turso.
Revisá antes que no haya nada que dependa del sistema de archivos local en producción y que el build pase con pnpm build.
```

### Prompt 16 — README final

```text
Escribí README.md: descripción, integrantes (dejá placeholders para nombre y legajo), URL pública, stack y justificación de lo "sugerido" que elegimos (sesión propia con jose en vez de Auth.js, etc.), pasos de instalación exactos de la sección 9, variables de entorno, usuarios de prueba por rol, estructura de carpetas explicada por capas, tabla de requerimientos (RF-*) -> archivo donde se implementa, y cómo correr pnpm verificar y pnpm test:concurrencia.
```

---

## Prompts para preparar la defensa

### Prompt D1 — Mapa de la defensa

```text
Generá docs/DEFENSA.md: para cada una de estas preguntas, respondé en 2-4 líneas y poné el archivo y la función exacta (con número de línea) donde está la respuesta en NUESTRO código:
[pegá acá todas las "Ejemplos de preguntas" del documento de la defensa]
No inventes: si algo no está implementado, decilo.
```

### Prompt D2 — Explicación guiada (usalo muchas veces)

```text
Explicame src/services/facturas.ts como si yo tuviera que defenderlo oralmente mañana. Recorré la función emitirFactura línea por línea, decime qué pasaría si quitara cada bloque importante, y al final hacéme 5 preguntas de examen sin darme las respuestas. Esperá mis respuestas y corregilas.
```

### Prompt D3 — Simulacro de cambio en vivo

```text
Voy a practicar un cambio en vivo. NO lo hagas vos. Decime solamente qué archivos tendría que tocar y en qué orden para: "[cambio]". Cuando yo termine, revisá mi diff y decime si me faltó algo (esquema, migración, servicio, UI, seed, validación en ambos lados).
```

