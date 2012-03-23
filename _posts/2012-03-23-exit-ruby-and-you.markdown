---
layout: post
title: Exit, Ruby and You
---

TL;DR
-----

_A lot is done by Ruby once your script terminates or once you call `exit`._
_`at_exit` hooks are evaluated, finalizers are run and even an exception might be thrown. Knowing
all of this can be useful when developing your next awesome Ruby application._

**Disclaimer:** lower-level details, if present, are valid for an UNIX-like operating system.

Process termination
-------------------

To start off, let's talk a little about process termination. I won't go into a greater level of detail since this
is not the main topic here (better save it for another post.) 

Every process has a parent, which is usually the process that created it. This constitutes what
is called the **process hierarchy** and can be visualized using the _pstree(1)_ command. Go and
try it right now to have a look at your process hierarchy.

Associated with every process termination is an **exit status**, used to indicate the parent process
whether or not it was a successful execution. Conventionally, an exit status of _0_ indicates success
while a non-zero value indicates an error or abnormal program termination.

Ruby
----

**Note:** In our context, _exit_ refers to _Kernel#exit_, which is different from what is executed
if you are using [irb](https://github.com/ruby/ruby/blob/trunk/lib/irb/context.rb#L264) or
[pry](https://github.com/pry/pry/blob/master/lib/pry/default_commands/navigating_pry.rb#L53).

In Ruby, it's pretty easy to exit when you reach an unrecoverable path in your script for example.

{% highlight ruby %}
    if greater_than_zero < 0
      error_message
      exit(false)
    end
{% endhighlight %}

You don't need to be in an error situation necessarily. This example, extracted from the 
[OptParser](http://ruby-doc.org/stdlib-1.9.3/libdoc/optparse/rdoc/OptionParser.html) class documentation
shows a common usage:

{% highlight ruby %}
    opts.on_tail("-h", "--help", "Show this message") do
      puts opts
      exit
    end
{% endhighlight %}

Calling _exit_ with no argument makes process terminate with a success exit status. You can verify
that _exit_ terminates your process with the given status with the following:

{% highlight ruby %}
    $ ruby -e 'exit 42'
    $ echo $?
    42
{% endhighlight %}

Worth noting is the fact that Ruby also provides the _exit!_ method and, as with all methods with
a bang at the end, it should be used only when you know what you are doing. The difference between 
_exit_ and _exit!_ is explained in the following sections.

### Digging Deeper

Time to take a look at the implementation of the _exit_ method. In this post, I'm going to show you
only some MRI code but the interested reader can later peek at the Rubinius or JRuby code to see what
is going on there too.

If we look at the [process.c](https://github.com/ruby/ruby/blob/trunk/process.c) file, we learn that
the _exit_ method is added to the _Kernel_ module invoking the _rb\_f\_exit_ function. Let's look at it

{% highlight c %}
    VALUE
    rb_f_exit(int argc, VALUE *argv)
    {
        /* some exit status logic */

        rb_exit(istatus);
        return Qnil;		/* not reached */
    }
{% endhighlight %}

The first lines of this function just assigns the appropriate value to the _istatus_ variable, according
to the argument passed (if any). Then, the function that actually exits is the _rb\_exit_, since the _return_
statement is never reached. So off we go!

{% highlight c %}
    void
    rb_exit(int status)
    {
        if (GET_THREAD()->tag) {
            VALUE args[2];

            args[0] = INT2NUM(status);
            args[1] = rb_str_new2("exit");
            rb_exc_raise(rb_class_new_instance(2, args, rb_eSystemExit));
        }
        ruby_finalize();
        exit(status);
    }
{% endhighlight %}

Now we start to see some interesting calls. First of all, there seems to be an exception being raised as
we can see by the call to _rb\_exc\_raise_. Is that truth?

To check if that exception is really raised, we will use the handy 
[set_trace_func](http://www.ruby-doc.org/core-1.9.3/Kernel.html#method-i-set_trace_func) method, allowing us to see what is called
when we try to _exit_.

{% highlight ruby %}
    set_trace_func proc { |event, file, line, id, binding, classname|
         printf "%8s %s:%-2d %10s %8s\n", event, file, line, id, classname
    }

    exit
{% endhighlight %}

If you run this script, you will have an output which is similar with the following:

    c-return test.rb:3  set_trace_func   Kernel
        line test.rb:5                     
      c-call test.rb:5        exit   Kernel
      c-call test.rb:5  initialize SystemExit
      c-call test.rb:5  initialize Exception
    c-return test.rb:5  initialize Exception
    c-return test.rb:5  initialize SystemExit
      c-call test.rb:5   exception Exception
    c-return test.rb:5   exception Exception
      c-call test.rb:5   backtrace Exception
    c-return test.rb:5   backtrace Exception
      c-call test.rb:5  set_backtrace Exception
    c-return test.rb:5  set_backtrace Exception
       raise test.rb:5        exit   Kernel
    c-return test.rb:5        exit   Kernel

Aha! As we suspected, an exception is raised, and it is a _SystemExit_. This allows us
to write some code to check if our program is terminating in a successful way:


{% highlight ruby %}
    begin
        call_exit_or_not
    rescue SystemExit => e
        log_unsuccessful_path unless e.success?
    end    
{% endhighlight %}

If we have a method that might call _exit_, like the _#parse_ method from _OptParser_ that
we mentioned before, the approach above might be useful.

What is left is the _ruby\_finalize_ function call, which is defined in the 
[eval.c](https://github.com/ruby/ruby/blob/ruby_1_9_3/eval.c) file.

{% highlight c %}
    static void
    ruby_finalize_0(void)
    {
        PUSH_TAG();
        if (EXEC_TAG() == 0) {
            rb_trap_exit();
        }
        POP_TAG();
        rb_exec_end_proc();
        rb_clear_trace_func();
    }

    static void
    ruby_finalize_1(void)
    {
        ruby_sig_finalize();
        GET_THREAD()->errinfo = Qnil;
        rb_gc_call_finalizer_at_exit();
    }

    void
    ruby_finalize(void)
    {
        ruby_finalize_0();
        ruby_finalize_1();
    }
{% endhighlight %}

A lot of new function calls! But I'll focus on just two of them. The _rb\_exec\_end\_proc_
executes all of the callbacks registered with the _at\_exit_ method. And there is also
_rb\_gc\_call\_finalizer\_at\_exit_, which finalizes the garbage collector. I will now
briefly describe each of these operations in a higher level.


### Exit hooks

C programmers might be used with the _atexit(3)_ function from _stdlib_. The idea is the same
in Ruby. You register a block of code and then it will be called on normal process termination.
As with the C function, the execution order is the reverse order in which the registrations occurred.

### Finalizers

Ruby supports the registration of finalizers associated with any object. This is done via the _define\_finalizer_
method in the [ObjectSpace](http://www.ruby-doc.org/core-1.9.3/ObjectSpace.html) module (you should take a look at
its documentation if you didn't know about it.) A finalizer is run after an object is garbage collected.

Basically a finalizer would serve to the purpose of freeing resources associated with an object.
In practice, their use is not encouraged in the Ruby community. If you have an object that needs to free
some resources, then write a method to do so, or expect a block to be passed and deal
with your resources transparently.

### exit bang!

Now that we saw what happens when the _exit_ method is called, let's investigate its meaner version:
_exit!_. It should be pretty straightforward to spot the difference:

{% highlight c %}
    static VALUE
    rb_f_exit_bang(int argc, VALUE *argv, VALUE obj)
    {
        /* exit status logic again */

        _exit(istatus);

        return Qnil;		/* not reached */
    }
{% endhighlight %}

That's exactly what you thought: instead of a call to the _rb\_exit_ function, we have a call
to the _\_exit(1)_ system call. What it means is that registered _at\_exit_ hooks will not be run,
and neither will your finalizers. The _\_exit_ system call is [guaranteed to finish](http://linux.die.net/man/2/_exit),
with no return value.

In the wild
-----------

To finish off, let's take a look at an interesting piece of code from [minitest](https://github.com/seattlerb/minitest),
since it demonstrates a useful way to use _at\_exit_ handlers. Here it is:

{% highlight ruby %}
    def self.autorun
      at_exit {
        next if $!

        exit_code = nil

        at_exit { exit false if exit_code && exit_code != 0 }

        exit_code = MiniTest::Unit.new.run ARGV
      } unless @@installed_at_exit
      @@installed_at_exit = true
    end
{% endhighlight %}

If you ever wondered how your tests are run in a script that just defined a class that inherited from _MiniTest::Unit::TestCase_,
here is how. In minitest, if you require _minitest/autorun_, the above method is executed, causing an _at\_exit_ handler to
be registered to run your tests. Minitest is even nice enough to make sure it returns a non-zero exit status (_EXIT\_FAILURE_)
if your tests fail. This allows you to write a shell script that sends an email to your boss when your tests pass, for example.

Also interesting is the fact that you can use _next_ in an _at\_exit_ handler, causing the next registered handler to be executed,
as if you were in an iteration. Sweet.

Tricky parts
------------

Let's take minitest as an example again. If you have some code you wish to execute after your tests are run, you can do that
using the _MiniTest::Unit.after\_tests_ method. You pass it a block and it is saved for later execution.

Presumably, this was implemented using an _at\_exit_ call with _yield_. However, what would happen if you had the following code:

{% highlight ruby %}
    require 'minitest/autorun'

    # pretend to have long tests so that we can check Twitter
    MiniTest::Unit.after_tests { puts "Calculating..."; sleep(10)  }

    class Complex < MiniTest::Unit::TestCase
        # ...
    end
{% endhighlight %}

You probably know where I'm trying to get: remember that when you require _minitest/autorun_ a new _at\_exit_ handler
is registered. The same used to happen when you called _MiniTest::Unit.after\_tests_. Because the handlers are run
in reverse order of registration, the code in the _after\_tests_ would be executed **before** your tests are run. That
might not be what you expected.

That's how the current implementation works:

{% highlight ruby %}
    def self.after_tests &block
      @@after_tests << block
    end

    def self.autorun
      at_exit {
        next if $!

        exit_code = nil

        at_exit {
          @@after_tests.reverse_each(&:call)
          exit false if exit_code && exit_code != 0
        }

        exit_code = MiniTest::Unit.new.run ARGV
      } unless @@installed_at_exit
      @@installed_at_exit = true
    end
{% endhighlight %}

No more _at\_exit_ in the _after\_tests_ method! Instead, the blocks are stored for later execution, imediately
after the tests are run. Now you can use the _after\_tests_ method before or after you require _minitest/autorun_
and be sure that it will be executed only after your tests.

The last tricky fact to be aware of regards the _SystemExit_ exception which is raised when you call _exit_. Let's
look at a small piece of code, this time from [rake](http://rake.rubyforge.org/):

{% highlight ruby %}
    def standard_exception_handling
      begin
        yield
      rescue SystemExit => ex
        raise
      rescue OptionParser::InvalidOption => ex
        $stderr.puts ex.message
        exit(false)
      rescue Exception => ex
        display_error_message(ex)
        exit(false)
      end
    end
{% endhighlight %}

This exception handling method tries to rescue different types of exceptions and take the appropriate action.
What is interesting here is the fact that when a _SystemExit_ is rescued, just a call to _raise_ is done. The
_raise_ method, when called with no arguments, raises the content of the _$!_ variable (which holds the last exception
raised) or _RuntimeError_ if it is _nil_. So in fact, it is ignored, allowing normal process termination. This is done
because later, _Exception_ is rescued and every exception in Ruby - including _SystemExit_ - inherits from _Exception_.
So, if we had something like:

{% highlight ruby %}
      begin
        yield
      rescue Exception => ex
        display_error_message(ex)
        exit(false)
      end
{% endhighlight %}

we would call _display\_error\_message_ and _exit(false)_ if we made a call to _exit(true)_ in the block passed to this method,
for example, which is probably not what we want.

Conclusion
----------

If you are still reading, thanks! I didn't expect this to be such a long post. I'll try to make shorter ones next time.

What is important to keep in mind from this, though, is what is done by Ruby on process termination. Being aware of
the order in which the _at\_exit_ handlers are executed is very important to avoid unexpected results. Also important
is knowing about the _SystemExit_ exception and how you can inadvertently execute some code if you just rescue _Exception_
without any further check.
