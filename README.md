# Webte2 server setup

Nastavenie LEMP webového servera pre predmet webte2 FEI STU. Server je dostupný na verejnej IP adrese v tvare 147.175.98.XX, ktorá je dostupná iba cez VPN. Server má priradené aj doménové meno v tvare siteXX.webte.fei.stuba.sk . Znaky XX v doménovom tvare adresy sú nahradené posledným číslom z IP adresy (môžu to byť 2 alebo 3 číslice).

Pred samotnou inštaláciou je nutné byť pripojený k univerzitnej sieti buď fyzicky alebo cez VPN. Návod pre nastavenie VPN je dostupný na stránke [https://www.stuba.sk/sk/pracoviska/centrum-vypoctovej-techniky/cinnosti-a-sluzby/vzdialeny-vpn-pristup-do-siete-stu.html?page_id=3750](https://www.stuba.sk/sk/pracoviska/centrum-vypoctovej-techniky/cinnosti-a-sluzby/vzdialeny-vpn-pristup-do-siete-stu.html?page_id=3750).

V celom návode je potrebné kľúčové slová **username** a **password** nahrádzať vlastným prihlasovacím menom (login) a heslom, ktoré ste obdržali mailom.

## Softvér a verzie
- Ubuntu 22.04
- Nginx
- PHP 8.2
- MySQL 8
- PhpMyAdmin

## Pripojenie k serveru pomocou SSH

Cez program putty (windows) alebo priamo cez terminál (OSx/Linux) sa pripojiť k svojmu pridelenému serveru.
```sh
ssh username@147.175.98.XX
```
Po prompte zadať heslo.

## Update systému
Po prihlásení je nutné systém aktualizovať
```sh
sudo apt update
sudo apt dist-upgrade
```

Ak sa po upgrade objaví táto notifikácia

![nginx](https://raw.githubusercontent.com/katarina02/webte2-installation/main/img/package_configuration_1.png)

spravte reštart systému pomocou príkazu
```sh
sudo reboot
```
Reštart systému ukončí reláciu, po niekoľkých sekundách treba znova nadviazať [ssh spojenie](#pripojenie-k-serveru-pomocou-ssh).

V prípade, že sa počas inštalácie zobrazí okno podobné tomuto

![nginx](https://raw.githubusercontent.com/katarina02/webte2-installation/main/img/package_configuration_2.png)

nemeňte žiadne nastavenia, len stlačte Ok a pokračujte ďalej.

Pridanie repozitárov pre novšie verzie softvéru php a phpmyadmin
```sh
sudo add-apt-repository ppa:ondrej/php
sudo add-apt-repository ppa:phpmyadmin/ppa
sudo apt update
```
## Nginx
Inštalácia balíkov webserveru nginx, textového editora vim.
```sh
sudo apt install nginx vim
```
> Poznámka: Miesto editora vim môžete použiť aj iný editor, ako napríklad nano.

Po navštívení IP adresy by webový prehliadač mal zobrazovať

![nginx](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/nginx.png)

Pridanie usera do skupiny www-data.
```sh
sudo usermod -aG www-data $USER
```
Zmena sa prejaví až pri novej relácii, systém je možné rovno reštartovať.
```sh
sudo reboot
```
Reštart systému ukončí reláciu, po niekoľkých sekundách treba znova nadviazať [ssh spojenie](#pripojenie-k-serveru-pomocou-ssh).

### Kontrola pridania používateľa do skupiny (Nepovinné)

Po zadaní príkazu

```sh
groups
```
by výstup mal vyzerať
```sh
username sudo www-data
```

## MySQL
Inštalácia MySQL databázového serveru.
```sh
sudo apt install mysql-server
```
Kvôli spusteniu skriptu na zabezpečenie databázy je potrebné zmeniť spôsob prihlásenia pre root užívateľa.
```sh
sudo mysql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
exit
```
Spustenie skriptu na zabezpečenie databázy.
```sh
sudo mysql_secure_installation
```
Odpovede na otázky počas konfigurácie:
- no
- Change the password for root? - no
- Remove anonymous user? - yes
- Disallow root login remotely? - yes
- Remove test database and access to it? - no
- Reload privilege tables now? - yes

Spôsob prihlásenia pre root uzivateľa zmeníme naspäť.
```sh
mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED WITH auth_socket;
exit
```

Pripojenie ku MySQL konzole.
```sh
sudo mysql
```
Prompt sa zmení na ```mysql>```
Vytvorenie nového používateľa pre prístup a srpávu databáz.
```sh
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
```
Pridanie privilégií pre prácu s databázami.
```sh
GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost';
FLUSH PRIVILEGES;
```
Opustenie konzoly MySQL pomocou ```Ctrl + d``` alebo ```exit```.


### Kontrola pridania používateľa pre prístup k databáze (Nepovinné)

Prihlásenie sa do MySQL konzoly pod novým používateľom
```sh
mysql -u username -p
```

## PHP
Inštalácia PHP 8.2.

```sh
sudo apt install php-fpm
```

Odpoveďou na príkaz ```php -v``` by malo byť
```sh
PHP 8.2.2 (cli) (built: Feb 7 2023 11:28:53) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.2.2, Copyright (c) Zend Technologies
    with Zend OPcache v8.2.2, Copyright (c), by Zend Technologies
```

### Vytvorenie Virtual host konfigurácie pre URL
Reťazec **XX** nahradiť prideleným číslom podľa URL

```sh
sudo vim /etc/nginx/sites-available/siteXX.webte.fei.stuba.sk
```
> Poznámka: Editor vim ukončíte príkazom `:wq` ak chcete uložiť zmeny alebo `:q!` ak zmeny ukladať nechcete.

Do súboru vložiť obsah a zameniť režazec **XX** za posledný číselný segment priradenej IP adresy:

```sh
server {
       listen 80;
       listen [::]:80;

       server_name siteXX.webte.fei.stuba.sk;

       root /var/www/siteXX.webte.fei.stuba.sk;
       index index.html index.php;
       
       location / {
               try_files $uri $uri/ =404;
       }
       
       location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
       }
}
```
> Poznámka: Príkaz `:%s/XX/16/g` vo Vim nahradí všetky výskyty reťazca XX číslom 16.

Vytvorenie symbolického odkazu súboru.
```sh
sudo ln -s /etc/nginx/sites-available/siteXX.webte.fei.stuba.sk /etc/nginx/sites-enabled/
```

Po spustení príkazu ```sudo service nginx restart``` by mal webový prehliadač ukazovať  chybu 404.
### Vytvorenie adresára pre webový server
Skripty a súbory v tomto adresáre sa zobrazia po navštívení pridelenej domény.

Aby bolo možné vytvárať zápisy do adresára musí patriť skupine www-data a mať prístup na zápis pre skupinu.
```sh
cd /var
sudo chown -R www-data:www-data www/
sudo chmod g+w -R www/
```
Po zmene oprávnení je možné zapisovať do adresára ```/var/www``` bez sudo privilégií.
```sh
cd /var/www
mkdir siteXX.webte.fei.stuba.sk
cd siteXX.webte.fei.stuba.sk
vim index.php
```

Vytvorte jednoduchý PHP skript, napr.
```php
<?php
	echo "Hello world!"
?>
```
Po navštívení pridellenej URL by sa mala načítať prázdna stránka s textom _Hello world!_.

### SSL certifikát pre HTTPS

Stiahnuť súbory:
- webte.fei.stuba.sk-chain-cert.pem
- webte.fei.stuba.sk.key

Skopírovať ich do:
- /etc/ssl/certs/webte.fei.stuba.sk-chain-cert.pem;
- /etc/ssl/private/webte.fei.stuba.sk.key;

Zmeniť konfiguráciu Nginx v súbore ```/etc/sites-available/siteXX.webte.fei.stuba.sk```

```sh
server {
       listen 80;
       listen [::]:80;

       server_name siteXX.webte.fei.stuba.sk;

       rewrite ^ https://$server_name$request_uri? permanent;
}

server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name siteXX.webte.fei.stuba.sk;

        access_log /var/log/nginx/access.log;
        error_log  /var/log/nginx/error.log info;

        root /var/www/siteXX.webte.fei.stuba.sk;
        index index.php index.html;

        ssl on;
        ssl_certificate /etc/ssl/certs/webte.fei.stuba.sk-chain-cert.pem;
        ssl_certificate_key /etc/ssl/private/webte.fei.stuba.sk.key;

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        }
}
```

Reštartovať Nginx príkazom
```sh
sudo service nginx restart
```

Po navštívení pridelenej domény vo webovom prehliadači sa stránka načíta pomcou https protokolu
![nginx](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/ssl.png)

## PhpMyAdmin

Inštalácia GUI utility pre správu databázy cez prehliadač.

```sh
sudo apt install phpmyadmin
```

Po spustení inštalácie sa zobrazí séria okien s otázkami, nikde nič nevypĺňať, len stlačiť enter
![phpmyadmin_1](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/phpmyadmin_1.png)
![phpmyadmin_2](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/phpmyadmin_2.png)
![phpmyadmin_3](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/phpmyadmin_3.png)


Vytvoriť súbor ```/etc/nginx/snippets/phpmyadmin.conf``` a vložiť obsah:

```sh
location /phpmyadmin {
    root /usr/share/;
    index index.php index.html index.htm;
    location ~ ^/phpmyadmin/(.+\.php)$ {
        try_files $uri =404;
        root /usr/share/;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include /etc/nginx/fastcgi_params;
    }

    location ~* ^/phpmyadmin/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ {
        root /usr/share/;
    }
}
```

Do konfiguračného súboru ```/etc/nginx/sites-available/siteXX.webte.fei.stuba.sk ``` pridať riadok

```sh
include snippets/phpmyadmin.conf;
```

Finálny konfiguračný súbor bude vyzerať

```sh
server {
    listen 80;
    listen [::]:80;

    server_name site16.webte.fei.stuba.sk;

    rewrite ^ https://$server_name$request_uri? permanent;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name site16.webte.fei.stuba.sk;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log info;

    root /var/www/site16.webte.fei.stuba.sk;
    index index.php index.html;

    ssl on;
    ssl_certificate /etc/ssl/certs/webte.fei.stuba.sk-chain-cert.pem;
    ssl_certificate_key /etc/ssl/private/webte.fei.stuba.sk.key;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
    }

    include snippets/phpmyadmin.conf;
}

```

Po reštarte nginx serveru príkazom ```sudo service nginx restart``` otvoriť stránku [https://siteXX.webte.fei.stuba.sk/phpmyadmin](https://siteXX.webte.fei.stuba.sk/phpmyadmin). Úvodná obrazovka by mala vyzerať takto:

![phpmyadmin_5](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/phpmyadmin_5.png)

Po zadaní prihlasovacích údajov zo sekcie [MySQL](#mysql) by sa mala zobraziť táto aplikácia

![phpmyadmin_6](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/phpmyadmin_6.png)


## License

MIT
