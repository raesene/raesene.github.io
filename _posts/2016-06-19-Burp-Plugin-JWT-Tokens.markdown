---
layout: post
title: "Burp Plugin for use with JWT Tokens"
date: 2016-06-19 10:01:39 +0100
comments: false
categories: Burp
---

One of the things that you get used to after using [Burp](https://portswigger.net/) for a while is that if there's any area that it doesn't have native functionality for, it's possible to use Extender to code up your own.  I had cause to do a bit of this recently and as with the previous time I looked at this ([for passive scanner checks](https://raesene.github.io/blog/2015/01/15/burp-passive-scanner-plugins-with-jruby/)) there were some gaps in the documentation for doing this with JRuby, so I thought I'd write it up.

This time, I was looking to automate the process of getting a new JWT token as a session handling rule for an application that needed a new token with each request.  Burp has pretty good session handling rules, but AFAIK at the moment they don't cover the scenario where you need to embed your value into an HTTP header (as you do with JWT tokens sometimes).

To do this what I did was write some code that implemented the ISessionHandlingAction interface using JRuby.  It's a pretty simple interface, just two methods, so it wasn't too tricky to get working.

The main action you need to look at is called ```performAction``` which takes two parameters which represent the request to be modified and the results of any macros that have been run.  That second part is important as using a macro is how we're going to get our token to inject into the request.

So first we set a couple of variables with the information from our two input requests.  We use these later on to get information about the requests that we need to extract data from them

``` ruby
#This analyses the request that we're going to modify
request_info = @helpers.analyzeRequest(baseRequestResponse)
#This gets the first response from a macro item... should work for the basic case
macro_response_info = @helpers.analyzeResponse(macroItems[0].getResponse())
```

The next bit is to get the token out of our macro response.  the idea of this process is that we use a burp macro to replay a request which generates a new token and then we extract it and insert it into new requests, so we'll need that token.

``` ruby
#Extract the JWT token from the macro response
macro_msg = macroItems[0].getResponse()
macro_body_offset = macro_response_info.getBodyOffset()
macro_body = macro_msg[macro_body_offset..-1]
macro_body_string = @helpers.bytesToString(macro_body)
jwt_token = JSON.parse(macro_body_string)
jwt = jwt_token["jwt"]
```
Next up we need to ensure that any existing Authorization header is removed before we add our new one.  This code is a bit brittle at the moment as it doesn't account for cases where you have other authorization headers you might want (e.g. an HTTP basic one) or cases where there are multiple headers to remove.

One thing I found a little tricky in this, coming from a ruby background, was the handling of removing an element from the array.  If you were doing this in ruby you might use ```delete!``` to remove the element, but it's important to note that the getHeaders call doesn't return a ruby array, it returns a Java ArrayList which has a different set of methods altogether.

``` ruby
#Get the headers from our base request
headers = request_info.getHeaders()
#we need a ref for the existing authorisation header if any to delete
auth_to_delete = ''
#So headers is an ArrayList so no ruby delete methods first iterate over and get our header
headers.each do |head|
  if head =~ /Authorization: JWT/
    auth_to_delete = head
  end
end
#then remove the header if it exists 
headers.remove(auth_to_delete)
```
Then with our new JWT token in hand and any previous one removed we just add our token into the request and send it on its way

``` ruby
#Add in our new authorization header
headers.add('Authorization: JWT ' + jwt)
#We need to get the body to add to our headers which is what the next three lines do
msg = baseRequestResponse.getRequest()
body_offset = request_info.getBodyOffset()
body = msg[body_offset..-1]
#Now we can create our new message with the headers and body that we need
new_message = @helpers.buildHttpMessage(headers, body)
#Lets just log something so we know it's doing something
@stdout.println("Changed a message")
#Set our Request to be the modified version from our code.
baseRequestResponse.setRequest(new_message)
```

Once we've got this plugin working, we can add it to a session handling rule in Burp's project options.  You set the action to be "run macro" and then in macro handling there's an option to invoke your extension after running the macro.  If you've loaded the plugin into burp ok, your plugin name should be on the drop down and you can insert it to have it run.

The final code is up on my [github](https://github.com/raesene/burp_sample_plugins/blob/master/test_header_injection.rb).