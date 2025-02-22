=======================
Django Background Tasks
=======================


.. image:: https://travis-ci.org/arteria/django-background-tasks.svg
    :target: https://travis-ci.org/arteria/django-background-tasks

Django Background Task is a databased-backed work queue for Django_, loosely based around Ruby's DelayedJob_ library. This project was adopted and adapted from this_ repo.

To avoid conflicts on PyPI we renamed it to django-background-tasks (plural). For an easy upgrade from django-background-task to django-background-tasks, the internal module structure were left untouched.

In Django Background Task, all tasks are implemented as functions (or any other callable).

There are two parts to using background tasks:

* creating the task functions and registering them with the scheduler
* setup a cron task (or long running process) to execute the tasks


Installation
============

1. Install from PyPI::

    pip install django-background-tasks

2. Add to INSTALLED_APPS::

    INSTALLED_APPS = (
        # ...
        'background_task',
        # ...
    )

(3.) Autodiscover tasks in your apps' ``tasks.py`` file. Add the following to your base ``urls.py`` so that autodiscover runs when your project starts up::

    from background_task import tasks
    tasks.autodiscover()
    
Do this in case you are not using the management command to execute the tasks.


Supported versions and compatibility
====================================
* Python: 2 & 3
* Django: 1.4 - 1.8

Full Django LTS to LTS compatibility through django-compat_.

Creating and registering tasks
==============================

To register a task use the background decorator::

    from background_task import background
    from django.contrib.auth.models import User

    @background(schedule=60)
    def notify_user(user_id):
        # lookup user by id and send them a message
        user = User.objects.get(pk=user_id)
        user.email_user('Here is a notification', 'You have been notified')

This will convert the notify_user into a background task function.  When you call it from regular code it will actually create a Task object and stores it in the database.  The database then contains serialised information about which function actually needs running later on.  This does place limits on the parameters that can be passed when calling the function - they must all be serializable as JSON.  Hence why in the example above a user_id is passed rather than a User object.

Calling notify_user as normal will schedule the original function to be run 60 seconds from now::

    notify_user(user.id)

This is the default schedule time (as set in the decorator), but it can be overridden::

    notify_user(user.id, schedule=90) # 90 seconds from now
    notify_user(user.id, schedule=timedelta(minutes=20)) # 20 minutes from now
    notify_user(user.id, schedule=timezone.now()) # at a specific time

Running tasks
=============

There is a management command to run tasks that have been scheduled::

    python manage.py process_tasks

This will simply poll the database queue every few seconds to see if there is a new task to run.

NB: to aid the management task in finding the registered tasks it is best to put them in a file called 'tasks.py'.  You can put them elsewhere, but you have to ensure that they will be imported so the decorator can register them with the scheduler.  By putting them in tasks.py they will be auto-discovered and the file automatically imported by the management command.

The process_tasks management command has the following options:

* `duration` - Run task for this many seconds (0 or less to run forever) - default is 0
* `sleep` - Sleep for this many seconds before checking for new tasks (if none were found) - default is 5
* `log-file` - Log file destination
* `log-std` - Redirect stdout and stderr to the logging system
* `log-level` - Set logging level (CRITICAL, ERROR, WARNING, INFO, DEBUG)

You can use the `duration` option for simple process control, by running the management command via a cron job and setting the duration to the time till cron calls the command again.  This way if the command fails it will get restarted by the cron job later anyway.  It also avoids having to worry about resource/memory leaks too much.  The alternative is to use a grown-up program like supervisord_ to handle this for you.

Settings
========

There are a few settings options that can be set in your `settings.py` file.

* `MAX_ATTEMPTS` - controls how many times a task will be attempted (default 25)
* `MAX_RUN_TIME` - maximum possible task run time, after which tasks will be unlocked and tried again (default 3600 seconds)
* Set ``BACKGROUND_TASK_RUN_ASYNC = True`` in the project setting will run the tasks asynchronous. This means the tasks will be processed in parallel (at the same time) instead of processing one by one (one after the other).

Task errors
===========

Tasks are retried if they fail and the error recorded in last_error (and logged).  A task is retried as it may be a temporary issue, such as a transient network problem.  However each time a task is retried it is retried later and later, using an exponential back off, based on the number of attempts::

    (attempts ** 4) + 5

This means that initially the task will be tried again a few seconds later.  After four attempts the task is tried again 261 seconds later (about four minutes).  At twenty five attempts the task will not be tried again for nearly four days!  It is not unheard of for a transient error to last a long time and this behavior is intended to stop tasks that are triggering errors constantly (i.e. due to a coding error) form dominating task processing.  You should probably monitor the task queue to check for tasks that have errors.  After `MAX_ATTEMPTS` the task will be marked as failed and will not be rescheduled again.

Known issues
============

* ``django.db.utils.OperationalError: database is locked`` when using SQLite. This is a SQLite specific error, see https://docs.djangoproject.com/en/dev/ref/databases/#database-is-locked-errors for more details.



Example project
===============

Hiroaki Nakamura has written an example project demonstrating how django-background-tasks works. You find it here_.



.. _Django: http://www.djangoproject.com/
.. _DelayedJob: http://github.com/tobi/delayed_job
.. _supervisord: http://supervisord.org/
.. _this: https://github.com/lilspikey/django-background-task
.. _compat:  https://github.com/arteria/django-compat
.. _django-compat:  https://github.com/arteria/django-compat
.. _25: https://github.com/arteria/django-background-tasks/issues/25
.. _here: https://github.com/hnakamur/django-background-tasks-example/
