 # Citas Médicas

Aplicación para la gestión de citas médicas desarrollada con Laravel.

El proyecto se inició sin Docker (PHP nativo + SQLite) y más adelante se incorporó [Laravel Sail](https://laravel.com/docs/sail) para poder trabajar con Docker. Ambas formas de instalación siguen siendo válidas; elige la que prefieras.

## Requisitos

### Con Docker (Sail)

- Docker y Docker Compose
- Git

### Sin Docker

- PHP >= 8.1
- Composer
- Node.js y npm
- Git
- MySQL, PostgreSQL o SQLite

## Instalación con Docker (Sail)

1. Clona el repositorio y accede al proyecto:

	```bash
	git clone <URL_DEL_REPOSITORIO>
	cd citas-medicas
	```

2. Crea el archivo de entorno:

	```bash
	cp .env.example .env
	```

3. Configura en `.env` la conexión a la base de datos para Sail (contenedor MySQL):

	```env
	DB_CONNECTION=mysql
	DB_HOST=mysql
	DB_PORT=3306
	DB_DATABASE=citas_medicas
	DB_USERNAME=sail
	DB_PASSWORD=password
	```

4. Instala las dependencias de PHP (necesario una vez, usando un contenedor temporal, ya que aún no tienes `vendor/bin/sail`):

	```bash
	docker run --rm -u "$(id -u):$(id -g)" -v "$(pwd):/var/www/html" -w /var/www/html laravelsail/php85-composer:latest composer install --ignore-platform-reqs
	```

5. Levanta los contenedores:

	```bash
	./vendor/bin/sail up -d
	```

6. Genera la clave de la aplicación y ejecuta las migraciones:

	```bash
	./vendor/bin/sail artisan key:generate
	./vendor/bin/sail artisan migrate
	```

7. Instala las dependencias de JavaScript e inicia el compilador de recursos frontend:

	```bash
	./vendor/bin/sail npm install
	./vendor/bin/sail npm run dev
	```

La aplicación estará disponible en `http://localhost`.

> Si `http://localhost` no carga aunque los contenedores estén corriendo, revisa [README-Docker.md](README-Docker.md) para el diagnóstico y solución de problemas comunes (conflictos de puerto, firewall, VPN).

## Instalación sin Docker

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

4. Configura en `.env` la conexión a la base de datos, especialmente `DB_DATABASE`, `DB_USERNAME` y `DB_PASSWORD` (por defecto usa SQLite).

5. Ejecuta las migraciones:

	```bash
	php artisan migrate
	```

### Ejecución en desarrollo (sin Docker)

Inicia el servidor de Laravel:

```bash
php artisan serve
```

En otra terminal, inicia el compilador de recursos frontend:

```bash
npm run dev
```

La aplicación estará disponible en `http://127.0.0.1:8000`.

> ⚠️ No ejecutes `php artisan serve` en el host al mismo tiempo que los contenedores de Sail: ambos intentarán usar el mismo puerto y causarán conflictos de conexión. Usa un solo método a la vez.
