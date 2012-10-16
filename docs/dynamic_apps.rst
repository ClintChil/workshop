.. _dynamic_apps:

Building Dynamic Applications
=============================

We've built awesome Twilio applications in the last two sections, but we've
been limited to static TwiML. The true power of Twilio can only be unlocked by
using a web application.

This section assumes you've completed the :ref:`setup` and have the Google App
Engine SDK running locally on your computer.

Your first web application
--------------------------

The first part of this guide walks you through running a sample application.
Before continuing, make sure your "Hello World" app is running and you have
"Hello World" displayed in your browser. If you can't remember how to run the
sample app, refer back to :ref:`setup`.


Before we write our dynamic Twilio application, let's first understand the
"Hello World" example. Let's go through the example line-by-line and see how it
works. Inside our ``main.py`` file:

.. literalinclude:: ../main.py
   :language: python
   :lines: 1


This line is the first part of our application. We use the `webapp2
<http://webapp-improved.appspot.com/>`_ Python module to create our web
application.  Before we can use it in our application, we must first import it.

.. literalinclude:: ../main.py
   :language: python
   :lines: 3-6

Whenever a user makes a request to our application, this is the code that will
be run. The output of the code gets displayed to the web browser. We will name
this block of code "HelloWorld". The name of a block of code is called the
`RequestHandler`.

We are also making a request called ``get``, which grabs the requested
resource.  This corresponds to the HTTP GET request. If you'd like to learn
more about HTTP, the language browsers use to talk to servers, take a look at
our :ref:`http` section.

.. literalinclude:: ../main.py
   :language: python
   :lines: 8-

In this part of our code, we create our application using the `webapp2` framework. 

The web applications is a mapping of the URL we specify with the listed request
handler.  The above mapping says "Whenever someone visits the front page of my
application (the ``/`` URL), process that request using the HelloWorld request
handler".

Your first task will be to change the message displayed in your browser. Open
up ``main.py`` in your text editor and change the "Hello World" message on line
6 to "Hello TwilioCon". Refresh the page to see your new message.

Congratulations! You've just created your first web application.

Responding with TwiML
---------------------

A simple message is great, but we want to use our application to serve TwiML.
How do we respond with TwiML instead of plain text?  First, let's change the
message we respond with to valid TwiML.

.. code-block:: python
   :emphasize-lines: 6

   import webapp2
    
   class HelloWorld(webapp2.RequestHandler):
    
       def get(self):
           self.response.write('<Response><Say>Hello TwilioCon</Say></Response>')
    
    
   app = webapp2.WSGIApplication([
       ('/', HelloWorld),
   ], debug=True)

When someone requests the front page of our application, they will now get TwiML
instead of HTML. If you refresh your page, nothing seems to have
changed. 

The problem is that while we're sending back TwiML, the browser still thinks
we're sending it HTML. Since we never tell the browser that we are sending XML,
which is the format of TwiML, it assumes that we are using HTML. To fix this
problem we'll include additional metadata via an HTTP header to tell the
browser we're sending valid TwiML.

.. code-block:: python
   :emphasize-lines: 6

   import webapp2
    
   class HelloWorld(webapp2.RequestHandler):
    
       def get(self):
           self.response.headers['Content-Type'] = "application/xml"
           self.response.write('<Response><Say>Hello TwilioCon</Say></Response>')

   app = webapp2.WSGIApplication([
       ('/', HelloWorld),
   ], debug=True)

When you refresh the page, you should now see the entire TwiML response, (it
may even be highlighted and formatted).


Using the Twilio Helper Library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Manually writing TwiML soon becomes very tiresome. If you miss a single ending
tag, your entire application can break. Instead, we'll use the
``twilio-python`` helper library to generate TwiML for us. This way we won't
have to worry about messing up the syntax. To use the ``twilio-python`` helper
library, we'll import it from Twilio with the code ``from twilio import twiml``
in line 2.

