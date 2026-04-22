---
title: "Building an Order Book in Python: From Broken to Production-Ready"
date: 2026-04-22
draft: false
tags: ["python", "trading", "algorithms", "concurrency"]
description: "An iterative walkthrough of building a limit order book, ground up."
---

Order books are the most fundamental data structure in the trading world, and like with most fundamental things, you need to understand them.

In this post, I'll build one iteratively, deliberately introducing real bugs along the way, so you can understand *why* each design decision exists. In the end we will have created a half decent limit order book.

So let's get into the implementation and learn about order books.

---

## What is an Order Book?

First let's cover some vocabulary:

An order book is a real-time record of all outstanding buy and sell orders for an asset. Buy orders (bids) are sorted descending by price, the highest bidder is first. Sell orders (asks) are sorted ascending, the cheapest seller is first.

When the best bid price meets or crosses the best ask price, a **trade executes** at the meeting point. When multiple ask price levels are crossed we fill the orders based on price order, cheaper asks get filled first. The same logic applies in reverse when ask prices cross the spread and meet bids.

### Some more definitions:
- The **spread**: the gap which needs to be crossed in order for a **trade** to execute.
- The **aggressor**: the order which crosses the spread to make the **trade** happen, it is the one who pays the spread, can be an ask or bid order.
- **Limit** book: this means that our book will support limit orders, limit orders sit at fixed prices and execute at prices:
    - Smaller then or equal, for bids.
    - Greater then or equal, for asks.

---

## Iteration 1: The Naive Book

Let's start simple. We'll track bids and asks as price -> quantity mappings, and match when prices cross.

```python
from sortedcontainers import SortedDict

class OrderBook:
    def __init__(self):
        self.bids = SortedDict(lambda x: -x)    # highest first
        self.asks = SortedDict()                # lowest first

    def add_order(self, side, price, qty):
        book = self.bids if side == "buy" else self.asks
        book[price] = book.get(price, 0) + qty
        self._match(aggressor_side=side)

    def cancel_order(self, side, price, qty):
        book = self.bids if side == "buy" else self.asks
        if price in book:
            book[price] -= qty
            if book[price] <= 0:
                del book[price]

    def _match(self, aggressor_side):
        while self.bids and self.asks:
            best_bid = next(iter(self.bids))
            best_ask = next(iter(self.asks))
            if best_bid < best_ask:             # didnt cross the spread.
                break
            trade_price = best_ask if aggressor_side == "buy" else best_bid # choose price based on the side which crossed.
            qty = min(self.bids[best_bid], self.asks[best_ask])             # quantity to be traded.
            print(f"Trade: {qty} @ {trade_price}")
            self.bids[best_bid] -= qty
            self.asks[best_ask] -= qty
            if self.bids[best_bid] == 0: del self.bids[best_bid]
            if self.asks[best_ask] == 0: del self.asks[best_ask]

    @property
    def spread(self):
        if self.bids and self.asks:
            return next(iter(self.asks)) - next(iter(self.bids)) # difference between best ask and bid.
        return None
```

This works for a toy example. But it has serious problems.

### Problems

**1. No order identity.**
Orders have no IDs. You cannot cancel a specific order, only subtract a quantity from a price level. If three traders all placed 10-unit orders at $100, you have no way to cancel just one of them. `cancel_order(side, 100, 10)` blindly subtracts 10 from the level, which might partially cancel the wrong order.

**2. Price levels lose structure.**
Orders at the same price are merged into one number. Real exchanges use **price-time priority** (FIFO): the first order placed at a price level gets filled first. Our book has no concept of time at all. Two orders at $100 are indistinguishable.

**3. No trade records.**
Trades just print to stdout. There's no record of what executed, at what price, between which parties, or at what time.

**4. No concurrency safety.**
If two threads call `add_order` simultaneously, they race on the same dict. Python's GIL protects individual bytecode operations but not compound read-modify-write sequences like `book[price] = book.get(price, 0) + qty`. Under concurrent load, quantity can be silently lost.

---

## Iteration 2: Orders as First-Class Objects

We fix the identity and FIFO problems by introducing `Order` and `Trade` dataclasses, and storing each price level as a queue of individual orders.

