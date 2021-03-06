# SYNOPSIS

SWAT is Simple Web Application Test ( Tool )

    $  swat examples/google/ google.ru
    /home/vagrant/.swat/reports/google.ru/00.t ..
    # start swat for google.ru/
    # try num 2
    ok 1 - successful response from GET google.ru/
    # data file: /home/vagrant/.swat/reports/google.ru/content.GET.txt
    ok 2 - GET / returns 200 OK
    ok 3 - GET / returns Google
    1..3
    ok
    All tests successful.
    Files=1, Tests=3, 12 wallclock secs ( 0.00 usr  0.00 sys +  0.02 cusr  0.00 csys =  0.02 CPU)
    Result: PASS

# WHY

I know there are a lot of tests tool and frameworks, but let me  briefly tell _why_ I created swat.
As devops I update a dozens of web application weekly, sometimes I just have _no time_ sitting and wait 
while dev guys or QA team ensure that deploy is fine and nothing breaks on the road. 
So I need a **tool to run smoke tests against web applications**. 
Not tool only, but the way to **create such a tests from the scratch in way easy and fast enough**. 

So this how I came up with the idea of swat. 

# Key features

SWAT:

- is very pragmatic tool designed for job to be done in a fast and simple way
- has simple and yet flexible DSL with low price mastering ( see my tutorial )
- produces [TAP](https://testanything.org/) output
- leverages famous [perl prove](http://search.cpan.org/perldoc?prove) and [curl](http://curl.haxx.se/) utilities

# Install

Swat relies on curl utility to make http requests. Thus first you need to install curl:

    $ sudo apt-get install curl

Also swat client is bash script so you need a bash. 

Then you install swat cpan module:

    sudo cpan install swat

## Install from source

    # useful for contributors
    perl Makefile.PL
    make
    make test
    make install

# Swat mini tutorial

For those who love to make long story short ...

## Create tests

    mkdir  my-app/ # create a project root directory to contain tests

    # define http URIs application should response to

    mkdir -p my-app/hello # GET /hello
    mkdir -p my-app/hello/world # GET /hello/world

    # define the content to return by URIs

    echo 200 OK >> my-app/hello/get.txt
    echo 200 OK >> my-app/hello/world/get.txt

    echo 'This is hello' >> my-app/hello/get.txt
    echo 'This is hello world' >> my-app/hello/world/get.txt

## Run tests

    swat ./my-app http://127.0.0.1

# DSL

Swat DSL consists of 2 parts. Routes and Swat Data.

## Routes

Routes are http resources a tested web application should have.

Swat utilize file system to get know about routes. Let we have a following project layout:

    example/my-app/
    example/my-app/hello/
    example/my-app/hello/get.txt
    example/my-app/hello/world/get.txt

When you give swat a run

    swat example/my-app 127.0.0.1

It will find all the _directories with get.txt or post.txt files inside_ and "create" routes:

    GET hello/
    GET hello/world

When you are done with routes you need to set swat data.

## Swat data

Swat data is DSL to describe/generate validation checks you apply to content returned from web application.

Swat data is stored in swat data files, named get.txt or post.txt. 

The validation process looks like:

- Swat recursively find files named **get.txt** or **post.txt** in the project root directory to get swat data.
- Swat parse swat data file and _execute_ entries found. At the end of this process swat creates a _final check list_ with 
["Check Expressions"](#check-expressions).
- For every route swat makes http requests to web application and store content into text file 
- Every line of text file is validated by every item in a _final check list_

_Objects_ found in test data file are called _swat entries_. There are _3 basic type_ of swat entries:

- Check Expressions
- Comments
- Perl Expressions and Generators

### Check Expressions

This is most usable type of entries you  may define at swat data file. _It's just a string should be returned_ when swat request a given URI. Here are examples:

    200 OK
    Hello World
    <head><title>Hello World</title></head>

Using regexps

Regexps are check expressions with the usage of <perl regular expressions> instead of plain strings checks.
Everything started with `regexp:` marker would be treated as perl regular expression.

    # this is example of regexp check
    regexp: App Version Number: (\d+\.\d+\.\d+)

### Comments

Comments entries are lines started with `#` symbol, swat will ignore comments when parse swat data file. Here are examples.

    # this http status is expected
    200 OK
    Hello World # this string should be in the response
    <head><title>Hello World</title></head> # and it should be proper html code

### Perl Expressions

Perl expressions are just a pieces of perl code to _get evaled_ by swat when parsing test data files.

Everything started with `code:` marker would be treated by swat as perl code to execute.
There are a _lot of possibilities_! Please follow [Test::More](https://metacpan.org/pod/search.cpan.org#perldoc-Test::More) documentation to get more info about useful function you may call here.

    code: skip('next test is skipped',1) # skip next check forever
    HELLO WORLD


    code: skip('next test is skipped',1) unless $ENV{'debug'} == 1  # conditionally skip this check
    HELLO SWAT

# Generators

Swat entries generators is the way to _create new swat entries on the fly_. Technically speaking it's just a perl code which should return an array reference:
Generators are very close to perl expressions ( generators code is also get evaled ) with major difference:

Value returned from generator's code should be  array reference. The array is passed back to swat parser so it can create new swat entries from it. 

Generators entries start with `:generator` marker. Here is example:

    # Place this in swat data file
    generator: [ qw{ foo bar baz } ]

This generator will generate 3 swat entries:

    foo
    bar
    baz

As you can guess an array returned by generator should contain _perl strings_ representing swat entries, here is another example:
with generator producing still 3 swat entities 'foo', 'bar', 'baz' :

    # Place this in swat date file
    generator: my %d = { 'foo' => 'foo value', 'bar' => 'bar value' }; [ map  { ( "# $_", "$data{$_}" )  } keys %d  ] 

This generator will generate 3 swat entities:

    # foo
    foo value
    # bar
    bar value

There is no limit for you! Use any code you want with only requirement - it should return array reference. 
What about to validate web application content with sqlite database entries?

    # Place this in swat data file
    generator:                                                          \
    
    use DBI;                                                            \
    my $dbh = DBI->connect("dbi:SQLite:dbname=t/data/test.db","","");   \
    my $sth = $dbh->prepare("SELECT name from users");                  \
    $sth->execute();                                                    \
    my $results = $sth->fetchall_arrayref;                              \
    
    [ map { $_->[0] } @${results} ]

As an example take a loot at examples/swat-generators-sqlite3 project

# Multiline expressions

Sometimes code looks more readable when you split it on separate chunks. When swat parser meets  `\` symbols it postpone entry execution and
add next line to buffer. This is repeated till no `\` found on next. Finally swat execute _"accumulated"_ swat entity.

Here are some examples:

    # Place this in swat data file
    generator:                  \
    my %d = {                   \
        'foo' => 'foo value',   \
        'bar' => 'bar value',   \
        'baz' => 'baz value'    \
    };                          \
    [                                               \
        map  { ( "# $_", "$data{$_}" )  } keys %d   \
    ]                                               \

    # Place this in swat data file
    generator: [            \
            map {           \
            uc($_)          \
        } qw( foo bar baz ) \
    ]

    code:                                                       \
    if $ENV{'debug'} == 1  { # conditionally skip this check    \
        skip('next test is skipped',1)                          \ 
    } 
    HELLO SWAT

Multiline expressions are only allowable for perl expressions and generators 

# Generators and Perl Expressions Scope

Swat uses _perl string eval_ when process generators and perl expressions code, be aware of this. 
Follow [http://perldoc.perl.org/functions/eval.html](http://perldoc.perl.org/functions/eval.html) to get more on this.

# PERL5LIB

Swat adds **$project\_root\_directory/lib** to PERL5LIB , so this is convenient convenient to place here custom perl modules:

    example/my-app/lib/Foo/Bar/Baz.pm

As an example take a loot at examples/swat-generators-with-lib/ project

# Anatomy of swat 

Once swat runs it goes through some steps to get job done. Here is description of such a steps executed in orders

## Run iterator over swat data files

Swat iterator look for all files named get.txt or post.txt under project root directory. Actually this is simple bash find loop.

## Parse swat data file

For every swat data file find by iterator parsing process starts. Swat parse data file line by line, at the end of such a process
_a list of Test::More asserts_ is generated. Finally asserts list and other input parameters are serialized as Test::More test scenario 
written into into proper \*.t file.

## Give it a run by prove

Once swat finish parsing all the swat data files there is a whole bunch of \*.t files kept under a designated  temporary directory,
thus every swat route maps into Test::More test file with the list of asserts. Now all is ready for prove run. Internally \`prove -r \`
command is issued to run tests and generate TAP report. That is it.

Below is example how this looks like

### project structure

    vagrant@Debian-jessie-amd64-netboot:~/projects/swat$ tree examples/anatomy/
    examples/anatomy/
    |----FOO
    |-----|----BARs
    |           |---- post.txt
    |--- FOOs
          |--- get.txt

    3 directories, 2 files

### swat data files

    # /FOOs 
    FOO
    FOO2
    generator: | %w{ FOO3 FOO4 }|

    # /FOO/BARs
    BAR
    BAR2
    generator: | %w{ BAR3 BAR4 }|
    code: skip('skip next 2 tests',2);
    BAR5
    BAR6
    BAR7

### Test::More Asserts list

    # /FOOs/0.t
    SKIP {
        ok($status, "successful response from GET $host/FOOs") 
        ok($status, "GET /FOOs returns FOO")
        ok($status, "GET /FOOs returns FOO2")
        ok($status, "GET /FOOs returns FOO3")
        ok($status, "GET /FOOs returns FOO4")
    }

    # /FOO/BARs0.t
    SKIP {
        ok($status, "successful response from POST $host/FOO/BARs") 
        ok($status, "POST /FOO/BARs returns BAR")
        ok($status, "POST /FOO/BARs returns BAR")
        ok($status, "POST /FOO/BARs returns BAR3")
        ok($status, "POST /FOO/BARs returns BAR4")
        skip('skip next 2 tests',2);
        ok($status, "POST /FOO/BARs returns BAR5")
        ok($status, "POST /FOO/BARs returns BAR6")
        ok($status, "POST /FOO/BARs returns BAR7")
    }

# Hooks

Hooks are files containing any perl code to be \`required\` into the beginning of every swat test. There are 2 types of hooks:

- **project based hooks**

    File located at `$project_root_directory/hook.pm`. Project based hooks are applied for every route in project and
    could be used for _project initialization_. For example one could define generators here:

        # place this in hook.pm file:
        sub list1 { | %w{ foo bar baz } | }
        sub list2 { | %w{ red green blue } | }


        # now we could use it in swat data file
        generator:  list() 
        generator:  list2()    

- **route based hooks**

    File located at `$project_root_directory/$route_directory/hook.pm`. Routes based hook are route specific hooks and
    could be used for _route initialization_. For example one could define route specific generators here:

        # place this in hook.pm file:
        # notices that we could tell GET from POST http methods here:

        sub list1 { 

            my $list;

            if ($method eq 'GET') {
                $list = | %w{ GET_foo GET_bar GET_baz } | 
            }elsif($method eq 'POST'){
                $list = | %w{ POST_foo POST_bar POST_baz } | 
            }else{
                die "method $method is not supported"
            }
            $list;
        }


        # now we could use it in swat data file
        generator:  list() 

# Post requests

Name swat data file as post.txt to make http POST requests.

    echo 200 OK >> my-app/hello/post.txt
    echo 200 OK >> my-app/hello/world/post.txt

You may use curl\_params setting ( follow ["Swat Settings"](#swat-settings) section for details ) to define post data, there are some examples:

- `-d` - Post data sending by html form submit.

         # Place this in swat.ini file or sets as env variable:
         curl_params='-d name=daniel -d skill=lousy'

- `--data-binary` - Post data sending as is.

         # Place this in swat.ini file or sets as env variable:
         curl_params=`echo -E "--data-binary '{\"name\":\"alex\",\"last_name\":\"melezhik\"}'"`
         curl_params="${curl_params} -H 'Content-Type: application/json'"

# Dynamic routes

There are possibilities to create a undetermined routes using `:path` placeholders. Let say we have application confirming GET /foo/:whatever 
requests where :whatever is arbitrary sting like: GET /foo/one or /foo/two or /foo/baz. Using dynamic routes we could write an swat test for it.

First let's create definition for `` `whatever` `` path in swat.ini file. This is as simple as create bash variable with a random sting value:

    # Place this in swat.ini file
    export whatever=`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 5  | head -n 1` 

Now we should inform swat to use bash variable $whatever when generating request for /foo/whatever

    $ mkdir foo/:whatever 

And finally drop some check expressions for it:

    $ echo 'generator [ $ENV{"whatever"} ]' > foo/:whatever/get.txt
    

Of course there are as many dynamic parts in http requests as you need:

    # Place this in swat.ini file
    export whatever=`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 5  | head -n 1` 
    export whenever=`date +%s` 

    $ mkdir -p foo/:whatever/:whenever 
    $ echo 'generator [ $ENV{"whatever"}, $ENV{"whenever"} ]' > foo/:whatever/:whenever/get.txt

# Swat Settings

Swat comes with settings defined in two contexts:

- Environment variables ( session settings )
- swat.ini files ( home directory , project based and route based settings  )

## Environment variables

Following variables define a proper swat settings.

- `debug` - set to `1` if you want to see some debug information in output, default value is `0`
- `debug_bytes` - number of bytes of http response  to be dumped out when debug is on. default value is `500`
- `swat_debug` - run swat in debug mode, default value is `0`
- `ignore_http_err` - ignore http errors, if this parameters is off (set to `1`) returned  _error http codes_ will not result in test fails, 
useful when one need to test something with response differ from  2\*\*,3\*\* http codes. Default value is `0`
- `try_num` - number of http requests  attempts before give it up ( useless for resources with slow response  ), default value is `2`
- `curl_params` - additional curl parameters being add to http requests, default value is `""`, follow curl documentation for variety of values for this
- `curl_connect_timeout` - follow curl documentation
- `curl_max_time` - follow curl documentation
- `port`  - http port of tested host, default value is `80`
- `prove_options` - prove options, default value is `-v`

## Swat.ini files

Swat checks files named `swat.ini` in the following directories

- **~/swat.ini** - home directory settings
- **$project\_root\_directory/swat.ini** -  project based settings 
- **$route\_directory/swat.ini** - route based settings 

Here are examples of locations of swat.ini files:

     ~/swat.ini # home directory settings 
     my-app/swat.ini # project based settings
     my-app/hello/get.txt
     my-app/hello/swat.ini # route based settings ( route hello )
     my-app/hello/world/get.txt
     my-app/hello/world/swat.ini # route based settings ( route hello/world )

Once file exists at any location swat simply **bash sources it** to apply settings.

Thus swat.ini file should be bash file with swat variables definitions. Here is example:

    # the content of swat.ini file:
    curl_params="-H 'Content-Type: text/html'"
    debug=1
    try_num=3

## Settings priority table

This table describes order in which settings are applied, starts from lowest priority settings

    | context                 | location                | settings type        | priority  level |
    | ------------------------|------------------------ | -------------------- | ----------------
    | swat.ini file           | ~/swat.ini              | home directory       |       1         |
    | swat.ini file           | project root directory  | project based        |       2         |
    | swat.ini file           | route directory         | route based          |       3         |
    | environment variables   | ---                     | session              |       4         |

# Settings merge algorithm

Swat applies settings in order for every route:

- Home directory settings are applied if exist.
- Project based settings are applied if exist.
- Route based settings are applied if exist.
- And finally environment settings aer applied if exist.

# TAP

Swat produces output in [TAP](https://testanything.org/) format , that means you may use your favorite tap parsers to bring result to
another test / reporting systems, follow TAP documentation to get more on this. Here is example for converting swat tests into JUNIT format

    swat <project_root> <host> --formatter TAP::Formatter::JUnit

See also ["Prove settings"](#prove-settings) section.

# Command line tool

Swat is shipped as cpan package, once it's installed ( see ["Install swat"](#install-swat) section ) you have a command line tool called **swat**, this is usage info on it:

    swat <project_root_dir|swat_package> <host:port> <prove settings>

- **host** - is base url for web application you run tests against, you also have to define swat routes, see DSL section.
- **project\_dir** - is a project root directory
- **swat\_package** - the name of swat package, see ["Swat Packages"](#swat-packages) section

## Default Host

Sometimes it is helpful to not setup host as command line parameter but define it at $project\_root/host file. For example:

    # let's create a default host for foo/bar project

    $ cat foo/bar/host
    foo.bar.com

    $ swat foo/bar/ # will run tests for foo.bar.com

# Prove settings

Swat utilize [prove utility](http://search.cpan.org/perldoc?prove) to run tests, so all the swat options _are passed as is to prove utility_.
Follow [prove](http://search.cpan.org/perldoc?prove) utility documentation for variety of values you may set here.
Default value for prove options is  `-v`. Here is another examples:

- `-q -s` -  run tests in random and quite mode

# Swat Packages

Swat packages is portable archives of swat tests. It's easy to create your own swat packages and share with other. 

This is mini how-to on creating swat packages:

## Create swat package

Swat packages are _just cpan modules_. So all you need is to create cpan module distribution archive and upload it to CPAN.

The only requirement for installer is that swat data files should be installed into _cpan module directory_ at the end of install process. 
[File::ShareDir::Install](http://search.cpan.org/perldoc?File%3A%3AShareDir%3A%3AInstall) allows you to install 
read-only data files from a distribution and considered as best practice for such a things.

Here is example of Makefile.PL for [swat::mongodb package](https://github.com/melezhik/swat-packages/tree/master/mongodb-http):

    use inc::Module::Install;

    # Define metadata
    name           'swat-mongodb';
    all_from       'lib/swat/mongodb.pm';

    # Specific dependencies
    requires       'swat'         => '0.1.28';
    test_requires  'Test::More'   => '0';

    install_share  'module' => 'swat::mongodb', 'share';    

    license 'perl';

    WriteAll;

Here we create a swat package swat::mongodb with swat data files kept in the project\_root directory ./share and get installed into
`auto/share/module/swat-mongodb` directory.

Once we uploaded a module to CPAN repository we can use it: 

    $ cpan install swat::mongodb
    $ swat swat::mongodb 127.0.0.1:28017

# Debugging

set `swat_debug` environment variable to 1

# Examples

./examples directory contains examples of swat tests for different cases. Follow README.md files for details.

# AUTHOR

[Aleksei Melezhik](mailto:melezhik@gmail.com)

# Swat Project Home Page

https://github.com/melezhik/swat

# Thanks

To the authors of ( see list ) without who swat would not appear to light

- perl
- curl
- TAP
- Test::More
- prove