.. code-block:: python
   :emphasize-lines: 2, 9-11

   import webapp2
   from twilio import twiml
    
   class HelloWorld(webapp2.RequestHandler):
    
       def get(self):
           self.response.headers['Content-Type'] = "application/xml"

           response = twiml.Response()
           response.say("Hello TwilioCon")
           self.response.write(str(response))

   app = webapp2.WSGIApplication([
       ('/', HelloWorld),
   ], debug=True)


When you refresh your page nothing should look different. The helper library
code we just wrote is equivalent to the static TwiML we had before. Let's
explain what the added code is actually doing.

.. code-block:: python

   response = twiml.Response()

Here we create a new Response object. Every Twilio application must begin with
the Response TwiML. We'll add additional TwiML verbs and nest them within the
Response TwiML.

.. code-block:: python

   response.say("Hello TwilioCon")

This methods adds a Say verb to the Response object. The other TwiML verbs
Play, Gather, Record, and Dial may also be used as methods on Response.

.. code-block:: python

   self.response.write(str(response))

Here we turn our response code into a string using Python's built in string
function. This will write a string to the response object and return a response
to Twilio.

The Weather Channel
-------------------

So far all our responses look the same. We're just returning static TwiML with
the same message. Now let's build a dynamic application that interacts with
your inputs. How about building "Weather Channel"! Instead of simply reading a
message, we'll inform the caller of the current weather in his or her ZIP code.

To begin, we need data on the weather. Let's import the weather data from
Yahoo! Weather API, ``from util import current_weather``.  Insert this code in
line 2 right before we import our helper library.

Now that we have data on the weather, let's make sure our application can get
the current weather of a particular ZIP code. Use the following code ``weather =
current_weather("94117")`` so that we can get the weather from San Francisco's
area code 94117. Let's also include the city so we can acknowledge to our
callers where they are getting their weather from, ``city = "San Francisco"``.

Finally, let's add our TwiML and so Twilio knows how to respond to the caller.
In the end your application should look like this:

.. code-block:: python
   :emphasize-lines: 2,10,11,14,15

	import webapp2
	from util import current_weather
	from twilio import twiml

	class HelloWorld(webapp2.RequestHandler):

	    def get(self):
	        self.response.headers['Content-Type'] = "application/xml"

	        weather = current_weather("94117")
	        city = "San Francisco"

	        response = twiml.Response()
	        response.say("Hello from " + city)
	        response.say("The current weather is " + weather)
	        self.response.write(str(response))

	app = webapp2.WSGIApplication([
	    ('/', HelloWorld),
	], debug=True)


When you visit your localhost:8080 page. You'll see the following message.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <Response>
      <Say>Hello from San Francisco</Say>
      <Say>The current weather is Partly Cloudy, 65 degrees</Say>
    </Response>


Let's revisit our application. Right now our application only gets the weather
for San Francisco for the ZIP code 94117.

Phone numbers contain a lot of data though. By using the phone number of your
caller, Twilio can pass the ZIP code and city information to your application.
Twilio labels this information "FromZip" and "FromCity".  Instead of just
getting the weather for San Francisco, let's make this application more
relevant to your caller and use this data. Keep in mind, this isn't actually
the ZIP code of the caller's live location though.

.. code-block:: python
   :emphasize-lines: 8,9

   import webapp2
   from util import current_weather
   from twilio import twiml
    
   class HelloWorld(webapp2.RequestHandler):
    
       def get(self):
           self.response.headers['Content-Type'] = "application/xml"

           weather = current_weather(self.request.get("FromZip", "94117"))
           city = self.request.get("FromCity", "San Francisco")

           response = twiml.Response()
           response.say("Hello from " + city)
           response.say("The current weather is " + weather)
           self.response.write(str(response))

   app = webapp2.WSGIApplication([
       ('/', HelloWorld),
   ], debug=True)


In the case we can't find your ZIP code or city, your application will default
back to providing the weather of San Francisco.

To test out the greeting, add the ``FromZip`` and ``FromCity`` parameter to your URL.

.. code-block:: bash

    http://localhost:8080/?FromZip=15601&FromCity=Greensburg

You should now see the weather for Greensburg, PA show up in your TwiML response.

