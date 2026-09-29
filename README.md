# Comedor típico: sistema de pedidos

Aplicación web de una sola página (`index.html`) para tomar y dar seguimiento a pedidos de un comedor de comida típica guatemalteca. Usa [Supabase](https://supabase.com) como base de datos, con sincronización en tiempo real y usuarios para el personal.

## Qué hace

- **Ordenar:** cualquier cliente ve el menú por categorías, arma su pedido y lo envía, para comer en mesa o para llevar. No necesita cuenta.
- **Pedidos enviados, Entregados y Menú:** vistas solo para el personal. Requieren iniciar sesión.
- Los pedidos y el menú se actualizan en tiempo real entre dispositivos.
- Categorías del menú: `Plato del día`, `Antojitos`, `Bebidas` y `Postres`.

## Estructura del proyecto

| Archivo | Para qué sirve |
|---|---|
| `index.html` | La aplicación completa (HTML, CSS y JavaScript en un solo archivo). |
| `auth_setup.sql` | Activa la seguridad por filas (RLS) y define quién puede leer y escribir. |
| `orders_import.sql` | Carga el historial de pedidos #041 a #106 en la tabla `orders`. Opcional. |
| `README.md` | Este documento. |

## Puesta en marcha

### 1. Crear el proyecto en Supabase

Creá un proyecto en [supabase.com](https://supabase.com) y anotá:

- **Project URL:** `https://TU-PROYECTO.supabase.co`
- **Anon public key:** en *Project Settings → API*.

### 2. Crear las tablas

**`menu_items`**

| Columna | Tipo sugerido |
|---|---|
| `id` | identity (autoincremental) |
| `cat` | text |
| `icon` | text |
| `name` | text |
| `description` | text |
| `price` | numeric |
| `available` | boolean |
| `enabled` | boolean |

**`orders`**

| Columna | Tipo sugerido | Notas |
|---|---|---|
| `id` | identity | Es el número de pedido. |
| `customer_name` | text | |
| `order_type` | text | `'mesa'` o `'llevar'`. |
| `table_number` | text | Solo el número, por ejemplo `'5'`. Vacío si es para llevar. |
| `notes` | text | |
| `items` | jsonb | Lista de platillos con `name`, `price`, `qty`, etc. |
| `total` | numeric | |
| `status` | text | Estado del pedido; `'entregado'` lo pasa al historial. |
| `paid` | boolean | |
| `ever_delivered` | boolean | |
| `current_batch` | jsonb | |
| `created_at` | timestamptz | Por defecto `now()`. |

Activá **Realtime** para `orders` y `menu_items` en *Database → Replication*.

### 3. Conectar la app

Abrí `index.html` y completá estas dos constantes:

```js
const SUPABASE_URL = "https://TU-PROYECTO.supabase.co";
const SUPABASE_ANON_KEY = "tu-anon-public-key";
```

La app quita sola un `/rest/v1/` sobrante al final de la URL, pero conviene dejar solo la URL base.

### 4. Activar permisos

En el **SQL Editor** de Supabase, ejecutá el contenido de `auth_setup.sql`. Con esto:

- Los clientes (sin sesión) pueden ver el menú y enviar pedidos.
- El personal (con sesión) tiene acceso total a pedidos y menú.

> Sin este paso, el login solo esconde las pestañas: cualquiera con la clave pública podría leer o modificar los datos.

### 5. Crear los usuarios del personal

1. Andá a **Authentication → Users → Add user → Create new user**.
2. Escribí un correo y una contraseña que vos elegís. Supabase no genera contraseñas ni las muestra después.
3. Marcá **Auto Confirm User**, para que no pida confirmar el correo.
4. Recomendado: desactivá *Allow new users to sign up* en *Authentication → Sign In / Providers*, para que nadie se registre solo.

La app no tiene pantalla de registro ni de recuperación de contraseña. Si alguien la olvida, cambiala desde el SQL Editor:

```sql
update auth.users
set encrypted_password = crypt('NuevaClave123', gen_salt('bf'))
where email = 'correo@ejemplo.com';
```

### 6. Usar la app

Abrí `index.html` en el navegador, o subilo a cualquier hosting estático (Netlify, Vercel, GitHub Pages, etc.). Para entrar como personal, usá el botón **🔐 Ingresar**.

## Cargar datos

**Agregar platillos al menú:** desde la pestaña **Menú** (con sesión iniciada), o con un `insert` en el SQL Editor:

```sql
insert into menu_items (cat, icon, name, description, price, available, enabled) values
('Plato del día', '🍲', 'Nombre', 'Descripción.', 38, true, true);
```

**Importar el historial de pedidos:** ejecutá `orders_import.sql`. La columna `id` es *identity*, así que el script usa `overriding system value` para conservar los números de pedido, y al final ajusta el contador para que el siguiente pedido no choque.

## Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| "No se pudo enviar el pedido" | URL de Supabase mal escrita, o falta una policy de `INSERT` | Revisá `SUPABASE_URL` y que `auth_setup.sql` esté ejecutado. Mirá el error real en la consola (F12). |
| "Could not find the 'X' column" | Falta una columna en la tabla | Compará con las tablas de arriba. |
| Login: "Correo o contraseña incorrectos" | Credenciales no coinciden | Restablecé la contraseña con el SQL de arriba. |
| Login: "El correo no está confirmado" | Usuario creado sin *Auto Confirm* | Confirmá el usuario en *Authentication → Users*, o recrealo. |
| Tras ingresar no se ven pedidos | Faltan policies para `authenticated` | Ejecutá `auth_setup.sql`. |
| `cannot insert a non-DEFAULT value into column "id"` | `id` es *identity* | Usá `overriding system value`, como en `orders_import.sql`. |

## Seguridad

- La **anon key** es pública por diseño y va dentro del HTML. Lo que protege los datos son las policies de RLS, así que no las omitas.
- Un cliente sin sesión puede leer pedidos de otros, porque la app necesita leer de vuelta el pedido recién creado para mostrar su número. Para cerrar ese hueco, se puede mover el envío a una función de Supabase que devuelva solo el número.
- Todos los usuarios con cuenta tienen los mismos permisos. Si se necesitan roles (cocina, caja, administrador), hay que agregarlos.
- Nunca pongas la **service_role key** en `index.html`.
