docker-compose.yml
services:
  stirling-pdf:
    image: stirlingtools/stirling-pdf:latest
    container_name: stirling-pdf
    ports:
      - '8080:8080'
    volumes:
      - ./stirling-data/tessdata:/usr/share/tessdata  # OCR language files
      - ./stirling-data/configs:/configs               # Settings & database
      - ./stirling-data/logs:/logs                     # Application logs
      - ./stirling-data/pipeline:/pipeline             # Automation configs
    environment:
      - SECURITY_ENABLELOGIN=false    # Set true to enable user authentication
      - LANGS=pt_BR                   # Interface language
      - SYSTEM_MAXFILESIZE=2000        # MB
      - SPRING_SERVLET_MULTIPART_MAX_FILE_SIZE=2000MB
      - SPRING_SERVLET_MULTIPART_MAX_REQUEST_SIZE=2000MB
      - JAVA_TOOL_OPTIONS="-Xms512m -Xmx4g"  # Min 512MB, Max 4GB RAM

    restart: unless-stopped


baixar pacotes de idiomas para OCR
https://github.com/tesseract-ocr/tessdata



cat: /etc/nginx/conf.d/pdf.jmsalles.homelab.br: No such file or directory
server {
    listen 80;
    server_name pdf.jmsalles.homelab.br;

    access_log /var/log/nginx/pdf.jmsalles.homelab.access.log;
    error_log  /var/log/nginx/pdf.jmsalles.homelab.error.log;

    location / {
        proxy_pass http://host.docker.internal:8080;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
