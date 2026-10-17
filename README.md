# Stock App: Control de depósito

Aplicación web para registrar la **entrada y salida de productos** de un depósito y visualizar el **stock actual en tiempo real**. Proyecto final del curso Full Stack (Grupo IDT), desarrollado con la **Opción 2 de la guía: Laravel + MySQL + Blade**, con autenticación de API mediante **Laravel Sanctum**.

## Funcionalidades

**Gestión de productos**
- Alta, edición y eliminación de productos (nombre, descripción, SKU único, categoría).
- Listado con búsqueda por nombre o SKU y filtro por categoría.

**Control de stock**
- Ingresos con cantidad, fecha y proveedor.
- Salidas con cantidad, fecha y motivo.
- El stock se calcula automáticamente y solo cambia mediante movimientos.
- Una salida mayor al stock disponible es rechazada.
- Actualización del stock **en tiempo real**, sin recargar la página (WebSockets).

**Usuarios y roles**
- Registro, inicio de sesión y cierre de sesión.
- Roles `admin` y `operario`.
- El operario puede consultar productos y registrar movimientos; solo el administrador crea, edita y elimina productos.

**Reportes y dashboard**
- Historial de movimientos con filtros por tipo, producto, rango de fechas y búsqueda.
- Stock disponible por categoría.
- Totales y últimos movimientos.

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| Backend | Laravel (PHP) |
| Base de datos | MySQL |
| Frontend | Blade, Tailwind CSS, Alpine.js |
| Autenticación web | Laravel Breeze (sesión y cookies) |
| Autenticación API | Laravel Sanctum (token Bearer) |
| Tiempo real | Laravel Reverb + Laravel Echo |
| Pruebas | PHPUnit y Postman |
| Seguridad | Protección CSRF, contraseñas con hash, validación en servidor, control de acceso por rol |

## Arquitectura

El proyecto sigue el patrón **MVC** de Laravel.

```
app/
├── Events/StockUpdated.php            # Evento de tiempo real
├── Http/
│   ├── Controllers/
│   │   ├── DashboardController.php    # Dashboard
│   │   ├── ProductController.php      # CRUD de productos (web)
│   │   ├── MovementController.php     # Movimientos e historial (web)
│   │   └── Api/                       # Controladores de la API
│   │       ├── AuthController.php
│   │       ├── ProductController.php
│   │       └── MovementController.php
│   └── Middleware/RoleMiddleware.php  # Restricción por rol
└── Models/                            # User, Category, Product, Movement
resources/views/                       # Vistas Blade (dashboard, products, movements)
routes/
├── web.php                            # Rutas web (sesión)
└── api.php                            # Rutas de la API (Sanctum)
tests/Feature/StockApiTest.php         # Pruebas automáticas
```

### Modelo de datos

- `users`: nombre, email, contraseña (hash), `role` (`admin` / `operario`).
- `categories`: nombre único.
- `products`: nombre, descripción, SKU único, `category_id`, `stock`.
- `movements`: tipo (`entrada` / `salida`), cantidad, `product_id`, `user_id`, proveedor, motivo, fecha.

Relaciones: una categoría tiene muchos productos; un producto tiene muchos movimientos; cada movimiento pertenece a un producto y a un usuario.

### Decisiones técnicas

1. **El stock no se edita a mano.** Solo cambia al registrar un movimiento, de modo que el historial siempre coincide con el stock.
2. **Transacción con bloqueo de fila** (`DB::transaction` + `lockForUpdate`) al registrar un movimiento, para evitar inconsistencias cuando dos usuarios operan sobre el mismo producto a la vez.
3. **El rol nunca lo decide el cliente.** Todo registro público por la API crea un usuario `operario`.
4. **El evento se emite después de confirmar la transacción**, para no notificar cambios que luego se revierten.
5. **Una sola lógica de negocio con dos entradas**: la web (Blade) y la API (Sanctum).
6. **Al eliminar un producto se eliminan también sus movimientos** (`onDelete('cascade')`).

## Requisitos

- PHP 8.2 o superior y Composer
- MySQL o MariaDB
- Node.js y npm
- Extensión `pdo_sqlite` de PHP (para las pruebas)

## Instalación

```bash
# 1. Clonar el repositorio
git clone <URL-DEL-REPOSITORIO>
cd stock-app

# 2. Instalar dependencias
composer install
npm install

# 3. Configurar el entorno
cp .env.example .env
php artisan key:generate
```

