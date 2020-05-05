Public Suffix List module for Python
====================================

This module allows you to get the public suffix, as well as the registrable domain,
of a domain name using the Public Suffix List from http://publicsuffix.org

A public suffix is a domain suffix under which you can register domain
names, or under which the suffix owner does not control the subdomains.
Some examples of public suffixes in the former example are ".com",
".co.uk" and "pvt.k12.wy.us"; examples of the latter case are "github.io" and
"blogspot.com".  The public suffix is sometimes referred to as the effective
or extended TLD (eTLD).
Accurately knowing the public suffix of a domain is useful when handling
web browser cookies, highlighting the most important part of a domain name
in a user interface or sorting URLs by web site. It is also used in a wide range
of research and applications that leverages Domain Name System (DNS) data.

This module builds the public suffix list as a Trie structure, making it more efficient
than other string-based modules available for the same purpose. It can be used
effectively in large-scale distributed environments, such as PySpark.

This Python module includes with a copy of the Public Suffix List (PSL) so that it is
usable out of the box. Newer versions try to provide reasonably fresh copies of
this list. It also includes a convenience method to fetch the latest list. The PSL does
change regularly.

The code is a fork of the publicsuffix package and includes the same base API. In
addition, it contains a few variants useful for certain use cases, such as the option to
ignore wildcards or return only the extended TLD (eTLD). You just need to import publicsuffix2 instead.

The public suffix list is now provided in UTF-8 format. To correctly process
IDNA-encoded domains, either the query or the list must be converted. By default, the
module converts the PSL. If your use case includes UTF-8 domains, e.g., '食狮.com.cn',
you'll need to set the IDNA-encoding flag to False on instantiation (see examples below).
Failure to use the correct encoding for your use case can lead to incorrect results for
domains that utilize unicode characters.

The code is MIT-licensed and the publicsuffix data list is MPL-2.0-licensed.

.. image:: https://api.travis-ci.org/nexB/python-publicsuffix2.png?branch=master
   :target: https://travis-ci.org/nexB/python-publicsuffix2
   :alt: master branch tests status

.. image:: https://api.travis-ci.org/nexB/python-publicsuffix2.png?branch=develop
   :target: https://travis-ci.org/nexB/python-publicsuffix2
   :alt: develop branch tests status

Usage
-----

Install with::

    pip install publicsuffix2

The module provides functions to obtain the base domain, or sld, of an fqdn, as well as one
to get just the public suffix. In addition, the functions a number of boolean parameters that
control how wildcards are handled. In addition to the functions, the module exposes a class that
parses the PSL, and allows for more control.

The module provides two equivalent functions to query a domain name, and return the base domain,
or second-level-doamin; get_public_suffix() and get_sld()::

    >>> from publicsuffix2 import get_public_suffix
    >>> get_public_suffix('www.example.com')
    'example.com'
    >>> get_sld('www.example.com')
    'example.com'
    >>> get_public_suffix('www.example.co.uk')
    'example.co.uk'
    >>> get_public_suffix('www.super.example.co.uk')
    'example.co.uk'
    >>> get_sld("co.uk")  # returns eTLD as is
    'co.uk'

This function loads and caches the public suffix list. To obtain the latest version of the
PSL, use the fetch() function to first download the latest version. Alternatively, you can pass
a custom list.

For more control, there is also a class that parses a Public
Suffix List and allows the same queries on individual domain names::

    >>> from publicsuffix2 import PublicSuffixList
    >>> psl = PublicSuffixList()
    >>> psl.get_public_suffix('www.example.com')
    'example.com'
    >>> psl.get_public_suffix('www.example.co.uk')
    'example.co.uk'
    >>> psl.get_public_suffix('www.super.example.co.uk')
    'example.co.uk'
    >>> psl.get_sld('www.super.example.co.uk')
    'example.co.uk'

Note that the ``host`` part of an URL can contain strings that are
not plain DNS domain names (IP addresses, Punycode-encoded names, name in
combination with a port number or a username, etc.). It is up to the
caller to ensure only domain names are passed to the get_public_suffix()
method.

The get_public_suffix() function and the PublicSuffixList class initializer accept
an optional argument pointing to a public suffix file. This can either be a file
path, an iterable of public suffix lines, or a file-like object pointing to an
opened list::

    >>> from publicsuffix2 import get_public_suffix
    >>> psl_file = 'path to some psl data file'
    >>> get_public_suffix('www.example.com', psl_file)
    'example.com'

Note that when using get_public_suffix() a global cache keeps the latest provided
suffix list data.  This will use the cached latest loaded above::

    >>> get_public_suffix('www.example.co.uk')
    'example.co.uk'

**IDNA-encoding.** The public suffix list is now in UTF-8 format. For those use cases that
include IDNA-encoded domains, the list must be converted. Publicsuffix2 includes idna
encoding as a parameter of the PublicSuffixList initialization and is true by
default. For UTF-8 use cases, set the idna parameter to False::

    >>> from publicsuffix2 import PublicSuffixList
    >>> psl = PublicSuffixList(idna=True)  # on by default
    >>> psl.get_public_suffix('www.google.com')
    'google.com'
    >>> psl = PublicSuffixList(idna=False)  # use UTF-8 encodings
    >>> psl.get_public_suffix('食狮.com.cn')
    '食狮.com.cn'

