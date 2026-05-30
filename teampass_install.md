Segue o tutorial final já sem o item “Limpar instalação anterior” e com a numeração ajustada.

# Guia final de instalação do TeamPass no Rocky Linux 10.0 

## Resumo

Instalação nativa do TeamPass no Rocky Linux 10.0 usando Apache, PHP-FPM, MariaDB e a versão do TeamPass que funcionou no ambiente.

```text
TeamPass: 3.1.4.33
Sistema: Rocky Linux 10.0
Web server: Apache
PHP: PHP do Rocky
Banco: MariaDB
Diretório da aplicação: /var/www/html/teampass
Secure path: /var/lib/teampass_secure
URL final: http://192.168.31.143/teampass
Usuário web inicial: admin
Senha web inicial: senha cadastrada no wizard de instalação
```

## 1. Atualizar o Rocky Linux

```bash
sudo dnf update -y
```

## 2. Instalar pacotes necessários

```bash
sudo dnf install -y httpd mariadb-server git unzip vim cronie
```

```bash
sudo dnf install -y php php-cli php-common php-fpm php-mysqlnd php-mbstring php-bcmath php-gd php-xml php-curl php-gmp php-ldap php-opcache php-process
```

## 3. Habilitar os serviços

```bash
sudo systemctl enable --now mariadb
```

```bash
sudo systemctl enable --now php-fpm
```

```bash
sudo systemctl enable --now httpd
```

```bash
sudo systemctl enable --now crond
```

## 4. Validar serviços e PHP

```bash
php -v
```

```bash
php -m | egrep -i 'mysqli|mbstring|bcmath|xml|gd|curl|gmp|ldap|openssl|opcache'
```

```bash
sudo systemctl status httpd
```

```bash
sudo systemctl status php-fpm
```

```bash
sudo systemctl status mariadb
```

## 5. Ajustar parâmetros do PHP

```bash
sudo vim /etc/php.ini
```

Ajustar ou confirmar as linhas abaixo:

```ini
memory_limit = 512M
max_execution_time = 120
max_input_time = 120
post_max_size = 64M
upload_max_filesize = 64M
date.timezone = America/Sao_Paulo
session.cookie_httponly = 1
```

Reiniciar os serviços:

```bash
sudo systemctl restart php-fpm
```

```bash
sudo systemctl restart httpd
```

## 6. Criar o banco do TeamPass

```bash
sudo mysql -u root -p
```

Dentro do MariaDB, execute:

```sql
CREATE DATABASE teampass CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE USER 'teampass_user'@'localhost' IDENTIFIED BY 'SENHA_FORTE_DO_BANCO';

GRANT ALL PRIVILEGES ON teampass.* TO 'teampass_user'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

Teste o acesso ao banco:

```bash
mysql -u teampass_user -p teampass -e "SHOW DATABASES;"
```

## 7. Baixar o TeamPass 3.1.4.33

```bash
cd /var/www/html
```

```bash
sudo git clone --branch 3.1.4.33 --depth 1 https://github.com/nilsteampassnet/TeamPass.git teampass
```

```bash
cd /var/www/html/teampass
```

```bash
sudo git describe --tags
```

Resultado esperado:

```text
3.1.4.33
```

## 8. Criar o secure path

```bash
sudo mkdir -p /var/lib/teampass_secure
```

```bash
sudo chown -R apache:apache /var/lib/teampass_secure
```

```bash
sudo chmod 750 /var/lib/teampass_secure
```

## 9. Ajustar permissões antes do wizard

```bash
sudo chown -R apache:apache /var/www/html/teampass
```

```bash
sudo find /var/www/html/teampass -type d -exec chmod 755 {} \;
```

```bash
sudo find /var/www/html/teampass -type f -exec chmod 644 {} \;
```

```bash
sudo chmod -R 775 /var/www/html/teampass/install
```

```bash
sudo chmod -R 775 /var/www/html/teampass/includes/config
```

```bash
sudo chmod -R 775 /var/www/html/teampass/includes/libraries/csrfp/libs
```

```bash
sudo chmod -R 775 /var/www/html/teampass/includes/libraries/csrfp/log
```

```bash
sudo chmod -R 775 /var/www/html/teampass/files
```

```bash
sudo chmod -R 775 /var/www/html/teampass/upload
```

## 10. Criar configuração do Apache

```bash
sudo vim /etc/httpd/conf.d/teampass.conf
```

Cole o conteúdo:

```apache
Alias /teampass /var/www/html/teampass

<Directory /var/www/html/teampass>
    Options FollowSymLinks
    AllowOverride All
    Require all granted
    DirectoryIndex index.php
</Directory>
```

Validar configuração:

```bash
sudo apachectl configtest
```

Reiniciar Apache e PHP-FPM:

```bash
sudo systemctl restart php-fpm
```

```bash
sudo systemctl restart httpd
```

## 11. Liberar firewall

```bash
sudo firewall-cmd --add-service=http --permanent
```

```bash
sudo firewall-cmd --reload
```

```bash
sudo firewall-cmd --list-all
```

## 12. Acessar o instalador web

Acesse no navegador:

```text
http://192.168.31.143/teampass/install/install.php
```

Na primeira tela, preencher:

```text
Absolute path to TeamPass folder:
/var/www/html/teampass

