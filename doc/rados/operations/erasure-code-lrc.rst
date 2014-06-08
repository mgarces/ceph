======================================
Locally repairable erasure code plugin
======================================

The *LRC* erasure code plugin recursively applies erasure code
techniques so that recovering from the loss of some chunks only
require a subset of the available chunks, most of the time.

For instance, when three coding steps are described as::

   chunk nr    01234567
   step 1      _cDD_cDD
   step 2      cDDD____
   step 3      ____cDDD

where *c* are coding chunks calculated from the data chunks *D*, the
loss of chunk *7* can be recovered with the last four chunks. And the
loss of chun *2* chunk can be recovered with the first four
chunks. This reduces the network bandwidth required to recover from
the loss of one chunk (which is the most frequent case) by 50%.

Erasure code profile examples
=============================

Minimal testing
---------------

It is strictly equivalent to using the default profile. The *DD*
implies *K=2*, the *c* implies *M=1* and the *jerasure* plugin is used
by default.::

	$ ceph osd erasure-code-profile set LRCprofile \
             plugin=LRC \
             mapping=DD_ \
             layers='[ [ "DDc", "" ] ]'
        $ ceph osd pool create lrcpool 12 12 erasure LRCprofile

Reduce recovery bandwidth between hosts
---------------------------------------

Although it is probably not an interesting use case when all hosts are
connected to the same switch, reduced bandwidth usage can actually be
observed.::

        $ ceph osd erasure-code-profile set LRCprofile \
             plugin=LRC \
             mapping=__DD__DD \
             layers='[
                       [ "_cDD_cDD", "" ],
                       [ "cDDD____", "" ],
                       [ "____cDDD", "" ],
                     ]'
        $ ceph osd pool create lrcpool 12 12 erasure LRCprofile


Reduce recovery bandwidth between racks
---------------------------------------

In Firefly the reduced bandwidth will only be observed if the primary
OSD is in the same rack as the lost chunk.::

        $ ceph osd erasure-code-profile set LRCprofile \
             plugin=LRC \
             mapping=__DD__DD \
             layers='[
                       [ "_cDD_cDD", "" ],
                       [ "cDDD____", "" ],
                       [ "____cDDD", "" ],
                     ]' \
             ruleset-steps='[
                             [ "choose", "rack", 2 ],
                             [ "chooseleaf", "host", 4 ],
                            ]'
        $ ceph osd pool create lrcpool 12 12 erasure LRCprofile


Recursive erasure coding and decoding
=====================================

The steps found in the layers description::

   chunk nr    01234567

   step 1      _cDD_cDD
   step 2      cDDD____
   step 3      ____cDDD

are applied in order. For instance, if a 4K object is encoded, it will
first go thru *step 1* and be divided in four 1K chunks (the four
uppercase D). They are stored in the chunks 2, 3, 6 and 7, in
order. From these, two coding chunks are calculated (the two lowercase
c). The coding chunks are stored in the chunks 1 and 4, respectively.

The *step 2* re-uses the content created by *step 1* in a similar
fashion and stores a single coding chunk *c* at position 0. The last four
chunks, marked with an underscore (*_*) for readability, are ignored.

The *step 3* stores a single coding chunk *c* at position 4. The three
chunks created by *step 1* are used to compute this coding chunk,
i.e. the coding chunk from *step 1* becomes a data chunk in *step 3*.

If chunk *2* is lost::

   chunk nr    01234567

   step 1      _c D_cDD
   step 2      cD D____
   step 3      __ _cDDD

decoding will attempt to recover it by walking the steps in reverse
order: *step 3* then *step 2* and finally *step 1*.

The *step 3* knows nothing about chunk *2* (i.e. it is an underscore)
and is skipped.

The coding chunk from *step 2*, stored in chunk *0*, allows it to
recover the content of chunk *2*. There are no more chunks to recover
and the process stops, without considering *step 1*.

Recovering chunk *2* required reading chunks *0, 1, 3* and writing
back chunk *2*.

If chunk *2, 3, 6* are lost::

   chunk nr    01234567

   step 1      _c  _c D
   step 2      cD  __ _
   step 3      __  cD D

The *step 3* can recover the conten of chunk *6*::

   chunk nr    01234567

   step 1      _c  _cDD
   step 2      cD  ____
   step 3      __  cDDD

The *step 2* fails to recover and is skipped because there are two
chunks missing (*2, 3*) and it can only recover from one missing
chunk.

The coding chunk from *step 1*, stored in chunk *1, 5*, allows it to
recover the content of chunk *2, 3*::

   chunk nr    01234567

   step 1      _cDD_cDD
   step 2      cDDD____
   step 3      ____cDDD

Controlling crush placement
===========================

The default crush ruleset provides OSDs that are on different hosts. For instance::

   chunk nr    01234567

   step 1      _cDD_cDD
   step 2      cDDD____
   step 3      ____cDDD

needs exactly *8* OSDs, one for each chunk. If the hosts are in two
adjacent racks, the first four chunks can be placed in the first rack
and the last four in the second rack. So that recovering from the loss
of a single OSD does not require using bandwidth between the two
racks.

For instance::

   ruleset-steps='[ [ "choose", "rack", 2 ], [ "chooseleaf", "host", 4 ] ]'

will create a ruleset that will select two crush buckets of type
*rack* and for each of them choose four OSDs, each of them located in
different bucket of type *host*.

The ruleset can also be manually crafted for finer control.

Create an LRC profile
=====================

To create a new LRC erasure code profile::

	ceph osd erasure-code-profile set {name} \
             plugin=LRC \
             mapping={mapping} \
             layers={description} \
             [ruleset-root={root}] \
             [ruleset-steps={steps}] \
             [directory={directory}] \
             [--force]

Where:

``mapping={mapping}``

:Description:

:Type: Integer
:Required: No.
:Default: 2

(TBD)
