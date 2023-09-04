## Introduction to Microservice Architecture

We are going to build a dockerized Django application with Django, Rabbitmq, celery, and Postgres to handle asynchronous tasks. 
Basically, the main idea here is to configure Django with docker containers, especially with Rabbitmq and celery and automate scrapping process.. 

At the end of this, you will able to create your own asynchronous apps which scrape articles from python blog sites. 

### To build this project we need following docker container
- Celery worker
- RabbitMq container
- Django webapp
- PostgreSql backend server 

### Advantage of using asynchronous taks
- No need to wait until task completes.
- We can run task parallely.

## Project Architecture
- Django site is mainly used for following things: 
  - Trigger asynchronous task.
  - View scraped data.
  - View task stats and logs.

- Once Django site trigger the task, It adds entry in task queue.
  - Here we used RabbitMq to handle the task.
- Celery pick the task from rabbitMQ and process the request.
- While processing the request, It saves data in postgres database.

![alt text](./project_model.png)

#### Note : This project is continuation of **workshop3** so do the changes in workshop3 project only.

### Step 1 : Create RabbitMq container
- Add below code in your docker-compose file which creates rabbitmq image
    ```yaml
     service-rabbitmq:
       container_name: "service_rabbitmq"
       image: rabbitmq:3.8-management-alpine
       environment:
         - RABBITMQ_DEFAULT_USER=myuser
         - RABBITMQ_DEFAULT_PASS=mypassword
         - RABBITMQ_DEFAULT_VHOST=extractor
         - BROKER_HOST=service-rabbitmq
       ports:
         - '5672:5672'
         - '15676:15672'
    ```
- Build docker container
    ```yaml
    docker-compose up -d
    ```
- To Verify rabbitmq container please open this link `http://localhost:15676/`
  - use username and password added in the docker image.


### Step 2 : Add RabbitMq credentials in webapp container
We need broker(rabbitmq) configuration to register celery task.  
- Add below environment variables to webapp container.  
    ```yaml
       environment:
         - RABBITMQ_DEFAULT_USER=myuser
         - RABBITMQ_DEFAULT_PASS=mypassword
         - BROKER_HOST=service-rabbitmq
         - RABBITMQ_DEFAULT_VHOST=extractor
         - BROKER_PORT=5672
    ```
- Build docker container 
    ```yaml
    docker-compose up -d
    ```
### Step 3: Configuring Rabbitmq and Celery Service
- Add celery.py file inside myworld directory.
  ```
  myworld/
  ├── __init__.py
  ├── asgi.py
  ├── celery.py
  ├── settings.py
  ├── urls.py
  └── wsgi.py
  ```
- Add blow content to celery.py
  ```python
  from celery import Celery
  
  # Set the default Django settings module for the 'celery' program.
  os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myworld.settings')
  
  app = Celery('myworld')
  
  # Using a string here means the worker doesn't have to serialize
  # the configuration object to child processes.
  # - namespace='CELERY' means all celery-related configuration keys
  #   should have a `CELERY_` prefix.
  app.config_from_object('django.conf:settings', namespace='CELERY')
  
  # Load task modules from all registered Django apps.
  app.autodiscover_tasks()
  
  
  @app.task(bind=True)
  def debug_task(self):
      print(f'Request: {self.request!r}')
  ```
- What's happening here?
  - First, we set a default value for the DJANGO_SETTINGS_MODULE environment variable so that Celery will know how to find the Django project.
  - Next, we created a new Celery instance, with the name core, and assigned the value to a variable called app.
  - We then loaded the celery configuration values from the settings object from django.conf. We used namespace="CELERY" to prevent clashes with other Django settings. All config settings for Celery must be prefixed with CELERY_, in other words.
  - Finally, app.autodiscover_tasks() tells Celery to look for Celery tasks from applications defined in settings.INSTALLED_APPS.