Full URL to TeamPass:
http://192.168.31.143/teampass

Absolute path to secure path:
/var/lib/teampass_secure
```

Na tela do banco, preencher:

```text
Database host:
localhost

Database name:
teampass

Database user:
teampass_user

Database password:
SENHA_FORTE_DO_BANCO

Database port:
3306

Table prefix:
teampass_
```

Na etapa do usuário administrador, criar:

```text
Usuário:
admin

Senha:
senha definida no wizard
```

Essa senha será usada depois para acessar a interface web.

## 13. Comando único para corrigir permissões se o wizard reclamar

Se aparecer erro de permissão durante a instalação, execute:

```bash
sudo bash -c 'mkdir -p /var/lib/teampass_secure && chown -R apache:apache /var/www/html/teampass /var/lib/teampass_secure && find /var/www/html/teampass -type d -exec chmod 755 {} \; && find /var/www/html/teampass -type f -exec chmod 644 {} \; && find /var/www/html/teampass -name ".htaccess" -exec chmod 644 {} \; && for d in /var/www/html/teampass/install /var/www/html/teampass/includes/config /var/www/html/teampass/includes/libraries/csrfp/libs /var/www/html/teampass/includes/libraries/csrfp/log /var/www/html/teampass/files /var/www/html/teampass/upload; do [ -d "$d" ] && chmod -R 775 "$d"; done && chmod -R 750 /var/lib/teampass_secure'
```

Reinicie os serviços:

```bash
sudo systemctl restart php-fpm httpd
```

Depois volte ao wizard e continue a instalação.

## 14. Finalizar após instalar com sucesso

Depois que o TeamPass abrir corretamente, remova o diretório de instalação:

```bash
sudo rm -rf /var/www/html/teampass/install
```

Ajuste permissões finais:

```bash
sudo chown -R apache:apache /var/www/html/teampass /var/lib/teampass_secure
```

```bash
sudo find /var/www/html/teampass -type d -exec chmod 755 {} \;
```

```bash
sudo find /var/www/html/teampass -type f -exec chmod 644 {} \;
```

```bash
sudo chmod -R 775 /var/www/html/teampass/files
```

```bash
sudo chmod -R 775 /var/www/html/teampass/upload
```

```bash
sudo chmod -R 775 /var/www/html/teampass/includes/libraries/csrfp/log
```

```bash
sudo chmod -R 750 /var/lib/teampass_secure
```

Reinicie os serviços:

```bash
sudo systemctl restart php-fpm
```

```bash
sudo systemctl restart httpd
```

## 15. Acessar o TeamPass

URL final:

```text
http://192.168.31.143/teampass
```

Credenciais:

```text
Usuário:
admin

Senha:
senha cadastrada no wizard de instalação
```

## 16. Comandos de validação

Validar HTTP local:

```bash
curl -I http://localhost/teampass
```

Validar logs do Apache:

```bash
sudo tail -n 100 /var/log/httpd/error_log
```

Validar logs do PHP-FPM:

```bash
sudo journalctl -u php-fpm -n 100 --no-pager
```

Validar logs do MariaDB:

```bash
sudo journalctl -u mariadb -n 100 --no-pager
```

Validar tabelas no banco:

```bash
mysql -u teampass_user -p teampass -e "SHOW TABLES;"
```

Validar arquivos principais de configuração:

```bash
sudo ls -la /var/www/html/teampass/includes/config
```

```bash
sudo ls -la /var/lib/teampass_secure
```

## 17. Backup básico

Criar diretório:

```bash
sudo mkdir -p /backup/teampass
```

Backup do banco:

```bash
sudo mysqldump -u root -p teampass > /backup/teampass/teampass_$(date +%F_%H%M).sql
```

Backup dos arquivos:

```bash
sudo tar -czf /backup/teampass/teampass_files_$(date +%F_%H%M).tar.gz /var/www/html/teampass /var/lib/teampass_secure
```

Listar backups:

```bash
sudo ls -lh /backup/teampass
```

## 18. Resumo final dos dados usados

```text
Versão TeamPass:
3.1.4.33

URL:
http://192.168.31.143/teampass

Diretório da aplicação:
/var/www/html/teampass

Secure path:
/var/lib/teampass_secure

Banco:
teampass

Usuário do banco:
teampass_user

Host do banco:
localhost

Porta do banco:
3306

Prefixo das tabelas:
teampass_

Usuário web:
admin

Senha web:
senha cadastrada no wizard
```

Criado por Jeferson Salles
LinkedIn: [https://www.linkedin.com/in/jmsalles/](https://www.linkedin.com/in/jmsalles/)
E-mail: [jefersonmattossalles@gmail.com](mailto:jefersonmattossalles@gmail.com)
GitHub: [https://github.com/jmsalles/](https://github.com/jmsalles/)
