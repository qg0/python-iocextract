iocextract
==========

.. image:: https://travis-ci.org/InQuest/python-iocextract.svg?branch=master
    :target: https://travis-ci.org/InQuest/python-iocextract
    :alt: Build Status
.. image:: https://readthedocs.org/projects/iocextract/badge/?version=latest
    :target: http://iocextract.readthedocs.io/en/latest/?badge=latest
    :alt: Documentation Status
.. image:: https://api.codacy.com/project/badge/Grade/920894593bde451c9277c56b7d9ab3e1
    :target: https://app.codacy.com/app/InQuest/python-iocextract
    :alt: Code Health
.. image:: https://api.codacy.com/project/badge/Coverage/920894593bde451c9277c56b7d9ab3e1
    :target: https://app.codacy.com/app/InQuest/python-iocextract
    :alt: Test Coverage
.. image:: http://img.shields.io/pypi/v/iocextract.svg
    :target: https://pypi.python.org/pypi/iocextract
    :alt: PyPi Version

Advanced Indicator of Compromise (IOC) extractor.

Overview
--------

This library extracts URLs, IP addresses, MD5/SHA hashes, and YARA rules from
text corpora. It includes obfuscated and "defanged" IOCs in the output, and
optionally deobfuscates them.

The Problem
-----------

It is common practice for malware analysts or endpoint software to "defang" IOCs
such as URLs and IP addresses, in order to prevent accidental exposure to live
malicious content. Being able to extract and aggregate these IOCs is often valuable
for analysts. Unfortunately, existing "IOC extraction" tools often pass right by them,
as they are not caught by standard REGEX.

For example, the simple defanging technique of surrounding periods with brackets::

    127[.]0[.]0[.]1

Existing tools that use a simple IP address REGEX will ignore this IOC entirely.

The Solution
------------

By combining specially crafted REGEX with some custom postprocessing, we are
able to both detect and deobfuscate "defanged" IOCs. This saves time and effort
for the analyst, who might otherwise have to manually find and convert IOCs into
machine-readable format.

A Simple Use Case
-----------------

Many Twitter users post C2s or other valuable IOC information with defanged URLs.
For example, `this tweet from @InQuest`_::

    Recommended reading and great work from @unit42_intel:
    https://researchcenter.paloaltonetworks.com/2018/02/unit42-sofacy-attacks-multiple-government-entities/ ...
    InQuest customers have had detection for threats delivered from hotfixmsupload[.]com
    since 6/3/2017 and cdnverify[.]net since 2/1/18.

If we run this through the extractor, we can easily pull out the URLs::

   https://researchcenter.paloaltonetworks.com/2018/02/unit42-sofacy-attacks-multiple-government-entities/
   hotfixmsupload[.]com
   cdnverify[.]net

Passing in ``refang=True`` at extraction time would remove the obfuscation, but
since these are real IOCs, let's leave them defanged in our documentation. :)

Installation
------------

Just get it from pip::

    pip install iocextract

Usage
-----

Try extracting some defanged URLS::

    >>> content = """
    ... I really love example[.]com!
    ... All the bots are on hxxp://example.com/bad/url these days.
    ... C2: tcp://example[.]com:8989/bad
    ... """
    >>> import iocextract
    >>> for url in iocextract.extract_urls(content):
    ...     print url
    ...
    hxxp://example.com/bad/url
    tcp://example[.]com:8989/bad
    example[.]com
    tcp://example[.]com:8989/bad

Note that some URLs may show up twice if they are caught by multiple REGEXes.

If you want, you can also "refang", or remove common obfuscation methods from
IOCs::

    >>> for url in iocextract.extract_urls(content, refang=True):
    ...     print url
    ...
    http://example.com/bad/url
    http://example.com:8989/bad
    http://example.com
    http://example.com:8989/bad

You can even extract and decode hex-encoded URLs::

    >>> content = '612062756e6368206f6620776f72647320687474703a2f2f6578616d706c652e636f6d2f70617468206d6f726520776f726473'
    >>> for url in iocextract.extract_urls(content):
    ...     print url
    ...
    687474703a2f2f6578616d706c652e636f6d2f70617468
    >>> for url in iocextract.extract_urls(content, refang=True):
    ...     print url
    ...
    http://example.com/path

