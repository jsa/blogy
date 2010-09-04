Self-testing AppEngine apps on Saucelabs
==

Intro/spoiler
--
You can very easily make you GAE application run Selenium tests on
itself with http://saucelabs.com/. It's actually trivial, so you could as well
go for a pony ride or something than continue reading.

Anyways. This is all still very early, but I think it may have some traces of
epic already. (I came up with this in the last few hours.)

<!-- http://twitter.com/hugs/status/22922750764 --> <style type='text/css'>.bbpBox22922750764 {background:url(http://s.twimg.com/a/1283473056/images/themes/theme1/bg.png) #1A1B1F;padding:20px;} p.bbpTweet{background:#fff;padding:10px 12px 10px 12px;margin:0;min-height:48px;color:#000;font-size:18px !important;line-height:22px;-moz-border-radius:5px;-webkit-border-radius:5px} p.bbpTweet span.metadata{display:block;width:100%;clear:both;margin-top:8px;padding-top:12px;height:40px;border-top:1px solid #fff;border-top:1px solid #e6e6e6} p.bbpTweet span.metadata span.author{line-height:19px} p.bbpTweet span.metadata span.author img{float:left;margin:0 7px 0 0px;width:38px;height:38px} p.bbpTweet a:hover{text-decoration:underline}p.bbpTweet span.timestamp{font-size:12px;display:block}</style> <div class='bbpBox22922750764'><p class='bbpTweet'>HOLY F'N COW! RT @<a class="tweet-url username" href="http://twitter.com/turbofunctor" rel="nofollow">turbofunctor</a> Wow. Epic. First @<a class="tweet-url username" href="http://twitter.com/netcycler" rel="nofollow">netcycler</a> “self-test” passed by driving @<a class="tweet-url username" href="http://twitter.com/saucelabs" rel="nofollow">saucelabs</a> *directly* from @<a class="tweet-url username" href="http://twitter.com/app_engine" rel="nofollow">app_engine</a>.<span class='timestamp'><a title='Fri Sep 03 21:46:09 +0000 2010' href='http://twitter.com/hugs/status/22922750764'>less than a minute ago</a> via web</span><span class='metadata'><span class='author'><a href='http://twitter.com/hugs'><img src='http://a1.twimg.com/profile_images/60653485/jason_normal.jpg' /></a><strong><a href='http://twitter.com/hugs'>Jason Huggins</a></strong><br/>hugs</span></span></p></div> <!-- end of tweet -->

Story time
--
Okay, so we have this startup thing going on at http://www.netcycler.fi/ and
we've been looking for ways to better automate the testing. Having the main
server infra maintained by the AE team, we have little server maintenance left
for ourselves and we'd like to keep it that way. (We have one little box in a
corner for backups and analytics.) We found Saucelabs pretty convincing and
went kicking the tires today.

<!-- http://twitter.com/turbofunctor/status/22896275646 --> <style type='text/css'>.bbpBox22896275646 {background:url(http://a1.twimg.com/profile_background_images/2677032/bg_19779.jpg) #9ae4e8;padding:20px;} p.bbpTweet{background:#fff;padding:10px 12px 10px 12px;margin:0;min-height:48px;color:#000;font-size:18px !important;line-height:22px;-moz-border-radius:5px;-webkit-border-radius:5px} p.bbpTweet span.metadata{display:block;width:100%;clear:both;margin-top:8px;padding-top:12px;height:40px;border-top:1px solid #fff;border-top:1px solid #e6e6e6} p.bbpTweet span.metadata span.author{line-height:19px} p.bbpTweet span.metadata span.author img{float:left;margin:0 7px 0 0px;width:38px;height:38px} p.bbpTweet a:hover{text-decoration:underline}p.bbpTweet span.timestamp{font-size:12px;display:block}</style> <div class='bbpBox22896275646'><p class='bbpTweet'>@<a class="tweet-url username" href="http://twitter.com/netcycler" rel="nofollow">netcycler</a> 's first unittest on @<a class="tweet-url username" href="http://twitter.com/saucelabs" rel="nofollow">saucelabs</a> passes -- still stunned by how well the service works. (w/ python client.) &lt;3 video features.<span class='timestamp'><a title='Fri Sep 03 15:24:37 +0000 2010' href='http://twitter.com/turbofunctor/status/22896275646'>less than a minute ago</a> via web</span><span class='metadata'><span class='author'><a href='http://twitter.com/turbofunctor'><img src='http://a2.twimg.com/profile_images/59091250/sweded2_normal.png' /></a><strong><a href='http://twitter.com/turbofunctor'>Janne Savukoski</a></strong><br/>turbofunctor</span></span></p></div> <!-- end of tweet -->

So, we have this web application running wild in cyberspace and Selenium hosted
somewhere there as well. The only missing part now is the guy who tells
the Selenium army what to do. (“The driver”, for the sake of techno parlance.)

It's some 5 minute task to set up a VM to run the selenium-driving unittest
scripts, but as we're these spoiled brats who want to avoid all maintenance
responsibility whatsoever, that would've been a significantly sub-optimal solution.

Basics
--

Solution? Selenium drivers communicate via HTTP and we can do HTTP request from
AppEngine. Easy peasy.

We're using Python (thank you very much) as are snippets below so written.
(I'm not mentioning the other option to avoid a cease and desist from a certain
leisure suit.)

Below is some code I scraped and packed for your entertainment, **but there are
some serious catches**, due to AE request handling deadlines etc. I don't
want to cover those here just to get this finished, but just take it with you
that **you can split the Selenium script across several tasks** as you don't
need to keep the connections open or anything. With this snippet of information,
you should be all set.

So, replace the `httplib` stuff in the `do_command(…)` of *selenium.py* (of
*selenium-python-client-driver-1.0.1*) with this snippet and you're almost
there:

    from google.appengine.api import urlfetch
    response = urlfetch.fetch("http://%s:%d/selenium-server/driver/" % (self.host, self.port),
                              payload=body,
                              headers={"Content-Type": "application/x-www-form-urlencoded; charset=utf-8"},
                              method='POST',
                              deadline=10)
    data = response.content

I modified also the `open(…)` to not die on `urlfetch.DownloadError` if
browser bootup takes longer than the maximum urlfetch deadline of 10 seconds.
(Just log it or something.)

Below is some pseudo-y code (I had to scrape it from our production code).

Then I have a *saucetest.py* with something like this: 

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


And finally a *urls.py* like this could do:

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

Misc
--
One other benefit I've come across early is that when you're driving the
Seleniums directly from AppEngine, you have a direct connection to datastore
from the unittests. This is a chief major benefit. Keep it in mind.

----

[twitter.com/turbofunctor](http://twitter.com/turbofunctor)
[Make a wish](http://www.netcycler.fi/)