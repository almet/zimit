#####################################
Create ZIM files out of HTTP websites
#####################################

This project provides an API and an user interface in order to convert any
website into a Zim file.

Exposed API
###########

All APIs are talking JSON over HTTP. As such, all parameters should be sent as
stringified JSON and the Content-Type should be set to "application/json".

POST /website-zim
=================

By posting to this endpoint, you are asking the system to start a new download
of a website and a conversion into a Zim format.

Required parameters
-------------------

- **url**: URL of the website to be crawled
- **title**: Title that will be used in the created Zim file
- **email**: Email address that will get notified when the creation of the file is over

Optional parameters
-------------------

- **language**: An `ISO 639-3 <https://en.wikipedia.org/wiki/ISO_639-3>`_ code
   representing the language
- **welcome**: the page that will be first shown in the Zim file
- **description**: The description that will be embedded in the Zim file
- **author**: The author of the content

Return values
-------------

- **job_id**: The job id is returned in JSON format. It can be used to know the
  status of the process.

Status codes
------------

- `400 Bad Request` will be returned in case you are not respecting the
  expected inputs. In case of error, have a look at the body of the response:
  it contains information about what is missing.
- `201 Created` will be returned if the process started.

Exemple
-------

::

  $ http POST http://0.0.0.0:6543/website-url url="https://refugeeinfo.eu/" title="Refugee Info" email="alexis@notmyidea.org"
  HTTP/1.1 201 Created

  {
      "job": "5012abe3-bee2-4dd7-be87-39a88d76035d"
  }


GET /status/{jobid}
===================

Retrieve the status of a job and displays the associated logs.

Return values
-------------

- **status**: The status of the job, it is one of 'queued', finished',
  'failed', 'started' and 'deferred'.
- **log**: The logs of the job.

Status codes
------------

- `404 Not Found` will be returned in case the requested job does not exist.
- `200 OK` will be returned in any other case.

Exemple
-------

::

    http GET http://0.0.0.0:6543/status/5012abe3-bee2-4dd7-be87-39a88d76035d
    HTTP/1.1 200 OK

    {
        "log": "<snip>",
        "status": "finished"
    }


Okay, so how do I install it on my server?
##########################################

Currently, the best way to install it is by retrieving the sources from github

::

  $ git clone https://github.com/almet/zimit.git
  $ cd zimit

Create a virtual environment and install the project in it::

  $ virtualenv venv
  $ venv/bin/pip install -e .
  $ venv/bin/pip install -e git+git@github.com:nvie/rq.git@master#egg=rq

Then, run it how you want, for instance with pserve::

  $ venv/bin/pserve zimit.ini


In a separate process, you also need to run the worker::

  $ venv/bin/rqworker


And you're ready to go. To test it::

  $ http POST http://0.0.0.0:6543/website-url url="https://refugeeinfo.eu/" title="Refugee Info" email="alexis@notmyidea.org"


Debian dependencies
####################

Installing the dependencies
===========================

::

    sudo apt-get install httrack libzim-dev libmagic-dev liblzma-dev libz-dev build-essential libtool libgumbo-dev redis-server automake pkg-config

Installing zimwriterfs
======================

::

    git clone https://github.com/wikimedia/openzim.git
    cd openzim/zimwriterfs
    ./autogen.sh
    ./configure
    make

Then upgrade the path to zimwriterfs executable in zimit.ini

::

  $ rqworker & pserve zimit.ini

How to deploy?
##############

There are multiple ways to deploy such service, so I'll describe how I do it
with my own best-practices.

First of all, get all the dependencies and the code. I like to have everything
available in /home/www, so let's consider this will be the case here::

  $ mkdir /home/www/zimit.notmyidea.org
  $ cd /home/www/zimit.notmyidea.org
  $ git clone https://github.com/almet/zimit.git

Then, you can change the configuration file, by creating a new one::

  $ cd zimit
  $ cp zimit.ini local.ini

From there, you need to update the configuration to point to the correct
binaries and locations.

Nginx configuration
===================

::

  # the upstream component nginx needs to connect to
    upstream zimit_upstream {
        server unix:///tmp/zimit.sock;
    }

    # configuration of the server
    server {
        listen      80;
        listen   [::]:80;
        server_name zimit.ideascube.org;
        charset     utf-8;

        client_max_body_size 200M;

        location /zims {
            alias /home/ideascube/zimit.ideascube.org/zims/;
            autoindex on;
        }

        # Finally, send all non-media requests to the Pyramid server.
        location / {
            uwsgi_pass  zimit_upstream;
            include     /var/ideascube/uwsgi_params;
        }
      }


UWSGI configuration
===================

::

  [uwsgi]
  uid = ideascube
  gid = ideascube
  chdir           = /home/ideascube/zimit.ideascube.org/zimit/
  ini             = /home/ideascube/zimit.ideascube.org/zimit/local.ini
  # the virtualenv (full path)
  home            = /home/ideascube/zimit.ideascube.org/venv/

  # process-related settings
  # master
  master          = true
  # maximum number of worker processes
  processes       = 4
  # the socket (use the full path to be safe
  socket          = /tmp/zimit.sock
  # ... with appropriate permissions - may be needed
  chmod-socket    = 666
  # stats           = /tmp/ideascube.stats.sock
  # clear environment on exit
  vacuum          = true
  plugins         = python


supervisord configuration
=========================

::

  [program:zimit-worker]
  command=/home/ideascube/zimit.ideascube.org/venv/bin/rqworker
  directory=/home/ideascube/zimit.ideascube.org/zimit/
  user=www-data
  autostart=true
  autorestart=true
  redirect_stderr=true

That's it!
