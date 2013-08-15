
# PyTexas 2013 - Using Nginx, Gunicorn, and Upstart to serve a WSGI app

By John-Michael Oswalt. Got a django/flask/other wsgi app and want the world to see it? Come learn all the necessary parts to get your app up and running on a server or a local VM.

This talk will cover the configuration of [Nginx](http://nginx.com/ ) (web server and reverse proxy), [Gunicorn](http://gunicorn.org/ ) (WSGI server), and [Upstart](http://upstart.ubuntu.com/)  (Ubuntu lightweight process manager) so that you can serve your Django or Flask app on a server. We will start with a vanilla Ubuntu 12.04 instance and take a pre-built Django app and setup all the necessary parts to get the app up and running. The code will be available for this talk so that you can follow along with your own server, or come back to it later to see how it's done. 

The instructions below were from the talk and presented at PyTexas 2013 in College Station on August 17th, 2013.

## Launch a box

This can be a [Vagrant](http://www.vagrantup.com/)  box (see Vagrantfile) or this could be a VPS somewhere like linode, digital ocean, AWS, Rackspace, etc. You will need root access to install packages and files.

Instructions below are for Ubuntu 12.04 64 bit, though they make for other Debian and Ubuntu install.

## Install packages

We will be using git to pull a package, pip for our virtualenv and packages, and nginx as our proxy server. First we will update our package list and then install our necessary packages:

    apt-get update
    apt-get install -y git python-pip nginx

**Note** In the event that nginx is not found on your distro, you will need to add the ppa with the commands below:

#### OPTIONAL

    apt-get install -y python-software-properties
    add-apt-repository ppa:nginx/stable
    apt-get update
    apt-get install nginx -y

Now that we have `pip` installed, we can use it to install `virtualenv`.

    pip install virtualenv

Once we have our Ubuntu packages installed, we can move on to installing our app.

## Clone and install our app

For this example, I am using the Django polls tutorial app. I am storing the app in `/opt/apps`, but this is purely preference. Another common location is `/var/www`. First, we make the directory for our app and clone the repo.

    mkdir /opt/apps
    cd /opt/apps
    git clone https://github.com/jmoswalt/django-polls.git 

Now that we have cloned our app, we will setup a virtual environment and install Django.

## Create virtualenv and install requirements

Now that we have our app, we need to create a virtual environment to store our python packages that this app will use. We utilize `virtualenv` so that we don't affect the system python packages (and they don't affect us). This also allows us to install multiple apps along side each other without impacting each other.

I put my virtualenvs in a folder named `venv` directly inside my app. I exclude this folder from git by adding it to the .gitignore so that in development these files aren't committed.

    cd /opt/apps/django-polls
    virtualenv venv --no-site-packages

We use the `--no-site-packages` flag in order to prevent unwanted packages from being installed in our virtual environment.

Next, we activate our environment, so that we can install our requirements:

    source venv/bin/activate

Now we can install our requirements with pip. In this case, we are installing Django and gunicorn. If we had other dependencies in our requirements file, they would be installed as well.

    pip install -r requirements.txt


#### Django only
The rest of this section is just specific to Django. If you are using a Flask or bottle.py or other WSGI-based app, you can skip these instructions.

We can check to see if Django installed correctly by running `python manage.py`. We should see a list of management commands from this:

    python manage.py

Since we are here, this is a good time to create our database and run `syncdb`. We are using a local SQLite database for this example. When prompted, you should probably create a superuser:

    python manage.py syncdb

We can also get our static media to move to a folder with `collectstatic`. I use the `--noinput` flag so it doesn't prompt me:

    python manage.py collectstatic --noinput

And finally, let's make sure our app runs by testing with the `runserver` command:

    python manage.py runserver

Now on to configuring nginx.

## Setting up nginx

We will need to create an nginx configuration file for our app. These files are typically found in `/etc/nginx/sites-available`. There is an example file there named `default`. We will make a new file named `polls`. The name is up to you, but typically the name of the app or the name of the domain are used so the can be quickly identified.

    cd /etc/nginx/sites-available/
    touch polls

I prefer to edit my files in `nano`, but you may want to use vim or some other editor. First, we will open the file to edit it, then we will paste in the text below and save the file:

    nano polls

Text to paste:

     server {
        listen 80;
        server_name _;
    
        root /opt/apps/django-polls/;
        access_log /var/log/nginx/polls_access.log;
        error_log /var/log/nginx/polls_errors.log;
    
        charset   utf-8;
        keepalive_timeout  65;
        client_max_body_size  30M;
    
        location /static/ {
            access_log off;
            expires 30d;
        }
    
        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_pass http://127.0.0.1:8000/;
        }
    }

**Note** in the above example, the `_` after server_name should be set to your domain name or your ip address.

Now we need to create a symlink to this file in the `/etc/nginx/sites-enabled/` folder, remove the default enabled site, and then restart nginx so it will recognize our changes.

    ln -s /etc/nginx/sites-available/polls /etc/nginx/sites-enabled/polls
    rm /etc/nginx/sites-enabled/default
    service nginx restart

Now, you should be able to load the site, but with a pretty big error (502 Bad Gateway). At this point, this is to be expected. We can test our static assets by going to /static/polls/style.css and we should see our stylesheet content load. This means that the /static/ path is being served, but our root location is not. For this, we now setup Gunicorn with Upstart

# Setting up Gunicorn with Upstart

Upstart is a process manager that comes bundles with Ubuntu. If you aren't using Ubuntu, a similar (and more robust) option is [supervisord](). We will create an upstart file to manage the process of serving the Django (or other WSGI) app with Gunicorn.

    nano /etc/init/polls.conf

In our polls.conf file, paste the following:

    description "Django Polls Upstart Script"
    start on runlevel [2345]
    stop on runlevel [06]
    kill timeout 300
    respawn
    respawn limit 10 5
    
    script
      cd /opt/apps/django-polls
      . venv/bin/activate
      exec gunicorn -w 2 -b 127.0.0.1:8000 mysite.wsgi:application
    end script

The `-w 2` portion of the file specifies 2 workers. If your app has lots of traffic, it will likely require more workers. You can also specify other types of workers like [gevent](http://docs.gunicorn.org/en/latest/configure.html#worker-class ), but we won't cover that in this talk.

**Note** Based on your project, the `mysite.wsgi:application` will likely change. This is the python path to your wsgi file, along with the wsgi_application to be run. In our case, we can actually run this without the `:application`, but it's better practice to specify the app.

Now, we can start our app with:

    service polls start

Once this is done, you should be able to load the app by visiting the url for your site. You can login at /admin with the username and password you created to start adding polls.

## Running multiple apps

If you plan on running multiple apps on a single server, your will need to repeat the process above with one tweak. The port numbers in the Nginx redirect and in the Upstart file for Gunicorn need to match, and can't be duplicated. In the example, we used port 8000. 

For your second app, you could use port 8001, making sure this change is made in both `/etc/nginx/sites-available/appname` and `/etc/init/appname.conf` files.

## Making updates in the future

In the future, when you need to update the code for your app, you will need to restart the gunicorn process by using the upstart file:

    service polls restart

This will regenerate the .pyc files and your app's changes can then be seen. **Note**, this is only for changes in python files. Changes to html files or static files (after `collectstatic` of course) do not need gunicorn to restart.

If you change your domain name or need to add a domain name, be sure to update the nginx configuration file (`/etc/nginx/sites-available/polls`) and restart nginx with:

    service nginx restart

Congrats! You are now able to host your own WSGI-based application with Nginx, Gunicorn, and Upstart.