.. code-block:: xml

   <?xml version="1.0" encoding="UTF-8"?>
   <Response>
     <Say>Hello from Greensburg</Say>
     <Say>The current weather is Cloudy, 59 degrees</Say>
   </Response>

Whenever an HTTP request is sent to your application from Twilio, it includes
data in query string and body of the request. The code we added when
constructing the Say verb pulls that data from the HTTP request parameter.

.. code-block:: python

   self.request.get('FromZip')

Incoming Twilio Data
~~~~~~~~~~~~~~~~~~~~

Adding this parameter to your URL mimics the request that Twilio will send to
your server. All TwiML requests made by Twilio include additional information
about the caller. Here is short list of some of the data that Twilio will send
to your server with every call.

=============== ===========
Parameter       Description
=============== ===========
``From``        The phone number or client identifier of the party that initiated the call. 
``To``          The phone number or client identifier of the called party.
``CallStatus``  A descriptive status for the call. The value is one of queued, ringing, in-progress, completed, busy, failed or no-answer
``FromCity``    The city of the caller.
``FromState``   The state or province of the caller.
``FromZip``     The postal code of the caller.
``FromCountry`` The country of the caller.
=============== ===========

Phone numbers are formatted in E164 format (with a '+' and the country code, e.g.
`+1617555121`).

For a complete list, check out `Twilio request parameters  
<http://www.twilio.com/docs/api/twiml/twilio_request#synchronous-request-parameters>`_ 
on the Twilio Docs.

Handling Server Errors
--------------------------------------------

Sometimes, errors will occur on the web application side of the code.

.. image:: _static/app_error.png

Don't panic if you see this. The stack trace will usually give you hints as to
what error the application encountered, and where it occurred.

Some errors may also appear on the AppEngine logs. If the errors on the browser
aren't too informative, try clicking on the "Logs" button on the AppEngine
Launcher.

Deploy your Twilio application
------------------------------

We're now ready to hook up your brand new application to a Twilio number. To do
this, we'll need to host your application live on the Internet, so that Twilio
can access it!

Open the Google App Engine Launcher application, highlight your application,
and hit the "Deploy" button. A window will pop up and show you the status of
your deployment. It should take less than a minute to deploy.

.. image:: _static/deployapp.png

Once it's deployed, copy the URL for your application,
``http://<your-application-name>.appspot.com`` and set it as the voice number
for your Twilio phone number. Configuring Twilio numbers is covered in more
detail in :ref:`configure-number`.

.. note:: 

   Since we have only implemented the GET endpoint, be sure to configure your
   number to use the GET method instead of the default POST*

Now give it a call. You should hear your custom message. Hooray!

Gathering Digits From the Caller
--------------------------------

Since not everyone's phone number is from the location they currently live,
it may be helpful to add a feature to our app for checking the weather of
any ZIP code. This allows us to interact with our application.

To achieve this, we're going to use a TwiML verb called `<Gather>
<http://www.twilio.com/docs/api/twiml/gather>`_.

Let's begin by adding a ``<Gather>`` menu:

.. code-block:: python
    :emphasize-lines: 11-14

    import webapp2
    from util import current_weather
    from twilio import twiml
    
    class HelloWorld(webapp2.RequestHandler):
    
        def get(self):
            self.response.headers['Content-Type'] = "application/xml"
            city = self.request.get("FromCity", "San Francisco")

            response = twiml.Response()
            gather = response.gather(numDigits=1)
            gather.say("Press one for the weather in " + city)
            gather.say("Press two to get the weather for another zip code.")
            self.response.write(str(response))

The TwiML we've generated so far for the menu looks like this:

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <Response>
        <Gather numDigits="1">
            <Say>Press one for the weather</Say>
            <Say>Press two to get the weather for another zip code.</Say>
        </Gather>
    </Response>

We now have a ``<Gather>`` verb which will allow the caller to type in exactly
1 digit (because of the attribute ``numDigits``). Next, we need to hook it up
to something that can process the input based on which digits were pressed.
When we use ``<Gather>``, these digits are passed to your application in the
next callback using the parameter ``Digits``.

