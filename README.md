# Setup For Local Development and Testing
###### or, how to python like you mean it


**TL;DR:** Testing locally can literally save you hours of headache. Virtualize your python environment against 2.6.6
and write unit tests before pushing to Woozle.

## Motivation

The hadoop cluster at U of C does not produce helpful stack traces to the console during an application run if something
goes wrong. Whether it's a simple syntax error or an index out of range exception, relying on the "run and pray"
debugging technique in this case will be useless. Not to mention its bad practice in general.

The hadoop cluster at U of C uses python 2.6.6, which further complicates things. Running your mappers and reducers
locally against python 2.7.0 won't help when it comes to syntax differences such as:

2.7.0 set syntax:
`primes = {1,3,5,7}`

2.6.6 set syntax:
`primes = set([1,3,5,7])`

Easy to get snagged by this sorta thing.

So, we need a way to have 2.6.6 installed and run locally in such a way that it doesnt blow away our nice python (2.7.0) 
and python3 installations. To do this, we'll use: [pyenv](https://github.com/pyenv/pyenv/), 
[pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) and [virtualenv](https://github.com/pypa/virtualenv)


## Disclaimer

The instructions here were written for OSX. The pure-linux flavours are similar. Windows folks, I'd suggest using an
ubuntu virtual machine and acting accordingly. All the cool kids are doing it. Peer pressure works.  

If you're an OSX user and aren't using homebrew, I recommend it. 


Likely subsitutions for linux folk:

* `brew install` -> `sudo apt install`
* `.bash_profile` -> `.bashrc` (or whatever)

## Tool Overview

### virtualenv

Read [this](https://virtualenv.pypa.io/en/stable/)!


### pyenv

Read [this](https://github.com/pyenv/pyenv)!
    

### pyenv-virtualenv

Surprise! Read [this](https://github.com/pyenv/pyenv-virtualenv).

### BONUS pipenv

Not used for hadoop stuff, but hella good practice. Read up on this bad boy [here](https://github.com/pypa/pipenv).
Never look back.

## Installation

Ok, we need to install pyenv and pyenv-virtaulenv. In a terminal, run:

~~~bash
# update homebrew
brew update

# get the stuff
brew install pyenv
brew install pyenv-virtualenv 
~~~

And do a little bash magic to make the shell shims (google dat, its how pyenv do) and autocompletion work correctly
~~~bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bash_profile
echo -e 'if which pyenv-virtualenv-init > /dev/null; then eval "$(pyenv virtualenv-init -)"; fi' >> ~/.bash_profile
~~~

now restart your shell or run `source ~/.bash_profile`. Your call.

## Use It!

OK! Now you're ready to do the thing. Still in your terminal, navigate to wherever you want to do your development and 
run the following:

~~~bash
# install python 2.6.6
pyenv install 2.6.6

# create a virtual environment named hadoop that uses python 2.6.6
pyenv virtualenv 2.6.6 hadoop

# this will activate the hadoop virtual environment
pyenv activate hadoop

# check the version of python currently active (should be 2.6.6)
python --version
~~~

You'll know the virtual environment is active because the shell prompt will start with (hadoop) or whatever name you
chose.

Now you can run your mapper and reducer modules locally in a python environment that won't fool you!

### Sanity Check

Try this. In a fresh directory, run each of these individually and observe the output:

`python --version` (you probably see 2.7.0 or 3.x.x)

`pyenv virtualenv 2.6.6 test`

`pyenv activate test` (now your shell will have the name 'test' in it)

`python --version` (Oh my gosh! we got python 2.6.6!)

`pyenv deactivate` (no more 'test')

`python --version` (and we're back!)

Neato.


## BONUS Unit Test Stuff

The real gold here is that having this enables some basic but realistic unit testing. Since Hadoop streaming
uses stdin and stdout, we need to [monkey patch](https://en.wikipedia.org/wiki/Monkey_patch) them. Lucky us, that's
easy!

From here on out, I assume you know about the unittest module and how to run tests.

Suppose you had the following directory structure:

```
.
├── mapper.py
├── reducer.py
└── tests.py

```



Nice and simple. So `tests.py` might look something like this:





~~~python
#!/usr/bin/env python
# tests.py

import StringIO
import ast
import sys
import unittest


# This is a helper function to take the our monkey-patched stdio and return a line-wise list of strings.
# This makes assertions easier.
def get_list(strio):
    return ast.literal_eval(str(strio.getvalue().strip().split('\n')))


class TestExample(unittest.TestCase):

    # this is run before every test_<whatever> method below
    def setUp(self):
        # hold onto references to legit stdx objects
        self._stdin = sys.stdin
        self._stdout = sys.stdout
        self._stderr = sys.stderr



    def test_mapper(self):
        # load up stdin with some data to be read by our mapper
        patched_input = StringIO.StringIO("""
Danny Vacon loves bacon. 
Bacon! 


bAcOn.;
    
""")
        sys.stdin = patched_input # monkey-patch!

        # patch up stdout to hang on to the result of print statements
        out = StringIO.StringIO()
        sys.stdout = out # monkey-patch!

        # "run" the mapper module.
        import mapper

        # collect the output as a list of strings.
        result = get_list(out)

        # force the interpreter to deeply "unimport" the mapper module
        # if you have multiple test methods for mapper, this is super important.
        del mapper
        sys.modules.pop("mapper")

        # clean up your messes.
        patched_input.close()
        out.close()

        # build your expectation
        expectation = ['danny 1', 'vacon 1', 'loves 1', 'bacon 1', 'bacon 1', 'bacon 1']

        # find out if you nailed it or not.
        self.assertEqual(result, expectation)


    # just like above!
    def test_reducer(self):
        patched_input = StringIO.StringIO("""bagel 1
bagel 1
bagel 1
high kicks 1
high kicks 1
bacon pancake 1
""")
        sys.stdin = patched_input
        out = StringIO.StringIO()
        sys.stdout = out

        import reducer
        result = get_list(out)
        del reducer
        sys.modules.pop("reducer")

        patched_input.close()
        out.close()

        expectation = ['bagel 3', 'high kicks 2', 'bacon pancake 1']
        self.assertEqual(result, expectation)

    # this is run after each test_<whatever> method above.
    def tearDown(self):
        # un-monkey-patch!
        sys.stdin = self._stdin
        sys.stdout = self._stdout
        sys.stderr = self._stderr


if __name__ == '__main__':
    unittest.main()

~~~

Now go have fun.
