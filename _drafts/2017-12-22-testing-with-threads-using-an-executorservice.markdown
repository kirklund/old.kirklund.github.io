---
layout: post
title:  "Testing With Threads: Using an ExecutorService"
date:   2017-12-22 00:00:00 -0000
categories: [Apache Geode, Testing]
tags: [Regression Test]
---
Goal: to reproduce a deadlock, infinite loop or other type of thread hang in a Regression Test so you can confirm that the bug exists and the fix corrects the problem. Since reusing a JVM across many tests is pretty common, you really don’t want to leave a hung thread hanging around to pollute the JVMs for subsequent tests.

In this post, I’ll detail how I’ve used `ExecutorService` in tests to provide a background thread that I can test with and then terminate during `tearDown`.

Since it’s easier to explain with an example, I’ll use [GEODE-4083](https://issues.apache.org/jira/browse/GEODE-4083) from Apache Geode Jira which is a bug involving an infinite loop that I paired on recently with one of my teammates. The bug was initially exposed in a Pivotal GemFire test. My team found a hot thread stuck in an infinite loop and filed GEODE-4083. We studied the class `RegionVersionVector` and determined that multiple threads can execute the code concurrently.

The bug involves the method `updateLocalVersion(long)`. Here’s the [method before we made any changes](https://github.com/apache/geode/blob/96684f3f801998edea9d64ce0f66cc9f73621c97/geode-core/src/main/java/org/apache/geode/internal/cache/versions/RegionVersionVector.java#L577):

{% highlight java %}
577:  private void updateLocalVersion(long version) {
578:    boolean repeat = false;
579:    do {
580:      long myVersion = this.localVersion.get();
581:      if (myVersion < version) {
582:        repeat = !this.localVersion.compareAndSet(myVersion, version);
583:      }
584:    } while (repeat);
585:  }
{% endhighlight %}

Thread-1 enters the method and sees a `myVersion` on line 580 which is older than the `version` passed into the method.

Thread-2 concurrently enters the method, also sees an older `myVersion` on line 4 and manages to execute line 582 before Thread-1. Thread-2 sets the value of `localVersion` and then exits the method.

When Thread-1 resumes and hits line 582, the `compareAndSet` fails which sets `repeat` to true. It then loops around and refetches `myVersion` from `localVersion`. This time the value of `myVersion` is equal to `version` because that’s what Thread-2 set it to. The condition on line 581 is false, so the value of `repeat` remains true. Thread-1 loops around forever burning up CPU. There’s our bug!

It’s pretty obvious we could set the flag back to false at the beginning of every loop iteration to fix the bug. My pairing partner and I decide to write a Regression Test which will verify the existence of the bug and thereafter verify the fix after we’ve fixed it.

We search for any tests for the `RegionVersionVector` class and we find a Unit Test called `RegionVersionVectorJUnitTest`. We read through the test and run it in IntelliJ to see what sort of coverage it provides for the class. The method has coverage but it’s not direct coverage, which is also pretty obvious since the method is private.

First, we rename the test to `RegionVersionVectorTest`. Next, in order to improve test coverage for `updateLocalVersion`, we decide to make the method package-private so we can write some new tests directly against it. First thing we notice is that `RegionVersionVector` is abstract so we decide to add a private static class called `TestableRegionVersionVector` to the Unit Test. IntelliJ implements the abstract methods with no-ops and that’s fine for our purpose:

{% highlight java %}
private static class TestableRegionVersionVector
    extends RegionVersionVector<VersionSource<InternalDistributedMember>> {

  TestableRegionVersionVector(VersionSource<InternalDistributedMember> ownerId, long version) {
    super(ownerId, null, version);
  }

  @Override
  protected RegionVersionVector createCopy(VersionSource ownerId, ConcurrentHashMap vector,
      long version, ConcurrentHashMap gcVersions, long gcVersion, boolean singleMember,
      RegionVersionHolder clonedLocalHolder) {
    return null;
  }

  @Override
  protected VersionSource<InternalDistributedMember> readMember(DataInput in)
      throws IOException, ClassNotFoundException {
    return null;
  }

  @Override
  protected void writeMember(VersionSource member, DataOutput out) throws IOException {

  }

  @Override
  public int getDSFID() {
    return 0;
  }
}
{% endhighlight %}

The first 3 tests added for `updateLocalVersion` are:

{% highlight java %}
@Test
public void usesNewVersionIfGreaterThanOldVersion() throws Exception {
  VersionSource<InternalDistributedMember> ownerId = mock(VersionSource.class);
  long oldVersion = 1;
  long newVersion = 2;

  RegionVersionVector rvv = new TestableRegionVersionVector(ownerId, oldVersion);
  rvv.updateLocalVersion(newVersion);
  assertThat(rvv.getVersionForMember(ownerId)).isEqualTo(newVersion);
}

@Test
public void usesOldVersionIfGreaterThanNewVersion() throws Exception {
  VersionSource<InternalDistributedMember> ownerId = mock(VersionSource.class);
  long oldVersion = 2;
  long newVersion = 1;

  RegionVersionVector rvv = new TestableRegionVersionVector(ownerId, oldVersion);
  rvv.updateLocalVersion(newVersion);
  assertThat(rvv.getVersionForMember(ownerId)).isEqualTo(oldVersion);
}

@Test
public void doesNothingIfVersionsAreSame() throws Exception {
  VersionSource<InternalDistributedMember> ownerId = mock(VersionSource.class);
  long oldVersion = 2;
  long sameVersion = 2;

  RegionVersionVector rvv = new TestableRegionVersionVector(ownerId, oldVersion);
  rvv.updateLocalVersion(sameVersion);
  assertThat(rvv.getVersionForMember(ownerId)).isEqualTo(oldVersion);
}
{% endhighlight %}

These new tests characterize the existing behavior of `updateLocalVersion`. They don’t expose our bug but they do help provide us with fast, early warning if any of our changes to `updateLocalVersion` breaks existing behavior.

Now with the help of these new tests, we safely make a few changes to `RegionVersionVector` to facilitate testing: 1) group the constructors together and add a new package-private constructor to use in testing that we’ll use to initialize the value of `localVersion`, 2) cleanup variable names in the `updateLocalVersion` method, and 3) we extract the call to `localVersion.compareAndSet` to a new package-private method so we can override it in our test to mimic the race condition:

