+++
date = "2018-02-25"
draft = false
description = "How properly to set up Django + Celery + Sentry logging"
title = "Getting logging to work with Django + Celery + Sentry"
+++

## Celery + Sentry logging is a mess

One of the most frustrating steps in setting up a Django + Celery + [Sentry](https://sentry.io) logging in a way that

- doesn't break your existing Django + Sentry configuration and
- sends log messages from Celery, with tracebacks, to Sentry and
- doesn't send messages from Celery twice.

If you look at the [Sentry](https://github.com/getsentry/sentry/search?utf8=%E2%9C%93&q=celery&type=Issues), [Raven](https://github.com/getsentry/raven-python/search?q=celery&type=Issues&utf8=%E2%9C%93) (a Sentry client), and [Celery](https://github.com/celery/celery/search?q=sentry&type=Issues&utf8=%E2%9C%93) bugtrackers, there are hundreds of issues on the topic. Information is scattered over [dozens](https://github.com/celery/celery/issues/4326) [of](https://github.com/getsentry/raven-python/issues/922) [threads](https://github.com/getsentry/raven-python/issues/903), with solutions ranging from changing some configuration, over creating a custom logging handler, to catching some event signal.

All of this while having [special support for Celery](https://github.com/getsentry/raven-python/tree/master/raven/contrib/celery) in Sentry/Raven.

The issues seem to stem from the fact that Django, Raven, and Celery all make complex changes to the Python runtime: They mess with the logging configuration, and modify tracebacks so they're nicer-to-read. While these changes are of great convenience it looks like they don't work together very well.

## How to get it working

What has worked for us is to disable Celery's logging modifications altogether as described [in this GitHub issue](https://github.com/getsentry/raven-python/issues/922).

Add this to our your Django project's [`celery.py`](http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html) (or some other piece of code that gets executed early in the Django boot sequence):

```py
from celery import Celery

# See https://github.com/getsentry/raven-python/issues/922
@signals.setup_logging.connect
def setup_logging(**kwargs):
    """Skip Celery logging modifications."""
```

This seems to be the recommended solution at the time of writing. We'll keep this tutorial up-to-date when things change in either of the projects.
