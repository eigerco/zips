::

  ZIP: 230
  Title: Version 6 Transaction Format
  Owners: Daira Emma Hopwood <daira@electriccoin.co>
          Jack Grigg <jack@electriccoin.co>
          Sean Bowe <sean@electriccoin.co>
          Kris Nuttycombe <kris@electriccoin.co>
          Greg Pfeil <greg@electriccoin.co>
          Deirdre Connolly <deirdre@zfnd.org>
  Credits: Ying Tong Lai
  Status: DRAFT
  Category: Consensus
  Created: 2023-04-18
  License: MIT
  Discussions-To: <https://github.com/zcash/zips/issues/686>


Terminology
===========

The key words "MUST" and "MAY" in this document are to be interpreted as described in
RFC 2119. [#RFC2119]_

The character § is used when referring to sections of the Zcash Protocol Specification
[#protocol]_.


Abstract
========

This proposal defines a new Zcash peer-to-peer transaction format, which includes data that supports Zcash Shielded Assets on the Orchard shielded pool [#protocol]_.  The
new transaction format defines well-bounded regions of the serialized form to
serve each of the existing pools of funds, and adds and describes a new elements
containing ZSA-specific elements.

This ZIP also depends upon and defines modifications to the computation of the values
**TxId Digest**, **Signature Digest**, and **Authorizing Data Commitment** defined by ZIP
244 [#zip-0244]_.


Motivation
==========

The new Orchard shielded pool requires serialized data elements that are distinct from
any previous Zcash transaction. Since ZIP 244 was activated in NU5, the
v5 and later serialized transaction formats are not consensus-critical. 
Thus, this ZIP defines format that can easily accommodate future extensions,
where elements or a given pool are kept separate.


Requirements
============

The new format must fully support the Orchard protocol.

The new format must fully support Zcash Shielded Assets (ZSAs).

The new format should lend itself to future extension or pruning to add or remove
value pools.

The computation of the non-malleable transaction identifier hash must include all
newly incorporated elements except those that attest to transaction validity.

The computation of the commitment to authorizing data for a transaction must include
all newly incorporated elements that attest to transaction validity.


Non-requirements
================

More general forms of extensibility, such as definining a key/value format that
allows for parsers that are unaware of some components, are not required.


Specification
=============

All fields in this specification are encoded as little-endian.

The Zcash transaction format for transaction version 6 is as follows:

Transaction Format
------------------

+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
| Bytes                       | Name                     | Data Type                              | Description                                                         |
+=============================+==========================+========================================+=====================================================================+
| **Common Transaction Fields**                                                                                                                                         |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``4``                        |``header``                |``uint32``                              |Contains:                                                            |
|                             |                          |                                        |  * ``fOverwintered`` flag (bit 31, always set)                      |
|                             |                          |                                        |  * ``version`` (bits 30 .. 0) – transaction version.                |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``4``                        |``nVersionGroupId``       |``uint32``                              |Version group ID (nonzero).                                          |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``4``                        |``nConsensusBranchId``    |``uint32``                              |Consensus branch ID (nonzero).                                       |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``4``                        |``lock_time``             |``uint32``                              |Unix-epoch UTC time or block height, encoded as in Bitcoin.          |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``4``                        |``nExpiryHeight``         |``uint32``                              |A block height in the range {1 .. 499999999} after which             |
|                             |                          |                                        |the transaction will expire, or 0 to disable expiry.                 |
|                             |                          |                                        |[ZIP-203]                                                            |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``8``                        |``fee``                   |``int64``                               |The fee to be paid by this transaction, in zatoshis.                 |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``8``                        |``zsf_deposit``           |``int64``                               |Funds deposited by this transaction to the Zcash Sustainability Fund.|
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
| **Transparent Transaction Fields**                                                                                                                                    |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``tx_in_count``           |``compactSize``                         |Number of transparent inputs in ``tx_in``.                           |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``tx_in``                 |``tx_in``                               |Transparent inputs, encoded as in Bitcoin.                           |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``tx_out_count``          |``compactSize``                         |Number of transparent outputs in ``tx_out``.                         |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``tx_out``                |``tx_out``                              |Transparent outputs, encoded as in Bitcoin.                          |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
| **Sapling Transaction Fields**                                                                                                                                        |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``nSpendsSapling``        |``compactSize``                         |Number of Sapling Spend descriptions in ``vSpendsSapling``.          |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``96 * nSpendsSapling``      |``vSpendsSapling``        |``SpendDescriptionV6[nSpendsSapling]``  |A sequence of Sapling Spend descriptions, encoded per                |
|                             |                          |                                        |protocol §7.3 ‘Spend Description Encoding and Consensus’.            |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``nOutputsSapling``       |``compactSize``                         |Number of Sapling Output Decriptions in ``vOutputsSapling``.         |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``756 * nOutputsSapling``    |``vOutputsSapling``       |``OutputDescriptionV6[nOutputsSapling]``|A sequence of Sapling Output descriptions, encoded per               |
|                             |                          |                                        |protocol §7.4 ‘Output Description Encoding and Consensus’.           |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``8``                        |``valueBalanceSapling``   |``int64``                               |The net value of Sapling Spends minus Outputs                        |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``32``                       |``anchorSapling``         |``byte[32]``                            |A root of the Sapling note commitment tree                           |
|                             |                          |                                        |at some block height in the past.                                    |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``192 * nSpendsSapling``     |``vSpendProofsSapling``   |``byte[192 * nSpendsSapling]``          |Encodings of the zk-SNARK proofs for each Sapling Spend.             |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``64 * nSpendsSapling``      |``vSpendAuthSigsSapling`` |``byte[64 * nSpendsSapling]``           |Authorizing signatures for each Sapling Spend.                       |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``192 * nOutputsSapling``    |``vOutputProofsSapling``  |``byte[192 * nOutputsSapling]``         |Encodings of the zk-SNARK proofs for each Sapling Output.            |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``64``                       |``bindingSigSapling``     |``byte[64]``                            |A Sapling binding signature on the SIGHASH transaction hash.         |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
| **Orchard Transaction Fields**                                                                                                                                        |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``nActionsOrchard``       |``compactSize``                         |The number of Orchard Action descriptions in                         |
|                             |                          |                                        |``vActionsOrchard``.                                                 |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``820 * nActionsOrchard``    |``vActionsOrchard``       |``OrchardAction[nActionsOrchard]``      |A sequence of Orchard Action descriptions, encoded per               |
|                             |                          |                                        |§7.5 ‘Action Description Encoding and Consensus’.                    |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``1``                        |``flagsOrchard``          |``byte``                                |An 8-bit value representing a set of flags. Ordered from LSB to MSB: |
|                             |                          |                                        | * ``enableSpendsOrchard``                                           |
|                             |                          |                                        | * ``enableOutputsOrchard``                                          |
|                             |                          |                                        | * The remaining bits are set to ``0``.                              |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``8``                        |``valueBalanceOrchard``   |``int64``                               |The net value of Orchard spends minus outputs.                       |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``32``                       |``anchorOrchard``         |``byte[32]``                            |A root of the Orchard note commitment tree at some block             |
|                             |                          |                                        |height in the past.                                                  |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``varies``                   |``sizeProofsOrchard``     |``compactSize``                         |Length in bytes of ``proofsOrchard``. Value is                       |
|                             |                          |                                        |:math:`2720 + 2272 \cdot \mathtt{nActionsOrchard}`.                  |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``sizeProofsOrchard``        |``proofsOrchard``         |``byte[sizeProofsOrchard]``             |Encoding of aggregated zk-SNARK proofs for Orchard Actions.          |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``64 * nActionsOrchard``     |``vSpendAuthSigsOrchard`` |``byte[64 * nActionsOrchard]``          |Authorizing signatures for each Orchard Action.                      |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+
|``64``                       |``bindingSigOrchard``     |``byte[64]``                            |An Orchard binding signature on the SIGHASH transaction hash.        |
+-----------------------------+--------------------------+----------------------------------------+---------------------------------------------------------------------+

* The fields ``valueBalanceSapling`` and ``bindingSigSapling`` are present if and only if
  :math:`\mathtt{nSpendsSapling} + \mathtt{nOutputsSapling} > 0`. If ``valueBalanceSapling``
  is not present, then :math:`\mathsf{v^{balanceSapling}}`` is defined to be 0.

* The field ``anchorSapling`` is present if and only if :math:`\mathtt{nSpendsSapling} > 0`.

* The fields ``flagsOrchard``, ``valueBalanceOrchard``, ``anchorOrchard``,
  ``sizeProofsOrchard``, ``proofsOrchard``, and ``bindingSigOrchard`` are present if and
  only if :math:`\mathtt{nActionsOrchard} > 0`. If ``valueBalanceOrchard`` is not present,
  then :math:`\mathsf{v^{balanceOrchard}}` is defined to be 0.

* The elements of ``vSpendProofsSapling`` and ``vSpendAuthSigsSapling`` have a 1:1
  correspondence to the elements of ``vSpendsSapling`` and MUST be ordered such that the
  proof or signature at a given index corresponds to the ``SpendDescriptionV6`` at the
  same index.

* The elements of ``vOutputProofsSapling`` have a 1:1 correspondence to the elements of
  ``vOutputsSapling`` and MUST be ordered such that the proof at a given index corresponds
  to the ``OutputDescriptionV6`` at the same index.

* The proofs aggregated in ``proofsOrchard``, and the elements of
  ``vSpendAuthSigsOrchard``, each have a 1:1 correspondence to the elements of
  ``vActionsOrchard`` and MUST be ordered such that the proof or signature at a given
  index corresponds to the ``OrchardAction`` at the same index.

* For coinbase transactions, the ``enableSpendsOrchard`` bit MUST be set to ``0``.

The encodings of ``tx_in``, and ``tx_out`` are as in a version 4 transaction (i.e.
unchanged from Canopy). The encodings of ``SpendDescriptionV6``, ``OutputDescriptionV6``
and ``OrchardAction`` are described below. The encoding of Sapling Spends and Outputs has
changed relative to prior versions in order to better separate data that describe the
effects of the transaction from the proofs of and commitments to those effects, and for
symmetry with this separation in the Orchard-related parts of the transaction format.

Sapling Spend Description (``SpendDescriptionV6``)
--------------------------------------------------

+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
| Bytes                       | Name                     | Data Type                            | Description                                                |
+=============================+==========================+======================================+============================================================+
|``32``                       |``cv``                    |``byte[32]``                          |A value commitment to the net value of the input note.      |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``nullifier``             |``byte[32]``                          |The nullifier of the input note.                            |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``rk``                    |``byte[32]``                          |The randomized validating key for the element of            |
|                             |                          |                                      |spendAuthSigsSapling corresponding to this Spend.           |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+

The encodings of each of these elements are defined in §7.3 ‘Spend Description Encoding
and Consensus’ of the Zcash Protocol Specification [#protocol-spenddesc]_.

Sapling Output Description (``OutputDescriptionV6``)
----------------------------------------------------

+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
| Bytes                       | Name                     | Data Type                            | Description                                                |
+=============================+==========================+======================================+============================================================+
|``32``                       |``cv``                    |``byte[32]``                          |A value commitment to the net value of the output note.     |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``cmu``                   |``byte[32]``                          |The u-coordinate of the note commitment for the output note.|
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``ephemeralKey``          |``byte[32]``                          |An encoding of an ephemeral Jubjub public key.              |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``580``                      |``encCiphertext``         |``byte[580]``                         |The encrypted contents of the note plaintext.               |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``80``                       |``outCiphertext``         |``byte[80]``                          |The encrypted contents of the byte string created by        |
|                             |                          |                                      |concatenation of the transmission key with the ephemeral    |
|                             |                          |                                      |secret key.                                                 |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+

The encodings of each of these elements are defined in §7.4 ‘Output Description Encoding
and Consensus’ of the Zcash Protocol Specification [#protocol-outputdesc]_.

Orchard Action Description (``OrchardAction``)
----------------------------------------------

+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
| Bytes                       | Name                     | Data Type                            | Description                                                |
+=============================+==========================+======================================+============================================================+
|``32``                       |``cv``                    |``byte[32]``                          |A value commitment to the net value of the input note minus |
|                             |                          |                                      |the output note.                                            |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``nullifier``             |``byte[32]``                          |The nullifier of the input note.                            |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``rk``                    |``byte[32]``                          |The randomized validating key for the element of            |
|                             |                          |                                      |spendAuthSigsOrchard corresponding to this Action.          |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``cmx``                   |``byte[32]``                          |The x-coordinate of the note commitment for the output note.|
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``32``                       |``ephemeralKey``          |``byte[32]``                          |An encoding of an ephemeral Pallas public key               |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``580``                      |``encCiphertext``         |``byte[580]``                         |The encrypted contents of the note plaintext.               |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+
|``80``                       |``outCiphertext``         |``byte[80]``                          |The encrypted contents of the byte string created by        |
|                             |                          |                                      |concatenation of the transmission key with the ephemeral    |
|                             |                          |                                      |secret key.                                                 |
+-----------------------------+--------------------------+--------------------------------------+------------------------------------------------------------+

The encodings of each of these elements are defined in §7.5 ‘Action Description Encoding
and Consensus’ of the Zcash Protocol Specification [#protocol-actiondesc]_.


Reference implementation
========================

TODO


References
==========

.. [#RFC2119] `RFC 2119: Key words for use in RFCs to Indicate Requirement Levels <https://www.rfc-editor.org/rfc/rfc2119.html>`_
.. [#protocol] `Zcash Protocol Specification, Version 2021.2.16 or later [NU5 proposal] <protocol/protocol.pdf>`_
.. [#protocol-spenddesc] `Zcash Protocol Specification, Version 2021.2.16 [NU5 proposal]. Section 4.4: Spend Descriptions <protocol/protocol.pdf#spenddesc>`_
.. [#protocol-outputdesc] `Zcash Protocol Specification, Version 2021.2.16 [NU5 proposal]. Section 4.5: Output Descriptions <protocol/protocol.pdf#outputdesc>`_
.. [#protocol-actiondesc] `Zcash Protocol Specification, Version 2021.2.16 [NU5 proposal]. Section 4.6: Action Descriptions <protocol/protocol.pdf#actiondesc>`_
.. [#zip-0222] `ZIP 222: Transparent Zcash Extensions <zip-0222.rst>`_
.. [#zip-0244] `ZIP 244: Transaction Identifier Non-Malleability <zip-0244.rst>`_
.. [#zip-0307] `ZIP 307: Light Client Protocol for Payment Detection <zip-0307.rst>`_