```python
from collections import deque
from dataclasses import dataclass
from sortedcontainers import SortedDict
from typing import Optional
import uuid

@dataclass
class Order:
    id: str
    side: str
    price: float
    qty: float

@dataclass
class Trade:
    buy_order_id: str
    sell_order_id: str
    price: float
    qty: float

class OrderBook:
    def __init__(self):
        self.bids = SortedDict(lambda x: -x)        # sorted bids, price to volume
        self.asks = SortedDict()                    # sorted asks, price to volume
        self._bid_levels: dict[float, deque] = {}   # price to bucket (ordered by time)
        self._ask_levels: dict[float, deque] = {}   # price to bucket (...)
        self._order_index: dict[str, Order] = {}    # ID to order

    def add_order(self, side: str, price: float, qty: float) -> tuple[str, list[Trade]]:
        order = Order(id=str(uuid.uuid4()), side=side, price=price, qty=qty)
        self._insert(order)
        trades = self._match(aggressor_side=side)
        return order.id, trades

    def cancel_order(self, order_id: str) -> bool:
        if order_id not in self._order_index:
            return False
        order = self._order_index.pop(order_id)
        self._remove(order)
        return True

    def _insert(self, order: Order):
        book, levels = (self.bids, self._bid_levels) if order.side == "buy" \
                       else (self.asks, self._ask_levels)           # insert into the respective book
        book[order.price] = book.get(order.price, 0) + order.qty    # keep track of total volume at each level
        levels.setdefault(order.price, deque()).append(order)       # add the order to the queue for the level
        self._order_index[order.id] = order                         # keep track of id

    def _remove(self, order: Order):
        book, levels = (self.bids, self._bid_levels) if order.side == "buy" \
                       else (self.asks, self._ask_levels)
        book[order.price] -= order.qty  # remove from volume
        if book[order.price] <= 0:      # delete the level
            del book[order.price]
        if order.price in levels:       # delete from the level volume
            levels[order.price] = deque(o for o in levels[order.price] if o.id != order.id)

    def _match(self, aggressor_side: str) -> list[Trade]:
        trades = []
        while self.bids and self.asks:
            best_bid = next(iter(self.bids))
            best_ask = next(iter(self.asks))
            if best_bid < best_ask:     # doesnt cross spread.
                break

            trade_price = best_ask if aggressor_side == "buy" else best_bid

            bid_order = self._bid_levels[best_bid][0]   # select the first bid at the level.
            ask_order = self._ask_levels[best_ask][0]   # select the first ask at the level.
            qty = min(bid_order.qty, ask_order.qty)

            trades.append(Trade(bid_order.id, ask_order.id, trade_price, qty))

            bid_order.qty -= qty
            ask_order.qty -= qty

            if bid_order.qty == 0:                          # bid order empty
                self._bid_levels[best_bid].popleft()        # remove it from the queue
                self._order_index.pop(bid_order.id, None)   # remove it from the index
                if not self._bid_levels[best_bid]:          # check whether price level is empty
                    del self.bids[best_bid]                 # delete it

            if ask_order.qty == 0:                          # symmetric
                self._ask_levels[best_ask].popleft()
                self._order_index.pop(ask_order.id, None)
                if not self._ask_levels[best_ask]:
                    del self.asks[best_ask]

        return trades
```

### What improved

Each order now has a UUID. Cancellation is exact - you cancel *that* order, not just a quantity at a level. FIFO is enforced via `deque`, the first order in is the first order filled. Trades are returned as structured objects, not print statements.

### What's still broken

Still no concurrency safety. Every method is a compound read-modify-write. Under concurrent access this book will corrupt silently and won't provide any meaningful FIFO behaviour.

---

## Iteration 3: Adding a Lock (Naively)

The obvious fix: wrap everything in a `threading.Lock`.

```python
import threading

class OrderBook:
    def __init__(self):
        # ... same as above ...
        self._lock = threading.Lock()

    def add_order(self, side: str, price: float, qty: float):
        with self._lock:
            order = Order(id=str(uuid.uuid4()), side=side, price=price, qty=qty)
            self._insert(order)
            trades = self._match()
            return order.id, trades

    def cancel_order(self, order_id: str) -> bool:
        with self._lock:
            if order_id not in self._order_index:
                return False
            order = self._order_index.pop(order_id)
            self._remove(order)
            return True

    @property
    def spread(self) -> Optional[float]:
        with self._lock:
            if self.bids and self.asks:
                return next(iter(self.asks)) - next(iter(self.bids))
            return None
```

This is **correct**, but naively so.

### Problems with naive locking

**1. Reads block writes.**
Every call to `spread`, `best_bid`, or `best_ask` acquires the same exclusive lock. In a real system, market data consumers read the books very frequently, it would be a throughput disaster to treat all reads equal to writes.

**2. You can't split the lock.**
Use one lock for bids, one for asks, sounds smart. But the eventual call to `_match()` would need both locks simultaneously. This sounds like a deadlock, lets analyse further:

Imagine the following lock setup:
```py
lock_bids = threading.Lock()
lock_asks = threading.Lock()
```

Now ideally this would release some contention since deleting and looking up orders can be split in an individual lock accesses and we get non serial threads!
But now consider how a `_match()` definition would look like:
```py
# Thread A, buy order arrives
def add_buy():
    with lock_bids:          # acquires lock_bids
        insert_bid(...)
        with lock_asks:      # tries to acquire lock_asks, BLOCKS
            match(...)

# Thread B - sell order arrives at the same time
def add_sell():
    with lock_asks:          # acquires lock_asks
        insert_ask(...)
        with lock_bids:      # tries to acquire lock_bids, BLOCKS
            match(...)
```

