# go2rtc + Nginx + Cloudflare Tunnel (com autenticação)

Este setup expõe o **go2rtc** de forma segura via **Cloudflare Tunnel**, com **proteção por senha (Basic Auth)** usando **Nginx**.  
Ideal para visualizar câmeras RTSP no celular ou navegador sem expor a rede local.

---

## 🧱 Estrutura

```
go2rtc-secure/
├── docker-compose.yml
├── go2rtc.yaml
├── nginx.conf
└── go2rtc.htpasswd
```

---

## ⚙️ Configuração

### 1. Criar arquivo `go2rtc.yaml`

Exemplo:
```yaml
streams:
  camera1: rtsp://usuario:senha@192.168.1.50:554/stream

api:
  listen: ":1984"

webrtc:
  candidates:
    - stun:stun.l.google.com:19302
```

Troque a URL RTSP pela da sua câmera.

---

### 2. Gerar senha (Basic Auth)

Instale `htpasswd` e crie o arquivo:
```bash
sudo apt install apache2-utils -y
htpasswd -c ./go2rtc.htpasswd usuario
```

Digite a senha quando solicitado.  
Este usuário/senha será exigido no login via navegador.

---

### 3. Configurar o `nginx.conf`

```nginx
server {
  listen 80;
  server_name _;

  auth_basic "Área restrita";
  auth_basic_user_file /etc/nginx/go2rtc.htpasswd;

  location / {
    proxy_pass http://go2rtc:1984;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

---

### 4. docker-compose.yml

```yaml
version: '3.8'

services:
  go2rtc:
    image: alexxit/go2rtc:latest
    container_name: go2rtc
    restart: unless-stopped
    volumes:
      - ./go2rtc.yaml:/config/go2rtc.yaml:ro
    networks:
      - camnet

  nginx:
    image: nginx:stable
    container_name: go2rtc-nginx
    restart: unless-stopped
    depends_on:
      - go2rtc
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./go2rtc.htpasswd:/etc/nginx/go2rtc.htpasswd:ro
    ports:
      - "1984:80"
    networks:
      - camnet

networks:
  camnet:
    driver: bridge
```

---

### 5. Subir os containers

Na pasta do projeto:
```bash
docker compose up -d
```

Teste local:
```
http://localhost:1984
```
O navegador pedirá usuário/senha e mostrará o painel do go2rtc.

---

### 6. Configurar o Cloudflare Tunnel

Arquivo `/etc/cloudflared/config.yml`:
```yaml
tunnel: nome_do_tunel
credentials-file: /etc/cloudflared/<id>.json

ingress:
  - hostname: camera.seudominio.com
    service: http://localhost:1984
  - service: http_status:404
```

Depois:
```bash
cloudflared tunnel route dns nome_do_tunel camera.seudominio.com
cloudflared tunnel run nome_do_tunel
```

Acesse:
```
https://camera.seudominio.com
```

Será exigido o mesmo usuário/senha definidos no `.htpasswd`.

---

## 📱 Acesso no Android

Abra o navegador no celular e acesse a URL do tunnel:  
```
https://camera.seudominio.com
```
O painel do go2rtc mostrará o feed da câmera via **WebRTC** ou **HLS**.

---

## 🔒 Segurança

- Use senha forte no `.htpasswd`.  
- O túnel Cloudflare fornece HTTPS automaticamente.  
- O RTSP fica interno — não é exposto.  
- Troque a porta 1984 se quiser evitar conflitos.  
- Opcional: combine com **Cloudflare Access** para autenticação MFA e controle granular.

---

## 🧰 Comandos úteis

Subir containers:
```bash
docker compose up -d
```

Ver logs:
```bash
docker compose logs -f
```

Atualizar imagens:
```bash
docker compose pull && docker compose up -d
```