- Update myworld/__init__.py so that the Celery app is automatically imported when Django starts:
  ```python
  # This will make sure the app is always imported when
  # Django starts so that shared_task will use this app.
  from .celery import app as celery_app
  
  __all__ = ('celery_app',)
  ```
- Within the project's settings `(settings.py)` module, add the following at the bottom to tell Celery to use RabbitMq as the broker and backend:
  ```python
  import os

  CELERY_TASK_SERIALIZER = 'json'
  CELERY_RESULT_SERIALIZER = 'json'
  CELERY_TIMEZONE = 'America/Los_Angeles'
  # This configures rabbitmq as the datastore between Django + Celery
  CELERY_BROKER_URL = 'amqp://{0}:{1}@{2}:{3}/{4}'.format(
              os.environ["RABBITMQ_DEFAULT_USER"], os.environ["RABBITMQ_DEFAULT_PASS"],
              os.environ["BROKER_HOST"], os.environ["BROKER_PORT"],
              os.environ["RABBITMQ_DEFAULT_VHOST"])
  ```

### Step 4 : Create celery worker
- Add below code in your docker-compose file which creates celery worker
    ```yaml
     worker:
       build:
         context: ./
         dockerfile: ./dockerfiles/Dockerfile
       image: workshop1_web
       container_name: worker
       stdin_open: true #  docker attach container_id
       tty: true
       environment:
         - RABBITMQ_DEFAULT_USER=myuser
         - RABBITMQ_DEFAULT_PASS=mypassword
         - BROKER_HOST=service-rabbitmq
         - RABBITMQ_DEFAULT_VHOST=extractor
         - BROKER_PORT=5672
       ports:
         - "4356:8000"
       volumes:
         - .:/root/workspace/site
    ```
- Build docker container
    ```yaml
    docker-compose up -d
    ```
- Verify celery worker.
    ```shell
    docker exec -it worker sh
    ```
  - Run celery command to verify celery worker.
      ```shell
      python -m celery -A myworld worker  -l info
      ```
#### Note : Sometimes we get error when running celery cmd. If you get any error please check rabbitMq configurations. 

### Step 5 : Add Job , Job stats and Job log models
We need following models to run and track asynchronous task.
- Job : Job is nothing but task which contains meta information required to run task.
- Job stats : This contains stats for each job runs.
- Job Logs : This contains logs which helps us to track asynchronous task. 

```python
class Job(models.Model):
    job_name = models.CharField(max_length=500)
    start_date = models.DateTimeField('Blog start date', null=True)
    end_date = models.DateTimeField('Blog end date', null=True)
    start_no = models.IntegerField(verbose_name="No of blogs to skip", null=True)
    no_of_blogs = models.IntegerField(verbose_name="No of blogs to extract", null=True)
    created_date = models.DateTimeField('Job created date', auto_now_add=True, null=True)

    def __str__(self):
        return self.job_name


class JobStats(models.Model):
    job = models.ForeignKey(Job, on_delete=models.CASCADE)
    status = models.CharField(max_length=50)
    total_blogs = models.IntegerField(verbose_name="Total blogs found", null=True)
    no_of_blogs_extracted = models.IntegerField(verbose_name='No of blogs extracted', null=True)
    start_date = models.DateTimeField('Extraction start date', null=True)
    end_date = models.DateTimeField('Extraction start date', null=True)

    def __str__(self):
        return self.job


class JobLogs(models.Model):
    job_stats = models.ForeignKey(JobStats, on_delete=models.CASCADE)
    log = models.TextField(verbose_name="job logs")
    function_name = models.TextField(verbose_name="Function name")
    date = models.DateTimeField('Log date', null=True, auto_now_add=True)
```

