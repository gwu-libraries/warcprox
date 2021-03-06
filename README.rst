Warcprox - WARC writing MITM HTTP/S proxy
*****************************************
.. image:: https://travis-ci.org/internetarchive/warcprox.svg?branch=master
    :target: https://travis-ci.org/internetarchive/warcprox

Warcprox is a tool for archiving the web. It is an http proxy that stores its
traffic to disk in `WARC
<https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/>`_
format. Warcprox captures encrypted https traffic by using the
`"man-in-the-middle" <https://en.wikipedia.org/wiki/Man-in-the-middle_attack>`_
technique (see the `Man-in-the-middle`_ section for more info).

The web pages that warcprox stores in WARC files can be played back using
software like `OpenWayback <https://github.com/iipc/openwayback>`_ or `pywb
<https://github.com/webrecorder/pywb>`_. Warcprox has been developed in
parallel with `brozzler <https://github.com/internetarchive/brozzler>`_ and
together they make a comprehensive modern distributed archival web crawling
system.

Warcprox was originally based on the excellent and simple pymiproxy by Nadeem
Douba. https://github.com/allfro/pymiproxy

.. contents::

Getting started
===============
Warcprox runs on python 3.4+.

To install latest release run::

    # apt-get install libffi-dev libssl-dev
    pip install warcprox

You can also install the latest bleeding edge code::

    pip install git+https://github.com/internetarchive/warcprox.git

To start warcprox run::

    warcprox

Try ``warcprox --help`` for documentation on command line options.

Man-in-the-middle
=================
Normally, http proxies can't read https traffic, because it's encrypted. The
browser uses the http ``CONNECT`` method to establish a tunnel through the
proxy, and the proxy merely routes raw bytes between the client and server.
Since the bytes are encrypted, the proxy can't make sense of the information
it's proxying. This nonsensical encrypted data would not be very useful to
archive.

In order to capture https traffic, warcprox acts as a "man-in-the-middle"
(MITM). When it receives a ``CONNECT`` directive from a client, it generates a
public key certificate for the requested site, presents to the client, and
proceeds to establish an encrypted connection with the client. Then it makes a
separate, normal https connection to the remote site. It decrypts, archives,
and re-encrypts traffic in both directions.

Although "man-in-the-middle" is often paired with "attack", there is nothing
malicious about what warcprox is doing. If you configure an instance of
warcprox as your browser's http proxy, you will see lots of certificate
warnings, since none of the certificates will be signed by trusted authorities.
To use warcprox effectively the client needs to disable certificate
verification, or add the CA cert generated by warcprox as a trusted authority.
(If you do this in your browser, make sure you undo it when you're done using
warcprox!)

API
===
For interacting with a running instance of warcprox.

* ``/status`` url
* ``WARCPROX_WRITE_RECORD`` http method
* ``Warcprox-Meta`` http request header and response header

See `<api.rst>`_.

Deduplication
=============
Warcprox avoids archiving redundant content by "deduplicating" it. The process
for deduplication works similarly to heritrix and other web archiving tools.

1. while fetching url, calculate payload content digest (typically sha1)
2. look up digest in deduplication database (warcprox supports a few different
   ones)
3. if found, write warc ``revisit`` record referencing the url and capture time
   of the previous capture
4. else (if not found),

   a. write warc ``response`` record with full payload
   b. store entry in deduplication database

The dedup database is partitioned into different "buckets". Urls are
deduplicated only against other captures in the same bucket. If specified, the
``dedup-bucket`` field of the ``Warcprox-Meta`` http request header determines
the bucket, otherwise the default bucket is used.

Deduplication can be disabled entirely by starting warcprox with the argument
``--dedup-db-file=/dev/null``.

Statistics
==========
Warcprox keeps some crawl statistics and stores them in sqlite or rethinkdb.
These are consulted for enforcing ``limits`` and ``soft-limits`` (see
`<api.rst#warcprox-meta-fields>`_), and can also be consulted by other
processes outside of warcprox, for reporting etc.

Statistics are grouped by "bucket". Every capture is counted as part of the
``__all__`` bucket. Other buckets can be specified in the ``Warcprox-Meta``
request header. The fallback bucket in case none is specified is called
``__unspecified__``.

Within each bucket are three sub-buckets:

* ``new`` - tallies captures for which a complete record (usually a ``response``
  record) was written to warc
* ``revisit`` - tallies captures for which a ``revisit`` record was written to
  warc
* ``total`` - includes all urls processed, even those not written to warc (so the
  numbers may be greater than new + revisit)

Within each of these sub-buckets we keep two statistics:

* ``urls`` - simple count of urls
* ``wire_bytes`` - sum of bytes received over the wire, including http headers,
  from the remote server for each url

For historical reasons, in sqlite, the default store, statistics are kept as
json blobs::

    sqlite> select * from buckets_of_stats;
    bucket           stats
    ---------------  ---------------------------------------------------------------------------------------------
    __unspecified__  {"bucket":"__unspecified__","total":{"urls":37,"wire_bytes":1502781},"new":{"urls":15,"wire_bytes":1179906},"revisit":{"urls":22,"wire_bytes":322875}}
    __all__          {"bucket":"__all__","total":{"urls":37,"wire_bytes":1502781},"new":{"urls":15,"wire_bytes":1179906},"revisit":{"urls":22,"wire_bytes":322875}}

Plugins
=======
Warcprox supports a limited notion of plugins by way of the ``--plugin``
command line argument. Plugin classes are loaded from the regular python module
search path. They will be instantiated with one argument, a
``warcprox.Options``, which holds the values of all the command line arguments.
Legacy plugins with constructors that take no arguments are also supported.
Plugins should either have a method ``notify(self, recorded_url, records)`` or
should subclass ``warcprox.BasePostfetchProcessor``. More than one plugin can
be configured by specifying ``--plugin`` multiples times.

`A minimal example <https://github.com/internetarchive/warcprox/blob/318405e795ac0ab8760988a1a482cf0a17697148/warcprox/__init__.py#L165>`__

License
=======

Warcprox is a derivative work of pymiproxy, which is GPL. Thus warcprox is also
GPL.

* Copyright (C) 2012 Cygnos Corporation
* Copyright (C) 2013-2018 Internet Archive

This program is free software; you can redistribute it and/or
modify it under the terms of the GNU General Public License
as published by the Free Software Foundation; either version 2
of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program; if not, write to the Free Software
Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.

