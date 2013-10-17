##Kojak: A simple JavaScript profiler

####What is Kojak?
Kojak is a simple utility that can help you figure out which of your JavaScript functions are running too slow. It tracks which of your
functions are called, how often they are called, how much time they are taking, how the functions were called.  It can also track your ajax calls
and help figure out how fast they are. (<a href="http://en.wikipedia.org/wiki/Kojak">Kojak</a> was a tv show detective)

####Why Kojak?
I've found that Chrome's developer tools or Firebug didn't usually help me figure out where my client side code was slow.
I wanted a tool that would weed out the noise of external / core JavaScript libraries.  Kojak allows you to define what
code you care about and it will ignore everything else.  It's really helped me get my JavaScript code faster. Hopefully
it can help you and your project.

####Dependencies
The core or Kojak has no external dependencies.  I've worked hard to avoid using any other libraries so that the tool is
light weight and easy to use. You do need a modern browser such as Chrome, Firefox or IE 8.0+.
If you want to profile ajax network requests you will need to include jQuery.

<br>
####How to use it (the short version)
To use Kojak copy/download the Kojak.min.js file.  Include it in the browser code you want to profile.  You can include it with
a normal script tag or you can also just copy and paste the Kojak.min.js file directly into a running browser console.
You can actually profile code in any web site as long as you know what the code root namespaces are (discussed later).

Kojak expects that your code is accessible via the window object.  A typical application might be assembled like this:

````
var myProject = {models: {}, views: {}, controllers: {}, utils: {}};
myProject.models.ModelA = function(){};
myProject.views.ViewA = function(){};
myProject.utils.sharedUsefulFunction = function(){};
````

The classes/functions live somewhere under the window object.  In the example above, the code lives under window.myProject.
(Kojak might be extended in the future to support code not directly accessible under window.)

To quickly get started with Kojak you could run the following commands in the browser's console:
````
  // replace with your own root package names
  kConfig.setIncludedPakages(['myProject']);

  // this will root recursively through all your code, starting with the included packages and wrap every
  // single function it finds to keep track of all of the function's runtime information
  kInst.instrument();

  // See which functions have been instrumented in your application
  kRep.instrumentedCode();

  // Now you should do something with your application that does not include a full page refresh

  // Now see which are your slowest functions.  This only includes the instrumented functions.
  kRep.funcPerf();
````

Kojak has a lot of other features that I'll explain later, but first I need to explain how Kojak makes sense of JavaScript.

<br>
####Supported code formats

<i>If this section is confusing I would recommend reading <a href="http://www.amazon.com/Secrets-JavaScript-Ninja-John-Resig/dp/193398869X/ref=sr_1_1?s=books&ie=UTF8&qid=1382038307&sr=1-1&keywords=secrets+of+the+javascript+ninja">chapters 1-6</a>.</i>

Kojak recognizes 3 types of data structures in your code
* `Pakage`
* `Clazz`
* `function`

A `Pakage` is anything that might contain your code. A `Pakage` might be a Plain Old JavaScript Object (POJO) that looks like {}.
A `Pakage` might also be a `Clazz` that contains other `Clazzes` or `Pakages`.
In the example below, myProject, myProject.models, myProject.models.ModelA and myProject.models.ModelA.NestedModelB are all considered Pakages:
````
var myProject = {models: {}, views: {}};
myProject.models.ModelA = function(){};
myProject.models.ModelA.NestedModelB = function(){};
````

A `Clazz` is a function that is expected to be used with the `new` operator.  `Clazzes` are expected to be named starting
with an upper case character. In the previous example ModelA and NestedModelB are both `Classes`.

A `function` is just a normal JavaScript function that Kojak does not think is a `Clazz`.  Kojak looks for functions
in `Pakage`, `Clazz` or under the `Clazz`.`prototype`.

For example, Kojak will profile the functions 'packageFunction`, `classLevelFunction` and the `prototypeLevelFunction`:
````
myProject.packageFunction = function(){};
myProject.models.ModelA = function(){};
myProject.models.ModelA.classLevelFunction(){};
myProject.models.ModelA.prototype.prototypeLevelFunction(){};
````

It's important for Kojak to understand when a function will be invoked with the `new` operator and to avoid wrapping
those type of functions directly.  If Kojak finds <b>any</b> reference to a function that starts with an upper case it will
assume the function is a `Clazz`.

The example below shows this:
````
myProject.models.ModelA = function(){};
myProject.someCodePackage = {
  refToModelA: myProject.models.ModelA
};
````

In the example above, ModelA is also referenced with the name refToModelA.  Kojak understands that ModelA and refToModelA both
reference the same function and that one of those references looks like a Clazz.  In that situation, Kojak will assume
all references should behave as a Clazz.  If Kojak accidentally wraps a function and the function is invoked with the
`new` operator Kojak will throw a runtime exception with a message explaining which function was instrumented incorrectly
as a function instead of a Clazz.

<br>
####How to use it (the long version)
Kojak needs to be told what code it is supposed to profile.  You tell Kojak via the command:
````kConfig.setIncludedPakages(['packageA', 'packageB'])````

Kojak will use these package names as entry points to find all of the code that you probably care about.

You can also tell Kojak to exclude functions or packages with this command:
```` kConfig.setExcludedPaths(['packageA.SomeClass.funcA', 'packageA.SomeClass.funcB']); ````

