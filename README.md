
# Linux Server Configuration

Configure Linux web server with Amazon Lightsail and deploy python web application.

- IP : 18.191.140.73
- HostName :  ec2-18-191-140-73.us-east-2.compute.amazonaws.com
- SSH port : 2200


## server configuration

### 1. Create instance from AWS

- goto Amazon Lightsail webpage
- create instance : choose option Linux platform, os only/ubuntu
- download deault private ssh key from account page

### 2. Connect to server with terminal
- open terminal
- place downloaded .pem file into ~/.ssh `$cp catalog.pem ~/.ssh`
- change the access permissions of .pem file `$ chmod 600 ~/.ssh/catalog.pem`
- connect to server with ssh `$ ssh -i ~/.ssh/catalog.pem ubuntu@18.191.140.73`

### 3. Add user grader
- add user grader `$ sudo adduser grader`
- set password for user 'grader'
- give user 'grader' sudo permission 
- open file `$sudo vi /etc/sudoers.d/grader`
- paste this and save file `grader ALL=(ALL) NOPASSWD:ALL`
- update 
```
$sudo apt-get update
$sudo apt-get upgrade
```
- install finger to checek user grader `$ sudo apt-get install finger`
- check user 'grader' `$finger grader`

### 4. Create SSH key

- create key pair from your local terminal `$ ssh-keygen -f ~/.ssh/catalog.rsa` 
- open public key  `$ cat ~/.ssh/catalog.rsa.pub`

- copy public key to server under /home/grader/.ssh
```
$ cd /home/grader
$ mkdir .ssh
$ sudo vi /home/grader/.ssh/authorized_keys
```
- change the permission of file and folder
```
$ sudo chmod 700 /home/grader/.ssh
$ sudo chmod 644 /home/grader/.ssh/authorized_keys 
```
- change the owner of .ssh
`$ sudo chown -R grader:grader /home/grader/.ssh`
- restart ssh `$ sudo service ssh restart`
- open another terminal and test ssh connection with newly created key as user grader
 `$ ssh -i ~/.ssh/catalog.rsa grader@18.191.140.73`


### 5. Change port & Configure Filewall
- edit file sshd_config `$ sudo nano /etc/ssh/sshd_config`
- set `PasswordAuthentication` to no
- set `port` 22 to 2200
- set `PermitRootLogin` to no
- restart ssh `$ sudo service ssh restart`
- from now when connect to ssh, use `-p 2200` option  `$ ssh -i ~/.ssh/catalog.rsa grader@18.191.140.73 -p 2200`

- set ufw
```
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw enable
```
- check current ufw setting `$ sudo ufw status`


## Application Deployment


### 1. install apache
- install apache `$ sudo apt-get install apache2`
- to check installation, type public ip in the browser  
Browser will display the apache ubuntu default page


### 2. install mod_wsgi and enable
WSGI (Web Server Gateway Interface) is an interface between web servers and web apps for python.  
Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications.  
- install mod_wsgi `$ sudo apt-get install libapache2-mod-wsgi python-dev`  
- enable mod_wsgi `$ sudo a2enmod wsgi `  

### 3. install and creating Flask App

- create and place app in the /var/www directory
- move to the /var/www directory `$ cd /var/www `
```
$ sudo mkdir catalog
$ cd catalog
$ sudo mkdir catalog
$ cd catalog
$ sudo mkdir static templates 
```

now directory structure look like this:  
|----catalog  
|---------catalog  
|--------------static  
|--------------templates 

- create the __init__.py file that will contain the flask application logic.
`$ sudo vi __init__.py `
```
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():
    return "Hello, world!"
if __name__ == "__main__":
    app.run()
```
Save and close the file.  

- create a virtual environment for flask application  
Setting up a virtual environment will keep the application and its dependencies isolated from the main system.  
- install pip `$ sudo apt-get install python-pip`
- use pip to install virtualenv and Flask then activate virtual enviroment
```
 $ sudo pip install virtualenv
 $ sudo virtualenv venv
 $ source venv/bin/activate 
 $ sudo pip install Flask 
```