{% highlight java %}
void updateLocalVersion(long version) {
  boolean needToTrySetAgain = false;
  do {
    long currentVersion = this.localVersion.get();
    if (currentVersion < version) {
      needToTrySetAgain = !compareAndSetVersion(currentVersion, version);
    }
  } while (needToTrySetAgain);
}

boolean compareAndSetVersion(long currentVersion, long newVersion) {
  return this.localVersion.compareAndSet(currentVersion, newVersion);
}
{% endhighlight %}

Now it’s ready to write a test that verifies the hang. We decide to use an `ExecutorService` to fork a thread that we’ll force into the infinite loop that causes the bug.

First we need another private static class which will override the `compareAndSetVersion` method to mimic the race caused by two threads executing `updateLocalVersion` concurrently:

{% highlight java %}
private static class VersionRaceConditionRegionVersionVector extends TestableRegionVersionVector {

  VersionRaceConditionRegionVersionVector(VersionSource<InternalDistributedMember> ownerId,
                                          long version) {
    super(ownerId, version);
  }

  @Override
  boolean compareAndSetVersion(long currentVersion, long newVersion) {
    super.compareAndSetVersion(currentVersion, newVersion);
    return false;
  }
}
{% endhighlight %}

Now it’s ready for a test that will verify the infinite loop experienced by Thread-1:

{% highlight java %}
@Test
public void hangsIfOtherThreadChangedVersion() throws Exception {
  ExecutorService executorService = Executors.newSingleThreadExecutor();
  VersionSource<InternalDistributedMember> ownerId = mock(VersionSource.class);
  long oldVersion = 1;
  long newVersion = 2;

  RegionVersionVector rvv = new VersionRaceConditionRegionVersionVector(ownerId, oldVersion);
  Future result = executorService.submit(() -> rvv.updateLocalVersion(newVersion));

  result.get(2, SECONDS);
}
{% endhighlight %}

