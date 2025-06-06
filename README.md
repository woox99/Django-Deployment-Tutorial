## How to permanently deploy django app on AWS(EC2) with static files for free

This is a walk-through tutorial on deploying a django application that uses SQLite database (with CSS and JS files). This deployment method will keep the server running on the AWS EC2 instance even after closing the instance.

This tutorial is NOT for you if:
* Your application uses file storage. (If the user is able to upload any type of file on the app. Example: profile picture upload)
* Your application uses a different database other than django's built in database SQLite
* Your application does NOT have any CSS or JS files (static files)

## What you will need:
* Github repo with your django project
* AWS account (free tier)

## Tutorial video:
Follow this video on youtube and use this README to copy the commands.
<a href="https://www.youtube.com/watch?v=U22oG_HO61s&t=3s" target="_blank">Tutorial Video</a>

## Launch AWS EC2 instance
* Name the instance (typically the same name as the project and repo)
* Select ```Ubuntu``` AMI
* Select ```Ubuntu Server 22.04 LTS``` server (free tier)

#### Instance Type
* Select ```t2.micro``` instance type (free tier)

#### Key Pair
* Select ```Proceed without a key pair``` for Key pair

#### Security Group
These settings will allow anyone with the link to your app to access your website. If you do not want anybody else to be able to access your application then you'll need to configure these accordingly.

* Select ```Allow SSH traffic``` from ```Anywhere 0.0.0.0/0```
* Select ```Allow HTTPS traffic from the internet```
* Select ```Allow HTTP traffic from the internet```

## django deployment:

Once connected to your ubuntu instance follow these commands precisely. Good Luck!

### Update instance:
``` 
sudo apt-get update 
```

### Upgrade Instance:

```
sudo apt-get upgrade
```

### Install pip package manager:
```
sudo apt-get install python3-venv
```

### Install virtual environment:
```
python3 -m venv env
```

### Activate virtual environment:
```
source env/bin/activate
```

### Install Django:
```
pip3 install django
```

### Clone repo from github:
* Make sure your repo is set to public on github
* Replace ```'HTTPS github address'``` with the address of your repo
```
git clone 'HTTPS github address'
```
* If it doesn't work, double check that you're including ```.git``` at the end of the address

### Edit ```settings.py``` file:
Your project folder (name of repo) will contain a file ```settings.py```. We are going to edit this file within our instance using Vim (or you can use nano). 

Reference the video if you are unfamilar with how to make changes to a file using Vim or nano.

* Change directory to your ```project``` directory

* Open ```settings.py``` with Vim:

```
sudo vim settings.py
```
This should open ```settings.py``` file within the instance. If it doesn't then double check you're in the correct directory


* Add this line near the top:
```import os```

* Replace this line:
```DEBUG = True```
with this line:
```DUBUG = False```

* Replace this line:
```ALLOWED_HOSTS = []```
with this line:
```ALLOWED_HOSTS = ['*']```

