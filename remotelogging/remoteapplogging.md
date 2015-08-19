# Cloud Logging, the perfect fit for mobile apps

During development of your app you have full control over what's happening in your code.
You have the debugger, console logging, performance profiling etc.
After you have released your app, you have almost no tools for checking how your app
behaves in the wild. In most cases, the feedback you get come from emails to your
support address, App Store reviews and crash reporting (if you chose to include
  Crashlytics or some similar tool in your app).
Crash reporting services are great. Lately they have also added methods to save extra info into
a state object, that will also be displayed together with the crash.
But, there is one obvious limitation with crash reporting: your app needs to crash before any information
is sent to you.

Wouldn't it be great if you could keep screening the log printouts that you already have in your app, also
after your app is released? The problem is, how can you access the logs and how can you find useful information
in the vast amount of logs that your users will create?

**Enter Cloud Logging services.**

Cloud logging services is a [short explanation of what cloud services is]

There are many players in the cloud logging market: [Splunk](http://www.splunk.com/), [Sumo Logic](https://www.sumologic.com/), [Loggly](https://www.loggly.com/) etc. Some of these are free up to a certain level of data volumes and retention periods. There are also open source log management systems that work the same way, but you will need to host the servers yourself.
An example of this is [Logstash](https://www.elastic.co/products/logstash) in combination with [Kibana](https://www.elastic.co/products/kibana) or [Graphite](http://graphite.wikidot.com/).

For some reason, Cloud logging services have mainly been used for replicating server logs.
This is extremely useful, because you can do aggregated searches over several server logs, set up automatic alarms and do all kinds
of log analysis. In the server case though, you already have the data somewhere. You can always read your logs on your individual servers if you really need to. For apps, you have nothing! You don't have the log source. So the incentive for cloud logging is much greater when it comes to
apps. For some reason though, it doesn't seem common to use cloud logging for apps.

A little over a year ago, I became interested in using one of the cloud logging services for my iPhone apps. To my surprise, there were
no iOS client libs available from any of the logging services themselves. I couldn't even find any open sourced libs on Github.
I decided to write a library myself. I chose to integrate against Loggly, because they have an excellent REST API for remote logging. First, I wrote an Objective-C library, which extends the amazing iOS logging framework [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack). The library formats and sends logs to Loggly. You can get the library at [LogglyLogger-CocoaLumberjack](https://github.com/melke/LogglyLogger-CocoaLumberjack).
Recently, I also wrote a minimalistic logging framework in Swift, that can be used for console logging only, or for uploading logs to Loggly, or both. The Swift framework is available at [SlimLogger](https://github.com/melke/SlimLogger).

Here, I will briefly describe how to do cloud logging using SlimLogger in a Swift iOS app.

##Prerequisites

  * Create an account at https://www.loggly.com

##Installation

  * `git clone https://github.com/melke/SlimLogger`
  * Add `SlimLogger.swift`, `SlimLogglyDestination.swift`, `SlimLoggerConfig.template` and `SlimLoggerDestinationConfig.template` to your project
  * Rename `SlimLoggerConfig.template` to `SlimLoggerConfig.swift`
  * Rename SlimLogglyDestinationConfig.template to SlimLogglyDestinationConfig.swift

##Configuration

Edit the SlimConfig struct in `SlimLoggerConfig.swift`

```swift
struct SlimConfig {
    // Enable or disable console logging. When releasing your app, you should set this to false.
    static let enableConsoleLogging = true

    // Log level for console logging, can be set during runtime
    static var consoleLogLevel = LogLevel.trace

    // Either let all logging through, or specify a list of enabled source files.
    // So, either let all files log:
    static let sourceFilesThatShouldLog:SourceFilesThatShouldLog = .All
    // Or let specific files log:
    static let sourceFilesThatShouldLog:SourceFilesThatShouldLog = .EnabledSourceFiles([
             "AppDelegate.swift",
             "AnotherSourceFile.swift"
    ])
    // Or don't let any class log (use to turn off all logging to for all destinations):
    static let sourceFilesThatShouldLog:SourceFilesThatShouldLog = .None
}
```

Edit SlimLogglyDestinationConfig.swift and change the api key and app name in the Loggly URL.

```swift
struct SlimLogglyConfig {
    // Replace your-loggly-api-key below with a "Customer Token" (you can create a customer token in the Loggly UI)
    // Replace your-app-name below with a short name for your app (no spaces or crazy characters). You can use this
    // tag in the Loggly UI to create Source Group for each app you have in Loggly.
    static let logglyUrlString = "https://logs-01.loggly.com/bulk/your-loggly-api-key/tag/your-app-name/"

    // Number of log entries in buffer before posting entries to Loggly. Entries will also be posted when the user
    // exits the app.
    static let maxEntriesInBuffer = 100

    // Loglevel for the Loggly destination. Can be set to another level during runtime
    static var logglyLogLevel = LogLevel.info
}
```

##Usage

In `didFinishLaunchingWithOptions` in your app delegate, add the Loggly destination:

```swift
Slim.addLogDestination(SlimLogglyDestination())
```

Now you can start logging from any class.

```swift
// I recommend logging something that can be casted to an NSDictionary
// This way, searchable fields will be created in Loggly for every json key.
Slim.trace(["logtype": "user-interaction", "tapped": "Remove-item-button", "longmsg": "Removing item number \(itemnumber)"])
Slim.debug(["logtype": "state", "state": "\(state)"])

// But you can also log any type that implements the Swift protocol Printable
Slim.trace("my message")
Slim.debug(4711)
Slim.info(["one","two","three"])
Slim.info(someNSURLvariable)
```

That is all there is to it. The log posts will include your log message plus some standard fields that SlimLogger will add automatically:

  - **level** - The log level
  - **timestamp** - Timestamp in iso8601 format (required by Loggly)
  - **sourcelocation** - Source file and line number in that file (nice for doing facet searches in Loggly)
  - **appname** - The name of your app
  - **appversion** - The version of your app
  - **devicemodel** - The device model
  - **devicename** - The device name
  - **lang** - The primary lang the app user has selected in Settings on the device
  - **osversion** - the iOS version
  - **rawmsg** - The log message that you sent, unparsed. This is also where simple non-JSON log messages will show up.
  - **sessionid** - A generated random id, to let you search in loggly for log statements from the same session.
  - **userid** - A userid string. Note, you must set this userid yourself in the SlimLogglyDestination object. No default value.

##Structured log messages

You can log any type that implements Printable, but if you want a better search experience in the Loggly UI, you should log a type that can be casted to an NSDictionary. If you do that, all dictionary keys will be logged as separate keys
in Loggly. This makes it much easier to do filtered and faceted field searches in Loggly.
Word of warning, don't use too many different keys, it will make it harder to get a good overlook of your data
in the Loggly UI. Also, Loggly will not log more that 100 distinct keys for free accounts. Figure out smart keys that you can reuse in many of your log statements.

You should think carefully about which fields you want to log with each request. Note that you don't have to be totally consistent and always
use the same fields for every log statement. If some fields don't make sense for a certain log message, just omit them.

Example:

Let's say that you want to log three types of log messages:
   * user interactions
   * user state
   * Bad stuff, warnings and errors

These types will require different kinds of fields. An error has an error object that you can log, a warning may not have an error object, and
user state and interactions serve other purposes altogether.

```swift
// User interaction
Slim.trace(["logtype": "user-interaction", "tapped": "Remove-item-button", "longmsg": "Removing item number \(itemnumber)"])

// State
Slim.debug(["logtype": "state", "state": "\(state)"])

// Bad stuff
Slim.warn(["logtype": "badstuff", "shortmsg": "No items", "longmsg": "No items for user \(username)"])
Slim.error(["logtype": "badstuff", "shortmsg": "User session timed out", "longmsg": "\(error)"])
```

As you can see, the field logtype exists in all log statements. Use this field as a filter when you
don't want to mix the different log types in the search results. There are also shortmsg and longmsg fields in some log statements. These are good when you want to see the sum of certain errors. The shortmsg should only be unique enough to identify the error. The longmsg can contain unique
data for that particular log statement.

##Use cases


###Find common errors
You can do aggregated searches in Loggly where you quickly can see the most common errors, warnings etc. [some more info about this?...]

###Alarms

In most cloud logging services, including Loggly, you can setup alarms for lots of different things. The most simple alarm condition would be if
the number of errors during a certain period of time rises above zero, or some other acceptable level of error count. You can also
be more specific, for example raising alarms whenever an In-App Purchase fails in your app.  

###Troubleshoot a session or a user

First of all, be very careful when logging anything that can be connected to a certain user. You always have to respect
the users privacy. Also, in some countries it is illegal to store such info altogether.

Anyway, to track a specific user you can set the userid property on the SlimLogglyDestination object. The userid
will then be included in every log statement until the app is terminated by iOS.

Let's say that a user complains about having problems in your app. You can then search the Loggly UI for all log entries
that this user has created. You can also have some secret button in your app, and when the user taps this
button, you can set the log level dynamically to a finer level in SlimLogglyConfig. If you don't want the secret button, you
could add a way in your backend to change the loglevel for a specific user. The app would react to this backend change and
set it's loglevel in the SlimLogglyConfig dynamically. Pretty nice, huh?

If you want your logs to be anonymized, you could use the sessionid that SlimLogger generates. This is useful when you want
to see all log messages from a certain session. It could also be combined with a secret button where the user can see the sessionid and
send it to the person trobleshooting the issue.

## Summary

There is nothing worse than getting error reports from users of your released app, but you can't reproduce the error and you have no information available to do any troubleshooting. Adding cloud logging for your app solves this problem. In fact, logging from apps may be the most appealing use case for cloud logging. I use it in my apps:

   * [Baby Names Mania](https://itunes.apple.com/app/baby-names-mania/id594784300?mt=8). Gives you loads of name suggestions for your baby.
   * [GTDT - Get Things Done Today](https://itunes.apple.com/app/gtdt-get-things-done-today/id977201254?mt=8). A pomodoro app that introduces a somewhat [new philosophy for personal efficiency](http://gtdt.melke.nu/).

Doing cloud logging in my apps has helped me to track down several issues that would otherwise have been hard to fix.
