[![Build Status](https://travis-ci.org/dmoody256/harfbuzz.svg?branch=master)](https://travis-ci.org/dmoody256/harfbuzz)
[![Build status](https://ci.appveyor.com/api/projects/status/wm9stjqj6g4pk2s4/branch/master?svg=true)](https://ci.appveyor.com/project/dmoody256/harfbuzz/branch/master)
# harfbuzz
harfbuzz with scons
```
sudo apt-get install binutils gcc g++ pkg-config ragel gtk-doc-tools libfreetype6-dev libglib2.0-dev libcairo2-dev libicu-dev libgraphite2-dev python python-pip gobject-introspection libgirepository1.0-dev 
sudo python -m pip install -U pip
sudo -H pip install -U wheel setuptools 
sudo -H pip install -U scons
scons --have-freetype --have-graphite2 --have-glib --have-icu --have-gobject --have-introspection
```
