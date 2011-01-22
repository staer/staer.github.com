---
layout: post
title: Multiple Authentication with Django-Piston
---

One of the problems that I hit with [django-piston](https://bitbucket.org/jespern/django-piston/wiki/Home)
recently was that I needed to support more than a single type of authentication 
for my REST API. In one case I needed an authentication handler that authenticated
against the the Django auth system itself so that I could use the API to dynamically
populate choice fields on the fly using the currently logged in website user.
An implementation of a DjangoAuthentication piston module can be found [here](https://bitbucket.org/yml/django-piston/src/dfb826a31ca8/piston/authentication.py).
The other type of auth handler I needed was the HttpBasicAuthentication (built into piston) handler so that 
3rd party tools could access the exact same API that the website was using. 

Django-Piston does not by default allow more than one authentication system to work together out
of the box, so the following is a quick and dirty implementation of piston authentication
handler that authenticates using a list of different auth modules:

	{% highlight python %}
	class MultiAuthentication(object):
	    """ Authenticated Django-Piston against multiple types of authentication """

	    def __init__(self, auth_types):
	        """ Takes a list of authenication objects to try against, the default
	        authentication type to try is the first in the list. """
	        self.auth_types = auth_types
	        self.selected_auth = auth_types[0]

	    def is_authenticated(self, request):
	        """ Try each authentication type in order and use the first that succeeds """
	        authenticated = False
	        for auth in self.auth_types:
	            authenticated = auth.is_authenticated(request)
	            if authenticated:
	                selected_auth = auth
	                break
	        return authenticated

	    def challenge(self):
	        """ Return the challenge for whatever the selected auth type is (or the default 
	        auth type which is the first in the list)"""
	        return self.selected_auth.challenge()
	{% endhighlight %}

The way this works is that you pass in a list of different authentication types
to the MultiAuthentication constructor which you want to test against. The first
item in the list is considered the 'default'. When authentication each auth module
in the list is tried in order until one succeeds or they all fail. If each method 
fails then the 'default' challenge will be returned to the user. Example usage:

	{% highlight python %}
	from piston.authentication import HttpBasicAuthentication
	from somewhere.else import DjangoAuthentication
	
	# Authenticate against HttpBasic and then Django if basic fails. If both 
	# methods fail, ask the user for username/password via HttpBasic
	basicAuth = HttpBasicAuthentication(realm="Test")
	djangoAuth = DjangoAuthentication()
	auth = MultiAuthentication([basicAuth, djangoAuth])
	{% endhighlight %}
	
This is available in gist form at [https://gist.github.com/790222](https://gist.github.com/790222)
