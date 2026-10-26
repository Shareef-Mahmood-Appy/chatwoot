# Deploying Chatwoot on Portainer

i used a single Docker Compose file that includes every service Chatwoot needs (Rails, Sidekiq, Postgres with pgvector, and Redis). It is based on `docker-compose.production.yaml` from the Chatwoot GitHub repo.

## Steps

1. Open the Portainer web UI home page.
2. Click **Local**.
3. Click **Stacks**.
4. Click **Add stack**.
5. Name the stack (i used `chatwoot`).
6. Paste the following Compose file into the **Web editor**:

```yaml
services:
  base: &base
    image: chatwoot/chatwoot:latest
    env_file: stack.env
    volumes:
      - storage_data:/app/storage

  rails:
    <<: *base
    depends_on:
      - postgres
      - redis
    ports:
      - '127.0.0.1:3000:3000'
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    entrypoint: docker/entrypoints/rails.sh
    command: ['bundle', 'exec', 'rails', 's', '-p', '3000', '-b', '0.0.0.0']
    restart: always

  sidekiq:
    <<: *base
    depends_on:
      - postgres
      - redis
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    command: ['bundle', 'exec', 'sidekiq', '-C', 'config/sidekiq.yml']
    restart: always

  postgres:
    image: pgvector/pgvector:pg16
    restart: always
    ports:
      - '127.0.0.1:5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=chatwoot
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

  redis:
    image: redis:alpine
    restart: always
    command: ["sh", "-c", "redis-server --requirepass \"$REDIS_PASSWORD\""]
    env_file: stack.env
    volumes:
      - redis_data:/data
    ports:
      - '127.0.0.1:6380:6379'

volumes:
  storage_data:
    external: true
    name: chatwoot_storage_data
  postgres_data:
    external: true
    name: chatwoot_postgres_data
  redis_data:
    external: true
    name: chatwoot_redis_data
```

7. Under **Environment variables**, either upload your `.env` file with **Load variables from .env file**, or add the variables one by one.
8. Click **Deploy the stack**.

## Notes

- **Create the external volumes first.** The Compose file marks all three volumes as `external: true`, so Docker will not create them automatically. Create `chatwoot_storage_data`, `chatwoot_postgres_data`, and `chatwoot_redis_data` in Portainer under **Volumes → Add volume** before deploying, or the deployment will fail.
- **Required variables.** At minimum, set `POSTGRES_PASSWORD` and `REDIS_PASSWORD`. Chatwoot also needs variables such as `SECRET_KEY_BASE`, `FRONTEND_URL`, `POSTGRES_HOST=postgres`, `POSTGRES_USERNAME`, and `REDIS_URL` (which should include the Redis password). Use the `.env.example` file from the Chatwoot repo as a reference.

## Expose Chatwoot on a Domain (Cloudflare + Nginx Proxy Manager)

### 1. Create the DNS record in Cloudflare

1. In Cloudflare, open the `appyerp.net` domain and go to **DNS → Records**.
2. Click **Add record** and fill in:
   - **Type:** `A`
   - **Name:** `appysocialsupport` (this becomes `appysocialsupport.appyerp.net`)
   - **IPv4 address:** your server's public IP
   - **Proxy status:** Proxied (orange cloud) or DNS only (grey cloud). Either works.
3. go to **SSL/TLS → Overview** and set the mode to **Full** or **Full (strict)**.

### 2. Add the proxy host in Nginx Proxy Manager

1. Open the Nginx Proxy Manager web UI.
2. Go to **Hosts → Proxy Hosts**.
3. Click **Add Proxy Host**.
4. On the **Details** tab, fill in:
   - **Domain Names:** `appysocialsupport.appyerp.net`
   - **Scheme:** `http`
   - **Forward Hostname / IP:** see the note below
   - **Forward Port:** `3000`
   - **Cache Assets:** off
   - **Block Common Exploits:** on
   - **Websockets Support:** **on**. Chatwoot needs this for live chat updates.
5. On the **SSL** tab, fill in:
   - **SSL Certificate:** Request a new SSL Certificate
   - **Force SSL:** on
   - **HTTP/2 Support:** on
   - **Email Address for Let's Encrypt:** your email
   - Agree to the Let's Encrypt terms
6. Click **Save**.

### 3. Set the Chatwoot frontend URL

In your Portainer stack's environment variables, set:

```
FRONTEND_URL=https://appysocialsupport.appyerp.net
```

Then redeploy the stack (**Stacks → chatwoot → Update the stack**) so Chatwoot generates correct links and emails.

### Forward Hostname / IP:

The Compose file binds Rails to `127.0.0.1:3000`.

In Portainer, go to Networks and find the network created by your stack. It is named chatwoot_default if the stack is called chatwoot (your stack name_default).
scroll to Containers and click your Nginx Proxy Manager container.
Scroll to Connected networks and click Join a network.
Select chatwoot_default and click Join network.