### Step 6 : Add models in admin.py
To display models in front we need to register models in admin.py
  ```shell
  from .models import Students, Blog, Job, JobLogs, JobStats
  from django.urls import reverse
  from django.utils.html import format_html
    
  class DjJob(admin.ModelAdmin):
     
    def view_stats(self, obj):
      path = "../jobstats/?q={}".format(obj.pk)
      return format_html(f'''<a class="button" href="{path}">stats</a>''')
  
    view_stats.short_description = 'Stats'
    view_stats.allow_tags = True
  
    list_display = ("job_name", "start_date", "end_date", "no_of_blogs", "start_no", "created_date", "view_stats")
    list_filter = ("job_name", "start_date")
    readonly_fields = ("created_date",)
  
  class DjJobStats(admin.ModelAdmin):
    def view_logs(self, obj):
      path = "../joblogs/?q={}".format(obj.pk)
      return format_html(f'''<a class="button" href="{path}">Logs</a>''')
  
    view_logs.short_description = 'Stats'
    view_logs.allow_tags = True
    list_display = ("job", "status", "view_logs", "total_blogs", "no_of_blogs_extracted", "start_date", "end_date")
    search_fields = ('job__pk',)
  
  class DjJobLogs(admin.ModelAdmin):
    list_display = ("date", "log", "function_name")
    search_fields = ('job_stats__pk',)
  
  
  admin.site.register(Job, DjJob)
  admin.site.register(JobStats, DjJobStats)
  admin.site.register(JobLogs, DjJobLogs)
  ```

### Step 7: Create tables for above models
- Exec into webapp container
    ```shell
    docker exec -it workshop_web_container sh
    ```
- Run makemigrations
    ```shell
    python manage.py makemigrations
    ```
- Run migrate
    ```shell
    python manage.py migrate
    ```
#### Note : Please verify models in front end by running webapp `http://0.0.0.0:8000/admin/`

### Step 8 : Add Task 
- Create new file called `tasks.py` inside members directory
  ```
  members/
  ├── __init__.py
  ├── admin.py
  ├── apps.py
  ├── models.py
  ├── urls.py
  ├── views.py
  ├── tasks.py
  └── test.py
  ```
- Add below contents
  ```python
  import datetime
  from myworld.celery import app
  from .models import Job, Blog, JobStats, JobLogs
  import requests
  from bs4 import BeautifulSoup
  from dateutil.parser import parse
  import pytz
  
  utc=pytz.UTC
  
  @app.task(bind=True, name="extract")
  def extract(self, job_id):
      job_obj = Job.objects.get(pk=job_id)
      job_stats_obj = JobStats(job=job_obj, status="IN PROGRESS", start_date=datetime.datetime.now(), no_of_blogs_extracted=0)
      job_stats_obj.save()
      JobLogs(job_stats=job_stats_obj, log="Extraction stated", function_name="extract", date=datetime.datetime.now()).save()
      start_date = job_obj.start_date
      end_date = job_obj.end_date
      start_id = job_obj.start_no
      no_of_articles = job_obj.no_of_blogs
      url = "https://blog.python.org/"
      try:
          data = requests.get(url)
          page_soup = BeautifulSoup(data.text, 'html.parser')
  
          blogs = page_soup.select('div.date-outer')
          article_count = 0
          counter = 1
          for blog in blogs:
              article_count += 1
              if start_id and article_count < int(start_id):
                  continue
              if no_of_articles and counter > int(no_of_articles):
                  continue
              date = blog.select('.date-header span')[0].get_text()
  
              converted_date = parse(date)
              JobLogs(job_stats=job_stats_obj, log=f"Extracting {article_count}", function_name="extract", date=datetime.datetime.now()).save()
              if start_date and utc.localize(converted_date) < start_date:
                  continue
              if end_date and utc.localize(converted_date) > end_date:
                  continue
  
              post = blog.select('.post')[0]
  
              title = ""
              title_bar = post.select('.post-title')
              if len(title_bar) > 0:
                  title = title_bar[0].text
              else:
                  title = post.select('.post-body')[0].contents[0].text
  
              # getting the author and blog time
              post_footer = post.select('.post-footer')[0]
  
              author = post_footer.select('.post-author span')[0].text
  
              time = post_footer.select('abbr')[0].text
  
              blog_obj = Blog(title=title, author=author, release_date=date, blog_time=time)
              blog_obj.save()
              job_stats_obj.no_of_blogs_extracted += job_stats_obj.no_of_blogs_extracted
              job_stats_obj.save()
  
              print("\nTitle:", title.strip('\n'))
              print("Date:", date, )
              print("Time:", time)
              print("Author:", author)
              counter += 1
          JobLogs(job_stats=job_stats_obj, log=f"Total {counter} articles extracted: ", function_name="extract", date=datetime.datetime.now()).save()
          job_stats_obj.end_date = datetime.datetime.now()
          job_stats_obj.total_blogs = article_count
          job_stats_obj.status = "COMPLETED"
          job_stats_obj.save()
          JobLogs(job_stats=job_stats_obj, log="Extraction Done", function_name="extract", date=datetime.datetime.now()).save()
      except Exception as ex:
          JobLogs(job_stats=job_stats_obj, log=str(ex), function_name="extract", date=datetime.datetime.now()).save()
          job_stats_obj.end_date = datetime.datetime.now()
          job_stats_obj.total_blogs = article_count
          job_stats_obj.status = "FAILED"
          job_stats_obj.save()
          JobLogs(job_stats=job_stats_obj, log="Extraction Done", function_name="extract", date=datetime.datetime.now()).save()

  ```