Edita el `.env` con los datos de tu base de datos:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=stock_app
DB_USERNAME=root
DB_PASSWORD=
```

Crea la base de datos `stock_app` en MySQL y luego:

```bash
# 4. Crear tablas y datos de prueba
php artisan migrate --seed
```

### Usuarios de prueba

| Rol | Correo | Contraseña |
|---|---|---|
| Administrador | `admin@stock.test` | `password` |
| Operario | `operario@stock.test` | `password` |

> Estas credenciales son solo para desarrollo. Cámbialas o elimina esos usuarios antes de publicar la aplicación.

## Ejecución en desarrollo

El tiempo real requiere **tres procesos**, cada uno en su propia terminal:

```bash
php artisan serve          # Aplicación en http://127.0.0.1:8000
npm run dev                # Compilación de assets (Vite)
php artisan reverb:start   # Servidor WebSocket
```

Si cambias el `.env`, ejecuta `php artisan config:clear` y reinicia `npm run dev`.

### Rutas web principales

| Ruta | Descripción | Acceso |
|---|---|---|
| `/dashboard` | Resumen y stock por categoría | Autenticado |
| `/stock` | Listado de productos y stock en tiempo real | Autenticado |
| `/products/create`, `/products/{id}/edit` | Alta y edición de productos | Solo admin |
| `/movements` | Registro e historial de movimientos | Autenticado |

## API REST (Sanctum)

Prefijo: `/api`. Enviar siempre el header `Accept: application/json`. Las rutas protegidas requieren `Authorization: Bearer <token>`.

| Método | Endpoint | Descripción | Acceso |
|---|---|---|---|
| POST | `/api/register` | Registro (rol `operario`) | Público |
| POST | `/api/login` | Inicio de sesión, devuelve `user` y `token` | Público |
| POST | `/api/logout` | Revoca el token actual | Autenticado |
| GET | `/api/products` | Listado paginado con búsqueda y filtros | Autenticado |
| GET | `/api/stock` | Lista simple (`id`, `name`, `sku`, `stock`) | Autenticado |
| GET | `/api/stock/by-category` | Stock total por categoría | Autenticado |
| GET | `/api/movements` | Historial con filtros (`type`, `product_id`, `from`, `to`) | Autenticado |
| POST | `/api/movements` | Registrar entrada o salida | Autenticado |
| POST | `/api/products` | Crear producto | Solo admin |
| PUT | `/api/products/{id}` | Editar producto | Solo admin |
| DELETE | `/api/products/{id}` | Eliminar producto | Solo admin |

### Ejemplo: login

```http
POST /api/login
Accept: application/json
Content-Type: application/json

{
  "email": "admin@stock.test",
  "password": "password"
}
```

### Ejemplo: registrar una entrada

```http
POST /api/movements
Authorization: Bearer <token>
Accept: application/json
Content-Type: application/json

{
  "product_id": 1,
  "type": "entrada",
  "quantity": 50,
  "supplier": "Proveedor SA",
  "date": "2026-10-16 10:00:00"
}
```

### Códigos de respuesta esperados

| Código | Cuándo ocurre |
|---|---|
| 200 / 201 | Operación correcta |
| 401 | Sin token o token inválido |
| 403 | Token válido, pero sin permiso (por ejemplo, operario creando productos) |
| 422 | Datos inválidos o stock insuficiente |

## Pruebas

### Automáticas (PHPUnit)

```bash
php artisan test
```

Las pruebas usan SQLite en memoria (configurado en `phpunit.xml`), por lo que **no modifican la base MySQL de desarrollo**. Cubren:

- Lista de stock con la estructura esperada.
- Acceso sin token (401).
- Login con devolución de token.
- Operario sin permiso para crear productos (403) y administrador con permiso (201).
- Entrada que aumenta el stock y salida que lo descuenta.
- Salida con stock insuficiente rechazada (422) sin alterar el stock.

### Manuales (Postman)

Se probó la API con Postman: autenticación (login y 401 sin token), permisos por rol (403 del operario y 201 del administrador), CRUD de productos y reglas de negocio (entrada y stock insuficiente).

## Guion de demostración

1. Preparar los datos: `php artisan migrate:fresh --seed`.
2. Iniciar sesión como **admin**.
3. Crear y editar un producto.
4. Abrir `/stock` y `/movements` en dos ventanas lado a lado.
5. Registrar una **entrada** con proveedor: el stock cambia sin recargar.
6. Registrar una **salida** con motivo: el stock baja.
7. Intentar una salida mayor al stock: se rechaza.
8. Mostrar el historial con filtros y el dashboard por categoría.
9. Entrar como **operario**: sin botones de editar y eliminar, y 403 en `/products/create`.
10. Mostrar Postman (login, 401, 403, 201) y `php artisan test` en verde.

## Seguridad

- Protección CSRF en todos los formularios (`@csrf`).
- Contraseñas almacenadas con hash.
- Validación de datos en el servidor.
- Control de acceso por rol mediante middleware.
- Tokens de API revocables (`logout`).
- El archivo `.env` no se versiona; se incluye `.env.example`.

## Despliegue

Según la guía: Laravel Forge, Laravel Cloud o un VPS, con MySQL en producción (por ejemplo, Digital Ocean o AWS). Pasos generales:

```bash
composer install --no-dev --optimize-autoloader
npm ci && npm run build
php artisan migrate --force
php artisan config:cache
```

Configuración mínima del `.env` de producción:

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://tu-dominio.com
```

