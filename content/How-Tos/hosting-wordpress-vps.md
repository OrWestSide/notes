# WordPress + MariaDB + Traefik on an Existing Docker VPS

This guide shows how to deploy a new, independent WordPress site on a VPS that already has **Docker + Traefik** running.

The intended architecture is:

```text
Internet
   │
   ▼
Domain
example.com
   │
   ▼
Traefik :443
   │
   │  traefik_default
   ▼
WordPress container :80
   │
   │  private wordpress network
   ▼
MariaDB :3306
```

Each WordPress project gets its own:

* WordPress container
* MariaDB container
* private Docker network
* Docker volumes
* Traefik hostname/router

Traefik itself is shared between all projects.

---

# 1. Prerequisites

The VPS should already have:

* Docker
* Docker Compose
* Traefik running
* Ports `80` and `443` available
* DNS pointing the domain to the VPS

For example:

```text
papanikolakis.innovia-labs.com
        │
        ▼
      VPS IP
```

Verify Traefik:

```bash
docker ps
```

Verify the Traefik network:

```bash
docker network ls
```

You should have something like:

```text
traefik_default
```

The exact network name depends on how the Traefik Compose project was created.

---

# 2. Create the WordPress project directory

For example:

```bash
mkdir -p ~/Projects/wordpress/papanikolakis
cd ~/Projects/wordpress/papanikolakis
```

A good structure for multiple sites is:

```text
~/Projects/wordpress/
├── papanikolakis/
│   └── compose.yml
├── project2/
│   └── compose.yml
└── project3/
    └── compose.yml
```

Each project is independent.

---

# 3. Create `compose.yml`

Use:

```yaml
services:

  wordpress:
    image: wordpress:php8.3-apache
    restart: unless-stopped

    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: CHANGE_ME_DB_PASSWORD

    volumes:
      - wordpress_data:/var/www/html

    networks:
      - wordpress
      - traefik

    labels:
      - "traefik.enable=true"

      # IMPORTANT:
      # WordPress is connected to two networks.
      # Explicitly tell Traefik to use the Traefik network.
      - "traefik.docker.network=traefik_default"

      # Router
      - "traefik.http.routers.blog.rule=Host(`papanikolakis.innovia-labs.com`)"
      - "traefik.http.routers.blog.entrypoints=websecure"
      - "traefik.http.routers.blog.tls=true"
      - "traefik.http.routers.blog.tls.certresolver=myresolver"

      # WordPress container's internal HTTP port
      - "traefik.http.services.blog.loadbalancer.server.port=80"

  db:
    image: mariadb:11
    restart: unless-stopped

    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress

      # MUST match WORDPRESS_DB_PASSWORD above
      MYSQL_PASSWORD: CHANGE_ME_DB_PASSWORD

      # This can and should be different
      MYSQL_ROOT_PASSWORD: CHANGE_ME_ROOT_PASSWORD

    volumes:
      - db_data:/var/lib/mysql

    networks:
      - wordpress

volumes:
  wordpress_data:
  db_data:

networks:
  wordpress:

  traefik:
    external: true
    name: traefik_default
```

---

# 4. Understand the passwords

There are three relevant credentials.

## WordPress database password

These two must be identical:

```yaml
WORDPRESS_DB_PASSWORD: CHANGE_ME_DB_PASSWORD
```

and:

```yaml
MYSQL_PASSWORD: CHANGE_ME_DB_PASSWORD
```

For example:

```yaml
WORDPRESS_DB_PASSWORD: my-db-password-123
```

and:

```yaml
MYSQL_PASSWORD: my-db-password-123
```

## MariaDB root password

This is separate:

```yaml
MYSQL_ROOT_PASSWORD: completely-different-root-password
```

So:

```text
MYSQL_PASSWORD
       │
       └── must equal WORDPRESS_DB_PASSWORD

MYSQL_ROOT_PASSWORD
       │
       └── can be completely different
```

Do **not** make all three passwords the same.

---

# 5. Important MariaDB behavior

MariaDB initialization happens when the database volume is first created.

This means changing:

```yaml
MYSQL_PASSWORD: new-password
```

later does **not necessarily change the password of an existing MariaDB user**.

The credentials in the Compose file are primarily used during initial database initialization.

Therefore:

### For a brand-new installation

Set the passwords correctly **before** starting the containers.

### For an existing database

Do not assume changing `MYSQL_PASSWORD` will update the existing database user.

You may need to change the MariaDB user's password directly.

---

# 6. Start the stack

From the project directory:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Expected:

```text
NAME                        SERVICE      STATUS
papanikolakis-db-1          db           Up
papanikolakis-wordpress-1   wordpress    Up
```

Check WordPress logs:

```bash
docker compose logs wordpress --tail=50
```

Normal startup should include something similar to:

```text
Apache/2.x ... configured -- resuming normal operations
```

