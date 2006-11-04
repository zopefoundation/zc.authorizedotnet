Authorize.Net Integration
=========================

Authorize.Net provides credit card (henceforth "CC") processing via a protocol
on top of HTTPS.  Authorize.Net's customers are "merchants".  The merchant is
the entity accepting a CC as payment.  This package provides a simple
interface to Authorize.Net's "Advanced Integration Method" (AIM).

Several terms used in this document:

    - authorize: check validity of CC information and for sufficient balance
    - capture: the approval of transfer of funds from the CC holder to the
      merchant
    - settlement: the actual transfer of funds from the CC holder
      to the merchant
    - credit: issuing a refund from the merchant to the card holder
    - voiding: canceling a previous transaction

Settlement is performed in daily batches.  The cut-off time for which is
specified in the merchant's settings available on the Authorize.Net merchant
interface.

There are many other settings which can be configured via the merchant
interface, but this module attempts to work independently of most of them.
Where specific settings are required they will be marked with the phrase
"required merchant interface setting".

Warning
-------

This package has NOT been tested in production.  It is made available as a
foundation for future work.  We consider it "alpha" quality.


Transaction Keys
----------------

Each AIM transaction must be accompanied by a merchant login and a
"transaction key".  This key is obtained from the merchant interface.  After
importing the CcProcessor class you must pass it your login and transaction
key:

    >>> from zc.authorizedotnet.processing import CcProcessor
    >>> cc = CcProcessor(server=SERVER_NAME, login=LOGIN, key=KEY)


Authorizing
-----------

To authorize a charge use the ``authorize`` method.  It returns a
``Transaction`` object.

    >>> result = cc.authorize(amount='1.00', card_num='4007000000027',
    ...                       exp_date='0530')

The result object contains details about the transaction.

    >>> result.response
    'approved'
    >>> result.response_reason
    'This transaction has been approved.'
    >>> result.approval_code
    '123456'
    >>> result.trans_id
    '123456789'

The server successfully processed the transaction:

    >>> transactions = server.getTransactions()
    >>> transactions[result.trans_id]
    'Authorized/Pending Capture'


Capturing Authorized Transactions
---------------------------------

Now if we want to capture the transaction that was previously authorized, we
can do so.

    >>> result = cc.captureAuthorized(trans_id=result.trans_id)
    >>> result.response
    'approved'

The server shows that the transaction has been captured and is awaiting
setttlement:

    >>> transactions = server.getTransactions()
    >>> transactions[result.trans_id]
    'Captured/Pending Settlement'


Voiding Transactions
--------------------

If we need to stop a transaction that has not yet been completed (like the
capturing of the authorized transaction above) we can do so with the ``void``
method.

    >>> result = cc.void(trans_id=result.trans_id)
    >>> result.response
    'approved'

The server shows that the transaction was voided:

    >>> transactions = server.getTransactions()
    >>> transactions[result.trans_id]
    'Voided'


Transaction Errors
------------------

If something about the transaction is erroneous, the transaction results
indicate so.

    >>> result = cc.authorize(amount='2.00', card_num='4007000000027',
    ...                       exp_date='0599')

The result object reflecs the error.

    >>> result.response
    'error'
    >>> result.response_reason
    'The credit card has expired.'

The valid values for the ``response`` attribute are 'approved', 'declined',
and 'error'.


Address Verification System (AVS)
---------------------------------

AVS is used to assert that the billing information provided for a transaction
must match (to some degree or another) the cardholder's actual billing data.
The gateway can be configured to disallow transactions that don't meet certain
AVS criteria.


    >>> result = cc.authorize(amount='27.00', card_num='4222222222222',
    ...                       exp_date='0530', address='000 Bad Street',
    ...                       zip='90210')
    >>> result.response
    'declined'
    >>> result.response_reason
    'The transaction resulted in an AVS mismatch...'


Duplicate Window
----------------

The gateway provides a way to detect and reject duplicate transactions within
a certain time window.  Any transaction with the same CC information (card
number and expiration date) and amount duplicated within the window will be
rejected.

The first transaction will work.

    >>> result = cc.authorize(amount='3.00', card_num='4007000000027',
    ...                       exp_date='0530')
    >>> result.response
    'approved'

A duplicate transaction will fail with an appropriate message.

    >>> result2 = cc.authorize(amount='3.00', card_num='4007000000027',
    ...                       exp_date='0530')
    >>> result2.response
    'error'
    >>> result2.response_reason
    'A duplicate transaction has been submitted.'

The default window size is 120 seconds, but any other value (including 0) can
be provided by passing ``duplicate_window`` to the transaction method.

    >>> cc.captureAuthorized(trans_id=result.trans_id).response
    'approved'

    >>> cc.captureAuthorized(trans_id=result.trans_id).response_reason
    'This transaction has already been captured.'

    >>> cc.captureAuthorized(trans_id=result.trans_id, duplicate_window=0
    ...                     ).response
    'approved'

But voiding doesn't report errors if the same transaction is voided inside
the duplicate window.

    >>> cc.void(trans_id=result.trans_id).response
    'approved'

    >>> cc.void(trans_id=result.trans_id).response
    'approved'


The MD5 Hash Security Feature
-----------------------------

Authorize.Net provides for validating transaction responses via an MD5 hash.
The required merchant interface setting to use this feature is under
"Settings and Profile" and then "MD5 Hash".  Enter a "salt" value in the
fields provided and submit the form.  You may then provide the ``salt``
parameter to the CcProcessor constructor to enable response validation.

WARNING: The format of the "amount" field is very important for this feature
to work correctly.  The field must be formatted in the "canonical" way for the
currency in use.  For the US dollar that means no leading zeros and two (and
only two) decimal places.  If the amount is not formatted properly in the
request, the hashes will not match and the transaction will raise an exception.

If you want to enable hash checking, provide a ``salt`` value to the
``CcProcessor`` constructor.  If an incorrect salt value is used, or the
hash given in the transaction doesn't match the true hash value an exception
is raised.

    >>> cc = CcProcessor(server=SERVER_NAME, login=LOGIN, key=KEY,
    ...                  salt='wrong')
    >>> result = cc.authorize(amount='10.00', card_num='4007000000027',
    ...                       exp_date='0530')
    Traceback (most recent call last):
        ...
    ValueError: MD5 hash is not valid (trans_id = ...)


Error Checking
--------------

If you don't pass a string for the amount when doing an authorization, an
exception will be raised.  This is to avoid charging the wrong amount due to
floating point representation issues.

    >>> cc.authorize(amount=5.00, number='4007000000027', expiration='0530')
    Traceback (most recent call last):
        ...
    ValueError: amount must be a string

TODO
----

 - explain and demonstrate duplicate transaction window

NOTES
-----

M2Crypto 0.16 appears to have issues on some platforms such as Centos.
Reverting to version 0.15 has alleviated them in some cases.
