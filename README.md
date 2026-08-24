 # Citas Médicas

Aplicación para la gestión de citas médicas desarrollada con Laravel.

## Requisitos

- PHP >= 8.1
- Composer
- Node.js y npm
- Git
- MySQL, PostgreSQL o SQLite

## Instalación

1. Clona el repositorio y accede al proyecto:

	```bash
	git clone <URL_DEL_REPOSITORIO>
	cd citas-medicas
	```

2. Instala las dependencias de PHP y JavaScript:

	```bash
	composer install
	npm install
	```

3. Crea el archivo de entorno y genera la clave de la aplicación:

	```bash
	cp .env.example .env
	php artisan key:generate
	```

4. Configura en `.env` la conexión a la base de datos, especialmente `DB_DATABASE`, `DB_USERNAME` y `DB_PASSWORD`.

5. Ejecuta las migraciones:

	```bash
	php artisan migrate
	```

## Ejecución en desarrollo

Inicia el servidor de Laravel:

```bash
php artisan serve
```

En otra terminal, inicia el compilador de recursos frontend:

```bash
npm run dev
```

La aplicación estará disponible en `http://127.0.0.1:8000`.
