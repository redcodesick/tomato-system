# tomato-system
 docker media stacks

There are two media stacks available.

`tomato-one` This stack contains Jellyfin, Radarr, Sonarr, Jackett and Deluge.

`tomato-two` This stack contains Jellyfin, Radarr, Sonarr, Jackett and Deluge.

Any one of them can be deployed using --profile option with docker-compose.

```
docker network create tomato-network

# Install Jellyfin, Radarr, Sonarr, Jackett and Deluge stack
docker-compose --profile tomato-one up -d

# Or, Install Jellyfin, Radarr, Sonarr, Deluge.

docker-compose -f docker-compose-nginx.yml up -d # OPTIONAL to use Nginx as reverse proxy
```

# Configure Deluge

# Add indexer to Jackett

- Open Jackett UI at http://localhost:9117
- Add indexer
- Search for torrent indexer (e.g. the pirates bay, YTS)
- Add selected

# Configure Radarr

- Open Radarr at http://localhost:7878
- Settings --> Media Management --> Check mark "Movies deleted from disk are automatically unmonitored in Radarr" under File management section --> Save
- Settings --> Indexers --> Add --> Add Rarbg indexer --> Add minimum seeder (4) --> Test --> Save
- Settings --> Indexers --> Add --> Torznab --> Follow steps from Jackett to add indexer
- Settings --> Download clients --> Transmission --> Add Host (transmission / qbittorrent) and port (9091 / 5080) --> Username and password if added --> Test --> Save 
- Settings --> General --> Enable advance setting --> Select AUthentication and add username and password

# Add a movie

- Movies --> Search for a movie --> Add Root folder (/downloads) --> Quality profile --> Add movie
- Go to transmission (http://localhost:9091) and see if movie is getting downloaded.

# Configure Jellyfin

- Open Jellyfin at http://localhost:8096
- Configure as it asks for first time.
- Add media library folder and choose /data/movies/

# Configure Jackett

- Add admin password

# Apply SSL in Nginx

- Open port 80 and 443.
- Get inside Nginx container and install certbot and certbot-nginx `apk add certbot certbot-nginx`
- Add URL in server block. e.g. `server_name  localhost armdev.navratangupta.in;` in /etc/nginx/conf.d/default.conf
- Run `certbot --nginx` and provide details asked.


# Configure Nginx

- Get inside Nginx container
- `cd /etc/nginx/conf.d`
- Add proxies as per below for all tools.
- OR, copy nginx.conf file to /etc/nginx/conf.d/default.conf and make necessary changes

`docker cp nginx.conf nginx:/etc/nginx/conf.d/default.conf && docker exec -it nginx nginx -s reload`
- Close ports of other tools in firewall/security groups except port 80 and 443.


# Radarr Nginx reverse proxy

- Settings --> General --> URL Base --> Add base (/radarr)
- Add below proxy in nginx configuration

```
location /radarr {
    proxy_pass http://radarr:7878;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
```

- Restart containers.

# Sonarr Nginx reverse proxy

- Settings --> General --> URL Base --> Add base (/sonarr)
- Add below proxy in nginx configuration

```
location /radarr {
    proxy_pass http://sonarr:8989;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
```

# Jackett Nginx reverse proxy

- Get inside jackett container and go to `/config/Jackett/`
- Add `"BasePathOverride": "/jackett"` in ServerConfig.json file.
- Add below proxy

```
location /jackett/ {
    proxy_pass         http://jackett:9117;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection keep-alive;
    proxy_cache_bypass $http_upgrade;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   X-Forwarded-Host $http_host;
}
```

- Restart containers

# Transmission Nginx reverse proxy

- Add below proxy in Nginx config

```
location /deluge {
    proxy_pass http://localhost:8112/;
    proxy_set_header X-Deluge-Base "/deluge/";
    include proxy-control.conf;
    add_header X-Frame-Options SAMEORIGIN;
}
```
# Jellyfin Nginx proxy

- Add base URL, Admin Dashboard -> Networking -> Base URL (/jellyfin)
- Add below config in Ngix config

```
 location /jellyfin {
        return 302 $scheme://$host/jellyfin/;
    }

    location /jellyfin/ {

        proxy_pass http://jellyfin:8096/jellyfin/;

        proxy_pass_request_headers on;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $http_host;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;

        # Disable buffering when the nginx proxy gets very resource heavy upon streaming
        proxy_buffering off;
    }
```
- Restart containers