Además: generar una nueva `APP_KEY`, crear el usuario administrador real (sin usar las credenciales del seeder) y mantener el proceso `php artisan reverb:start` activo en el servidor (con Supervisor o el daemon de Forge).

## Mejoras futuras

- Alertas cuando el stock baje de un mínimo definido.
- Exportación de reportes a PDF y Excel.
- Canales privados en Reverb con autenticación.
- Reportes por usuario, por período y gráficos adicionales.
- Frontend en React que consuma la API de Sanctum.
- Integración con APIs externas (proveedores, facturación).
- Escalabilidad: caché, colas y soporte para varios depósitos.

### Comentarios y sugerencias recibidos en la demostración

_Completar después de la presentación ante el grupo e instructor._

-
-
-

## Autor

- **Nombre:** _completar_
- **Curso:** Full Stack, Grupo IDT
- **Año:** 2026<p align="center"><a href="https://laravel.com" target="_blank"><img src="https://raw.githubusercontent.com/laravel/art/master/logo-lockup/5%20SVG/2%20CMYK/1%20Full%20Color/laravel-logolockup-cmyk-red.svg" width="400" alt="Laravel Logo"></a></p>

<p align="center">
<a href="https://github.com/laravel/framework/actions"><img src="https://github.com/laravel/framework/workflows/tests/badge.svg" alt="Build Status"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/dt/laravel/framework" alt="Total Downloads"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/v/laravel/framework" alt="Latest Stable Version"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/l/laravel/framework" alt="License"></a>
</p>

## About Laravel

Laravel is a web application framework with expressive, elegant syntax. We believe development must be an enjoyable and creative experience to be truly fulfilling. Laravel takes the pain out of development by easing common tasks used in many web projects, such as:

- [Simple, fast routing engine](https://laravel.com/docs/routing).
- [Powerful dependency injection container](https://laravel.com/docs/container).
- Multiple back-ends for [session](https://laravel.com/docs/session) and [cache](https://laravel.com/docs/cache) storage.
- Expressive, intuitive [database ORM](https://laravel.com/docs/eloquent).
- Database agnostic [schema migrations](https://laravel.com/docs/migrations).
- [Robust background job processing](https://laravel.com/docs/queues).
- [Real-time event broadcasting](https://laravel.com/docs/broadcasting).

Laravel is accessible, powerful, and provides tools required for large, robust applications.

## Learning Laravel

Laravel has the most extensive and thorough [documentation](https://laravel.com/docs) and video tutorial library of all modern web application frameworks, making it a breeze to get started with the framework. You can also check out [Laravel Learn](https://laravel.com/learn), where you will be guided through building a modern Laravel application.

If you don't feel like reading, [Laracasts](https://laracasts.com) can help. Laracasts contains thousands of video tutorials on a range of topics including Laravel, modern PHP, unit testing, and JavaScript. Boost your skills by digging into our comprehensive video library.

## Laravel Sponsors

We would like to extend our thanks to the following sponsors for funding Laravel development. If you are interested in becoming a sponsor, please visit the [Laravel Partners program](https://partners.laravel.com).

### Premium Partners

- **[Vehikl](https://vehikl.com)**
- **[Tighten Co.](https://tighten.co)**
- **[Kirschbaum Development Group](https://kirschbaumdevelopment.com)**
- **[64 Robots](https://64robots.com)**
- **[Curotec](https://www.curotec.com/services/technologies/laravel)**
- **[DevSquad](https://devsquad.com/hire-laravel-developers)**
- **[Redberry](https://redberry.international/laravel-development)**
- **[Active Logic](https://activelogic.com)**

## Contributing

Thank you for considering contributing to the Laravel framework! The contribution guide can be found in the [Laravel documentation](https://laravel.com/docs/contributions).

## Code of Conduct

In order to ensure that the Laravel community is welcoming to all, please review and abide by the [Code of Conduct](https://laravel.com/docs/contributions#code-of-conduct).

## Security Vulnerabilities

If you discover a security vulnerability within Laravel, please send an e-mail to Taylor Otwell via [taylor@laravel.com](mailto:taylor@laravel.com). All security vulnerabilities will be promptly addressed.

## License

The Laravel framework is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
