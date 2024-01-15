# Instalación de una Arquitectura en 2 niveles con Ansible.

## Instalación de la pila Lamp con Ansible.

### Pero,¿De que consta una pila LAMP?
Muy simple, con esto describimos un sistema de infraestructura de internet, lo que estamos buscando es desplegar una serie de aplicaciones en la web, desde un unico sistema operativo, esto quiere decir que, buscamos desplegar aplicaciones en la web de forma cómoda y rápida ejecutando un único script, el cual hay que configurar previamente.

### 1. Que representa cada letra de la palabra --> LAMP.

#### L --> Linux (Sistema operativo).
#### A --> Apache (Servidor web).
#### M --> MySQL/MariaDB (Sistema gestor de base de datos).
#### P --> PHP (Lenguaje de programación).

### Con esto, buscamos hacer un despligue de aplicaciones.

# Pasos previos.
FICHERO INVENTARIO:
```
[frontend]
44.195.253.227 # IP pública del frontend.

[backend]
3.223.203.182 # Ip publica del backend.

# Son necesarios tenerlos para poder comunicar con ssh con ellos y manejarlos via ssh.

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/arquitectura_2niveles_ansible_wp/wordpress/labsuser.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'

#Esto permite que se puede conectar automaticamente mediante ssh a las maquinas y poder operar con ellas mediante los ficheros yml.#

```

Directorio Templates:

Contienen los fichero 000-default necesario para apache, y .htaccess para establecer correctamente la reescritura de contenido.

```
000-default:

ServerSignature Off
ServerTokens Prod

<VirtualHost *:80>
        DocumentRoot /var/www/html
        DirectoryIndex index.php index.html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Directory "/var/www/html">
            AllowOverride All
        </Directory>
</VirtualHost>
```

```
.htaccess:

# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPress
```

### Fichero con las variables.

```
installation_path: /var/www/html
db:
    name: wordpress_db
    user: wordpress_user
    password: wordpress_passwd
    MYSQL_PRIVATE: 172.31.86.216

wordpress:
    wordpress_title: "Sitio web de IAW"
    wordpress_admin_user: admin
    wordpress_admin_pass: admin
    wordpress_admin_email: demo@demo.es
    WORDPRESS_DB_HOST: 172.31.86.216
    WORDPRESS_HIDE_LOGIN: acceso

certbot:
    email: admin_wordpress@email.es
    domain: hipherion1219.ddns.net

phpmyadmin:
    user: pma_user
    password: pma_password
    db_name: db_name
```

Contienen las variables que se irán asignando en los ficheros yml una vez se vayan ejecutando.

``Y por ultimo un fichero .pem el cual no voy a mostrar el contenido por seguridad.``

# Fichero de instalación de la pila lamp en el frontend.

```python
- name: Playbook para instalar la pila LAMP en el FrontEnd
  hosts: frontend
  become: yes

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present
      
    - name: Copiar el archivo de configuración de Apache
      template: # Si se hace con template se modifica el contenido con variables y además manda el contenido a la máquina destino.
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/
        mode: 0755

    - name: Instalar PHP y los módulos necesarios
      apt: 
        name:
          - php
          - php-mysql
          - libapache2-mod-php
        state: present

    - name: Habilitar el módulo rewrite de Apache
      apache2_module:
        name: rewrite
        state: present

    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
```

Lo vamos a dividir por partes para explicarlo al detalle.

### Con esto actualizamos los repositorios.
    
   ```python

     name: Actualizar los repositorios
      apt:
        update_cache: yes
    
```
Empleando el modulo ``apt``.

### Instalamos apache.
    
```
    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present
```
Empleando ``state: present``.

### Copiamos el archivo de configuración al host remoto.
```
    - name: Copiar el archivo de configuración de Apache
      template: # Si se hace con template se modifica el contenido con variables y además manda el contenido a la máquina destino.
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/
        mode: 0755
```
### Instalamos los módulos necesarios de apache.
```python
    - name: Instalar PHP y los módulos necesarios
      apt: 
        name:
          - php
          - php-mysql
          - libapache2-mod-php
        state: present
```

### Habilitamos el modulo rewrite de apache.
```
    - name: Habilitar el módulo rewrite de Apache
      apache2_module:
        name: rewrite
        state: present
```
Necesario para evitar errores de ``reescritura``.

### Reiniciamos el servicio.
```python
    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
```

## Método de ejecutarlo.

```
ansible-playbook -i /home/ubuntu/arquitectura_2niveles_ansible_wp/wordpress/inventory/inventory install_lamp_fronted.yml
```

Independientemente del nuevo del fichero ``.yml`` ese comando vale para todo.

## Fichero de instalación de la pila lamp para el backend.

Con esto damos inicio fichero de automatización para el backend
```
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes
```
Cogiendo como host el backend del inventario.

### Llamamos a las variables.
```
  vars_files:
    - ../vars/variables.yml
```
### Actualizamos repositorios.
```
  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes
```

### Instalamos mysql-server.
```
    - name: Instalar el sistema gestor de bases de datos MySQL
      apt:
        name: mysql-server
        state: present
```

### Instalamos python pip3 para conectar equipos con bases de datos.

```
    - name: Instalamos el gestor de paquetes de Python pip3
      apt: 
        name: python3-pip
        state: present
```

### Instalamos el modulo de pymysql.
```
    - name: Instalamos el módulo de pymysql
      pip:
        name: pymysql
        state: present
```

## Creación de la base de datos.

### Creamos la base de datos para wordpress.
```
    - name: Crear una base de datos
      mysql_db:
        name: "{{ db.name }}"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 
```

