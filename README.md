

# Installieren von Smartstore Core auf CentOS 8.5

## Voraussetzungen

 - Smartstore Core für Linux X64
 - CentOS 8.5
 - Nicht root Benutzer mit sudo-Rechten

## Ablauf

 - NGINX Reverse Proxy installieren
 - FTP-Server installieren
 - Firewall anpassen
 - MySQL installieren
 - Smartstore installieren
 - Tipps & Tricks

## NGINX installieren

 ### NGINX installieren
   ```bash
   sudo dnf install nginx
   ```

 ### NGINX starten, stoppen, neu starten und Konfiguration neu laden:
   ```bash
sudo systemctl start nginx
```
 ```bash
sudo systemctl stop nginx
  ```
```bash
sudo systemctl restart nginx
 ```
 ```bash
sudo systemctl reload nginx
   ```

 ### Die installierte Version von NGINX prüfen:
   ```bash
sudo nginx -v
   ```
 ### Die NGINX-Konfiguration auf Fehler prüfen:
 ```bash
sudo nginx -t
  ```

### NGINX als Dienst beim Systemstart starten:
   ```bash
sudo systemctl enable nginx
  ```

### Firewallregeln für HTTP und HTTPS anpassen:
   ```bash
sudo firewall-cmd --permanent --zone=public --add-service=https --add-service=http
  ```
   ```bash
sudo firewall-cmd --reload
  ```


## FTP-Server installieren und konfigurieren
### Installation von ```vsftp```
   ```bash
sudo dnf install vsftpd
  ```
```vsftp``` als Dienst aktivieren:
   ```bash
sudo systemctl enable vsftpd --now
  ```
 Den Status prüfen:
   ```bash
sudo systemctl status vsftpd
  ```

### Konfiguration
Die Konfigurationsdatei im Editor öffnen:
   ```bash
sudo nano /etc/vsftpd/vsftpd.conf
  ```

 - FTP Zugriff
	Anonymen FTP Zugriff ausschließen und lokalen Benutzern Zugriff gewähren:
	```
	anonymous_enable=NO
    local_enable=YES
	  ```

 - Uploads aktivieren
	```
	write_enable=YES
	```
	
 - FTP-Benutzer im ```home```-Ordner einschließen
	 ```
	chroot_local_user=YES
	allow_writeable_chroot=YES
	```
- Ports für Passive FTP-Verbindungen angeben
	```
	pasv_min_port=30000
	pasv_max_port=31000
	```

- ```vsftp```-Dienst neu starten
	Nachdem die Datei gespeichert wurde, den Dienst neu starten:
   ```bash
	sudo systemctl restart vsftpd
  ```

### Firewall für ```vsftp```öffnen
Port 21, 20 und 30000-31000 (Bereich des passiven Ports) öffnen:
   ```bash
sudo firewall-cmd --permanent --add-port=20-21/tcp
sudo firewall-cmd --permanent --add-port=30000-31000/tcp
  ```


Firewall Regeln neu laden:
   ```bash
sudo firewall-cmd --reload
  ```

----


### FTP Benutzer erstellen
Für den FTP Zugriff ein neuer Benutzer angelegt oder ein vorhandener Benutzer genutzt werden. Wenn ein vorhandener Benutzer genutzt werden soll, kann der folgende Schritt übersprungen werden.

Einen neuen Benutzer anlegen und das Kennwort setzen:
   ```bash
sudo adduser newftpuser
sudo passwd newftpuser
  ```

Einen Benutzer zur Liste der erlaubten FTP-Benutzer hinzufügen:
   ```bash
echo "newftpuser" | sudo tee -a /etc/vsftpd/user_list
  ```
  
FTP-Verzeichnisbaum erstellen und die richtigen Rechte setzen:
   ```bash
sudo mkdir -p /home/newftpuser/ftp/upload
sudo chmod 550 /home/newftpuser/ftp
sudo chmod 750 /home/newftpuser/ftp/upload
sudo chown -R newftpuser: /home/newftpuser/ftp
  ```

### Bei Bedarf: Shell-Zugang für FTP-Benutzer sperren
```bash
echo -e '#!/bin/sh\necho "This account is limited to FTP access only."' | sudo tee -a  /bin/ftponly
sudo chmod a+x /bin/ftponly
echo "/bin/ftponly" | sudo tee -a /etc/shells
sudo usermod newftpuser -s /bin/ftponly
  ```


  
 ## NGINX einrichten
 ### Standard NGINX-Seite aufrufen
 - IP-Adresse herausfinden, wenn unbekannt
    ```bash
   hostname -I
   ```
