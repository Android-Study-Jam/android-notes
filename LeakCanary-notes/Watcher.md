## LeakCanary: Watcher

#### HeapDump

Data Structure used for holding information about a heap dump.

Listener recieves the heap dump to analyze.
```java
    public interface Listener {
    Listener NONE = new Listener() {
      @Override public void analyze(HeapDump heapDump) {
      }
    };

    void analyze(HeapDump heapDump);
  }
```

```java
/** The heap dump file, which you might want to upload somewhere. **/
public final File heapDumpFile;

public final String referenceKey;

/**
* User defined name to help identify the leaking instance.
*/
public final String referenceName;

/** References that should be ignored when analyzing this heap dump. */
public final ExcludedRefs excludedRefs;

/** Time from the request to watch the reference until the GC was triggered. */
public final long watchDurationMs;
public final long gcDurationMs;
public final long heapDumpDurationMs;
```

#### ExcludedRefs

Prevents specific references from being taken into account when computing the shortest strong reference path from a suspected leaking instance to the GC roots.

Returns an object of BuilderWithParams class.
```java
public static Builder builder() {
    return new BuilderWithParams();
}
```
Ignores instance fields, static fields, threads and class.
```java
public BuilderWithParams instanceField(String className, String fieldName)
public BuilderWithParams staticField(String className, String fieldName)
public BuilderWithParams thread(String threadName)
public BuilderWithParams clazz(String className)
```
#### WatchExecutor
A WatchExecutor is in charge of executing a Retryable in the future, and retry later if needed.
```java
public interface WatchExecutor {
  WatchExecutor NONE = new WatchExecutor() {
    @Override public void execute(Retryable retryable) {
    }
  };

  void execute(Retryable retryable);
}
```

#### RefWatcher

Watches references that should become weakly reachable. When RefWatcher detects that a reference might not be weakly reachable when it should, it triggers the HeapDumper.

Watches the provided references and checks if it can be GCed. This method is non blocking, the check is done on the WatchExecutor this RefWatcher has been constructed with.
```java
public void watch(Object watchedReference, String referenceName)
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference)
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime)
```
