# Celery + Redis + Django (under development)


Celery is a task queue with focus on real-time processing, while also supporting task scheduling. \

Redis is a message broker. This means it handles the queue of "messages" between Django and Celery. 

Django is a web framework made for perfectionists with deadlines. 

All three work together to make real-time magic. 


============

#### Do you have a Django Project? If not, use [this](http://kirr.co/itznm2/).


1. Download Redis
    - Using [Homebrew](http://brew.sh):
        ```
        brew install redis

        brew services start redis
        ```
        Brew permission errors? Try `sudo chown -R "$USER":admin /usr/local`

    - Direct [Download](http://redis.io/download)


2. Open & Test Redis:
    - open Terminal

    - **redis-cli ping**
        ```
        $ redis-cli ping
        PONG
        ```

    - **redis-server**
        ```
        $ redis-server
        86750:C 08 Nov 08:17:21.431 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
        86750:M 08 Nov 08:17:21.433 * Increased maximum number of open files to 10032 (it was originally set to 256).
                        _._                                                  
                   _.-``__ ''-._                                             
              _.-``    `.  `_.  ''-._           Redis 3.2.5 (00000000/0) 64 bit
          .-`` .-```.  ```\/    _.,_ ''-._                                   
         (    '      ,       .-`  | `,    )     Running in standalone mode
         |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
         |    `-._   `._    /     _.-'    |     PID: 86750
          `-._    `-._  `-./  _.-'    _.-'                                   
         |`-._`-._    `-.__.-'    _.-'_.-'|                                  
         |    `-._`-._        _.-'_.-'    |           http://redis.io        
          `-._    `-._`-.__.-'_.-'    _.-'                                   
         |`-._`-._    `-.__.-'    _.-'_.-'|                                  
         |    `-._`-._        _.-'_.-'    |                                  
          `-._    `-._`-.__.-'_.-'    _.-'                                   
              `-._    `-.__.-'    _.-'                                       
                  `-._        _.-'                                           
                      `-.__.-'                                               

        86750:M 08 Nov 08:17:21.434 # Server started, Redis version 3.2.5
        86750:M 08 Nov 08:17:21.434 * The server is now ready to accept connections on port 6379

        ```
        **Close Redis** with `control` + `c` to quit

3. Install Celery + Redis in your virtualenv.
    ```
    pip install celery
    pip install redis
    pip install django-celery-beat
    pip install django-celery-results
    pip freeze > requirements.txt
    ```

4. Update Django `settings.py`:
    ```
    INSTALLED_APPS = [
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
        'django_celery_beat',
        'django_celery_results',
    ]


    CELERY_BROKER_URL = 'redis://localhost:6379'
    CELERY_RESULT_BACKEND = 'redis://localhost:6379'
    CELERY_ACCEPT_CONTENT = ['application/json']
    CELERY_TASK_SERIALIZER = 'json'
    CELERY_RESULT_SERIALIZER = 'json'
    CELERY_TIMEZONE = TIME_ZONE
    ```

5. Create `celery.py` to setup `Celery app`:
    Navigate to project config module (where `settings` and `urls` modules are) and create a `celery.py` file with the contents:
    ```
    from __future__ import absolute_import, unicode_literals
    import os
    from celery import Celery
    

    # set the default Django settings module for the 'celery' program.
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'red.settings')

    app = Celery('proj')

    # Using a string here means the worker don't have to serialize
    # the configuration object to child processes.
    # - namespace='CELERY' means all celery-related configuration keys
    #   should have a `CELERY_` prefix.
    app.config_from_object('django.conf:settings', namespace='CELERY')

    # Load task modules from all registered Django app configs.
    app.autodiscover_tasks()

    @app.task(bind=True)
    def debug_task(self):
        print('Request: {0!r}'.format(self.request))

    ```

6. Create `tasks.py` in your Django app (a valid app in `INSTALLED_APPS`):
    ```
    from __future__ import absolute_import, unicode_literals
    import random
    from celery.decorators import task

    @task(name="sum_two_numbers")
    def add(x, y):
        return x + y


    @task(name="multiply_two_numbers")
    def mul(x, y):
        total = x * (y * random.randint(3, 100))
        return total


    @task(name="sum_list_numbers")
    def xsum(numbers):
        return sum(numbers)
    ```
7. Run migrations: 
    `python manage.py makemigrations` 
    and 
    `python manage.py migrate`


8. Test tasks:
    1. Open a terminal window, `Run Celery` with in your `project root` where `manage.py` lives:

        ```
        celery -A yourproject worker -l info
        # like 
        celery -A cfehome worker -l info
        ```

    2. Open another terminal window, in your Django project `python manage.py shell`:

        ```
        >>> from yourapp.tasks import add, mul, xsum
        >>> add(1,3)
        4
        >>> add.delay(1,3)
        <AsyncResult: 7bb03f9a-5702-4661-b737-2bc54ed9f558>
        ```

        If you look at your celery worker, you should see something like:
        ```
        [2016-11-08 22:05:22,686: INFO/PoolWorker-5] Task sum_two_numbers[7bb03f9a-5702-4661-b737-2bc54ed9f558] succeeded in 0.0004559689841698855s: 21
        ```


9. Setup Schedule to Run Tasks [Docs](http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html):
    ```
    # celery.py
    
    from celery.schedules import crontab
    app.conf.beat_schedule = {
        'add-every-minute-contrab': {
            'task': 'multiply_two_numbers',
            'schedule': crontab(),
            'args': (16, 16),
        },
        'add-every-5-seconds': {
            'task': 'multiply_two_numbers',
            'schedule': 5.0,
            'args': (16, 16)
        },
        'add-every-30-seconds': {
            'task': 'tasks.add',
            'schedule': 30.0,
            'args': (16, 16)
        },
    }
    ```
