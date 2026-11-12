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