Estableciendo como puerto el de mysql y usando la variable para insertar el nombre.

### Creamos el usuario de la base de datos.
```
    - name: Crear el usuario de la base de datos
      mysql_user:         
        name: "{{ db.user }}"
        password: "{{ db.password }}"
        priv: "{{ db.name }}.*:ALL"
        host: "%"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 
```
Estableciendo variables para ello, sus privilegios, desde donde se puede conectar, etc.

### Configuramos el servidor que recibe peticiones sql en este caso la ip privada del backend.
```
    - name: Configuramos MySQL para permitir conexiones desde cualquier interfaz
      replace:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: 127.0.0.1
        replace: "{{ db.MYSQL_PRIVATE }}" # Ip privada del backend.
```
Con este modulo de ansible podemos reemplazar contenido facilmente.

### Reiniciamos el servicio de base de datos.
```
    - name: Reiniciamos el servicio de base de datos
      service:
        name: mysql
        state: restarted
```
Empleando el módulo ``restarted``.

# Cifrado seguro en las peticiones HTTP

Aquí voy a mostrar el contenido del fichero config_https.yml para automatizar el proceso de un certificado validado por una autoridad certificadora.


### Desintalamos previamente certbot.
```
tasks:

    - name: Desinstalar instalaciones previas de Certbot
      snap:
        name: certbot
        state: absent
```
Empleando ``state: absent``.

### Instalamos certbot empleando snap.
```
    - name: Instalar Certbot con snap
      snap:
       name: certbot
       classic: yes
       state: present
```      

Para instalar algo con snap necesitamos, ``classic: yes``.

### Creamos un enlace simbolico a bin para que certbot actúe como un comando.

```
    - name: Crear un alias para el comando certbot
      command: ln -s -f /snap/bin/certbot /usr/bin/certbot
```

Necesario para ejecutarlo como si un comando se tratara.

### Solicitamos y configuramos el certificado a Let´s Encrypt para securizar http.
```
    - name: Solicitar y configurar certificado SSL/TLS a Let's Encrypt con certbot
      command:
        certbot --apache \
        -m {{ certbot.email }} \
        --agree-tos \
        --no-eff-email \
        --non-interactive \
        -d {{ certbot.domain }}
```

Incluyendo las variables ``certbot.email`` y ``certbot.domain`` las cuales contienen el correo y el dominio al que se va a poder realizar búsquedas via internet, insertando el dominio en el buscador.

```python
certbot:
    email: admin_wordpress@email.es
    domain: hipherion1219.ddns.net
```

Estas son las variables con las que trabaja el fichero .yml de certbot.

# Explicación del fichero de despligue de wordpress.

Aquí se va a explicar las instrucciones del despliegue automatizado de wordpress.

```python
name: Playbook para hacer el deploy de la aplicación web PrestaShop
  hosts: frontend
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Borrar archivos previos de wordpress.
      file:
        path: /tmp/wp-cli.phar
        state: absent

    - name: Descargar el código fuente de Wordpress.
      get_url:
        url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
        dest: /tmp
        mode: 0755

    - name: Movemos el fichero a bin para incluirlo en la lista de comandos.
      command: mv /tmp/wp-cli.phar /usr/local/bin/wp 

    - name: Eliminamos instalaciones previas de wordpress.
      shell: rm -rf /var/www/html/*

    - name: Descargamos el codigo fuente de wordpress en /var/www/html 
      command: wp core download --path=/var/www/html --locale=es_ES --allow-root
    
    - name: Creación del archivo wp-config
      command: wp config create \
              --dbname="{{ db.name }}" \
              --dbuser="{{ db.user }}"  \
              --dbpass="{{ db.password }}" \
              --dbhost="{{ db.MYSQL_PRIVATE }}" \
              --path=/var/www/html \
              --allow-root

    - name: Instalo wordpress
      command: wp core install \
                --url="{{ certbot.domain }}" \
                --title="{{ wordpress.wordpress_title }}" \
                --admin_user="{{ wordpress.wordpress_admin_user }}" \
                --admin_password="{{ wordpress.wordpress_admin_pass }}" \
                --admin_email="{{ wordpress.wordpress_admin_email }}" \
                --path=/var/www/html \
                --allow-root
                
    - name: Actualizamos el core de wordpress.
      command: wp core update --path=/var/www/html --allow-root

    - name: Instalamos un tema para wordpress
      command: wp theme install sydney --activate --path=/var/www/html --allow-root
   
    - name: Instalamos un plugin para wordpress. 
      command: wp plugin install bbpress --activate --path=/var/www/html --allow-root
   
    - name: Instalación del plugin para ocultar login.
      command: wp plugin install wps-hide-login --activate --path=/var/www/html --allow-root

    - name: Habilitamos reescritura para mejorar el seo.
      command:  wp rewrite structure '/%postname%/' \
               --path=/var/www/html \
               --allow-root
               
    - name: Actualizar el plugin de wordpress para cambiar el fichero de la página de logín a otro y no aparezca en el navegador.            
      command: wp option update whl_page "{{ wordpress.WORDPRESS_HIDE_LOGIN }}" --path=/var/www/html --allow-root
    
    - name: Copiar el archivo .htacces
      template: # Si se hace con template se modifica el contenido con variables y además manda el contenido a la máquina destino.
         src: ../templates/.htaccess
         dest: /var/www/html/
         mode: 0755

    - name: Cambiar el propietario de html. 
      file: 
        dest: /var/www/html
        owner: www-data
        group: www-data
        recurse: yes
```