This leads to a textbook deadlock. The fix is **lock ordering**, always acquire locks in the same fixed order, never the reverse. This eliminates the circular wait and prevents deadlock, but serialises both threads on the first lock acquisition regardless of side, no better than a single lock.
```py
def add_order(self, side, price, qty):
    with lock_bids:           # always first
        with lock_asks:       # always second
            insert(...)
            match(...)
```

**3. No fairness.**
`threading.Lock` offers no ordering guarantees. Under high contention, a thread can be starved indefinitely. For guaranteed ordering, `queue.Queue` is the practical answer, FIFO by design, and it's how most real order processing pipelines are built anyway.

---

## Iteration 4: Final

Keep the single lock, but be deliberate about it. Public methods hold the lock; private methods assume it is already held.
This is a common pattern in concurrency design, public methods acquire locks and private atomic methods assume the lock is already held. It keeps the locking boundary explicit and the private methods clean.

```python
import threading
from collections import deque
from dataclasses import dataclass
from sortedcontainers import SortedDict
from typing import Optional
import uuid

@dataclass
class Order:
    id: str
    side: str
    price: float
    qty: float

@dataclass
class Trade:
    buy_order_id: str
    sell_order_id: str
    price: float
    qty: float

class OrderBook:
    def __init__(self):
        self.bids = SortedDict(lambda x: -x)
        self.asks = SortedDict()
        self._bid_levels: dict[float, deque] = {}
        self._ask_levels: dict[float, deque] = {}
        self._order_index: dict[str, Order] = {}
        self._lock = threading.Lock()

    def add_order(self, side: str, price: float, qty: float) -> tuple[str, list[Trade]]:
        order = Order(id=str(uuid.uuid4()), side=side, price=price, qty=qty)
        with self._lock:
            self._insert(order)
            trades = self._match(aggressor_side=side)
        return order.id, trades

    def cancel_order(self, order_id: str) -> bool:
        with self._lock:
            if order_id not in self._order_index:
                return False
            order = self._order_index.pop(order_id)
            self._remove(order)
            return True

    def best_bid(self) -> Optional[float]:
        with self._lock:
            return next(iter(self.bids), None)

    def best_ask(self) -> Optional[float]:
        with self._lock:
            return next(iter(self.asks), None)

    @property
    def spread(self) -> Optional[float]:
        with self._lock:
            if self.bids and self.asks:
                return next(iter(self.asks)) - next(iter(self.bids))
            return None

    def _insert(self, order: Order):
        book, levels = (self.bids, self._bid_levels) if order.side == "buy" \
                       else (self.asks, self._ask_levels)
        book[order.price] = book.get(order.price, 0) + order.qty
        levels.setdefault(order.price, deque()).append(order)
        self._order_index[order.id] = order

    def _remove(self, order: Order):
        book, levels = (self.bids, self._bid_levels) if order.side == "buy" \
                       else (self.asks, self._ask_levels)
        book[order.price] -= order.qty
        if book[order.price] <= 0:
            del book[order.price]
        if order.price in levels:
            levels[order.price] = deque(
                o for o in levels[order.price] if o.id != order.id
            )

    def _match(self, aggressor_side: str) -> list[Trade]:
        trades = []
        while self.bids and self.asks:
            best_bid = next(iter(self.bids))
            best_ask = next(iter(self.asks))
            if best_bid < best_ask:
                break

            trade_price = best_ask if aggressor_side == "buy" else best_bid

            bid_order = self._bid_levels[best_bid][0]
            ask_order = self._ask_levels[best_ask][0]
            qty = min(bid_order.qty, ask_order.qty)

            trades.append(Trade(bid_order.id, ask_order.id, trade_price, qty))

            bid_order.qty -= qty
            ask_order.qty -= qty

            if bid_order.qty == 0:
                self._order_index.pop(bid_order.id, None)
                self._remove(bid_order)

            if ask_order.qty == 0:
                self._order_index.pop(ask_order.id, None)
                self._remove(ask_order)

        return trades
```

### Complexity

| Operation | Time |
|---|---|
| `add_order` | O(log n) - SortedDict insert |
| `cancel_order` | O(1) lookup + O(k) deque filter |
| `_match` | O(m log n) where m = number of trades |
| `best_bid / best_ask` | O(1) |
| `spread` | O(1) |

The `_order_index` dict gives O(1) cancel lookup, without it you'd scan every level to find an order by ID.

---

## What's Still Missing

A production exchange would add:

- **Order types** - limit, market, IOC (immediate-or-cancel), FOK (fill-or-kill) - we only support fill.
- **Timestamps** - for true price-time priority and audit logs - we use locking order which is unreliable and doesnt reflect Order placement order.
- **Persistent trade log** - append-only record of all fills - allows for recovery if crash took place.
- **Reader-writer lock** - separate read/write paths for high-frequency market data
- **Async support** - swap `threading.Lock` for `asyncio.Lock` - this also ensures fair scheduling by the event loop instead of relying on the OS thread scheduler.

---

## Takeaway

Started naive, found the flaws, fixed them one by one. The final version is correct and production-reasonable for a single-process system. The missing pieces above are what separates it from something you'd run on a real exchange.