Kojak will then ignore funcA and funcB.  The excluded paths can be fully qualified function paths or namespaces etc.

There are several other options in `kConfig` that are discussed in a later section.  All options set with `kConfig` are persisted
automatically in the browser's local storage.  So, the next time you refresh a page etc. your Kojak options are saved.

<br>
After you've told Kojak what it should are about and what to exclude you need to run this command:
```` kInst.instrument(); ````

The `instrument` function will locate every single `Pakage`, `Clazz` and `function` that can be found recursively through
what you specified in the kConfig. Kojak will inject a `_kPath` string in each `Pakage` and `Clazz` that is the fully qualified
path to the `Package` or `Clazz` so that they can be self aware of where they live.

The `instrument` function replace each `function` it finds with a wrapper function.  The wrapper function helps Kojak to
track everything that happens with that function. The new wrapper function contains a `_kFProfile` property.  The `_kProfile`
property keeps track of all of the information of what is a happening with the function.

<br>
After you have told Kojak to instrument your code you can now invoke your code.  Typically you will click on something etc.
After some of your code has ran you want to see why it was so slow.  To determine which functions are too slow run the command:
```` kRep.funcPerf(); ````

By default, this function will return the 20 slowest functions that have been profiled. For example:

````
Top 20 functions displayed sorted by IsolatedTime'
--KPath--                                                                    --IsolatedTime--  --WholeTime--  --CallCount--
MyProject.views.schedule.ResourceListView.prototype._positionAppointments     392               426            77
MyProject.views.schedule.ResourceListView.prototype._createGridLines          351               367            35
MyProject.domain.BaseModel.prototype.get                                       99                230            14,679
MyProject.domain.BaseModel.prototype._getResolvedAttributeTypes                98                98             14,926
MyProject.views.schedule.ScheduleView.prototype._sizeWeekViewResourceColumns   78                78             28
...
````

The funcPerf() will report 3 different fields for each function
 * IsolatedTime
 * WholeTime
 * CallCount

IsolatedTime is how much cumulative time was spent inside the function itself.  Whole time includes other functions.

For example, funcA takes 1 second and funcB takes 2 seconds.  If we modify funcA to internally call funcB the results would
look like this:
````
--KPath--  --IsolatedTime--  --WholeTime--  --CallCount--
funcA         1,000              3,000         1
funcB         2,000              2,000         1
````

funcA's IsolatedTime would be 1,000 milliseconds but it's whole time would be 3,000 milliseconds.  Most of the time, you
 will probably care more about IsolatedTime than WholeTime.

The funcPerf() function can take the following options:
* sortBy - kRep.funcPerf({sortBy: 'WholeTime'});
* max - kRep.funcPerf({max: 30}); // I want 30 results instead of the default of 20
* filter - kRep.funcPerf({filter: ['BaseModel', 'BaseView']}); // I only want results that contain 'BaseModel' or 'BaseView' in their path

<br>
After seeing what your slowest functions are you might want to know how they are being invoked.  This is particularly
important for when a function's CallCount is unacceptably high.

To figure out who is calling a function run this command:
````
kRep.callPaths(MyProject.someFunc);

// Example results
--Call Count--  --Invocation Path--
1,380           MyProject.foo > MyProject.models.ModelA.prototype.bar > MyProject.something
...
````

The invocation paths will only include functions that Kojak has instrumented.  These call paths can help you track down
why a function is called too much.

<br>
####Tracking performance in between actions
Sometimes it's helpful to take performance checkpoints between actions and not include any previous results.  Sometimes it
can take a long time to set up a test and you don't want to have to repeat steps.  You can only run kInst.instrument() one time.

To do this use this command:
```` kInst.takeCheckpoint(); ````

Then perform your action.

Then run this command:
```` kRep.funcPerfAfterCheckpoint(); ````

The funcPerfAfterCheckpoint() is identical to funcPerf() and can accept the same parameters.  The difference is that
  the function results (IsolatedTime etc.) are only since the last time takeCheckpoint() was called.

Sometimes it's particularly interesting to watch the CallCount's for functions when running the identical actions.  Most of
the time the CallCount numbers should be identical.  If they are not, you probably have some type of memory leak.


<br>
####Tracking Network Requests
Kojak can also track all of your network ajax requests.  To use the NetWatcher you must use jQuery.  To enable run this command:
```` kConfig.setEnableNetWatcher(true); ````

The Kojak NetWatcher will then watch any ajax requests made.  To see a consolidated view the results use this command:
````
kRep.netCalls();
// Example output
--urlBase--            --urlParameters--    --When Called--  --Call Time--  --Size (bytes)--  --Obj Count--
/kpi/SOME_KPI [GET]      <none>                01:04:22 PM      3,442          386               8
...
````

The results are consolidated by the urlBase.  The results are sorted by the Call Time.


<br>
####Full API and Options
todo, ugh.


<br>
####How to compile it locally
Normally you won't need to do this unless you are forking the code.  If you do want to fork the code.
* Install NodeJS
* Install GIT
* Fork the code (git clone https://github.com/theironcook/Kojak/)
* Navigate to the directory you forked the code and type: npm install

This will install grunt, phantom, jasmine, uglify etc. in the local node_modules directory.

To build a prod version type: grunt buildProd
To build a dev version type: grunt buildDev
You can also type: grunt watch
This will run grunt buildDev whenever a source file or a unit test changes.