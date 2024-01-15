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