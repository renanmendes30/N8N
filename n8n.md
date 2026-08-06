---

# n8n + PostgreSQL + Nginx na Oracle Cloud

## Requisitos

* Oracle Cloud Free Tier ou superior
* Ubuntu 24.04 LTS
* Docker
* Docker Compose
* Portas abertas na Security List:

  * 22
  * 80
  * 443

> Não é necessário abrir a porta 5678 para a Internet. O acesso será feito pelo Nginx.

---

# 1. Atualizar o servidor

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Reconecte via SSH.

---

# 2. Instalar o Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Adicionar o usuário ao grupo Docker:

```bash
sudo usermod -aG docker $USER
```

Feche a conexão SSH e conecte novamente.

Verifique:

```bash
docker --version
docker compose version
```

---

# 3. Criar a estrutura

```bash
sudo mkdir -p /opt/n8n
sudo chown -R $USER:$USER /opt/n8n

cd /opt/n8n
```

---

# 4. Criar o arquivo `.env`

```bash
nano .env
```

Conteúdo:

```env
POSTGRES_DB=n8n
POSTGRES_USER=n8n
POSTGRES_PASSWORD=TroquePorUmaSenhaForte

N8N_HOST=SEU_IP_PUBLICO
N8N_PROTOCOL=http
N8N_PORT=5678
N8N_WEBHOOK_URL=http://SEU_IP_PUBLICO/

GENERIC_TIMEZONE=America/Sao_Paulo
```

Exemplo:

```env
N8N_HOST=163.176.xxx.xxx
N8N_WEBHOOK_URL=http://163.176.xxx.xxx/
```

---

# 5. Criar o docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
services:

  postgres:
    image: postgres:16
    restart: unless-stopped

    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest

    restart: unless-stopped

    depends_on:
      - postgres

    expose:
      - "5678"

    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: ${POSTGRES_DB}
      DB_POSTGRESDB_USER: ${POSTGRES_USER}
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}

      N8N_HOST: ${N8N_HOST}
      N8N_PROTOCOL: ${N8N_PROTOCOL}
      N8N_PORT: ${N8N_PORT}
      N8N_WEBHOOK_URL: ${N8N_WEBHOOK_URL}

      GENERIC_TIMEZONE: ${GENERIC_TIMEZONE}
      TZ: America/Sao_Paulo

      N8N_SECURE_COOKIE: "false"

    volumes:
      - n8n_data:/home/node/.n8n

  nginx:
    image: nginx:latest

    restart: unless-stopped

    depends_on:
      - n8n

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf

volumes:
  postgres_data:
  n8n_data:
```

---

# 6. Criar a configuração do Nginx

```bash
mkdir nginx

nano nginx/default.conf
```

Conteúdo:

```nginx
server {

    listen 80;

    server_name _;

    location / {

        proxy_pass http://n8n:5678;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;

        proxy_cache_bypass $http_upgrade;
    }
}
```

> **Importante:** utilizar `server_name _;` enquanto estiver acessando pelo IP. Depois que configurar um domínio, substitua `_` pelo nome do domínio.

---

# 7. Iniciar os containers

```bash
docker compose pull

docker compose up -d
```

Verificar:

```bash
docker ps
```

Saída esperada:

```
postgres
n8n
nginx
```

---

# 8. Testar

Abra no navegador:

```
http://SEU_IP_PUBLICO
```

O n8n deverá exibir a tela inicial.

---

# 9. Atualizar o n8n

```bash
cd /opt/n8n

docker compose pull

docker compose up -d
```

---

# 10. Comandos úteis

Ver logs do n8n:

```bash
docker compose logs -f n8n
```

Ver logs do PostgreSQL:

```bash
docker compose logs -f postgres
```

Ver logs do Nginx:

```bash
docker compose logs -f nginx
```

Parar os containers:

```bash
docker compose down
```

Iniciar novamente:

```bash
docker compose up -d
```

Reiniciar:

```bash
docker compose restart
```

---

# Problemas encontrados durante a instalação

### PostgreSQL incompatível

Erro:

```
database files are incompatible with server
```

**Causa:** volume criado com PostgreSQL 17 e tentativa de iniciar com PostgreSQL 16.

**Solução:**

```bash
docker compose down -v
```

ou remover o volume manualmente:

```bash
docker volume rm n8n_postgres_data
```

---

### Porta 80 ocupada

Erro:

```
failed to bind host port 80
```

Verificar:

```bash
sudo ss -tulpn | grep :80
```

Parar o serviço que estiver utilizando a porta (Apache, Nginx ou outro container) antes de iniciar o Nginx do Docker.

---

### 502 Bad Gateway

**Causa:** o Nginx iniciou, mas o n8n ainda não estava respondendo ou havia configuração incorreta.

Verificar:

```bash
docker compose logs n8n
```

---

### Cannot GET

**Causa:** configuração incompleta do Nginx ou `server_name` apontando para um domínio inexistente.

**Solução:**

* Utilizar `server_name _;`
* Configurar corretamente o `proxy_pass`
* Reiniciar o Nginx:

```bash
docker compose restart nginx
```

---

# Próximos passos (produção)

Quando tiver um domínio configurado:

1. Atualize o `.env`:

```env
N8N_HOST=n8n.seudominio.com
N8N_PROTOCOL=https
N8N_WEBHOOK_URL=https://n8n.seudominio.com/
```

2. Atualize o Nginx:

```nginx
server_name n8n.seudominio.com;
```

3. Instale o Certbot:

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d n8n.seudominio.com
```

4. Altere no `docker-compose.yml`:

```yaml
N8N_SECURE_COOKIE: "true"
```

---

Eu também adicionaria ao repositório um `README.md` com esse passo a passo, um `.env.example` (sem senhas) e a estrutura de pastas pronta para que qualquer pessoa consiga clonar o projeto, editar o `.env` e executar apenas:

```bash
docker compose up -d
```

Isso deixa o projeto muito mais fácil de usar e manter.