All ``extract_*`` functions in this library return iterators, not lists. The
benefit of this behavior is that ``iocextract`` can process extremely large
inputs, with a very low overhead. However, if for some reason you need to iterate
over the IOCs more than once, you will have to save the results as a list::

    >>> list(iocextract.extract_urls(content))
    ['hxxp://example.com/bad/url', 'tcp://example[.]com:8989/bad', 'example[.]com', 'tcp://example[.]com:8989/bad']

A command-line tool is also included::

    $ iocextract -h
    usage: iocextract [-h] [--input INPUT] [--output OUTPUT] [--extract-ips]
                      [--extract-urls] [--extract-yara-rules] [--extract-hashes]
                      [--refang]

    Advanced Indicator of Compromise (IOC) extractor. If no arguments are
    specified, the default behavior is to extract all IOCs.

    optional arguments:
      -h, --help            show this help message and exit
      --input INPUT         default: stdin
      --output OUTPUT       default: stdout
      --extract-ips
      --extract-urls
      --extract-yara-rules
      --extract-hashes
      --refang              default: no

Only URLs and IPv4 addresses can be "refanged".

More Details
------------

This library currently supports the following IOCs:

* IP Addresses
    * IPv4 fully supported
    * IPv6 partially supported
* URLs
    * With protocol specifier: http, https, tcp, udp, ftp, sftp, ftps
    * With ``[.]`` anchor, even with no protocol specifier
    * IPv4 and IPv6 (RFC2732) URLs are supported
    * Hex-encoded URLs with protocol specifier: http, https, ftp
* Emails
    * Partially supported, anchoring on ``@``
* YARA rules
* Hashes
    * MD5
    * SHA1
    * SHA256
    * SHA512

For IPv4 addresses, the following defang techniques are supported:

+-----------------+---------------+-----------+
| Technique       | Defanged      | Refanged  |
+=================+===============+===========+
| ``. -> [.]``    | 1[.]1[.]1[.]1 | 1.1.1.1   |
+-----------------+---------------+-----------+
| ``. -> (.)``    | 1(.)1(.)1(.)1 | 1.1.1.1   |
+-----------------+---------------+-----------+
| Partial         | 1[.1[.1.]1    | 1.1.1.1   |
+-----------------+---------------+-----------+
| Any combination | 1.)1[.1.)1    | 1.1.1.1   |
+-----------------+---------------+-----------+

For URLs, the following defang techniques are supported:

+-----------------+----------------------------------------------------+-----------------------------+
| Technique       | Defanged                                           | Refanged                    |
+=================+====================================================+=============================+
| ``. -> [.]``    | ``example[.]com/path``                             | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| ``. -> (.)``    | ``example(.)com/path``                             | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| Partial         | ``http://example[.com/path``                       | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| ``/ -> [/]``    | ``http://example.com[/]path``                      | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| `Cisco ESA`_    | ``http:// example .com /path``                     | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| ``:// -> __``   | ``http__example.com/path``                         | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| ``hxxp``        | ``hxxp://example.com/path``                        | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| Any combination | ``hxxp__ example( .com[/]path``                    | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+
| Hex encoded     | ``687474703a2f2f6578616d706c652e636f6d2f70617468`` | ``http://example.com/path`` |
+-----------------+----------------------------------------------------+-----------------------------+

Note that the table above is not exhaustive, and other URL/defang patterns may
also be extracted correctly. If you notice something missing or not working
correctly, feel free to let us know via the GitHub Issues_.

Contributing
------------

If you have a defang technique that doesn't make it through the extractor, or
if you find any bugs, PRs and Issues_ are always welcome. The library is
released under a "BSD-New" (aka "BSD 3-Clause") license.

.. _Issues: https://github.com/inquest/python-iocextract/issues
.. _this tweet from @InQuest: https://twitter.com/InQuest/status/969469856931287041
.. _Cisco ESA: https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118775-technote-esa-00.html
