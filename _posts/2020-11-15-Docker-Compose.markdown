---
layout: post
title:  "Docker Compose"
date:   2020-11-15 15:00:50 +0530
categories: docker-compose
---
# Docker Compose
Docker Compose is a tool for defining and running multi-container Docker applications. You make use of a YAML file to configure your application's services, which you can then create and start with a single command.

Using Compose essentially consists of 3 steps:

1. Creating a `Dockerfile` for your app ([for the uninitiated](https://mlndshh.github.io/docker/2020/11/14/Docker-Tutorial.html))
2. Define the `services` that make up your app in a file called `docker-compose.yml` so they can be run together in an isolated environment.
3. Run the command `docker-compose up` and Compose starts and runs your app

*(The steps above were picked right from Compose's [docs](https://docs.docker.com/compose/))*

To install Compose, follow their steps [here](https://docs.docker.com/compose/install/).

We will now quickly go through how one would use Compose for a simple Django/PostgreSQL app.

 1. Create an empty directory for your app. In my case it's `django-app`
 2. Create a new `Dockerfile` in the directory with the following content: 

		FROM python:3
		ENV PYTHONUNBUFFERED=1
		RUN mkdir /code
		WORKDIR /code
		COPY requirements.txt /code/
		RUN pip install -r requirements.txt
		COPY . /code/
	

	 - `FROM python:3` - To start with a Python 3 base image
	 -  `ENV PYTHONUNBUFFERED=1` - [See here](https://github.com/aws/amazon-sagemaker-examples/issues/319#issuecomment-405749895)
	 - `RUN mkdir /code` - creates a new directory called code
	 - `WORKDIR /code` - Makes code the working directory.
	 - `COPY requirements.txt /code/` - copies the requirements.txt file from the project directory into the container.
	 - `RUN pip install -r requirements.txt` - runs `pip install` to install the requirements.
	 - `COPY . /code/` - copy the project directory into the container's code folder.

3. Create the `requirements.txt` file in your project directory with the following content:

		Django>=3.0,<4.0
		psycopg2-binary>=2.8

4. Create a file called docker-compose.yml in your project directory.

		version: "3.8"
   
		services:
		  db:
		    image: postgres
		    environment:
		      - POSTGRES_DB=postgres
		      - POSTGRES_USER=postgres
		      - POSTGRES_PASSWORD=postgres
		  web:
		    build: .
		    command: python manage.py runserver 0.0.0.0:8000
		    volumes:
		      - .:/code
		    ports:
		      - "8000:8000"
		    depends_on:
		      - db

5. Now we will create the Django project (in a slightly different way) by running the following command:

		sudo docker-compose run web django-admin startproject composeexample .

	Output:
		
		(base) milind@Milinds-MacBook-Air django-app % sudo docker-compose run web django-admin startproject composeexample .
		Password:
		Creating network "django-app_default" with the default driver
		Pulling db (postgres:)...
		latest: Pulling from library/postgres
		bb79b6b2107f: Pull complete
		e3dc51fa2b56: Pull complete
		f213b6f96d81: Pull complete
		2780ac832fde: Pull complete
		ae5cee1a3f12: Pull complete
		95db3c06319e: Pull complete
		475ca72764d5: Pull complete
		8d602872ecae: Pull complete
		cd38e50fcd6d: Pull complete
		090a5b74af73: Pull complete
		1a4356e41724: Pull complete
		886a3b55e0f4: Pull complete
		0aa963dfbf32: Pull complete
		1403f0786df1: Pull complete
		Digest: sha256:5344d1b6f33169b56f450c0586ffc3497f0479e1fc499bed76857912d2fe5f98
		Status: Downloaded newer image for postgres:latest
		Building web
		Step 1/7 : FROM python:3
		3: Pulling from library/python
		e4c3d3e4f7b0: Pull complete
		101c41d0463b: Pull complete
		8275efcd805f: Pull complete
		751620502a7a: Pull complete
		0a5e725150a2: Pull complete
		397dba5694db: Pull complete
		b1d09d0eabcb: Pull complete
		475299e7c7f3: Pull complete
		ede3eeefc571: Pull complete
		Digest: sha256:399f5241afed6952ed9ad5663993aa0f4e12a5f98f362e8ffd00788034f09285
		Status: Downloaded newer image for python:3
		 ---> 768307cdb962
		Step 2/7 : ENV PYTHONUNBUFFERED=1
		 ---> Running in bb37fe0c07ac
		Removing intermediate container bb37fe0c07ac
		 ---> 4e5a84c72006
		Step 3/7 : RUN mkdir /code
		 ---> Running in 072915a20d94
		Removing intermediate container 072915a20d94
		 ---> 880927257032
		Step 4/7 : WORKDIR /code
		 ---> Running in aaf360ea03dd
		Removing intermediate container aaf360ea03dd
		 ---> 512f964ccee9
		Step 5/7 : COPY requirements.txt /code/
		 ---> 1d0ac1ee513c
		Step 6/7 : RUN pip install -r requirements.txt
		 ---> Running in 9d73e6628b76
		Collecting Django<4.0,>=3.0
		  Downloading Django-3.1.3-py3-none-any.whl (7.8 MB)
		Collecting psycopg2-binary>=2.8
		  Downloading psycopg2_binary-2.8.6-cp39-cp39-manylinux1_x86_64.whl (3.0 MB)
		Collecting asgiref<4,>=3.2.10
		  Downloading asgiref-3.3.1-py3-none-any.whl (19 kB)
		Collecting sqlparse>=0.2.2
		  Downloading sqlparse-0.4.1-py3-none-any.whl (42 kB)
		Collecting pytz
		  Downloading pytz-2020.4-py2.py3-none-any.whl (509 kB)
		Installing collected packages: asgiref, sqlparse, pytz, Django, psycopg2-binary
		Successfully installed Django-3.1.3 asgiref-3.3.1 psycopg2-binary-2.8.6 pytz-2020.4 sqlparse-0.4.1
		Removing intermediate container 9d73e6628b76
		 ---> dc92e13c0922
		Step 7/7 : COPY . /code/
		 ---> ae51ba14be86

		Successfully built ae51ba14be86
		Successfully tagged django-app_web:latest
		WARNING: Image for service web was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
		Creating django-app_db_1 ... done
		Creating django-app_web_run ... done

	The above command essentially instructs Compose to run `django-admin startproject composeexample` in a container using the `web` service's image configuration. Since `web` does not exist yet, it builds the image too using the current directory (since `build: .` was mentioned in the yml file).

	Once the `web` service image is built, Compose runs it and executes `django-admin startproject` in the container. 

	If you are confused as to why running the command in the container caused files to be made in the project directory, it is because of:
	
		volumes:
		      - .:/code
	We created a volume so that any changes in the project directory reflect in the code directory of the container and vice versa.
	
6. Let us now set up the database connection for Django
	
	In `composeexample/settings.py`, replace `DATABASES = ` with:

		# settings.py
   
		DATABASES = {
		    'default': {
		        'ENGINE': 'django.db.backends.postgresql',
		        'NAME': 'postgres',
		        'USER': 'postgres',
		        'PASSWORD': 'postgres',
		        'HOST': 'db',
		        'PORT': 5432,
		    }
		}

	*If you notice the host name, it's `db`. This is because the postgres service is named `db` and Compose sets up a network for all services to be in. The services can communicate to each other simply by using their names*

7. We can now run `docker-compose up` and see the app in action.

		(base) milind@Milinds-MacBook-Air django-app % docker-compose up
		django-app_db_1 is up-to-date
		Creating django-app_web_1 ... done
		Attaching to django-app_db_1, django-app_web_1
		db_1   | The files belonging to this database system will be owned by user "postgres".
		db_1   | This user must also own the server process.
		db_1   | 
		db_1   | The database cluster will be initialized with locale "en_US.utf8".
		db_1   | The default database encoding has accordingly been set to "UTF8".
		db_1   | The default text search configuration will be set to "english".
		db_1   | 
		db_1   | Data page checksums are disabled.
		db_1   | 
		db_1   | fixing permissions on existing directory /var/lib/postgresql/data ... ok
		db_1   | creating subdirectories ... ok
		db_1   | selecting dynamic shared memory implementation ... posix
		db_1   | selecting default max_connections ... 100
		db_1   | selecting default shared_buffers ... 128MB
		db_1   | selecting default time zone ... Etc/UTC
		db_1   | creating configuration files ... ok
		db_1   | running bootstrap script ... ok
		db_1   | performing post-bootstrap initialization ... ok
		db_1   | syncing data to disk ... ok
		db_1   | 
		db_1   | initdb: warning: enabling "trust" authentication for local connections
		db_1   | You can change this by editing pg_hba.conf or using the option -A, or
		db_1   | --auth-local and --auth-host, the next time you run initdb.
		db_1   | 
		db_1   | Success. You can now start the database server using:
		db_1   | 
		db_1   |     pg_ctl -D /var/lib/postgresql/data -l logfile start
		db_1   | 
		db_1   | waiting for server to start....2020-11-15 09:03:03.734 UTC [46] LOG:  starting PostgreSQL 13.1 (Debian 13.1-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
		db_1   | 2020-11-15 09:03:03.737 UTC [46] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
		db_1   | 2020-11-15 09:03:03.749 UTC [47] LOG:  database system was shut down at 2020-11-15 09:03:03 UTC
		db_1   | 2020-11-15 09:03:03.759 UTC [46] LOG:  database system is ready to accept connections
		db_1   |  done
		db_1   | server started
		db_1   | 
		db_1   | /usr/local/bin/docker-entrypoint.sh: ignoring /docker-entrypoint-initdb.d/*
		db_1   | 
		db_1   | waiting for server to shut down....2020-11-15 09:03:03.793 UTC [46] LOG:  received fast shutdown request
		db_1   | 2020-11-15 09:03:03.795 UTC [46] LOG:  aborting any active transactions
		db_1   | 2020-11-15 09:03:03.827 UTC [46] LOG:  background worker "logical replication launcher" (PID 53) exited with exit code 1
		db_1   | 2020-11-15 09:03:03.828 UTC [48] LOG:  shutting down
		db_1   | 2020-11-15 09:03:03.868 UTC [46] LOG:  database system is shut down
		db_1   |  done
		db_1   | server stopped
		db_1   | 
		db_1   | PostgreSQL init process complete; ready for start up.
		db_1   | 
		db_1   | 2020-11-15 09:03:03.957 UTC [1] LOG:  starting PostgreSQL 13.1 (Debian 13.1-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
		db_1   | 2020-11-15 09:03:03.958 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
		db_1   | 2020-11-15 09:03:03.958 UTC [1] LOG:  listening on IPv6 address "::", port 5432
		db_1   | 2020-11-15 09:03:03.962 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
		db_1   | 2020-11-15 09:03:03.971 UTC [55] LOG:  database system was shut down at 2020-11-15 09:03:03 UTC
		db_1   | 2020-11-15 09:03:03.982 UTC [1] LOG:  database system is ready to accept connections
		web_1  | Watching for file changes with StatReloader
		web_1  | Performing system checks...
		web_1  | 
		web_1  | System check identified no issues (0 silenced).
		web_1  | 
		web_1  | You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
		web_1  | Run 'python manage.py migrate' to apply them.
		web_1  | November 15, 2020 - 09:40:40
		web_1  | Django version 3.1.3, using settings 'composeexample.settings'
		web_1  | Starting development server at http://0.0.0.0:8000/
		web_1  | Quit the server with CONTROL-C.
		web_1  | Not Found: /login/
		web_1  | [15/Nov/2020 09:41:01] "GET /login/ HTTP/1.1" 404 1965
		web_1  | Not Found: /favicon.ico
		web_1  | [15/Nov/2020 09:41:01] "GET /favicon.ico HTTP/1.1" 404 1980
		web_1  | [15/Nov/2020 09:41:03] "GET / HTTP/1.1" 200 16351
		web_1  | [15/Nov/2020 09:41:03] "GET /static/admin/css/fonts.css HTTP/1.1" 200 423
		web_1  | [15/Nov/2020 09:41:03] "GET /static/admin/fonts/Roboto-Light-webfont.woff HTTP/1.1" 200 85692
		web_1  | [15/Nov/2020 09:41:03] "GET /static/admin/fonts/Roboto-Regular-webfont.woff HTTP/1.1" 200 85876
		web_1  | [15/Nov/2020 09:41:03] "GET /static/admin/fonts/Roboto-Bold-webfont.woff HTTP/1.1" 200 86184

	![](/images/docker-compose/ss.png)

8. To see a list of running containers: `docker ps`

		(base) milind@Milinds-MacBook-Air django-app % docker ps
		CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
		5e96429cdf83        django-app_web      "python manage.py ru…"   2 minutes ago       Up 2 minutes        0.0.0.0:8000->8000/tcp   django-app_web_1
		66df35a22a83        postgres            "docker-entrypoint.s…"   39 minutes ago      Up 39 minutes       5432/tcp                 django-app_db_1

9. To shut the containers down, run `docker-compose down` from the project directory.
		
		(base) milind@Milinds-MacBook-Air django-app % docker-compose down
		Stopping django-app_web_1 ... done
		Stopping django-app_db_1  ... done
		Removing django-app_web_1                ... done
		Removing django-app_web_run_35b56718eeab ... done
		Removing django-app_db_1                 ... done
		Removing network django-app_default 

And that's about it! This was a quick tutorial to help get anyone started, but Compose has a lot more to it (going through the docs is always a great idea).

**Sources:**

- [https://docs.docker.com/compose/](https://docs.docker.com/compose/)