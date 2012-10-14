.. _browser:

Into the Browser
================

Using Twilio Client
-------------------

Using Twilio Client, we can be hooked into the full power of the Twilio API
from your Web Browser. This includes the ability to make and receive phone
calls, opening up the world of telephony to your dynamic web applications.

Let's try writing a web app that is capable of answering phone calls in a
Twilio <Queue>. This way, our Radio DJ won't be required to use their personal
phone when answering queues.

How Client Works
-----------------

This is what an outbound Client call looks like:

1. Your server creates a Capability Token with the Application Sid you would
   like to use to handle outbound calls. 
2. User triggers a `connect()` action in the `twilio.js` Javascript library.
3. Twilio looks up the Application Sid for the Client, and retrieves the Voice
   URL for that application.
4. Twilio makes an HTTP request to the Voice URL and plays the TwiML it
   retrieves back to the user's browser.

To connect an inbound call to your Client browser, generate a Capability Token
that allows incoming connections. Then return this TwiML in response to an
inbound call:

.. code-block:: xml

    <Response>
        <Dial>
            <Client>client-name</Client>
        </Dial>
    </Response>

Creating an Application
-----------------------

We are going to reuse the application we created in the previous example. Set
the Voice Request URL to a new endpoint for the TwiML we want to be executed
when the DJ's browser connects (this should be the same URL as the DJ's dial-in
number).

Generating a Token
------------------

Since Twilio Client applications are being run on the browser, we need to grant
the end user temporary privileges on our Twilio Account. This is the job of
`Capability Tokens <https://www.twilio.com/docs/client/capability-tokens>`_.
Capability Tokens allow us to lock down access to what we want the end user's
session to be able to do. For our needs we only need to add access to making an
outgoing connection to our new Application.

Here is the function we'll use for generating the Capability Token.

.. code-block:: python

    from twilio.util import TwilioCapability

    def gen_token(account_sid, auth_token, application_sid):
        capability = TwilioCapability(account_sid, auth_token)
        # Allow access to the Call-in ApplicationSid we created
        capability.allow_client_outgoing(application_sid)
        return capability.generate()


Answering Queues in the Browser
-------------------------------

The first thing we'll need to build is a web interface. Let's start by adding a
new AppEngine RequestHandler into ``main.py``.

.. code-block:: python

    class IndexPage(webapp2.RequestHandler):

        def get(self):
            params = {
                "token": gen_token(ACCOUNT_SID, AUTH_TOKEN, APP_SID)
            }
            self.response.out.write(render_template("index.html", params))


Here is the ``index.html`` file we are rendering.

.. code-block:: html

	<!DOCTYPE html>
	<html>
	  <head>
		<title>Hello Client Monkey 4</title>
		<script type="text/javascript"
		  src="http://static.twilio.com/libs/twiliojs/1.0/twilio.min.js"></script>
		<script type="text/javascript"
		  src="https://ajax.googleapis.com/ajax/libs/jquery/1.7.2/jquery.min.js">
		</script>
		<link href="http://static0.twilio.com/packages/quickstart/client.css"
		  type="text/css" rel="stylesheet" />
		<script type="text/javascript">

		  // Render the token we generated on the server side. 
		  Twilio.Device.setup("{{ token }}");

		  Twilio.Device.ready(function (device) {
			$("#log").text("Ready");
		  });

		  Twilio.Device.error(function (error) {
			$("#log").text("Error: " + error.message);
		  });

		  Twilio.Device.connect(function (conn) {
			$("#log").text("Successfully established call");
		  });

		  Twilio.Device.disconnect(function (conn) {
			$("#log").text("Call ended");
		  });

		  Twilio.Device.incoming(function (conn) {
			$("#log").text("Incoming connection from " + conn.parameters.From);
			// accept the incoming connection and start two-way audio
			conn.accept();
		  });

		  function call() {
			// get the phone number to connect the call to
			params = {"PhoneNumber": $("#number").val()};
			Twilio.Device.connect(params);
		  }

		  function hangup() {
			Twilio.Device.disconnectAll();
		  }
		</script>
	  </head>
	  <body>
		<button class="call" onclick="call();">
		  Call
		</button>

		<button class="hangup" onclick="hangup();">
		  Hangup
		</button>

		<input type="text" id="number" name="number"
		  placeholder="Enter a phone number to call"/>

		<div id="log">Loading pigeons...</div>
	  </body>
	</html>

There are two important lines in the Javascript that make this work:

.. code-block:: javascript

    Twilio.Device.setup("{{ token }}");

The above line of code calls ``Twilio.Device.setup`` and uses our templating
engine to pass in a valid Capability Token. When ``setup`` finishes, the
function passed into ``Twilio.Device.ready`` will fire to let the browser know
that Twilio has initialized access to the microphone, speakers, and we've
started listening for incoming calls (if applicable).

.. code-block:: javascript

    function call() {
      Twilio.Device.connect();
    }

This code defines a new function called ``call`` that just wraps
``Twilio.Device.connect``, which initiates an outgoing call to the Application
we created earlier. In this case, calling ``call()`` should execute the TwiML
below. We've made a small change so that the DJ can press "#" to end the
current call, and dial the next person in the queue.

.. code-block:: xml

    <?xml version="1.0" encoding="UTF-8"?>
    <Response>
        <Dial finishOnKey="#">
            <Queue>radio-callin-queue</Queue>
        </Dial>
        <Redirect></Redirect>
    </Response>

Change your application's Voice URL so it serves this TwiML when dialed.

Getting the Next Caller From the <Queue>
-----------------------------------------

With <Queue>s, we can hangup on the current caller and move to the next one by
pressing the "#" key on the phone. Luckily, Twilio Client has a feature for
sending DTMF tones programmatically.

First, we need to hold on to the response of ``Twilio.Device.connect()`` so
let's add a global variable called ``connection`` and have every ``call()``
command set it. Replace the existing ``call`` function with something like
this:

.. code-block:: javascript

    var connection = null;
    function call() {
        connection = Twilio.Device.connect();
    }

Now, we can add a new function, called ``next()``:

.. code-block:: javascript

    function next() {
        if (connection) {
            connection.sendDTMF("#");
        }
    }

Because we added a `finishOnKey` attribute to our TwiML, sending a "#" symbol
via DTMF tone will hang up on the current caller, and connect the browser to
the next caller. 

Now we just need to add another button to trigger the hangup.

.. code-block:: html

    <button class="next" onclick="next();">
        Next Caller
    </button>

Adding UI To Display the Queue
------------------------------

Let's add a feature where we can see a visualization of the queue. We'll add
a new queue status endpoint, which will return the current queue status as
JSON.

.. code-block:: python

    import json
    from twilio import TwilioRestClient

    class QueueStatusPage(webapp2.RequestHandler):

        queue_sid = "QQ123"
        def get(self):
            client = TwilioRestClient(ACCOUNT_SID, AUTH_TOKEN)
            q_data = {"queues": client.queues.get(queue_sid)}
            self.response.out.write(json.dumps(q_data))


Add this QueueStatusPage into the WSIApplication's routing map as
``/queue-status``. Now we need some Javascript to poll the state of the queue
and update the UI.

.. code-block:: javascript

    $.get("/queue-status", function(result) {
        // update your UI here
    });

