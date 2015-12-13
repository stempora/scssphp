---
layout: default
title: Documentation
---

# scssphp {{ site.current_version }} Documentation

## PHP Interface

### Including

The project can be loaded through a `composer` generated auto-loader.

Alternatively, the entire project can be loaded through a utility file.
Just include it somewhere to start using it:

{% highlight php startinline=true %}
require_once 'scssphp/scss.inc.php';
{% endhighlight %}

### Compiling

In order to manually compile code from PHP you must create an instance of the
`Compiler` class. The typical flow is to create the instance, set any compile time
options, then run the compiler with the `compile` method.

{% highlight php startinline=true %}
use Leafo\ScssPhp\Compiler;

$scss = new Compiler();

echo $scss->compile('
  $color: #abc;
  div { color: lighten($color, 20%); }
');
{% endhighlight %}

* `compile($scssCode)` will attempt to compile a string of SCSS code. If it
  succeeds then the CSS will be returned as a string. If there is any error, an
  exception is thrown with an appropriate error message.

### Import Paths

When you import a file using the `{% raw %}@{% endraw %}import` directive, the current path of your
PHP script is used as the search path by default. This is often not what
you want, so there are two methods for manipulating the import path:
`addImportPath`, and `setImportPaths`.

* `addImportPath($path)` will append `$path` to the list of the import
  paths that are searched.

* `setImportPaths($pathArray)` will replace the entire import path with
  `$pathArray`. The value of `$pathArray` will be converted to an array if it
  isn't one already.

If the import path is set to `array()` then importing is effectively disabled.
The default import path is `array('')`, which means the current directory.

{% highlight php startinline=true %}
use Leafo\ScssPhp\Compiler;

$scss = new Compiler();
$scss->setImportPaths('assets/stylesheets/');

// will search for 'assets/stylesheets/mixins.scss'
echo $scss->compile('{% raw %}@{% endraw %}import "mixins.scss";');
{% endhighlight %}

Besides adding static import paths, it's also possible to add custom import
functions. This allows you to load paths from a database, or HTTP, or using
files that SCSS would otherwise not process (such as vanilla CSS imports).

{% highlight php startinline=true %}
use Leafo\ScssPhp\Compiler;

$scss = new Compiler();
$scss->addImportPath(function($path) {
    if (!file_exists('stylesheets/'.$path)) return null;
    return 'stylesheets/'.$path;
});

// will import 'stylesheets/vanilla.css'
echo $scss->compile('{% raw %}@{% endraw %}import "vanilla.css";');
{% endhighlight %}

A list of the compiled files (both the primary file and its imports) can be
retrieved using the `getParsedFiles` method.

* `getParsedFiles()` returns an associative array where the keys are
  the file names and the values are the corresponding file's last-modified
  timestamp.

### Preset Variables

You can preset variables before compilation by using the `setVariables($vars)`
method. If the variable is also defined in your scss source, use the `!default`
flag to prevent your preset variables from being overridden.

{% highlight php startinline=true %}
use Leafo\ScssPhp\Compiler;
-
$scss = new Compiler();
$scss->setVariables(array(
    'var' => 'false',
));

echo $scss->compile('$var: true !default;');
{% endhighlight %}

Likewise, you can retrieve the preset variables using the `getVariables()`
method, and unset a variable using the `unsetVariable($name)` method.

### Output Formatting

It's possible to customize the formatting of the output CSS by changing the
default formatter.

Five formatters are included:

* `Leafo\ScssPhp\Formatter\Expanded`
* `Leafo\ScssPhp\Formatter\Nested` *(default)*
* `Leafo\ScssPhp\Formatter\Compressed`
* `Leafo\ScssPhp\Formatter\Compact`
* `Leafo\ScssPhp\Formatter\Crunched`

We can change the formatting using the `setFormatter` method.

* `setFormatter($formatterName)` sets the current formatter to `$formatterName`,
  the name of a class as a string that implements the formatting interface. See
  the source for `Leafo\ScssPhp\Formatter\Expanded` for an example.

Given the following SCSS:

{% highlight scss %}
/*! Comment */
.navigation {
    ul {
        line-height: 20px;
        color: blue;
        a {
            color: red;
        }
    }
}

.footer {
    .copyright {
        color: silver;
    }
}
{% endhighlight %}

The formatters output the following:

`Leafo\ScssPhp\Formatter\Expanded`:

{% highlight css %}
/*! Comment */
.navigation ul {
  line-height: 20px;
  color: blue;
}
.navigation ul a {
  color: red;
}
.footer .copyright {
  color: silver;
}
{% endhighlight %}

`Leafo\ScssPhp\Formatter\Nested`:

{% highlight css %}
/*! Comment */
.navigation ul {
  line-height: 20px;
  color: blue; }
    .navigation ul a {
      color: red; }

.footer .copyright {
  color: silver; }
{% endhighlight %}

`Leafo\ScssPhp\Formatter\Compact`:

{% highlight css %}
/*! Comment */
.navigation ul { line-height:20px; color:blue; }

.navigation ul a { color:red; }

.footer .copyright { color:silver; }
{% endhighlight %}

`Leafo\ScssPhp\Formatter\Compressed`:

{% highlight css %}
/* Comment*/.navigation ul{line-height:20px;color:blue;}.navigation ul a{color:red;}.footer .copyright{color:silver;}
{% endhighlight %}

`Leafo\ScssPhp\Formatter\Crunched`:

{% highlight css %}
.navigation ul{line-height:20px;color:blue;}.navigation ul a{color:red;}.footer .copyright{color:silver;}
{% endhighlight %}

You can also change the number precision using the `setNumberPrecision($precision)`
method. (The default precision is 5.)

### Source Line Debugging

You can output the original SCSS line numbers within the compiled CSS file for better frontend debugging.

This works well in combination with frontend debugging tools such as https://addons.mozilla.org/de/firefox/addon/firecompass-for-firebug/

To activate this feature, call the `setLineNumberStyle` method after creating a new instance of class `Compiler`.

{% highlight php startinline=true %}
use Leafo\ScssPhp\Server;
use Leafo\ScssPhp\Compiler;

$directory = 'css';

$scss = new Compiler();
$scss->setLineNumberStyle(Compiler::LINE_COMMENTS);

$server = new Server($directory, null, $scss);
$server->serve();
{% endhighlight %}

You can also call the `compile` method directly (without using an instance of `Server` like above)

{% highlight php startinline=true %}
use Leafo\ScssPhp\Server;
use Leafo\ScssPhp\Compiler;

$scss = new Compiler();
$scss->setLineNumberStyle(Compiler::LINE_COMMENTS);

echo $scss->compile('
  $color: #abc;
  div { color: lighten($color, 20%); }
');
{% endhighlight %}

### Custom Functions

It's possible to register custom functions written in PHP that can be called
from SCSS. Some possible applications include appending your assets directory
to a URL with an `asset-url` function, or converting image URLs to an embedded
data URI to reduce the number of requests on a page with a `data-uri` function.

We can add and remove functions using the methods `registerFunction` and
`unregisterFunction`.

* `registerFunction($functionName, $callable, $prototype)` assigns the callable value to
  the name `$functionName`. The name is normalized using the rules of SCSS.
  Meaning underscores and dashes are interchangeable. If a function with the
  same name already exists then it is replaced. The optional `$prototype` is an
  array of parameter names.

* `unregisterFunction($functionName)` removes `$functionName` from the list of
  available functions.

The `$callable` can be anything that PHP knows how to call using
`call_user_func`. The function receives two arguments when invoked. The first
is an array of SCSS typed arguments that the function was sent. The second is an
array of SCSS values corresponding to keyword arguments (aka kwargs).

The SCSS *typed arguments* and *kwargs* are actually just arrays that represent
SCSS values.  SCSS has different types than PHP, and this is how **scssphp**
represents them internally.

For example, the value `10px` in PHP would be `array('number', 1, 'px')`. There
is a large variety of types. Experiment with a debugging function like `print_r`
to examine the possible inputs.

The return value of the custom function can either be a SCSS type or a basic
PHP type (such as a string or a number). If it's a PHP type, it will be converted
automatically to the corresponding SCSS type.

As an example, a function called `add-two` is registered, which adds two numbers
together. PHP's anonymous function syntax is used to define the function.

{% highlight php startinline=true %}
use Leafo\ScssPhp\Compiler;

$scss = new Compiler();

$scss->registerFunction(
  'add-two',
  function($args) {
    list($a, $b) = $args;

    return $a[1] + $b[1];
  }
);

$scss->compile('.ex1 { result: add-two(10, 10); }');
{% endhighlight %}

Using a prototype and kwargs, functions can take named parameters. In this next example,
we register a function called `divide` which divides a named dividend by a named divisor.

{% highlight php startinline=true %}
use Leafo\ScssPhp\Compiler;

$scss = new Compiler();

$scss->registerFunction(
  'divide',
  function($args, $kwargs) {
    return $kwargs['dividend'][1] / $kwargs['divisor'][1];
  },
  array('dividend', 'divisor')
);

$scss->compile('.ex2 { result: divide($divisor: 2, $dividend: 30); }');
{% endhighlight %}

Note: in the above examples, we lose the units of the number, and we
also don't do any type checking. This will have undefined results if we give it
anything other than two numbers.

For feature detection via the `feature-exists()` built-in function, you can register
your custom feature using the `addFeature` method:

* `addFeature($name)` registers the `$name`.

## SCSS Server

The SCSS server is a small class that helps with automatically compiling SCSS.

It's an endpoint for your web application that searches for SCSS files in a
directory then compiles and serves them as CSS. It will only compile
files if they've been modified (or one of the imports has been modified).

### Using `serveFrom`

`Server::serveFrom` is a simple to use function that should handle most cases.

For example, create a file `style.php`:

{% highlight php startinline=true %}
use Leafo\ScssPhp\Server;

$directory = 'stylesheets';

Server::serveFrom($directory);
{% endhighlight %}

Going to the URL `example.com/style.php/style.scss` will attempt to compile
`style.scss` from the `stylesheets` directory, and serve it as CSS.

* `Server::serveFrom($directory)` will serve SCSS files out of
  `$directory`. It will attempt to get the path to the file out of
  `$_SERVER['PATH_INFO']`. (It also looks at the GET parameter `p`)

If it can not find the file it will return an HTTP 404 page:

{% highlight text %}
/* INPUT NOT FOUND scss v0.0.1 */
{% endhighlight %}

If the file can't be compiled due to an error, then an HTTP 500 page is
returned. Similar to the following:

{% highlight text %}
Parse error: failed at 'height: ;' stylesheets/test.scss on line 8
{% endhighlight %}

By default , the SCSS server must have write access to the style sheet
directory. It writes its cache in a special directory called `scss_cache`.

Also, because SCSS server writes headers, make sure no output is written before
it runs.

### Using `Leafo\ScssPhp\Server`

Creating an instance of `Server` is just another way of accomplishing what
`serveFrom` does. It lets us customize the cache directory and the instance
of the `Compiler` that is used to compile

* `new Server($sourceDir, $cacheDir, $scss)` creates a new server that
  serves files from `$sourceDir`. The cache dir is where the cached compiled
  files are placed. When `null`, `$sourceDir . '/scss_cache'` is used. `$scss`
  is the instance of `scss` that is used to compile.

Just call the `serve` method to let it render its output.

Here's an example of creating a SCSS server that outputs compressed CSS:

{% highlight php startinline=true %}
use Leafo\ScssPhp\Compiler;
use Leafo\ScssPhp\Server;

$scss = new Compiler();
$scss->setFormatter('Leafo\ScssPhp\Formatter\Compressed');

$server = new Server('stylesheets', null, $scss);
$server->serve();
{% endhighlight %}

## Command Line Tool

A really basic command line tool is included for integration with scripts. It
is called `pscss`. It reads SCSS from either a named input file or standard in,
and returns the CSS to standard out.

Usage: bin/pscss [options] [input-file]

### Options

If passed the flag `-h` (or `--help`), input is ignored and a summary of the command's usage is returned.

If passed the flag `-v` (or `--version`), input is ignored and the current version is returned.

If passed the flag `-T`, a formatted parse tree is returned instead of the compiled CSS.

The flag `-f` (or `--style`) can be used to set the [formatter](#output-formatting):

{% highlight bash %}
$ bin/pscss -f compressed < styles.scss
{% endhighlight %}

The flag `-i` (or `--load_paths`) can be used to set import paths for the loader. On Unix/Linux systems,
the paths are colon separated.

The flag `-p` (or `--precision`) can be used to set the decimal number precision. The default is 5.

The flag `--debug-info` can be used to annotate the selectors with CSS {% raw %}@{% endraw %}media queries that identify the source file and line number.

The flag `--line-comments` (or `--line-numbers`) can be used to annotate the selectors with comments that identify the source file and line number.