The following warning is generally harmless:

```text
AH00558: apache2: Could not reliably determine the server's fully qualified domain name
```

It does not normally prevent WordPress from working.

---

# 7. Verify the Docker networks

WordPress must be connected to **both** networks:

```text
papanikolakis_wordpress
traefik_default
```

Check:

```bash
docker inspect papanikolakis-wordpress-1 \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'
```

You should see two IP addresses.

For example:

```text
172.19.0.3 172.18.0.3
```

The `172.18.x.x` address in this example is the address on the Traefik network.

---

# 8. Verify Traefik discovered the router

Traefik should discover the Docker labels automatically.

The router should correspond to:

```text
Host(`papanikolakis.innovia-labs.com`)
```

and:

```text
entrypoint: websecure
```

with:

```text
TLS
certificate resolver: myresolver
```

If you have Traefik's API available, you can inspect the routers.

The important thing is that you should see something conceptually like:

```text
blog@docker
Host(`papanikolakis.innovia-labs.com`)
websecure
```

---

# 9. Test WordPress directly from Traefik

This is one of the most useful troubleshooting techniques.

First find the WordPress IP on `traefik_default`:

```bash
docker inspect papanikolakis-wordpress-1 \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'
```

Then test it from inside the Traefik container:

```bash
docker exec traefik-reverse-proxy-1 \
  wget -S -O- http://WORDPRESS_TRAEFIK_NETWORK_IP:80
```

For example:

```bash
docker exec traefik-reverse-proxy-1 \
  wget -S -O- http://172.18.0.3:80
```

If this returns:

```text
HTTP/1.1 200 OK
```

and WordPress HTML, then:

* Docker networking works
* Apache works
* WordPress works
* Traefik can reach WordPress

At that point, investigate the Traefik configuration rather than WordPress.

---

# 10. The critical Traefik label

Because WordPress is connected to two networks:

```yaml
networks:
  - wordpress
  - traefik
```

Traefik needs to know which network to use.

Use:

```yaml
- "traefik.docker.network=traefik_default"
```

Without this, Traefik can potentially select the wrong Docker network.

The complete important section is:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.docker.network=traefik_default"

  - "traefik.http.routers.blog.rule=Host(`papanikolakis.innovia-labs.com`)"
  - "traefik.http.routers.blog.entrypoints=websecure"
  - "traefik.http.routers.blog.tls=true"
  - "traefik.http.routers.blog.tls.certresolver=myresolver"

  - "traefik.http.services.blog.loadbalancer.server.port=80"
```

---

# 11. Access the site

Once everything is working, open:

```text
https://papanikolakis.innovia-labs.com
```

You should see the standard WordPress installation page.

Complete:

* Site title
* Admin username
* Admin password
* Admin email

The WordPress installation is then ready.

---

# 12. Adding another WordPress site

Create another directory:

```bash
mkdir -p ~/Projects/wordpress/project2
cd ~/Projects/wordpress/project2
```

Create another `compose.yml`.

Use the same Traefik network:

```yaml
networks:
  traefik:
    external: true
    name: traefik_default
```

Give the new site its own private network:

```yaml
networks:
  wordpress:
  traefik:
    external: true
    name: traefik_default
```

Use a different hostname:

```yaml
- "traefik.http.routers.project2.rule=Host(`project2.example.com`)"
```

And explicitly use:

```yaml
- "traefik.docker.network=traefik_default"
```

Each site can therefore look like:

```text
Internet
   │
   ├── site1.example.com ──► WordPress 1 ──► MariaDB 1
   │
   ├── site2.example.com ──► WordPress 2 ──► MariaDB 2
   │
   └── site3.example.com ──► WordPress 3 ──► MariaDB 3
                 │
                 ▼
              Traefik
```

---

# Troubleshooting

## 1. Gateway Timeout

If the browser says:

```text
Gateway Timeout
```

do **not immediately assume Traefik is broken**.

Test WordPress directly from the Traefik container:

```bash
docker inspect papanikolakis-wordpress-1 \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'
```

Then:

```bash
docker exec traefik-reverse-proxy-1 \
  wget -S -O- http://WORDPRESS_IP:80
```

### If you get `200 OK`

WordPress is working.

Check:

```yaml
- "traefik.docker.network=traefik_default"
```

and verify that the WordPress container is actually connected to:

```text
traefik_default
```

### If you get `500 Internal Server Error`

The problem is inside WordPress/PHP/database.

Check:

```bash
docker compose logs wordpress --tail=100
```

and:

```bash
docker exec papanikolakis-wordpress-1 \
  tail -100 /var/log/apache2/error.log