- Im Browser NGINX-Startseite per IP aufrufen
	 ```bash
	http://ip-adresse
	```
- Es wird die Standard-Landingpage für NGINX angezeigt

	![NGINX-Landingspage](https://www.smartstore.com/news/images/Qs7PlUtvga.png)

### NGINX als Reverse-Proxy konfigurieren
Auf CentOS 8 ist der Ordner ```/usr/share/nginx/html``` als Standard-WWW-Ordner konfiguriert. Wir werden diesen ORdner nicht nutzen und erstellen stattdessen einen Ordner unter ```/var/www```.
Neuen Ordner für Smartstore erstellen:
 ```bash
sudo mkdir -p /var/www/smartstore/
```
Als nächstes passen wir die Eigentumsrechte an:
 ```bash
sudo chown -R $USER:$USER /var/www/smartstore
```
>Hinweis: $USER ist eine Umgebungsvariable

Damit Nginx den Inhalt des Ordners bereitstellen kann, muss ein Serverblock konfiguriert werden. Dazu wird eine neue Konfigurations-Datei erstellt:
 ```bash
sudo nano /etc/nginx/conf.d/smartstore.conf
```


Den folgenden Codeausschnitt in die Datei einfügen:
	 ```bash
	/etc/nginx/sites-available/default
	```
```bash
server {
	listen        80;
	server_name   example.com *.example.com;
	location / {
				proxy_pass         http://127.0.0.1:5000;
				proxy_http_version 1.1;
				proxy_set_header   Upgrade $http_upgrade;
				proxy_set_header   Connection keep-alive;
				proxy_set_header   Host $host;
				proxy_cache_bypass $http_upgrade;
				proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
				proxy_set_header   X-Forwarded-Proto $scheme;
			    }
	}		    
```

> ** Hinweis**: Wenn noch keine Domain vorhanden ist, kann auch eine IP-Adresse eingetragen werden:```server_name 172.16.1.17;```

Hiernach muss die NGINX-Konfiguration neu geladen werden:
```bash
sudo systemctl reload nginx
sudo systemctl restart nginx
```
Mit diesem Befehl kann die NGINX-Konfiguration geprüft werden:
```bash
sudo nginx -t
```
todo
todo
todo


## MySQL installieren
### Installieren von MySQL

Das Paket ```mysql-server``` und seine Abhängigkeiten installieren
   ```bash
sudo dnf install mysql-server
   ```
Hiermit ist MySQL installiert, aber nicht konfiguriert.

MySQL-Dienst starten:
   ```bash
sudo systemctl start mysqld.service
   ```

MySQL-Dienst-Status anzeigen:
   ```bash
sudo systemctl status mysqld
   ```

MySQL-Dienst zum Systemstart hinzufügen:
   ```bash
sudo systemctl enable mysqld
   ```

### Sichern von MySQL

MySQL enthält ein Sicherheitsscript, mit dem die Sicherheit von MySQL verbessert werden kann.
Sicherheitsskript ausführen:
   ```bash
   sudo mysql_secure_installation
   ```
> Bitte treffen Sie eine, Ihren Bedürfnissen entsprechende Auswahl.

Um die Benutzerauthentifizierung- und Berechtigungen anzupassen MySQL-Eingabeaufforderung öffnen:
   ```bash
   mysql -u root -p
   ```
Authentifizierungsverfahren prüfen:
```bash
SELECT user,authentication_string,plugin,host FROM mysql.user;
   ```
Wird der ```root``` Benutzer über das ```auth-socket```-Plugin authentifiziert, muss das ```root```-Konto umkonfiguriert werden. Mit diesem Befehl wird das vorherige ```root```-Passwort geändert. Es sollte ein starkes Passwort gewählt werden (```password``` ersetzen durch eigenes).
```bash
   ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password';
   ```
Berechtigungstabellen neu laden:
```bash
FLUSH PRIVILEGES;
   ```
MySQL-Shell verlassen:
```bash
exit
   ```
Einen dedizierten MySQL-Benutzer für die Nutzung mit Smartstore erstellen:
```bash
mysql -u root -p
   ```
```bash
CREATE USER 'smartstore'@'localhost' IDENTIFIED BY 'password';
   ```
> ```smartstore``` und ```password``` nach belieben ändern

Benutzerberechtigungen erteilen:
```bash
GRANT ALL PRIVILEGES ON *.* TO 'smartstore'@'localhost' WITH GRANT OPTION;
   ```
MySQL-Shell verlassen:
```bash
exit
   ```

## Smartstore installieren
### Dateien übertragen
Die Dateien aus dem Release per FTP auf den Debian-Server in den Ordner
   ```bash
/var/www/smartstore
``` 
übertragen.
      	
### App als Dienst einrichten
Erstellen einer Dienstdefinitionsdatei für ```systemd```:
```bash
sudo nano /etc/systemd/system/kestrel-smartstore.service
``` 
Folgenden Code-Ausschnitt einfügen und speichern:
> Bitte die Hinweise unter dem Codeblock beachten!
```bash
[Unit]
Description=Smartstore Core Web App running on Linux

[Service]
WorkingDirectory=/var/www/html
ExecStart=/usr/bin/dotnet /var/www/html/Smartstore.Web.dll
Restart=always
#Restart service after 10 seconds if dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=smartstore-core
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
``` 
> **Hinweis**: Pfade in ```WorkingDirectory``` und ```ExecStart``` ggf. anpassen.

> **Wichtig**: 
> Code bei **frameworkabhängiger Bereitstellung**:
>```bash
>ExecStart=/usr/bin/dotnet /var/www/html/Smartstore.Web.dll
>```
> Code bei **eigenständiger Bereitstellung**:
>```bash
>ExecStart=/var/www/html/Smartstore.Web
>```

### Dienst aktivieren und starten
Dienst aktivieren:
```bash
sudo systemctl enable kestrel-smartstore.service
``` 
Dienst starten:
```bash
sudo systemctl start kestrel-smartstore.service
``` 
Dienst stoppen:
```bash
sudo systemctl stop kestrel-smartstore.service
```

### ```wkhtmltopdf``` installieren
```bash
sudo apt-get update
```
```bash
sudo apt-get -y install wkhtmltopdf
```

### Festlegen der Ordnerberechtigungen
Eigenen Benutzer als Besitzer des Website-Ordners mit vollen Lese-, Schreib- und Ausführungsrechten setzen:

> **Hinweis:** ```smartstore``` ist beispielhaft der eigene Benutzer
```bash
chown -R smartstore /var/www/html/
```
Webserver als Gruppen-Besitzer setzen:
```bash
chgrp -R www-data /var/www/html/
```
Rekursiv für alle Dateien und Ordner Lese-, Schreib- und Ausführungsrechte für den Besitzer, Lese- und Ausführungsrechte für den Gruppen-Besitzer und keine Rechte für andere festlegen:

```bash
chmod -R 750 /var/www/html/
```
Gruppenbesitz auf neue Dateien und Ordner vererben:

```bash
chmod g+s /var/www/html/
```
Spezielle Ordner rekursiv mit Schreibrechten für Webserver versehen:
```bash
chmod -R g+w /var/www/html/App_Data
```
```bash
chmod -R g+w /var/www/html/Modules
```

### Smartstore installieren
Die Website per IP-Adresse oder Domainname aufrufen und die erforderlichen Daten eingeben.

![Startseite der Installation](https://www.smartstore.com/news/images/smartstore_installation_de_640px.png)

> Der MySQL-Server ist per localhost erreichbar. Als Anmeldename für die Datenbank ist der für diese Installation eigens angelegte MySQL-Benutzer zu verwenden

Nachdem alle erforderlichen Daten eingegeben wurden, wird die Installation per Klick auf **Installieren** gestartet.
Nach der Fertigstellung der Installation erscheint die Startseite mit den Demo-Daten:

![Startseite](https://www.smartstore.com/news/images/smartstore_core_Startseite-640px.png)

### Tipps & Tricks
Um die **maximale Dateiuploadgröße** zu  ändern wird die Datei ```/etc/nginx/nginx.conf``` mit einem Editor geöffnet.
Die Einstellung kann an zwei Stellen vorgenommen werden:

Im Http-Block: Einstellung gilt für alle virtuellen Hosts.
```bash
http {
    ...
    client_max_body_size 100M;
}
```

Im Server-Block: Einstellung gilt nur für diese spezielle Site/App.
```bash
server {
    ...
    client_max_body_size 100M;
}
```


to be continued...


 
 



