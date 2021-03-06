(ARM-3) Puppet Templates - EPP
==============================

Summary
-------
This ARM defines Embedded Puppet Templates (EPP) - ERB "compatible" templates where the logic
is written in the Puppet Language instead of Ruby.

Goals
-----
Offer a Puppet Language based replacement for ERB templates.

Non Goals
---------
It is not a goal to enhance the ERB template syntax.

Motivation
----------

It is currently possible to evaluate templates using ERB by calling
functions that either process an external template (an .erb file), or an
inline template passed as a string. In the ERB template the template
author must use the puppet ruby API to obtain variable values. The
author must be aware of quite a few rules and knowledgeable in Ruby to
be able to avoid problems.

By adding support for templates with puppet logic the problems
associated with using ERB/Ruby/Puppet API are eliminated.

Longer term, the support for ERB can be dropped from Puppet.

Description
===========

EPP is an ERB "compatible" template solution. It is compatible with respect to the tags used to embed logic, but
supports Puppet DSL instead of Ruby. The support is compatible with the configuration currently used for ERB (options turned on/off)
as shown in the following table of supported tags.

EPP Tags
--------

When in text mode:

<table>
<tr>
  <td><tt>&lt;%</tt></td>
  <td>Switches to puppet mode</td>
</tr>
<tr>
  <td><tt>&lt;%=</tt></td>
  <td>Switches to puppet expression mode. (Left trimming is not possible)</td>
</tr>
<tr>
  <td><tt>&lt;%%</tt></td>
  <td>Literal <tt>&lt;%</tt></td>
</tr>
<tr>
  <td><tt>%%></tt></td>
  <td>Literal <tt>%&gt;</tt></td>
</tr>
<tr>
  <td><tt>&lt;%-</tt></td>
  <td>Trim Left<br/>
  When the opening tag is <tt>&lt;%-</tt> any whitespace preceding the tag, up to and
  including a new line is not included in the output.
  </td>
</tr>
<tr>
  <td><tt>&lt;%#</tt></td>
  <td>Comment<br/>
  A comment not included in the output (up to the next <tt>%&gt;</tt>, or right
  trimming <tt>-%&gt;</tt>). Continues in text mode after having skipped the comment
  (observes right trimming semantics).
  </td>
</tr>
</table>

When in Puppet mode:

<table>
<tr>
  <td><tt>%&gt;</tt></td>
  <td>End puppet mode</td>
</tr>
<tr>
  <td><tt>-%&gt;</tt></td>
  <td>Trim Right and end puppet mode</td>
</tr>
</table>

When the closing tag is `-%>` any whitespace following the tag, up to and
including a new line is not included in the output.

When ending puppet mode the result of the puppet logic is rendered to
the output for a puppet expression tag `<%=`, but not for `<%`.

Invocation
----------

Currently an ERB template is evaluated with the `template()` or `inline_template()` functions.
These functions cannot be modified in a backwards compatible way to support both ERB and EPP templates since
type of template can not be derived from filenames (since it is not required to use .erb or .epp as extension). The same
applies to the inline function where it is not possible to detect if a string is .erb or .epp in a straight forward way.

Instead, two new functions `epp()` and `inline_epp()` are introduced to process EPP templates.

These functions work on one template string/file at the time (in contrast to the existing template
functions that handles multiple templates and joins them). In addition EPP supports passing a hash with named arguments.