To test if the installation is successful and app is running, run \_\_init\_\_.py
`$ sudo python __init__.py `

- It should display “Running on http://localhost:5000/” or "Running on http://127.0.0.1:5000/". 
If you see this message, you have successfully configured the app.

- To deactivate the environment `$ deactivate`

### 4. Configure and enable virtual host

- create VirtualHost files, name it 'catalog.conf'   `$ sudo vi /etc/apache2/sites-available/catalog.conf`
- in the '.conf' file paste below
```
<VirtualHost *:80>
        ServerName 18.191.140.73
        ServerAdmin admin@mywebsite.com
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog/catalog/>
            Order allow,deny
            Allow from all
        </Directory>
        Alias /static /var/www/catalog/catalog/static
        <Directory /var/www/catalog/catalog/static/>
            Order allow,deny
            Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
- Enable the virtual host `$ sudo a2ensite catalog`


### 5. Create .wsgi file

- Apache uses the .wsgi file to serve the Flask app. 
- Move to the /var/www/catalog directory and create a file named catalog.wsgi
```
$ cd /var/www/catalog
$ sudo nano catalog.wsgi 
```

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from FlaskApp import app as application
application.secret_key = 'super_secret_key'
```

- now directory structure look like this  
|--------catalog  
|----------------catalog  
|-----------------------static  
|-----------------------templates  
|-----------------------venv  
|-----------------------\_\_init\_\_.py  
|----------------catalog.wsgi  


### 6. Clone the Github Repository
- install git `$ sudo apt-get install git`
- clone source from git repository `$ git clone [repository url] catalog`
- place 'catalog' directory under /www/var/catalog/catalog 
- delete \_\_init\_\_.py then rename application.py to \_\_init\_\_.py

now directory look like this:  
|----catalog  
|---------catalog  
|--------------static  
|--------------templates  
|--------------venv  
|--------------\_\_init\_\_.py  
|--------------client_secret.json  
|--------------fb_client_secret.json  
|--------------model.py  
|---------catalog.wsgi  

### 7. Install other pakages
```
$ source venv/bin/activate
$ pip install httplib2
$ pip install requests
$ sudo pip install --upgrade oauth2client
$ sudo pip install sqlalchemy
$ pip install Flask-SQLAlchemy
$ sudo pip install flask-seasurf
```
 
### 8. Install Postgresql

- install psycopg `$ sudo apt-get install python-psycopg2`
- install PostgreSQL `$ sudo apt-get install postgresql postgresql-contrib`
- update \_\_init\_\_.py, model.py

from `sqlite://catalog.db` to 
`create_engine('postgresql://catalog:catalog-pw@localhost/catalog')`

connect to psql
```
$ sudo su - postgres
$ psql
```

Create user catalog with password

```
postgres=# CREATE USER catalog WITH PASSWORD [password];
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog with OWNER catalog;
postgres=# \c catalog
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
catalog=# \q
$ exit
```

- restart postgresql: `$ sudo service postgresql restart`

### 9. Run the Application

- update \_\_init\_\_.py

from `CLIENT_ID = json.loads( open('client_secrets.json', 'r').read())['web']['client_id']` to
```
open('/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']
```

from `app_id = json.loads(open('fb_client_secret.json', 'r').read())['web']['app_id']` to
```
    app_id = json.loads(open('/var/www/catalog/catalog/fb_client_secret.json', 'r')
                        .read())['web']['app_id']
```

- restart apache `$ sudo service apache2 restart `
- run application `$ python /var/www/catalog/catalog/__init__.py`
- open browser then put public ip address
- check your errors in /var/log/apache2/error.log files. `$ tail -10 /var/log/apache2/error.log` 

## Resource

- https://github.com/anumsh/Linux-Server-Configuration

- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
