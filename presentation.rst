:css: hovercraft.css

.. title:: asyncio

----

=======
Asyncio
=======

----

Standard
========


.. note::

    * short scripts
    * your service binding
    * selectors

----

Yield Points
============

----

.. code-block:: python

    @coroutine
    def transfer(amount, payer, payee, server):
        if not payer.sufficient_funds_for_withdrawl(amount):
            raise InsufficientFunds()
        log("{payer} has sufficient funds.", payer=payer)
        payee.deposit(amount)
        log("{payee} received payment", payee=payee)
        payer.withdraw(amount)
        log("{payer} made payment", payer=payer)
        yield from server.update_balances([payer, payee])

----

.. image:: van_rossum.png

----

Using gevent:

.. code-block:: python

    def transfer(amount, payer, payee, server):
        with payee.lock_deposit():
            if not payer.sufficient_funds_for_withdrawl(amount):
                raise InsufficientFunds()
            payee.deposit(amount)
            payer.withdraw(amount)
            server.update_balances([payer, payee])
        log("{payer} has sufficient funds.", payer=payer)
        log("{payee} received payment", payee=payee)
        log("{payer} made payment", payer=payer)

----

Using asyncio:

.. code-block:: python

    @coroutine
    def transfer(amount, payer, payee, server):
        # WARNING! WARNING! WARNING!
        # Do not insert yield points between following lines
        if not payer.sufficient_funds_for_withdrawl(amount):
            raise InsufficientFunds()
        payee.deposit(amount)
        payer.withdraw(amount)
        yield from server.update_balances([payer, payee])
        # End of careful code block
        log("{payer} has sufficient funds.", payer=payer)
        log("{payee} received payment", payee=payee)
        log("{payer} made payment", payer=payer)

----

Exceptions:

.. code-block:: python

    @coroutine
    def transfer(amount, payer, payee, server):
        # WARNING! WARNING! WARNING!
        # Do not insert yield points between following lines
        if not payer.sufficient_funds_for_withdrawl(amount):
            raise InsufficientFunds()
        payee.deposit(amount)
        currency_rate = get_currency_rate()
        payer.withdraw(amount/currency_rate)  # !!!
        yield from server.update_balances([payer, payee])
        # End of careful code block

----

Pipelining
==========

----

gevent code:

.. code-block:: python

    def request(self, req):
        self.socket.write(req)
        return self.socket.readline()

----

asyncio code:

.. code-block:: python

    # not-a-coroutine
    def request(self, req):
        self.writer.write(req)  # (!)this is not-yielding
        fut = Future()
        self.requests.append(fut)
        return fut

----

asyncio response reader:

.. code-block:: python

    @coroutine
    def reader(self):
        while True:
            line = yield from self.reader.readline()
            self.requests.popleft().set(line)

----

asyncio bad example:

.. code-block:: python

    # not-a-coroutine
    def request(self, req):
        self.writer.write(req)
        yield from self.writer.drain()
        value = yield from self.reader.readline()
        return value

----

another awful example:

.. code-block:: python

    # not-a-coroutine
    def do_something(redis);
        yield from redis.multi()
        yield from redis.set('a', 'b')
        yield from redis.set('c', 'd')
        yield from redis.exec()

----

why this is bad:

.. code-block:: python

    # not-a-coroutine
    def update_price(redis, dollars, rate);
        yield from redis.multi()
        yield from redis.set('price_dollars', str(dollars))
        yield from redis.set('price_uah', str(dollars/rate)) # rate = 0 ??
        yield from redis.exec()

----

locks in asyncio:

.. code-block:: python

    @coroutine
    def something(self):
        with (yield from self.lock()):
            yield from self.do_work_during_lock()
        # The following is executed concurrently with self.release()
        yield from self.do_work_after_lock()

----

lock implementation:

.. code-block:: python

    # not-a-couroutine
    def __exit__(self, et, ev, tb):
        Task(self.release())

----

bad implementation of release():

.. code-block:: python

    @coroutine
    def release(self):
        yield from self.release_request()

----

good implementation of release():

.. code-block:: python

    # not-a-couroutine
    def release(self):
        self.transport.write(release_request())
        self.requests.push(Future())

----

Problems with Asyncio
=====================

----

Can't use generators
---------------------

----

.. code-block:: python

    @coroutine
    def get_items():
        resp = yield from request()
        for line in resp.splitlines():
            yield line

----

Use list:

.. code-block:: python

    @coroutine
    def get_items():
        resp = yield from request()
        return list(resp.splitlines())

----

But not list comprehensions:

.. code-block:: python

    @coroutine
    def get_urls(urls):
        return [(yield from request(url)) for url in urls]

----

"yield from" comprehensions:

.. code-block:: pycon

    >>> [(yield from '') for _ in '']
    <generator object <listcomp> at 0x7f5dbb02ecf0>
    >>> list((yield from 'ab') for _ in 'cd')
    ['a', 'b', None, 'a', 'b', None]
    >>> list({(yield from 'ab') for _ in 'cd'})
    ['a', 'b', 'a', 'b']
    >>> list({a: (yield from 'ab') for a in 'cd'})
    ['a', 'b', 'a', 'b']
    >>> list((a, (yield from 'ab')) for a in 'cd')
    ['a', 'b', ('c', None), 'a', 'b', ('d', None)]


----

yield list of coroutines:

.. code-block:: python

    # not-a-coroutine
    def get_lines()
        while True:
            yield request()

    @coroutine
    def function():
        for fut in get_lines():
            line = yield from fut()

----

bad example (asyncio-redis):

.. code-block:: python

    yield from (yield from redis.smembers('name')).asset()

good example:

.. code-block:: python

    for future in redis.scan('name'):
        key = yield from future()

----

Common Bugs
===========

----

Async task:

.. code-block:: python

    Task(redis.set('a', 'b'))

----

Exception handling:

.. code-block:: python

    def task_wrapper():
        try:
            yield from redis.set('a', 'b')
        except Exception as e:
            log.exception("Task error")

    Task(task_wrapper())

----

Forgetting yield from::

    PYTHONASYNCIODEBUG=1

.. code-block:: pycon

    >>> asyncio.coroutine(a)()
    <asyncio.tasks.CoroWrapper object at 0x7f5fe4cdbb38>

----

Waiting first completed::

    done, left = yield from asyncio.wait(map(asyncio.Task, coroutines),
        return_when=asyncio.FIRST_COMPLETED)
    for coro in left:
        coro.cancel()


----

Rewrite Everything
==================

----

Special Methods
===============

----

asyncio vs gevent
=================