The entries in the hash become available as variables in the local scope used when evaluating the
template. When specified, they replace access to the calling context. (This is explained further in [Passing Arguments to the Template](#passing-arguments-to-the-template)).

File Extensions
---

EPP templates should be given an extension that identities them as such.
EPP templates should be named with the extension `.epp`.

    404page.epp
    404page.html.epp

Both of these signal that this is an epp file, and could be used like
this:

    epp ('404page.epp')

The `epp()` functions will appended `.epp` to the given name if it is missing.

Passing Arguments to the Template
---

To make it easier to create reusable templates, EPP offers a way to declare required and
optional parameters directly in a template. The caller of the template must supply the
values of the required parameters (it is an error if they are not given).

When a template declares parameters, the call must supply a hash. Additionally, when a hash
is given, it replaces access to the calling scope. The template still has access to global
scope. This applies both to `inline_epp` and `epp`.

Declaring parameters provides additional isolation between templates and the scope/closure
where they are evaluated.
Security concerned users may want to go as far as dictating that templates must always use parameters.

A template has access to the scope in which it is defined (unless it has declared parameters),
plus the global scope. For a file based template, the defining scope is global scope
(access is never given to the scope where a call to `app` is made), and
for an inline template, the defining scope is the scope where `inline_epp` is called.

These are equivalent:

    $x = 'droid'
    $message = "This is not the $x you are looking for."

    $x = 'droid'
    $message = inline_epp('This is not the <%= $x %> you are looking for.')

This only works if there is a `$x` in global scope:

    $message = epp('404page.epp') # Assuming it contains the text above

We can pass arguments even if there are no parameters declared:

    $x = 'page'
    inline_epp('This is not the <%= $x %> you are looking for.', { 'x' => 'droid'})
    
    # => 'This is not the droid you are looking for.'


Note that it is possible to pass arguments (i.e. set variables in the template's local scope) irrespective of the template
having declared parameters or not.

Also note that it is not possible to pass qualified names (they can not be set in a local scope). Thus, if there is a desire
to shadow a name, the template should be designed to take an (unqualified) parameter and it is the callers responsibility to override a default setting as in the following example:

    <%- |$foo = $some::where::foo| -%>

Now, a user will get the default `$some::where::foo` or can pass an overriding value for `foo`.

When a templates does not declare parameters, any parameters given in a hash in the call to
`epp` or `inline_epp` are added to the template's scope, and they may shadow variables in the
defining and global scopes.

Declaring Template Parameters
-----------------------------

It is possible to declare required and optional parameters (with default values) inside the template 
by starting the template with a pipe separated parameters list in the same fashion parameters are
declared for a `lambda`.

Since the "call" passes arguments by name instead of by position it is allowed to place required and optional parameters in any order (although the convention is to place optional parameters last).

    <%- |$x, $y, $z = 'unicorn'| -%>
    Here are your values: <%= $x %>, <%= $y %>. Don't forget to feed the <%= $z %>

In this example, the template is declared to take three parameters, the required `$x` and `$y`, and 
the optional `$z`, which if not given is assigned the value `unicorn`.

Note the use of `<%-` and `-%>` which suppresses the leading and trailing whitespace (if there is 
leading whitespace before the epp tag containing the parameters they are not recognized as parameters
to the EPP template and will result in a syntax error).

As stated earlier, access to variables in the defining scope is not available when parameters are declared.

Nesting
-------
As in ERB it is not possible to nest template tags. It is however possible to call the epp template functions.

No EPP in regular logic
-----------------------

It is not possible to switch to EPP/text mode when parsing regular manifests. (One could imagine 
turning the epp tags "inside out", and switch mode, but this is not supported.

The following is illegal and will result in Syntax error when parsed with --parser future as a regular manifest:

    $a = 10
    $b = %>This is template text with <%= $a %> %<

In fact, any EPP tags in a manifest (.pp) will result in a syntax error. The EPP tags are only 
recognized when performing template parsing invoked via the two EPP functions.

Mixing Text and Logic
---------------------

It is important to consider how text and logic combine in a template.
A text segment in the template results in evaluation of a "render" operation
that appends the text to the overall rendered result. The rendering always returns nil as
it is of questionable value to return the appended text (probably just a
source of errors). Thus, if an attempt is made to assign the result of
rendered text as in this example:

    Hello <% $x = %>world<%= $x %>

The result is simply 'Hello world' since $x is assigned nil/undef
which renders as an empty string.

Switching to text mode is only supported at positions in the puppet
language where a function call may appear. This limitation should have
no practical consequence, but is required since the lexer/parser
combination needs to be able to look ahead in certain circumstances, and
the semantics for evaluation order of certain grammar positions is simply not
defined (or there is nothing there to evaluate in the first place).

As an (illegal) example:

    Here is a list of servers:
    <ul>
    <% $array.each %> TEXT <% |$x|
      <li><%= $x %></li>
    <% } %>
    </ul>
    </p>

This will produce a syntax error.


Allowed expressions
-------------------

Since templates can contain puppet code and thus can contain any type of
expression it is important to constrain the set of legal
expressions/statements.

Variable assignment is by virtue of the implementation already protected
(assigned variables go out of scope when the template has been
evaluated, and it is not possible to refer to them from other scopes).

Defines, classes and nodes can not be created in logic embedded in EPP as they
must appear in "toplevel" constructs (which an EPP template is not).

There is however no protection against users creating resources inside
the template (nor if they do this via function calls to `create_resources`).
There is also no protection against realizing/collecting resources.

Arguably, these operations are not intended to be used inside templates (as a side effect of
producing text) and should probably be validated as not being allowed.
The current implementation does not validate these expressions, but this can be
added. (As a comparison, it is possible to create resources by side-effect in an
interpolated string). If someone chooses to use these questionable expressions inside a template, 
there is no real harm; only poor design.

Alternatives and Recommendation
===============================

Alternative: Use Puppet Style Interpolation
-------------------------------------------

Why not simply use Puppet string interpolation? The `<%= $x %>` syntax
is very heavy compared to puppet's simple `$x`, or `${x}`?

The idea is that the EPP tags are different enough to not clash with
most types of file content. If we pick `$x`, or `${x}`, the author may
instead have to use lots of escapes.

The proposal is based on using the ERB tags to make it easy to transition from
existing ERB templates by just replacing variable references from the `@form`, to `$form`.

Alternative: Recognize `@x` as a Variable
-----------------------------------------

One consideration was to allow `@x` to be a variable reference in EPP. This would mean
that some ERB templates would immediately work as EPP templates. There is however grammar problems
with using `@` in this fashion and the idea was abandoned. (Having more than one way to reference variables is
also unwanted, so they would come with deprecation warnings. Doing a search/replace is easy enough).


Risks and Assumptions
---------------------

The implementation mostly concerns lexing. The model objects (EppExpression, RenderStringExpression, RenderExpression)
are straight forward and small. Since the functionality is isolated it is easy to test, and
should not have any far-reaching implication.

The implementation makes use of a scope variable `@epp` that is impossible to access from the puppet language. It is
assumed that scope will continue to not enforce variable naming rules (it does not do this today). If at some point
this is added to scope, it will need to provide direct support for collecting render operations.

Dependencies
------------

The implementation is built on the `--parser future` / `--evaluator future` implementation.


Impact
------

- Compatibility:
  EPP does not introduce any non backwards compatible constructs.

- Security:
  Potentially a positive effect on security as ERB templates have access to anything in puppet. EPP does not.

- Performance/scalability:
  Should be on par with ERB.

- User experience:
  Non Rubyist users of Puppet will welcome EPP.

- I18n/L10n:
  Neutral.

- Portability:
  Helps with portability since EPP is based on Puppet. When Puppet is ported so is EPP. There is not dependencies on ERB.

- Packaging/installation:
  Neutral. The implementation is part of Puppet.

- Documentation:
  The documentation describing the ERB support is easily edited to use Puppet Language. It should be recommended
  in favor of the ERB support.

- Spin-offs/Future work:
  This makes it possible to deprecate the use of ERB. It reduces the dependency on Ruby thus removing one (out of many) issues
  if there is a future desire to implement a puppet runtime on something other than Ruby.


