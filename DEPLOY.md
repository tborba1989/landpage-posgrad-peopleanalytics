# Deploy — peopleanalytics.persoa.tec.br

## Estrutura final
- URL pública: https://peopleanalytics.persoa.tec.br
- Container: nginx:alpine na porta interna 8080
- Isolamento: rede própria, sem interferência no n8n

---

## Passo 1 — DNS (painel do seu registrador de domínio)

Adicione um registro tipo **A** apontando para o IP da sua VPS Oracle:

| Tipo | Nome              | Valor             | TTL  |
|------|-------------------|-------------------|------|
| A    | peopleanalytics   | <IP_DA_VPS>       | 3600 |

Aguarde a propagação (geralmente 5 a 30 minutos).

Verifique com:
```bash
dig peopleanalytics.persoa.tec.br +short
# Deve retornar o IP da VPS
```

---

## Passo 2 — Subir os arquivos na VPS

```bash
# No seu computador local
scp -r ./persoa-landing ubuntu@<IP_DA_VPS>:/opt/persoa-landing
```

---

## Passo 3 — Subir o container

```bash
# Na VPS
cd /opt/persoa-landing
docker compose up -d
```

Verificar se está rodando:
```bash
docker ps | grep persoa
# Esperado: persoa-landing   Up X minutes   0.0.0.0:8080->80/tcp
```

---

## Passo 4 — Proxy reverso (escolha uma opção)

### Opção A — Nginx no host (mais comum)

Crie o arquivo de configuração do subdomínio:
```bash
sudo nano /etc/nginx/sites-available/peopleanalytics
```

Conteúdo:
```nginx
server {
    listen 80;
    server_name peopleanalytics.persoa.tec.br;

    location / {
        proxy_pass         http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

Ativar e testar:
```bash
sudo ln -s /etc/nginx/sites-available/peopleanalytics \
           /etc/nginx/sites-enabled/peopleanalytics
sudo nginx -t && sudo systemctl reload nginx
```

### Opção B — Caddy (SSL automático, mais simples)

Adicione ao seu Caddyfile:
```
peopleanalytics.persoa.tec.br {
    reverse_proxy localhost:8080
}
```

Recarregar:
```bash
caddy reload
```
O SSL é provisionado automaticamente via Let's Encrypt.

---

## Passo 5 — SSL com Certbot (apenas se usar Nginx)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d peopleanalytics.persoa.tec.br
```

Siga as instruções interativas. O Certbot configura o HTTPS
e renova automaticamente a cada 90 dias.

Verificar renovação automática:
```bash
sudo systemctl status certbot.timer
```

---

## Passo 6 — Verificação final

```bash
curl -I https://peopleanalytics.persoa.tec.br
# Esperado: HTTP/2 200
```

Abra no navegador:
```
https://peopleanalytics.persoa.tec.br
```

---

## Isolamento do n8n — confirmação

O container da landing page usa a rede interna `persoa-net`.
O n8n opera em sua própria rede e porta (tipicamente 5678).
Não há colisão de portas, redes ou volumes.

| Serviço       | Porta interna | Porta host | Rede Docker  |
|---------------|--------------|------------|--------------|
| persoa-landing| 80           | 8080       | persoa-net   |
| n8n           | 5678         | 5678       | (própria)    |

Nenhuma configuração do n8n será alterada em nenhuma etapa deste deploy.

---

## Atualizar o conteúdo da página no futuro

```bash
# Copiar o novo index.html para a VPS
scp ./index.html ubuntu@<IP_DA_VPS>:/opt/persoa-landing/index.html

# O nginx serve o arquivo diretamente — sem necessidade de restart
```