Let's add some additional code to handle this callback.

.. code-block:: python
    :emphasize-lines: 12,17-29

    import webapp2
    from util import current_weather
    from twilio import twiml
    
    class HelloWorld(webapp2.RequestHandler):
    
        def get(self):
            self.response.headers['Content-Type'] = "application/xml"
            city = self.request.get("FromCity", "San Francisco")

            response = twiml.Response()
            gather = response.gather(method="POST", numDigits=1)
            gather.say("Press one for the weather in " + city)
            gather.say("Press two to get the weather for another zip code.")
            self.response.write(str(response))

        def post(self):
            response = twiml.Response()

            weather = current_weather(self.request.get("FromZip", "94117"))

            digit_pressed = self.request.get("Digits")
            if digit_pressed == "1":
                response.say("The current weather is " + weather)
                response.redirect("/", method="GET")
            else:
                gather = response.gather(numDigits=5)
                gather.say("Please enter a 5 digit zip code.")
            self.response.write(str(response))

    app = webapp2.WSGIApplication([
        ('/', HelloWorld),
    ], debug=True)


First, we've specified that the action of the ``<Gather>`` should be an HTTP ``POST``. 

Next, we added some code to our ``webapp2.RequestHandler`` to respond to a
``POST`` request.  Because our first ``<Gather>`` specifies a ``POST`` method
and no ``action``, the default ``action`` is the current URL (In this case,
"/"). So, this code is what will get run after the first ``<Gather>``.

We pull the value ``digit_pressed`` from ``self.request.get("Digits")`` which
corresponds to what the caller pressed. 

In the case that the ``digit_pressed`` was ``"1"``, the behavior looks quite
similar to our earlier example. We then redirect the user back to the beginning
to the menu, so they can try again.

If the caller presses 2, we ask them for 5 more digits. We don't yet have the
logic to process what to do with these 5 more digits, so nothing interesting
will happen when they finish entering these digits.

.. note::

    In this example, we use the ``numDigits`` attribute to know when the is done pressing digits.
    This works because we know we're looking for exactly one digit. If we didn't know this, we could
    use another attribute called ``finishOnKey``.

Let's find out how to read the ZIP code.

.. code-block:: python
    :emphasize-lines: 27,30-42,46

    import webapp2
    from util import current_weather
    from twilio import twiml
    
    class HelloWorld(webapp2.RequestHandler):
    
        def get(self):
            self.response.headers['Content-Type'] = "application/xml"
            city = self.request.get("FromCity", "San Francisco")

            response = twiml.Response()
            gather = response.gather(method="POST", numDigits=1)
            gather.say("Press one for the weather in " + city)
            gather.say("Press two to get the weather for another zip code.")
            self.response.write(str(response))

        def post(self):
            response = twiml.Response()

            weather = current_weather(self.request.get("FromZip", "94117"))

            digit_pressed = self.request.get("Digits")
            if digit_pressed == "1":
                response.say("The current weather is " + weather)
                response.redirect("/", method="GET")
            else:
                gather = response.gather(action="/weather_for_zip", method="POST", numDigits=5)
                gather.say("Please enter a 5 digit zip code.")
            self.response.write(str(response))

    class GetWeather(webapp2.RequestHandler):
        
        def post(self):
            response = twiml.Response()

            zipcode = self.request.get("Digits")
            weather = current_weather(zipcode)

            response.say("The current weather is " + weather)
            response.redirect("/", method="GET")
            
            self.response.write(str(response))

    app = webapp2.WSGIApplication([
        ('/', HelloWorld),
        ('/weather_for_zip', GetWeather),
    ], debug=True)

We've added a second ``webapp2.RequestHandler`` class. We also configure this
handler to respond to the URL ``/weather_for_zip`` and changed the ``<Gather>``
to point to it.

When the caller has entered 5 digits, Twilio will do a ``POST`` request to
``/weather_for_zip`` with the digits pressed passed as the ``Digits`` argument.
We use these digits to look up the weather, just as we did for the original app
with the ``FromZipCode`` passed in by Twilio.

Now, deploy your application again, and try it out!
