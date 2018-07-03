---
layout: post
title: Rate Limiting Part 2
subtitle: Guava Implementation
tags: [Algorithm, Java, Rate Limiting]
---

# Overview
In [previous pos](/2018/06/25/rate-limiter-part1), we learned some rate limiting
algorithms. In this article, we will see how rate limiting is implemented in
[Google Guava library](https://github.com/google/guava).

# RateLimiter
`RateLimiter` class is the base class for all types of rate limiters. Currently
Guava only has two implementations: `SmoothWarmingUp` and `SmoothBursty`.

## Static Creators Methods
`RateLimiter` provides two static creators that will create a `SmoothBursty`
rate limiter and a `SmoothWarmingUp` rate limiter respectively. We will talk
about these two rate limiters later.

```java
static RateLimiter create(double permitsPerSecond)
static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit)
```

The time unit for the configured rate is per second. If you want to use any
other time unit, you have to convert it to permits per second. For example, 1
permit per minute equals to 1 / 60 permits per second.

## Concurrency
`RateLimiter` class takes care of multi-thread access to methods by using a
mutex.
```java
@MonotonicNonNull private volatile Object mutexDoNotUseDirectly;

private Object mutex() {
  Object mutex = mutexDoNotUseDirectly;
  if (mutex == null) {
    synchronized (this) {
      mutex = mutexDoNotUseDirectly;
      if (mutex == null) {
        mutexDoNotUseDirectly = mutex = new Object();
      }
    }
  }
  return mutex;
}
```
In each method, `syschronized(mutex())` is called around operations, including
invocation to abstract methods implemented by subclasses so that they don't need
to worry about concurrency.

{: .box-note}
Think: why it uses two if statements to check the mutex value?

## Abstract Methods
Everything else is taken care by the base class `RateLimiter` so that a subclass
only has to implement the following four methods:

```java
abstract void doSetRate(double permitsPerSecond, long nowMicros);
abstract double doGetRate();
abstract long queryEarliestAvailable(long nowMicros);
abstract long reserveEarliestAvailable(int permits, long nowMicros);
```

The first two methods are straightforward and they imply that a user can change
the rate at runtime.

### queryEarliestAvailable
`queryEarliestAvailable` takes current time and returns the earliest available
time of next permit. It is called when checking whether a permit can be granted
with certain timeout:

```java
private boolean canAcquire(long nowMicros, long timeoutMicros) {
  return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}
```

Personally I think this method will be easier to understand if the return statement
is changed to: `nowMicros + timeoutMicros >= queryEarliestAvailable(nowMicros)`,
which means return true if and only if we can pass the earliest available time
of next permit after waiting for at most timeout microseconds from now on.

![queryEarliestAvailable](/img/queryEarliestAvailable.png)

### reserveEarliestAvailable
`reserveEarliestAvailable` reserves the requested number of permits and returns
the earliest time when all permits are available. This method is called to
calculate the duration that the caller has to wait before getting all permits.

`reserveAndGetWaitLength` is the method used to calculate the time to wait by
calling `reserveEarliestAvailable`.
```java
final long reserveAndGetWaitLength(int permits, long nowMicros) {
  long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
  return max(momentAvailable - nowMicros, 0);
}
```

## SleepingStopwatch
`RateLimiter` depends on a `SleepingStopwatch` to mesure the time elapsed since
start as well as sleep the thread whenever needed.
```java
abstract static class SleepingStopwatch {
  protected abstract long readMicros(); // Return time elpsed since start
  protected abstract void sleepMicrosUninterruptibly(long micros); // Sleep
}
```
This class also provides a static creator method which creates an implementation
with the help of another two classes in Guava: `Stopwatch` and
`Uninterruptibles`.

## tryAcquire and acquire
`tryAcquire()` and `acquire()` are the two most important methods, which are
responsible to calculate the throttling time and sleep for that long.

### tryAcquire
`tryAcquire()` has four overload methods, but all of them finally call to the
following one:

```java
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
  long timeoutMicros = max(unit.toMicros(timeout), 0);
  checkPermits(permits); // Parameter validation
  long microsToWait;
  synchronized (mutex()) {
    long nowMicros = stopwatch.readMicros();
    if (!canAcquire(nowMicros, timeoutMicros)) {
      return false;
    } else {
      microsToWait = reserveAndGetWaitLength(permits, nowMicros);
    }
  }
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return true;
}
```
The method first calls `canAcquire` to check whether the requested permits can
be acquired within the timeout value specified by the user. If not then it
directly returns false.  Otherwise it calculates the time it needs to wait and
sleeps for that long before returning true.

### acquire
`acquire()` has two overload methods and they finally call to the following one:
```java
public double acquire(int permits) {
  long microsToWait = reserve(permits);
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}
```
Different from `tryAcquire()`, `acquire()` will block until all requested
permits can be granted. It returns the time spent sleeping in seconds.

# SmoothRateLimiter
`SmoothRateLimiter` is a subclass of `RateLimiter` and the superclass of
`SmoothWarmingUp` and `SmoothBursty`.

## Design
The design is based on leaky bucket algorithm. But it also borrows some ideas
from token bucket.

One problem of native leaky bucket algorithm is, it doesn't keep the memory of
past underutilization - when lots of water drops after the bucket being empty
for long, it still leaks at the same constant rate as usual.

![Leaky Bucket Problem](/img/leaky-bucket-problem.png)

To deal with the underutilization problem, a `storedPermits` is used to model
the past underutilization. Specifically, this variable is 0 when there is no
underutilization and it is increased by 1 every second that the rate limiter is
unused. It can grow up to `maxPermits`.  Requested permits are first served by
the `storedPermits` and then served by fresh permits. When `storedPermits` is 3
and `acquire(10)` is called, then the first 3 permits are served by
`storedPermits` while the rest 7 permits are served by fresh permits.

Now the question is, when we serve the permits using `storedPermits`, how to
calculate the time to wait? When there is no underutilization, the interval
between requests should be `1 / rate`. When a user requests `n` permits, if they
are all served by fresh permits, the waiting time should be `n / rate`. But if
the `n` permits are all or partially served by `storedPermits`, what should be
the throttling time?

Basically, we need a function that maps `storedPermits` and `permitsToTake` to
throttling time. `SmoothRateLimiter` defines an abstract method
`storedPermitsToWaitTime(double storedPermits, double permitsToTake)` that needs
to be implemented by the subclasses to play this role.

Another abstract method is `coolDownIntervalMicros()` which returns the interval
between two increments of `storedPermits` when the rate limiter is unused.  In
other words, it measures how fast we cumulate the `storedPermits`. For example,
if it returns `10^6` microseconds (1 second), then we increase `storedPermits`
by 1 every second. The return value can be same as or different from the stable
interval. Actually it is the same as stable interval in `SmoothBursty`
implementation but different from stable interval in `SmoothWarmingUp`
implementation. We will talk about that later.

## Implementation

### Variables
A `SmoothRateLimiter` has four member variables:
* `storedPermits` - used to model past underutilization as described in Design
  section.
* `maxPermits` - maximum number of `storedPermits`.
* `stableIntervalMicros` - the interval between two requests at the stable rate.
  Its value should be `1 / rate`.
* `nextFreeTicketMicros` - the time when next permit can be granted. It is
  updated after each permit is granted.

### rsync
`resync()` method updates the value of `nextFreeTicketMicros` and
`storedPermits` if `nextFreeTicketMicros` is in the past.

```
void resync(long nowMicros) {
  // if nextFreeTicket is in the past, resync to now
  if (nowMicros > nextFreeTicketMicros) {
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    storedPermits = min(maxPermits, storedPermits + newPermits);
    nextFreeTicketMicros = nowMicros;
  }
}
```

Here we see that it calls subclass's `coolDownIntervalMicros()` to determine how
many permits we need to add to `storedPermits`. This is similar to lazily
refilled tokens in token bucket algorithm we discussed in [previous
article](2018/06/25/Rate-Limiter-Part1/).

`resync()` is called before reserving permits. See `reserveEarliestAvailable`
below.

### Override Methods
Since `SmoothRateLimiter` is a subclass of `RateLimiter`, it needs to implement
the abstract methods. As we mentioned earlier, two most important ones are
`queryEarliestAvailable` and `reserveEarliestAvailable`.

Apparently, `queryEarliestAvailable` just needs to return `nextFreeTicketMicros`
variable.
```java
@Override
final long queryEarliestAvailable(long nowMicros) {
  return nextFreeTicketMicros;
}
```

`reserveEarliestAvailable` also returns current `nextFreeTicketMicros`. But
it also needs to update `nextFreeTicketMicros` to a new value.
```java
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  resync(nowMicros);
  long returnValue = nextFreeTicketMicros;
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  double freshPermits = requiredPermits - storedPermitsToSpend;
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
          + (long) (freshPermits * stableIntervalMicros);

  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}
```
This method first saves the old `nextFreeTicketMicros` to be the return value.
Then it calculates how many stored permits and how many fresh permits will be
used to serve this the required permits. The waiting time incurred by the stored
permits is gotten from `storedPermitsToWaitTime` implemented by the subclass
while the waiting time incurred by the fresh permits is just `freshPermits *
stableInterval`. The total waiting time is the sum of these two value and is
added to `nextFreeTicketMicros`, which means next request will have to wait for
at most that long before being granted.

## SmoothBursty
`SmoothBursty` implementation is very straightforward -
`storedPermitsToWaitTime` simply returns 0, meaning that `storedPermits` are
translated to zero throttling. That's why it's called "bursty".
```java
@Override
long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
  return 0L;
}
```

`coolDownIntervalMicros` simply returns `stableIntervalMicros`, meaning that
`storedPermits` cumulates at the stable rate while unused.
```java
@Override
double coolDownIntervalMicros() {
  return stableIntervalMicros;
}
```

`SmoothBursty` has a member variable `maxBurstSeconds` that is used to calculate
`maxPermits`.
```java
maxPermits = maxBurstSeconds * permitsPerSecond;
```
By default `maxBurstSeconds` is 1 and there is no way for the user to change its
value.

## SmoothWarmingUp
To understand how `SmoothWarmingUp` rate limiter converts `storedPermits` to
throttling time, let's first look at the following figure:

![SoothWarmingUp](/img/smooth-warm-up.png)

The x-axis is `storedPermits` and the y-axis is the throttling interval.

Suppose there is a vertical line `x = k` representing the state of the rate
limiter:

* When the rate limiter is fully used, the line stays at `x = 0`.
* When the rate limiter is unused, the line goes from left to right.
* When the rate limiter is used again after an underutilization, the line goes
  from right to left.

The trapezoid in the figure denotes a warm up period. When a `SmoothWarmingUp`
rate limiter is reused after an underutilization, it is said to be in a warm up
period until `storedPermits` reaches `thresholdPermits`. In the warm up period,
the throttling interval is longer than stable interval, which means the rate
limiter becomes slower after a period of underutilization. Warm up period
duration can be specified by the user.

How to decide the value of `thresholdPermits` and `maxPermits`? In the comment,
they claim that to preserve the behavior of original implementation, the time it
takes to go from `thresholdPermits` to 0 has to be half the time it takes to go
from `maxPermits` to `thresholdPermits`, which is the `warmUpPeriod`. In other
words, in the figure above, the area of the trapezoid is two times the area of
the rectangle. Therefore, to calculate `thresholdPermits`:
```
warmupPeriod = 2 * stableInterval * thresholdPermits
thresholdPermits = 0.5 * warmupPeriod / stableInterval
```

`warmupPeriod` is specified by the user while creating this rate limiter.

To calculate `maxPermits`:
```
  warmupPeriod
= 0.5 * (stableInterval + coldInterval) * (maxPermits - thresholdPermits)

  maxPermits
= thresholdPermits + 2.0 * warmupPeriod / (stableInterval + coldInterval)
```

`coldInterval` is the maximum throttling interval, which is calculated by a
parameter `coldFactor` that has a default value 3.0 and is not modifiable by
the user.
```
coldInterval = stableInterval * coldFactor;
```

### coolDownIntervalMicros
```java
@Override
double coolDownIntervalMicros() {
  return warmupPeriodMicros / maxPermits;
}
```
As we mentioned before, `coolDownIntervalMicros` is the interval between two
increments of `storedPermits`. In the figure above, it measures how fast our
imaginary vertical line goes from left to right. This value is chose to be
`warmupPeriod / maxPermits` so that the time it takes to go from 0 to
`maxPermits` is equal to `warmupPeriod`.

### storedPermitsToWaitTime

Suppose currently there are K `storedPermits` and we request P permits (P <= K).
Then the total throttling time should be the area between `x = K` and `x = K -
P`.

![Smooth Warm Up Request](/img/smooth-warm-up-request.png)

Thus, `storedPermitsToWaitTime` just needs to calculate this area.

```java
@Override
long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
  double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
  long micros = 0;
  // measuring the integral on the right part of the function (the climbing line)
  if (availablePermitsAboveThreshold > 0.0) {
    double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
    double length =
        permitsToTime(availablePermitsAboveThreshold)
            + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
    micros = (long) (permitsAboveThresholdToTake * length / 2.0);
    permitsToTake -= permitsAboveThresholdToTake;
  }
  // measuring the integral on the left part of the function (the horizontal line)
  micros += (long) (stableIntervalMicros * permitsToTake);
  return micros;
}
```

It first checks whether `storedPermits` is above the threshold. If so, then we
need to calculate the area of the trapezoid. Otherwise we only need to calculate
the partial area of the rectangle. `permitsToTime` is used to query "given x,
what is the y value of the climbing line?"
```java
private double permitsToTime(double permits) {
  return stableIntervalMicros + permits * slope;
}
```

The value of `slope` should be `(coldInterval - stableInterval) / (maxPermits -
thresholdPermits)`.

## Problem
After understanding the implementation, we can find one problem - when we only
request 1 permit at a time, the `SmoothRateLimiter` works as a leaky bucket with
consideration of past underutilization. However, if we request many permits at a
time, it has the same problem as token bucket. That is, the real-time rate might
be higher than the configured rate. And it may starve following requests.

![Token Bucket Problem](/img/token-bucket-problem.png)

This is because the throttling time is paid by next request instead of current
request. And larger requests push next available time further than smaller
requests. For example, suppose the rate is 1 per seconds. If we request 100
permits at the beginning, the permits will be granted right away but the next
request needs to wait for at most 100 seconds. Since most of the use cases I
have seen only require to request 1 permit at a time, I am not sure whether this
is a serious problem. But you may need to keep it in mind while requesting many
requests at a time.

# Reference
[1] [RateLimiter.java](https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/RateLimiter.java)<br>
[2] [SmoothRateLimiter.java](https://github.com/google/guava/blob/master/guava/src/com/google/common/util/concurrent/SmoothRateLimiter.java)<br>