We execute the test and it now fails with a `TimeoutException` because the thread invoking `updateLocalVersion` is stuck in an infinite loop.

There’s one problem though — after the test completes that background thread is still there, looping forever, and it's hot. So, we hoist the `ExecutorService` out to a member field in the test and invoke `shutdownNow` on it in `tearDown` to ensure that the thread is interrupted and terminates:

{% highlight java %}
private ExecutorService executorService;

@Before
public void setUp() throws Exception {
  executorService = Executors.newSingleThreadExecutor();
}

@After
public void tearDown() throws Exception {
  assertThat(executorService.shutdownNow()).isEmpty();
}
@Test
public void hangsIfOtherThreadChangedVersion() throws Exception {
  VersionSource<InternalDistributedMember> ownerId = mock(VersionSource.class);
  long oldVersion = 1;
  long newVersion = 2;

  RegionVersionVector rvv = new VersionRaceConditionRegionVersionVector(ownerId, oldVersion);
  Future result = executorService.submit(() -> rvv.updateLocalVersion(newVersion));

  result.get(2, SECONDS);
}
{% endhighlight %}

One of the members of my team prefers using `CompletableFuture` and `Future<Void>` so we make one more change:

{% highlight java %}
@Test
public void hangsIfOtherThreadChangedVersion() throws Exception {
  VersionSource<InternalDistributedMember> ownerId = mock(VersionSource.class);
  long oldVersion = 1;
  long newVersion = 2;

  RegionVersionVector rvv = new VersionRaceConditionRegionVersionVector(ownerId, oldVersion);
  Future<Void> result =
      CompletableFuture.runAsync(() -> rvv.updateLocalVersion(newVersion), executorService);

  result.get(2, SECONDS);
}
{% endhighlight %}

Since we’re passing `executorService` to `CompletableFuture.runAsync`, the background thread is still interruptible for proper `tearDown`.

Now for the fix. This is pretty easy since we already identified the problem and its solution. We need to reset the value of the flag which we’ve renamed to `needToTrySetAgain` at the start of each loop iteration:

{% highlight java %}
void updateLocalVersion(long version) {
  boolean needToTrySetAgain = false;
  do {
    needToTrySetAgain = false; <-- here's the fix!
    long currentVersion = this.localVersion.get();
    if (currentVersion < version) {
      needToTrySetAgain = !compareAndSetVersion(currentVersion, version);
    }
  } while (needToTrySetAgain);
}
{% endhighlight %}

The new test now passes! Next we rename the test and add a couple assertions for proper validation:

{% highlight java %}
@Test
public void doesNotHangIfOtherThreadChangedVersion() throws Exception {
  VersionSource<InternalDistributedMember> ownerId = mock(VersionSource.class);
  long oldVersion = 1;
  long newVersion = 2;

  RegionVersionVector rvv = new VersionRaceConditionRegionVersionVector(ownerId, oldVersion);
  Future<Void> result =
      CompletableFuture.runAsync(() -> rvv.updateLocalVersion(newVersion), executorService);

  assertThatCode(() -> result.get(2, SECONDS)).doesNotThrowAnyException();
  assertThat(rvv.getVersionForMember(ownerId)).isEqualTo(newVersion);
}
{% endhighlight %}

The bug is fixed, we’ve improved unit test coverage for the section of code that was involved in the bug and we have a new Regression Test that will ensure this bug never returns. If you’d like to view the full diff, take a look at [the main commit](https://github.com/apache/geode/commit/b1486ae6d830d6c707264f1a03509a621cb95570) and [this followup commit](https://github.com/apache/geode/commit/0ca3c8cbdf63c77b921041951601a4eb6766cfb6) in the Apache Geode repo on GitHub.

We’ve used an `ExecutorService` for thread testing in several tests in Apache Geode, so next time I’ll share the creation of a JUnit Rule that makes this testing pattern easy to drop in and reuse in any test!
