# Docker / Sail — Troubleshooting

Este proyecto usa [Laravel Sail](https://laravel.com/docs/sail) (`compose.yaml`) para levantar la app en Docker. Este documento registra un problema real de "no puedo ver mi app en localhost" y cómo se diagnosticó y resolvió.

## Atajo `sail`

Para no escribir `./vendor/bin/sail` cada vez, agrega esta función a tu `~/.bashrc` o `~/.zshrc`:

```bash
sail() {
    if [[ -x "./vendor/bin/sail" ]]; then
        ./vendor/bin/sail "$@"
    elif [[ -x "./sail" ]]; then
        ./sail "$@"
    else
        echo "No se encontró Laravel Sail en este proyecto."
        return 1
    fi
}
```

Después de agregarla, recarga la shell (`source ~/.bashrc` / `source ~/.zshrc`) y podrás usar `sail up -d`, `sail artisan migrate`, `sail npm run dev`, etc. desde cualquier subcarpeta del proyecto donde exista `vendor/bin/sail` o `./sail`.

## Síntomas

- `http://localhost` no carga (timeout, "connection reset" o "connection refused").
- Los contenedores de Sail aparecen como `Up` y `healthy` en `docker ps`.
- Dentro del contenedor la app responde bien (HTTP 200).

## Diagnóstico paso a paso

1. **Confirmar que los contenedores están arriba:**

   ```bash
   docker ps -a --filter "name=citas-medicas" --format "table {{.Names}}\t{{.Status}}"
   ```

2. **Revisar si hay un servidor duplicado en el host compitiendo por el puerto 80** (por ejemplo, un `php artisan serve` corriendo fuera de Docker al mismo tiempo que Sail):

   ```bash
   ps aux | grep "artisan serve"
   ```

   Si aparece un proceso local además del contenedor, mátalo (usa el PID que te devuelva `ps aux`):

   ```bash
   kill <PID>
   ```

3. **Probar la app directamente dentro del contenedor** (para descartar que el problema sea la app y no la red):

   ```bash
   docker exec citas-medicas-laravel.test-1 curl -sv http://127.0.0.1:80/
   ```

   Si esto responde `HTTP 200`, el problema está en el reenvío de puertos host → contenedor, no en Laravel.

4. **Si el contenedor no responde ni internamente**, revisa los logs y reinícialo:

   ```bash
   docker logs --tail 40 citas-medicas-laravel.test-1
   docker restart citas-medicas-laravel.test-1
   ```

5. **Probar desde el host contra el puerto publicado y contra la IP del contenedor:**

   ```bash
   curl -sv http://127.0.0.1:80/
   docker port citas-medicas-laravel.test-1
   docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' citas-medicas-laravel.test-1
   curl -sv http://<ip-del-contenedor>:80/
   ```

   - Si el paso 3 funciona pero estos fallan, el problema es de red/firewall en el host, no de Docker ni de Laravel.

## Cambiar el usuario/password de la base de datos

Las credenciales de MySQL salen de `DB_USERNAME` y `DB_PASSWORD` en `.env`, que `compose.yaml` pasa al contenedor como `MYSQL_USER`/`MYSQL_PASSWORD`. **MySQL solo aplica esas variables la primera vez que crea el volumen de datos** (`citas-medicas_sail-mysql`), así que editar el `.env` con el contenedor ya inicializado no cambia el usuario/password existente.

Para aplicar el cambio hay dos caminos:

**A) Recrear el volumen (borra los datos actuales, útil en desarrollo):**

```bash
# 1. Edita DB_USERNAME / DB_PASSWORD en .env

# 2. Baja los contenedores y borra el volumen de MySQL
sail down
docker volume rm citas-medicas_sail-mysql

# 3. Levanta de nuevo (MySQL se inicializa con las nuevas credenciales)
sail up -d

# 4. Corre las migraciones
sail artisan migrate
```

**B) Conservar los datos actuales (actualizar el usuario existente):**

```bash
# 1. Edita DB_PASSWORD en .env

# 2. Dentro del contenedor de MySQL, actualiza la contraseña del usuario
docker exec -it citas-medicas-mysql-1 mysql -u root -p"$(grep ^DB_PASSWORD .env | cut -d= -f2)" \
  -e "ALTER USER 'sail'@'%' IDENTIFIED BY 'nueva_password'; FLUSH PRIVILEGES;"
```

> Nota: en este proyecto se usó la opción A (recrear el volumen) porque era un entorno de desarrollo sin datos importantes.

## Causa raíz encontrada en este proyecto

En este equipo (Fedora) el problema fue que **Mullvad VPN tenía bloqueado "Local network sharing"**. Ese ajuste bloquea el tráfico del host hacia rangos de IP privados (`10.x`, `172.16-31.x`, `192.168.x`), lo que incluye las redes *bridge* virtuales que crea Docker (por ejemplo `172.19.0.0/16`). Por eso el contenedor respondía internamente pero `localhost` no.

### Verificar el estado del ajuste

```bash
mullvad lan get
```

### Solución

```bash
mullvad lan set allow
```

Después de habilitarlo, probar de nuevo:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost/
```

### Nota de seguridad

Habilitar "Local network sharing" hace que tu equipo vuelva a ser visible/alcanzable por otros dispositivos en la misma red física (LAN) mientras el VPN está conectado. Es un ajuste razonable en redes de confianza (casa/oficina), pero conviene desactivarlo en redes públicas no confiables:

```bash
mullvad lan set block
```

## Otras causas posibles del mismo síntoma (no aplicaron aquí, pero vale revisarlas)

- **`firewalld` interfiriendo con las reglas de red de Docker**: en Fedora/RHEL, si `firewalld` recarga sus reglas después de que Docker configuró las suyas (NAT/forwarding), el reenvío de puertos deja de funcionar. Reiniciar el servicio de Docker suele regenerar las reglas:

  ```bash
  sudo systemctl restart docker
  ```

  Luego volver a levantar los contenedores:

  ```bash
  ./vendor/bin/sail up -d
  ```

- **Conflicto de puertos**: otro proceso (local o de otro proyecto) usando el puerto 80 o 5173. Revisar con:

  ```bash
  ss -tlnp | grep -E ":80 |:5173 "
  ```
