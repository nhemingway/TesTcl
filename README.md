# Introduction

**TesTcl** is a [Tcl](http://en.wikipedia.org/wiki/Tcl) library for unit testing
[iRules](https://devcentral.f5.com/HotTopics/iRules/tabid/1082202/Default.aspx) which 
are used when configuring [F5 BigIP](http://www.f5.com/products/big-ip/) devices.
The goal of this library is to make it easy to unit test iRules used when load balancing HTTP traffic.

## The challenge

Configuring BigIP devices is no trivial task, and typically falls in under a DevOps kind of role.
In order to make your system perform the best it can, you need:

- In-depth knowledge about the BigIP system (typically requiring at least a [$1,995 3-day course](http://www.f5.com/services/global-training/course-descriptions/big-ip-ltm-essentials.html))
- In-depth knowledge about the web application being load balanced 
- The Tcl language and the iRule extensions
- And finally: _A way to test your iRules_

## Testing iRules

Most shops test iRules [manually](http://en.wikipedia.org/wiki/Manual_testing), the procedure typically being a variation of the following:

- Create/edit iRule
- Add log statements that show execution path
- Push iRule to staging/QA environment
- Bring backend servers up and down **manually** as required to test fallback scenarios
- Generate HTTP-traffic using a browser and verify **manually** everything works as expected
- Verify log entries **manually**
- Remove or disable log statements
- Push iRule to production environment
- Verify **manually** everything works as expected 

There are lots of issues with this **manual** approach:

- Using log statements for testing and debugging messes up your code, and you still have to look through the logs **manually**
- Potentially using different iRules in QA and production make automated deployment procedures harder
- Bringing servers up and down to test fallback scenarios can be quite tedious
- **Manual** verification steps are prone to error
- **Manual** testing takes a lot of time
- Development roundtrip-time is forever, since deployment to BigIP sometimes can take several minutes

Clearly, **manual** testing is not the way forward!

Enough said about manual testing. Let's talk about unit testing iRules using TesTcl!

## Getting started

If you're familiar with unit testing and [mocking](http://en.wikipedia.org/wiki/Mock_object) in particular,
using TesTcl should't be to hard. Check out the examples below:

### Simple example ###

Let's say you want to test the following simple iRule found in *simple_irule.tcl*:

    rule simple {

      when HTTP_REQUEST {
        #starts_with "/foo" 
        if { [regexp {^/foo} [HTTP::uri]] } {
          pool foo
        } else {
          pool bar
        }
      }

    }

Now, create a file called let's say *test_simple_irule.tcl* containing the following lines:

    package require -exact testcl 0.8
    namespace import ::testcl::*

    # Comment in to enable logging
    #log::lvSuppressLE info 0
    
    it "should handle request using pool bar" {
      event HTTP_REQUEST
      on HTTP::uri return "/bar"
      endstate pool bar
      run simple_irule.tcl simple
    }

    it "should handle request using pool foo" {
      event HTTP_REQUEST
      on HTTP::uri return "/foo/admin"
      endstate pool foo
      run simple_irule.tcl simple
    }

Next, put testcl on your library path. If you use JTcl, you can add the directory containing all the 
files found in this project (zip and tar.gz can be downloaded from this page) to 
the [TCLLIBPATH](http://jtcl.kenai.com/gettingstarted.html) environment variable.

In order to run this example, type in the following at the command-line:

    >jtcl test_simple_irule.tcl

This should give you the following output:

    **************************************************************************
    * it should handle request using pool bar
    **************************************************************************
    -> Test ok

    **************************************************************************
    * it should handle request using pool foo
    **************************************************************************
    -> Test ok

#### Explanations

- Require the **testcl** package and import the commands and variables found in the **testcl** namespace to use it.
- Enable or disable logging
- Add the specification tests
  - Describe every _it_ statement as precisely as possible.  
  - Add an _event_ . This is mandatory.
  - Add one or several _on_ statements to setup expectations/mocks. If you don't care about the return value, return "".
  - Add an _endstate_. This could be a _pool_, _HTTP::respond_ or _HTTP::redirect_ call . This is mandatory.
  - Add a _run_ statement in order to actually run the Tcl script file containing your iRule. This is mandatory.

#### Avoiding code duplication using the before command

In order to avoid code duplication, one can use the _before_ command.
The argument passed to the _before_ command will be executed _before_ the following _it_ specifications.

Using the _before_ command, the *test_simple_irule.tcl* can be rewritten as:

    package require -exact testcl 0.8
    namespace import ::testcl::*

    # Comment in to enable logging
    #log::lvSuppressLE info 0

    before {
      event HTTP_REQUEST
    }

    it "should handle request using pool bar" {
      on HTTP::uri return "/bar"
      endstate pool bar
      run simple_irule.tcl simple
    }

    it "should handle request using pool foo" {
      on HTTP::uri return "/foo/admin"
      endstate pool foo
      run simple_irule.tcl simple
    }


## How stable is this code?
This work is still undergoing quite some development so you can expect minor breaking changes.

## Gotchas
- If you try testing iRules that contain iRule extensions to the Tcl language, this stuff won't work. I'm working on an extension to [JTcl](http://jtcl.kenai.com/) to make this work.
  - Operators like equals, starts_with etc

## TODOs

- Implement irule extensions to Tcl (operators like *starts_with* etc)
  - contains	Tests if one string contains another string
  - ends_with	Tests if one string ends with another string
  - equals	Tests if one string equals another string
  - matches_glob	Implement glob style matching within a comparison
  - matches_regex	Tests if one string matches a regular expression
  - starts_with	Tests if one string starts_with another string
  - and	Performs a logical "and" comparison between two values
  - not	Performs a logical "not" on a value
  - or	Performs a logical "or" comparison between two values
- Add advanced example
- Improve error handling / logging
- Add support for *HTTP_RESPONSE*