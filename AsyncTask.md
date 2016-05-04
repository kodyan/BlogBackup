title: 不如自己读一遍AsyncTask源码
date: 2016/4/28 11:45:37
categories: 
tags: Android
---

>看多少总结都不如自己试着分析一遍，RTFSC。

<!--more-->

## AsyncTask用法
直接上[官网的例子](https://developer.android.com/training/articles/perf-anr.html):

```java
private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
    // Do the long-running work in here
    protected Long doInBackground(URL... urls) {
        int count = urls.length;
        long totalSize = 0;
        for (int i = 0; i < count; i++) {
            totalSize += Downloader.downloadFile(urls[i]);
            publishProgress((int) ((i / (float) count) * 100));
            // Escape early if cancel() is called
            if (isCancelled()) break;
        }
        return totalSize;
    }

    // This is called each time you call publishProgress()
    protected void onProgressUpdate(Integer... progress) {
        setProgressPercent(progress[0]);
    }

    // This is called when doInBackground() is finished
    protected void onPostExecute(Long result) {
        showNotification("Downloaded " + result + " bytes");
    }
}
```
写一个类继承AsyncTask，重写三个方法：

- doInBackground：在这里面执行耗时的异步任务
- onProgressUpdate：更新进度在这个方法里写
- onPostExecute：任务执行后的操作放这里面

此外还有两个方法可以重写：
- onPreExecute
- onCancelled

AsyncTask类指定三个泛型参数，这三个参数的用途如下：

- Params——在执行AsyncTask时需要传入的参数，可用于在后台任务中使用。
- Progress——后台任务执行时，如果需要在界面上显示当前的进度，则使用这里指定的泛型作为进度单位。
- Result——当任务执行完毕后，如果需要对结果进行返回，则使用这里指定的泛型作为返回值类型。

启动这个任务：
```java
new DownloadFilesTask().execute(url1, url2, url3);
```

## 源码分析

我的SDK版本时API 23，各版本的AsyncTask的具体代码可能会有出入，但基本的原理是一样的。

先看看构造器里都做了什么：

```java
 /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

首先用匿名内部类的形式new了一个WorkerRunnable\<Params, Result>实例，WorkerRunnable是实现了Callable接口的一个抽象类，Callable代码如下，包含一个带反回类型的call()方法：

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```
回构造器看看匿名内部类里是如何实现call()方法的，在call()方法中将线程的优先级设置为了`THREAD_PRIORITY_BACKGROUND`，官网说了：

>If you don't set the thread to a lower priority this way, then the thread could still slow down your app because it operates at the same priority as the UI thread by default.

意思是如果不降低线程的优先级，那么子线程默认和UI线程优先级一样，执行起来app可能就卡了。

接下来就调用了doInBackground方法，返回一个result，这都是在子线程里的操作，那怎么把result传递给主线程呢？追进postResult看看：

```java
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
熟悉的Message出现了，还调用了sendToTarget()方法，这不就是通过Hadler传递的方式吗。再看getHandler方法返回的是静态内部类InternalHandler的实例，InternalHandler继承自Handler：

```java
private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
InternalHandler构造器中拿的是主线程的looper，handleMessage是接收到不同消息进行相应的处理，finish：

```java
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
接收到finish消息后，如果调用了isCancelled就回调onCancelled方法，否则回调onPostExecute方法。如果消息的更新progress那么就回调onProgressUpdate方法。这样就把result传递给了主线程。

再回到构造器接着往下看：

```java
mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
```
同样用匿名内部类生成一个FutureTask\<Result>实例，FutureTask实现了RunnableFuture接口，RunnableFuture的源码如下，它同时继承了Runnable接口和Future接口，既可以作为Runnable的引用也可以作为Future的引用，Future接口中定义cancel方法和get方法用于取出线程的执行结果。

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```
前面的FutureTask匿名内部类重写了done方法，调用postResultIfNotInvoked，如果之前WorkerRunnable的call方法抛异常了，则再次把result传递出去。

好了，构造器里定义好了一个实现Callable接口的WorkerRunnable实例和一个实现了Runnable接口和Future接口的FutureTask实例。

接下来看看调用execute方法后都做了什么：

```java
@MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
再追进executeOnExecutor方法：

```java
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
首先回调onPreExecute()方法，这还是在主线程，接下来调用传递进来的Executor实例的execute方法开始执行任务：
```java
exec.execute(mFuture);
```

传进来的sDefaultExecutor是定义的静态内部类SerialExecutor的实例，去源码看到它实现的execute方法中使用一个双端队列来串行执行传进来的Runnable引用的run方法，具体引用是谁呢？看上面，是mFuture。在mFuture的run方法中会调用Callable引用的call方法，具体引用是谁呢？回AsyncTask构造器看，是mWorker。真正执行任务的是线程池THREAD\_POOL_EXECUTOR。

### AsyncTask默认线程池参数
把THREAD\_POOL_EXECUTOR具体参数和定义也贴上：

```java
 private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
                    
public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```
可以看到线程池的核心线程数是（CPU数＋1），最大容量是（2*CPU数＋1），阻塞队列容量128。记得AsyncTask中的线程池核心线程数是5啊，对吗？什么时候改的呢，这是API 23即Android 6.0，我们去历史版本看看吧，我一般在[androidxref.com](http://androidxref.com/)看各个版本的源码。

搜索之后可以得知，在Android 4.3的源码中AsyncTask的配置还是如下所示：

```java
private static final int CORE_POOL_SIZE = 5;
    private static final int MAXIMUM_POOL_SIZE = 128;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);

```
核心线程数5，线程池最大容量128，阻塞队列容量10。从Android 4.4开始就变成上面看到的根据CPU数的配置了，涨了一点点知识，不能再说AsyncTask默认线程池核心线程数是5啦。

此外，我们还可以通过setDefaultExecutor来使用自定义的Executor和线程池来执行任务。

以上分析下来确实学到了不少啊。