- Crate new view to trigger the task
  - Add below code in views.py file
    ```python
    from members.tasks import extract
    from django.shortcuts import redirect
  
    def python_blog_scraping(request, job_id):
        extract.delay(job_id)
        return redirect('/admin/members/job/')

    ```

- Add Url for above view
  ```python
  path('python_blog_scraping/<int:job_id>', views.python_blog_scraping, name="scraping")
  ```

- Run below curl cmd to trigger the task.
  ```shell
  curl http://0.0.0.0:8000/members/python_blog_scraping/1
  ```
  - Please make sure celery is running in worker container.

### Step 9 : Add Run button in job table to trigger the task
- Add below code inside Job class (admin.py)
  ```shell
  def run(self, obj):
    return format_html('<a class="button" href="{}">RUN</a>', reverse('scraping', args=(str(obj.pk))))
  
  run.short_description = 'Run'
  run.allow_tags = True
  list_display = ("job_name", "start_date", "end_date", "no_of_blogs", "start_no", "created_date", "run", "view_stats")
  ```
- click On **RUN** to trigger the task. `http://0.0.0.0:8000/admin/members/job/`

# Periodic Tasks
Using the celery Beat, we can configure tasks to be run periodically. This can be defined either implicitly or explicitly. The thing to keep in mind is to run a single scheduler at a time. Otherwise, this would lead to duplicate tasks. The scheduling depends on the time zone (CELERYTIMEZONE = "Asia/NewYork") configured in the settings.py
We can configure periodic tasks either by manually adding the configurations to the celery.py module or using the django-celery-beat package which allows us to add periodic tasks from the Django Admin by extending the Admin functionality to allow scheduling tasks.
## Manual Configuration
- Add below code to celery.py
  ```shell
  app.conf.beat_schedule = {
      #Scheduler Name
      'run-task-ten-seconds': {
          # Task Name (Name Specified in Decorator)
          'task': 'extract',
          # Schedule
          'schedule': 60.0,
          # Function Arguments
          'args': (1,)
      }
  }
  ```
  Note : Add Proper **job id** in **args**
- Restart your worker
  ```shell
  python -m celery -A myworld worker  -l info
  ```

## Using django-celery-beat
This extension enables the user to store periodic tasks in a Django database and manage the tasks using the Django Admin interface.  
Please check here : https://django-celery-beat.readthedocs.io/en/latest/