**Ignore wildcards.** In some use cases, particularly those related to large-scale domain processing,
the user might want to ignore wildcards to create more aggregation. This is possible by setting
the parameter wildcard=False.::

    >>> psl.get_public_suffix('telinet.com.pg', wildcard=False)
    'com.pg'
    >>> psl.get_public_suffix('telinet.com.pg', wildcard=True)
    'telinet.com.pg'

**Require valid eTLDs (strict).** In the publicsuffix2 module, a domain with an invalid TLD will still return
return a base domain, e.g,::

    >>> psl.get_public_suffix('www.mine.local')
    'mine.local'

This is useful for many use cases, while in others, we want to ensure that the domain includes a
valid eTLD. In this case, the boolean parameter strict provides a solution. If this flag is set,
an invalid TLD will return None.::

    >>> psl.get_public_suffix('www.mine.local', strict=True) is None
    True

Keep in mind that 'valid' is determined by the list that the PSL object is created from, and is not necessarily the
Mozilla list. You can create a version with your own lists, for example, using only the ICANN public suffixes, or
ICANN TLDS, or some other custom list.  In all cases, strict=True will verify that the right most label of the lookup
item is contained in the list.

**Return eTLD only.** The standard use case for publicsuffix2 is to return the registrable,
or base, domain
according to the public suffix list. In some cases, however, we only wish to find the eTLD
itself. This is available via the get_tld() method.::

    >>> psl.get_tld('www.google.com')
    'com'
    >>> psl.get_tld('www.google.co.uk')
    'co.uk'

All of the methods and functions include the wildcard and strict parameters.

For convenience, the public method get_sld() is available. This is identical to the method
get_public_suffix() and is intended to clarify the output for some users.

**Accelerated Functions**

If your data is already normalized to lowercase with trailing dots, e.g., 'com.' or '.com' removed, you can use
functions that will perform much faster by avoiding these element-wise operations on the input data.::

    psl.get_sld_unsafe('www.google.com')
    'google.com'
    psl.get_tld_unsafe('www.google.co.uk')
    'co.uk'

**Edge Case Processing**

Domain names occur in many sources and varieties, including use cases from urls, dns data, and emails. All of these,
particularly at large volume, may have improperly formatted data and noise. In order to provide backward compatibility
across these cases, the user is left to deal with some edge cases in their own data. The library chooses to lowercase
and remove trailing and leading dots from domains. This means that inputs that are technically invalid DNS domains
will generally return a value. For example, empty labels are invalid in DNS and thus '..google.com' is not a valid
domain, however, this library will return a value::

    psl.get_sld('..google.com')
    'google.com'

Users who wish to remove invalid DNS labels will need to clean their data prior to using the library functions.
However, domains that contain empty labels in the middle will return None.::

    psl.get_sld('google..com') ==  None
    True

To **update the bundled suffix list** use the provided setup.py command::

    python setup.py update_psl

The update list will be saved in `src/publicsuffix2/public_suffix_list.dat`
and you can build a new wheel with this bundled data.

Alternatively, there is a fetch() function that will fetch the latest version
of a Public Suffix data file from https://publicsuffix.org/list/public_suffix_list.dat
You can use it this way::

    >>> from publicsuffix2 import get_public_suffix
    >>> from publicsuffix2 import fetch
    >>> psl_file = fetch()
    >>> get_public_suffix('www.example.com', psl_file)
    'example.com'

Note that the once loaded, the data file is cached and therefore fetched only
once.

The extracted public suffix list, that is the tlds and their modifiers, is put into
an instance variable, tlds, which can be accessed as an attribute, tlds.::

    >>> psl = PublicSuffixList()
    >>> psl.tlds[:5]
    ['ac',
    'com.ac',
    'edu.ac',
    'gov.ac',
    'net.ac']

**Using the module in large-scale processing**
If using this library in large-scale pyspark processing, you should instantiate the class as
a global variable, not within a user function. The class methods can then be used within user
functions for distributed processing.

Source
------

Get a local copy of the development repository. The development takes
place in the ``develop`` branch. Stable releases are tagged in the ``master``
branch::

    git clone https://github.com/nexB/python-publicsuffix2.git


History
-------
This code is forked from Tomaž Šolc's fork of David Wilson's code.

Tomaž Šolc's code originally at:

https://www.tablix.org/~avian/git/publicsuffix.git

Copyright (c) 2014 Tomaž Šolc <tomaz.solc@tablix.org>

David Wilson's code was originally at:

http://code.google.com/p/python-public-suffix-list/

Copyright (c) 2009 David Wilson


License
-------

The code is MIT-licensed.
The vendored public suffix list data from Mozilla is under the MPL-2.0.

Copyright (c) 2015 nexB Inc. and others.

Copyright (c) 2014 Tomaž Šolc <tomaz.solc@tablix.org>

Copyright (c) 2009 David Wilson

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the "Software"),
to deal in the Software without restriction, including without limitation
the rights to use, copy, modify, merge, publish, distribute, sublicense,
and/or sell copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
DEALINGS IN THE SOFTWARE.
