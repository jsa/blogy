** I think this still looks better in [wiki](http://github.com/jsa/blogy/wiki/Self-testing-AppEngine-apps-with-Saucelabs).**

Self-testing AppEngine apps with Saucelabs
==

Intro
--
You can very easily make you GAE application run Selenium tests on itself with [Saucelabs](http://saucelabs.com/). This is all still very early, but I think the direction may lead to something funky. I tested this idea out just moments ago and wrote this quick note just to let people know of this kind a possibility. I'll write about the implications later when we have more experience. The basic promise is parallel testing fully in the cloud.

![turbofunctor](http://tweetpong.appspot.com/22915523028-500.png)

Story time
--
Okay, so we have this startup thing going on at [Netcycler](http://www.netcycler.fi/) and we've been looking for ways to better automate testing. Having main server infra maintained by the AE team, we have little server maintenance left for ourselves and we'd like to keep it that way. (We have one little box in a corner for backups and analytics.) We found Saucelabs pretty convincing and went kicking the tires today.

The web application is running wild in cyberspace and Selenium is hosted somewhere there as well. The only missing part now is the guy who tells the Selenium army what to do. It's some 5 minute task to set up a VM to run the selenium-driving unittest scripts, but as we're these spoiled brats who want to avoid all maintenance responsibility whatsoever, that would've been a significantly sub-optimal solution.

Basics
--

Selenium drivers communicate via HTTP and we can do HTTP requests from AppEngine. Easy peasy.

Below is some code I scraped and packed from our production code, **but there are some serious catches** — due to AE request handling deadlines, for example. I don't want to cover those here just to get this finished, but just take it with you that **you can split the Selenium script across several tasks** as you don't need to keep the connections open or anything. With this snippet of information, you should be all set.

So, replace the `httplib` stuff in the `do_command(…)` of selenium.py (of selenium-python-client-driver-1.0.1) with this snippet and you're almost there:

    from google.appengine.api import urlfetch
    response = urlfetch.fetch("http://%s:%d/selenium-server/driver/" % (self.host, self.port),
                              payload=body,
                              headers={"Content-Type": "application/x-www-form-urlencoded; charset=utf-8"},
                              method='POST',
                              deadline=10)
    data = response.content

I modified also the `open(…)` to not die on `urlfetch.DownloadError` if browser bootup takes longer than the maximum urlfetch deadline of 10 seconds. (Just log it or something.) And missing the response doesn't matter much in this case as the method doesn't return anything, and Saucelabs doesn't seem to bother.

Then I have a saucetest.py with something like this: 

    import unittest
    
    from django.conf import settings
    from django.utils.simplejson import dumps
    from selenium import selenium
    
    class Smoke(unittest.TestCase):
    
        def setUp(self):
            self.selenium = selenium(
                "saucelabs.com",
                4444,
                dumps({'username': settings.SAUCELABS['ACCOUNT'],
                       'access-key': settings.SAUCELABS['KEY'],
                       'os': "Windows 2003",
                       'browser': "iexplore",
                       'browser-version': "7.",
                       'job-name': self.__class__.__name__,
                       'record-video': True}),
                'http://www.netcycler.fi/')
            self.selenium.start()
    
        def test_login(self):
            sel = self.selenium # This referencing is a bit clumsy, they
                                # could have made it at least fluent...
            sel.open('http://www.netcycler.fi/')
            sel.type("name=email", "tom@te.st") # Doesn't work, no need to try
            sel.type("name=password", "test")
            sel.click("css=#login_form [type='submit']")
            sel.wait_for_page_to_load(10000) # Login is XHR & redirect
            self.assertEqual("Tom Yum", sel.get_text("css=#login .user"))
    
        def tearDown(self):
            self.selenium.stop()

And finally a urls.py like this could do:

    import logging
    from StringIO import StringIO
    import unittest
    from django import http
    from django.conf.urls.defaults import patterns
    
    def smokerun(request):
        suite = unittest.makeSuite(Smoke)
        log_stream = StringIO()
        rs = unittest.TextTestRunner(stream=log_stream, verbosity=2).run(suite)
        return http.HttpResponse("%s:\n%s" % (testcase.__name__, log_stream.getvalue()))
    
    urlpatterns = patterns('',
        (r'^smoke/$', smokerun),
    )

Voilà! Be happy and all.

Future directions
--
1. I'm experimenting on creating a taskqueue chain of testcases.
1. Running tons of different testcases in parallel is trivial.
1. How to automate deployment from Github without middle men? If AppEngine just could deploy on itself.

Misc
--
One other benefit I've come across early is that when you're driving the Seleniums directly from AppEngine, you have a direct connection to datastore from the unittests. This is a chief major benefit. Keep it in mind.

----

[twitter.com/turbofunctor](http://twitter.com/turbofunctor)

[Make a wish](http://www.netcycler.fi/)