* Change Database to MySQL (if you're using mysql db)
```
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        'NAME': 'db_name', // Keep as string
        'USER': 'db_username', // Keep as string
        'PASSWORD': 'db_password', // Keep as string
        'HOST': 'localhost', // Keep as string
        'PORT': 3306,
    }
}
```

* Add this line: ```STATIC_ROOT = os.path.join(BASE_DIR, 'static')```
* Now save and exit: ```Esc``` -> ```:wq``` -> ```Enter```

### Install dependencies:
Now is a good time to install any dependencies or requirements your application may have. For example my tutorial application in the video uses ```bcrypt``` so I'm installing that now.

```
pip install -r requirements.txt
```

### Install MySQL:
```
sudo apt-get install mysql-server     
// answer yes
sudo apt-get update
sudo apt update
sudo apt install pkg-config python3-dev libmysqlclient-dev default-libmysqlclient-dev build-essential
pip install mysqlclient
```

enter the same password as MySQL on your computer when prompted
```
sudo mysql -uroot -p
 ```

Replace ```your_mysql_password```

 ```
 mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_mysql_password';
 mysql> FLUSH PRIVILEGES;
 mysql> quit
 ```

 ```
sudo mysql_secure_installation
```


```
ANSWER Y/N IN THIS ORDER:
NO
NO 
YES
YES
YES
YES
```

enter password when prompted
```
mysql -uroot -p
```

Now, export mysql db file with  `create scheme`.

Open db file in editor and copy/paste the file into the mysql> command line.

Verify that the database was created with:
```
mysql> SHOW DATABASES;
mysql> exit
```


### Install Nginx:
We are using Nginx for our web server. We will also be using Nginx to serve our static CSS and/or JS files later.

```
sudo apt-get install -y nginx
```

### Install gunicorn:
We are using gunicorn (Green Unicorn) for our gateway interface. I do not know what it does or how it works, however, I do know that we need it
```
pip install gunicorn
```

### Collect static files
We are using Nginx to server our static files, so we need to run this command so Nginx will know where to find them.
* Navigate to the directory that contains a file ```manage.py```
* Run this collect static
```
python manage.py collectstatic
```
* You should see a message that your static files have been collected. You should also see a file path. **This path is import soon so save it somewhere for now**
    * The file path will typically be ```/home/ubuntu/name_of_project/name_of_static_folder/```
* If you do not see this then check if you're in the correct directory

### Install supervisor:
We want our web server to keep running even if we close out of our instance, therefore, we'll install supervisor to handle this:
```
sudo apt-get install supervisor
```

### Create ```gunicorn.conf``` file:
* Change directory to our ```supervisor``` directory:
```
cd /etc/supervisor/conf.d
```

* Create ```gunicorn.conf``` file:
```
sudo touch gunicorn.conf
```

* Open the newly created file in Vim (or nano):
```
sudo vim gunicorn.conf
```

* Copy the ```gunicorn.conf``` file from this tutorial repo **WITHOUT THE COMMENTS**:

* **Anywhere the file says ```'name'```, replace it with the name of your project (repo name) WITHOUT the apostrophes.**

* Now save and exit the file: ```Esc``` -> ```:wq``` -> ```Enter```

### Create log directory:
Let's setup supervisor that we installed earlier.
* Run these commands in order:
```
sudo mkdir /var/log/gunicorn
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status
```

* Check the feedback message after running the ```status``` command, and make sure it's running ```successfully```.

### Edit ```nginx.conf``` file:
Let's make a small change to our ```nginx.conf``` file:
* Change directory to ```nginx``` directory:
```
cd /etc/nginx
```

* Edit ```nginx.conf``` with Vim:
```
sudo vim nginx.conf
```
* Replace: ```user www-data;``` with ```user root;```
* Now save and exit: ```Esc``` -> ```:wq``` -> ```Enter```

### Create ```django.conf``` file:
We need to tell our server where to find both; our django app and our static files. Earlier when we ran the ```collectstatic``` command, we saved the path where they're collected at.
* You should already be in the ```nginx``` directory; if not, navigate there.
* Change directory to ```sites-available```
```
cd sites-available
```
* Create ```django.conf``` file
```
sudo touch django.conf
```
* Open the file with Vim
```
sudo vim django.conf
```
1) Copy the ```django.conf``` file from this tutorial repo **WITHOUT THE COMMENTS**

2) Replace ```'name'``` with the name of your project (repo name) **WITHOUT THE APOSTROPHES**

3) **Replace the path in the ```locations /static/``` block with the path to your static files that you saved earlier**

   * The path will typically be ```/home/ubuntu/name_of_project/name_of_static_folder/```

4) Now save and exit: ```Esc``` -> ```:wq``` -> ```Enter```

### Almost done!
Finally, we're going to test and restart our nginx server.
* Test nginx server
```
sudo nginx -t
```

* Enable our site
```
sudo ln django.conf /etc/nginx/sites-enabled
```

* Restart our server
```
sudo service nginx restart
```

Congratulations! You should now be able to navigate to the Public IPv4 address of the AWS instance in the browser and see your deployed django application. 

You can now close the AWS instance and your application will continue to run (do not stop or terminate it).

# Pulling Repo Update 
If you make changes to the application and want to pull those changes to your instance follow this series of steps.

### Activate Virtual Environment
```
source env/bin/activate
```
### Git Pull
Make sure you github repository is public
```
cd 'project'
```
```
git pull origin main
```
### Updating Database?
Either make changes to database or drop database and copy paste updated database into client.

Enter password when prompted
```
sudo mysql -uroot -p
EXIT;
SHOW DATABASES;
XX DROP DATABASE 'database_name'; XX
```

Navigate to `manage.py` file and run migrations (if models changed):
```
python manage.py makemigrations
```
```
python manage.py migrate
```

### Collect Static Files
Navigate to `manage.py` file:
```
python manage.py collectstatic
```
### Check settings.py
Check `settings.py` file. You may need to reconfigure this file to deployment ready state.
```
sudo vim settings.py
```
### Restart Servers
```
sudo supervisorctl reread
```
```
sudo supervisorctl update
```
```
sudo supervisorctl status
```
You will get the status of of gunicor that looks something like this:
`guni:gunicorn` RUNNING pid 657, uptime 0:54:15

Use the program name to restart
```
sudo supervisorctl restart guni:gunicorn
```

Restart nginx
```
sudo service nginx restart
```