```

---

# 2. MariaDB says `Access denied`

For example:

```text
Access denied for user 'wordpress'@'172.19.0.2'
```

This means WordPress reached MariaDB, but authentication failed.

Check that:

```yaml
WORDPRESS_DB_PASSWORD
```

and:

```yaml
MYSQL_PASSWORD
```

are identical.

Remember that changing `MYSQL_PASSWORD` after MariaDB has already been initialized may not change the existing database user's password.

---

# 3. Root login fails

If you see:

```text
Access denied for user 'root'@'localhost'
```

the root password you're trying does not match the password stored in the existing MariaDB database.

Again, changing:

```yaml
MYSQL_ROOT_PASSWORD
```

doesn't necessarily change an already-initialized MariaDB root account.

If this is a production database, do **not** delete the volume just to fix credentials.

---

# 4. Clean installation and credentials are messed up

If this is genuinely a brand-new installation and there is **nothing to preserve**, the easiest solution is to destroy the project's volumes and recreate it.

From the project directory:

```bash
docker compose down -v
```

Then:

```bash
docker compose up -d
```

This deletes the project's:

* WordPress volume
* MariaDB volume

and creates them again.

**Never do this on a production installation unless you have confirmed you have backups or intentionally want to destroy the data.**

---

# 5. Check container status

```bash
docker compose ps
```

Both should be:

```text
Up
```

If something is restarting:

```bash
docker compose logs SERVICE_NAME --tail=100
```

For example:

```bash
docker compose logs wordpress --tail=100
```

or:

```bash
docker compose logs db --tail=100
```

---

# 6. Check the database

MariaDB should eventually report:

```text
ready for connections
```

If it does, MariaDB itself is running.

You can also check:

```bash
docker compose logs db --tail=100
```

---

# 7. Check WordPress → MariaDB connectivity

WordPress uses:

```yaml
WORDPRESS_DB_HOST: db:3306
```

The hostname `db` comes from the Docker Compose service name.

Do **not** use:

```text
localhost
```

for the MariaDB host.

Inside the WordPress container:

```text
db:3306
```

means:

```text
WordPress container
       │
       ▼
Docker DNS
       │
       ▼
db container
       │
       ▼
MariaDB :3306
```

---

# 8. Check the WordPress container networks

```bash
docker inspect papanikolakis-wordpress-1 \
  --format '{{json .NetworkSettings.Networks}}'
```

You should see both:

```text
papanikolakis_wordpress
traefik_default
```

MariaDB should only need:

```text
papanikolakis_wordpress
```

It does **not** need to be exposed to Traefik.

---

# 9. Check Traefik logs

```bash
docker logs traefik-reverse-proxy-1 --tail=100
```

If there are no errors, that doesn't necessarily mean the request is working. The direct `wget` test from inside Traefik is more useful for determining whether Traefik can actually reach the backend.

---

# 10. Check the router configuration

If the router isn't appearing in Traefik, inspect the WordPress labels:

```bash
docker inspect papanikolakis-wordpress-1 \
  --format '{{json .Config.Labels}}'
```

Verify:

```text
traefik.enable=true
traefik.docker.network=traefik_default
```

and the router rule:

```text
Host(`papanikolakis.innovia-labs.com`)
```

---

# 11. Don't confuse HTTP 500 with Gateway Timeout

These indicate different problems.

### HTTP 500

```text
Traefik
   ↓
WordPress
   ↓
PHP/application error
```

The request reached WordPress.

### Gateway Timeout

Usually means Traefik cannot successfully complete the request to the backend.

First test:

```bash
docker exec traefik-reverse-proxy-1 \
  wget -S -O- http://WORDPRESS_IP:80
```

This immediately tells you whether the backend itself is reachable.

---

# 12. Don't delete the database volume casually

This command is destructive:

```bash
docker compose down -v
```

It is fine for a disposable clean installation.

It is **not** a normal troubleshooting command for an established WordPress site.

For a real site, investigate and repair the credentials instead.

---

# 13. Recommended production improvements

The basic setup works, but there are a few things worth improving later.

## Don't expose the Traefik dashboard publicly

If Traefik is currently configured with:

```text
--api.insecure=true
```

and:

```yaml
ports:
  - "8080:8080"
```

the dashboard/API is exposed directly.

For production, disable insecure API access and expose the dashboard through a properly protected Traefik route, or don't expose it at all.

## Use `.env` or Docker secrets

Instead of committing passwords directly into `compose.yml`, consider:

```text
.env
```

or Docker secrets for production deployments.

## Back up MariaDB

Your important WordPress data is in:

```text
db_data
```

and WordPress files/uploads are in:

```text
wordpress_data
```

Back up both appropriately.

## Keep each project isolated

For unrelated WordPress sites, the recommended structure is:

```text
/srv/wordpress/site1
/srv/wordpress/site2
/srv/wordpress/site3
```

with each project having its own:

```text
WordPress
MariaDB
private network
volumes
Traefik router
```

while sharing only:

```text
traefik_default
```

with the reverse proxy.
