<!--
.. title: Setting up python's Windows embeddable distribution (properly)
.. slug: setting-up-python-windows-embeddable-environment-properly
.. date: 2019-09-09 14:48:18 UTC+08:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

# Embeddable distribution
If there is anything I like about Windows as a pythonist, it must be that 
you can use [embedded distribution](https://docs.python.org/3/using/windows.html#windows-embeddable) of python.  

>The embedded distribution is a ZIP file containing a minimal Python environment. 
It is intended for acting as part of another application, rather than being directly 
accessed by end-users.

In my opinion, it is a portable, ready-to-ship virtualenv. However, the 
embedded distribution comes with some limitation:

>Third-party packages should be installed by the application installer alongside 
the embedded distribution. Using pip to manage dependencies as for a regular Python 
installation is not supported with this distribution, though with some care it may be 
possible to include and use pip for automatic updates. In general, third-party packages 
should be treated as part of the application (“vendoring”) so that the developer can 
ensure compatibility with newer versions before providing updates to users.

Sounds scary right? It said it doesn't even support pip. Don't worry, follow
these simple steps, you will have a fully workable embedded environment.

## Get the distribution

1. Go to <https://www.python.org/downloads/windows/>
2. Choose the version python you like and download the corresponding `Windows x86-64 embeddable zip file`.
3. Unzip the file. 

To make this tutorial easy to follow, I am assuming that you have downloaded `Python3.7` and unzipped it to
`C:\python\`.

## Get pip

The distribution does not have pip installed in place, you need to install it yourself:
1. Download `get-pip.py` at <https://bootstrap.pypa.io/get-pip.py>
2. Save it to `c:\python\get-pip.py`
3. In command-line run `C:\python\python get-pip.py`
4. pip is now installed

## Config path

The runtime of this distribution doesn't have an empty string `''` added in `sys.path`,
so that the `current directory` is not added into `sys.path`, to solve the problem,
you need to:

1. Open `C:\python\python37._pth`.
2. Uncomment the line `#import site` and save.
3. Create the following python file at `c:\python\sitecustomize.py`:
```python
import sys
sys.path.insert(0, '')
```

## lib2to3 issue
 
You will encounter the following error when you install some packages:
```python
error: [Errno 0] Error: 'lib2to3\\Grammar3.6.5.final.0.pickle'
```

1. Unzip `C:\python\python37.zip` to a `new folder`
2. Delete `C:\python\python37.zip`
3. Rename the `new folder` to `python37.zip` 

Python's import module is able to treat zip file as folder however, it cannot
read pickle file inside a zip file, so unzip it and rename it.

## Running pip

If you don't want to mess with you PATH, you can simply do the following in your window command prompt:

1. `CD C:\python\Scripts`
2. `C:\python\python pip install xxxxx`

## Running Scripts

Again if you don't want to mess with you PATH, you can simply do in your window command prompt:

1. `C:\python\python <path to your script>`



