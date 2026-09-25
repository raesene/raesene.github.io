---
layout: post
title: "Burp Passive Scanner Plugins with JRuby"
date: 2015-01-15 20:45:28 +0000
comments: true
categories: Burp
---

Like most web application testers, I'm a fan of using [Burp Suite](http://www.portswigger.net) for automation and generally making the process of completing a test a whole lot easier.

In recent versions the Portswigger team have opened up the Burp API and we've started to see more 3rd party plugins coming along which is awesome as it lets Burp users contribute, whilst making their own testing lives easier.  Plugins can be written in Java, Jython, or JRuby.  Looking at the [BApp store](https://pro.portswigger.net/bappstore) most of the entries at the moment are either in Java or Jython, so I thought it was worth documenting a basic passive scanner check in JRuby as there are definitely a couple of gotchas to getting everything working, including one that had me stumped (thanks to Timur and Ryan for pointing me in the right direction!)

First up to use JRuby you need to point Burp at your JRuby jar file in the extender--> options tab, and after that you should be able to load an appropriately configured ruby file in the main Extensions window.

The code for this basic check is on [github](https://github.com/raesene/burp_sample_plugins) so here are a couple of points to note.

First up there's a load of information on the topic of Burp plugins on the [Portswigger blog](http://blog.portswigger.net/search?updated-min=2012-01-01T00:00:00Z&updated-max=2013-01-01T00:00:00Z&max-results=16)

In general structure the basic plugin is a ruby class called BurpExtender, with a method which implements the callback (registerExtenderCallback) and for the passive scan we need a doPassiveScan method as well.

The main meat of the check happens in doPassiveScan.  This method is called whenever Burp encounters a URL that it's set to passively scan (either just URLs in scope or all URLs depending on the configuration you've set).

Within that you can extract information about the request that triggered the call and analyze that information to see if an issue is present.

So in the sample check we're looking for an HTTP header, which if it's not found, will add a low severity issue to the burp scanner.

``` ruby

  def doPassiveScan(baseRequestResponse)
    service_info = baseRequestResponse.getHttpService()
    host_name = service_info.getHost()
    response_info = @helpers.analyzeResponse(baseRequestResponse.getResponse)
    headers = response_info.getHeaders()

```

This section of code extracts some information from the request-response object that Burp passes in to the method.  It's pretty easy to extract the various parts of the HTTP request to analyze and once you've done it they appear as ruby objects (e.g. strings or arrays) which can be analyzed with standard ruby methods.

``` ruby

  header_found = false

    headers.each do |header|
      if header.downcase =~ /your_value_here/
          header_found = true
      end
    end

```

the next section of code is really the only analysis that we're doing in this basic example.  The headers variable is an array, so we can iterate over it and check each element to see if it's found. In this case our header is called 'your_value_here' which is pretty unlikely to show up, but in a real check this could be replaced with something like 'Strict-Transport-Security' or 'Public-Key-Pins' to check for the presence of security related HTTP headers.

Once we've got our finding of course the next step is to add it to the scanner results, so we'll need to create a Scan Issue and pass it back to Burp.

``` ruby

   findings = Java::JavaUtil::ArrayList.new

```

This line was one of the places I tripped up initially.  Burp expects an array of results back and not a single Scan issue so creating an array to hold the findings is handy.

``` ruby

    finding_message = CustomHttpRequestResponse.new
    finding_message.setResponse(baseRequestResponse.getResponse())
    finding_message.setRequest(baseRequestResponse.getRequest())
    finding_message.setHttpService(baseRequestResponse.getHttpService())

```

This section is necessary if you want to return the request/response information into the scanner and is another place where I had some issues. The key point is that you need to implement a class that conforms to the interface for the IHttpRequestResponse class that the Burp API describes.  In the file on github there's a class that seems to work for me.  It looks a bit boilerplate heavy, but at the moment the more rubyish setup with attr_accessor helpers for the class properties doesn't seem to fly for me, so manual getter/setter methods it is.

``` ruby

    unless header_found
      finding = CustomScanIssue.new
      finding.httpMessages=finding_message
      finding.httpService=baseRequestResponse.getHttpService()
      finding.url=@helpers.analyzeRequest(baseRequestResponse.getHttpService(), baseRequestResponse.getRequest).getUrl()
      finding.name = "Header Not Set"
      finding.detail = "A header that should be set isn't"
      finding.severity = "Low"
      finding.confidence = "Certain"
      finding.remediation_detail = "Lorem Ipsum"
      finding.issue_background = "Sit Dolor Amet"
      findings.add finding
    end

```

This code just populates another class which you need to implement which is the Scan Issue.  In general that class is pretty straight forward like the RequestResponse class but a key point is that Burp expects an array of RequestResponse issues to be returned in the messages property of the scan issue.

Last bit in this basic example is a brief method to do some basic handling of duplicate instances of the same issue (saves the scanner repeatedly finding the same issue for a URL)

``` ruby

  def consolidateDuplicateIssues(existing_issue, new_issue)
    if existing_issue.getIssueName == new_issue.getIssueName
      return -1
    else
      return 0
    end
  end

```

This is obviously quite a basic example but hopefully shows that once you know the structure of the code, implementing scanner checks in Burp should be a relatively straightforward task, and can have some cool benefits in terms of automating the testing process.