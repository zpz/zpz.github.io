---
layout: post
title: Sending Pipeline Alerts from Python to Slack
---

For recurrent pipelines, it is a common requirement to send notifications,
or alerts, especially when error occurs. Where to send the notifications?
E-mail comes to mind. However, everyone knows e-mail is a *bad* channel
for error alerts. In addition, sending emails from code is not entirely trivial---one needs to deal with SMTP server, as well as e-mail account and authentication, and stuff.

My colleage pointed me to a solution: first, let the program write a *status file* containing a status value, either 'OK' or 'CRITICAL'; then totally separately, configure [Icinga 2](https://www.icinga.com/) to scan this status file at specified times and frequencies, and send out alerts to e-mail, Slack, or wherever, under rules specified based on the status.

The crux of the idea---separate alerting from the program---was enlightening to me. When configuring Icinga to do the job, however, I encountered a number of inflexibilities and restrictions. 

I decided to take control of sending the alerts myself (my code itself, that is). This would allow me to write whatever I like in the message, format it however I like, and specify arbitrary alerting rules in the code. The idea of notifying a Slack channel rather than an e-mail account stayed; this would turn out to be easy, as I'll show below. The idea of writing a status file also stayed, as this would enable alerting rules based on both the current status and the *previous* status, as well as other info of the previous status (mainly the timestamp).

The notifier is designed as a Python decorator to be applied to any function that could raise an exception:


```python
from datetime import datetime
import functools
import inspect
import logging
import os
from pathlib import Path
from traceback import format_exc
from typing import Union, Tuple, List, Callable


def notify(exception_classes: Union[Exception, Tuple[Exception], List[Exception]] = None,
           slack_channel: str = 'alerts',
           silent_seconds: Union[float, int] = 1,
           ok_silent_hours: Union[float, int] = 0.99) -> Callable:
    '''
    A decorator for writing a status file for a function, and sending alerts to Slack.
   
    `slack_channel`, `silent_seconds`, `ok_silent_hours`, in combination with
    the current and previous statuses, determine whether to send alert to Slack.
    See `should_send_alert` for details.
   
    Args:
        exception_classes: exception class object, or tuple or list of multiple classes,
            to be captured; if `None`, all exceptions will be captured.
        
        slack_channel: if value is a supported channel, send alert to Slack if certain conditions
            are met. To suppress Slack notification, pass in '' or `None`.
        
        silent_seconds: if new status is identical to the previous one and the previous
            status was written within the last `silent_seconds` seconds, do not send alert.
        
        ok_silent_hours: if new and previous statuses are both 'OK' and the previous status
            was written within the last `ok_silent_hours` hours, do not send alert.
    
    The status file is located in the directory specified by the environ variable `NOTIFYDIR`.
    The file name is constructed by the package/module of the decorated function as well as the function's name.
    For example, if function `testit` in package/module `models.regression.linear` is being decorated,
    then the status file is `models.regression.linear.testit` in the notify directory.
    
    If the decorated function is in a launching script (hence its `module` is named `__main__`),
    then the full path of the script is used to construct the status file's name.
    For example, if function `testthat` in script `/home/docker-user/work/scripts/do_this.py` is being decorated,
    then the status file is `home.docker-user.work.scripts.do_this.py.testthat` in the notify directory.
    
    This decorator writes 'OK' in the status file if the decorated function returns successfully.
    If the decorated function raises an exception of the types specified by `exception_classes`
    (or any exception if `exception_classes` is `None`), `ERROR` is written along with some additional info.
    
    This decorator does not write logs. If you wish to log the exception, you must handle that separately.
    If you handle the exception within the function, make sure you re-`raise` the exception so that this decorator
    can capture it.
    
    Usually you only need to decorate the top-level function in a pipeline's launching script, like this:
        
        # launcher.py
        
        @notify()
        def main():
            # do things that could raise exeptions
            # ...
        
        if __name__ == '__main__':
            main()
    
    Use this decorator at more refined places when you want to handle a certain exception
    and then continue the program, but also want to notify about that exception, like this:
    
        # module `abc.py` in package `proj1.component2`
    
        class MySpecialError(Exception):
            pass
    
        @notify(MySpecialError)    
        def func1():
            # do things that could raise MySpecialError
            # ...
            # result = ...
            if result is None:
                raise MySpecialError('omg!')
           
            #...
            #...
    
        def func2():
            try:
                func1()
                #...
            except MySpecialError as e:
                logger.info('MySpecialError has occurred!')
                return 3    # let program continue to run
    '''
    if not exception_classes:
        exception_classes = (Exception, )
    elif issubclass(exception_classes, Exception):
        exception_classes = (exception_classes, )
    else:
        if isinstance(exception_classes, list):
            exception_classes = tuple(exception_classes)
        assert isinstance(exception_classes, tuple)
        assert all(issubclass(v, Exception) for v in exception_classes)

    def decorator(func):
        module = func.__module__
        if module == '__main__':
            module = str(Path.cwd() / inspect.getsourcefile(func))
        decloc = module + ' :: ' + func.__name__
        fname = module.strip('/').replace('/', '.') + '.' + func.__name__
        fdir = os.environ['NOTIFYDIR']
        Path(fdir).mkdir(parents=True, exist_ok=True)

        notifile = Path(fdir) / fname

        @functools.wraps(func)
        def decorated(*args, **kwargs):
            dt = datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S:%f')
            try:
                z = func(*args, **kwargs)
                status = 'OK'
                msg = '{} UTC\n{}\n'.format(dt, decloc)
                return z
            except exception_classes as e:
                status = 'ERROR'
                msg = '{} UTC\n{}\n\n{}\n'.format(dt, decloc, format_exc())
                raise
            except:
                status = 'OK'
                msg = '{} UTC\n{}\n'.format(dt, decloc)
                raise
            finally:
                if should_send_alert(status, notifile, silent_seconds,
                                     ok_silent_hours):
                    notify_slack(slack_channel, status, msg)
                open(notifile, 'w').write(status + '\n' + msg)

        return decorated

    return decorator
```

Two things are worth noting:

1. If the user specifies particular types of exceptions, then the status is 'ERROR'
   only when an exception of the specified type(s) is raised.
2. The name of the status file is constructed based on the names of the package/module/file and the name of the function being decorated. This ensures that two applications of this decorator function will write different status files.

This decorator function calls `should_send_alert` to determine whether a notification should be sent:

```python
def should_send_alert(status: str, 
                      ff: Path, 
                      silent_seconds: Union[float, int],
                      ok_silent_hours: Union[float, int]) -> bool:
    if not ff.exists():
        return True

    old_status, old_dt, *_ = open(ff).read()[:50].split('\n')
    old_date, old_time, *_ = old_dt.split()

    if old_status != status:
        return True

    t0 = datetime.strptime(old_date + ' ' + old_time, '%Y-%m-%d %H:%M:%S:%f')
    t1 = datetime.utcnow()
    lapse = (t1 - t0).total_seconds()

    if lapse < float(silent_seconds):
        return False

    if status != 'OK':
        return True

    if lapse > float(ok_silent_hours) * 3600.:
        return True

    return False
```

In words, the alerting rule can be summarized as follows:

- If status file does not exist (e.g. this is the first time the decorated function is executed), alert.
- If the current status differs from the previous status (i.e. one is 'OK' and the other is 'ERROR'), alert.
- If both statuses are 'OK', and the last status file is older than a specified age, alert; otherwise, do not alert.
- If both statuses are 'ERROR', and the last status file is older than a specified (short) age, alert; otherwise, do not alert.

Sending notification to Slack is handled by the function `notify_slack`:

```python
import json
import  logging
import threading
import urllib.request

logger = logging.getLogger(__name__)

SLACK_NOTIFY_CHANNELS = ['alerts']
# Add other supported channels.

def notify_slack(slack_channel: str, status: str, msg: str) -> None:
    if slack_channel not in SLACK_NOTIFY_CHANNELS:
        logger.error(
            'the requested Slack notification channel, %s, is not supported',
            slack_channel)
        return

    slack_channel = slack_channel.replace('-', '').replace('_', '').upper()
    url = os.environ['SLACK_' + slack_channel + '_WEBHOOK_URL']
    json_data = json.dumps({
        'text': '--- {} ---\n{}'.format(status, msg)
        }).encode('ascii')
    req = urllib.request.Request(
        url, data=json_data, headers={'Content-type': 'application/json'})
    thr = threading.Thread(target=urllib.request.urlopen, args=(req, ))
    try:
        thr.start()
    except Exception as e:
        logger.error('failed to send alert to Slack:\n%s', str(e))
```

There are Python packages for interacting with the Slack API.
Our use case is the simplest: we do not need to appear like a human being and interact with other users; we just need to post a message to a particular channel in a one-off fashion.
For this purpose, using the Slack API is not necessary. A simpler solution involves 
creating a Slack 'bot' that is authorized to post messages to a specified channel.
Simply follow [these instructions](https://get.slack.help/hc/en-us/articles/115005265703-Create-a-bot-for-your-workspace).
After a few steps, you will get a URL.
This URL is styled like a random string or hashed string---the fact that you know it indicates you are an "insider". Subsequently, HTTP-posting to this URL amounts to posting messages to the specific Slack channel,
and no authentication is needed.
In the function above, we assume this URL is stored in an environment variable.

Two other note-worthy considerations in `notify_slack` are:

- Posting to Slack is handled in a separate thread in an async fashion, to avoid blocking the flow of the main program.
- If this posting fails for some reasion, log some info and let the program continue.

And that's it.

This solution does not use anything outside of the standard library.
The structure of this solution is quite general.
To send notifications to places other than Slack, we only need to replace `notify_slack`.
The code that determines whether to send notification, and what notification to send, stays unchanged.
