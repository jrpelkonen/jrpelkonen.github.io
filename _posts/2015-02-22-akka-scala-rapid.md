---
layout: post
title: Rapid Development Cycle with Scala, Akka, and sbt
tags: [Scala, Akka, sbt]
---
Performance is a feature. Studies have proven than today's fickle consumers lose interest at a rate which can be measured in hundreds of milliseconds.
While attention span of a motivated developer is considerably longer than that, it is equally important than all unnecessary delays and activities will be removed from the development process. This allows the developer to stay focused due to the instant feedback.

I have been recently working with Akka and sbt. One of the nice features of sbt is the [triggered execution](http://www.scala-sbt.org/0.13/docs/Triggered-Execution.html) feature that can be used
to automatically compile and run the application when the source code has changed. However, this feature is not very useful with Akka applications, as they will stay active 
until explicitly shutdown. For these situations it would be nice to have a mechanism that would shut down the actor system when the source has changed. This would allow sbt triggered execution to automatically recompile and rerun the application. In this post I present one such solution for the problem. 

Java 7 introduced a class called java.nio.file.WatchService, which makes it relatively straightforward to monitor changes in a directory. The watchservice monitors directories that have been registered and returns events when changes occurs. Unfortunately, the Java API is not very natural fit for a well designed Scala application and it is rather verbose, but in the end, it does its job. The watch service can be then used to create a higher order function that executes a function when a filesystem event occurs. I called the function onFileSystemChange, and it is shown below: 

{% highlight scala %}
object onFileSystemChange extends ((Path, () => Unit) => Unit) {
  import scala.collection.JavaConversions._
  def apply(root: Path, onChange: () => Unit) = {
    val watcher = root.getFileSystem.newWatchService

    // registers the directory and its subdirectories recursively to
    // watch filesystem events
    def register(dir: Path): Map[WatchKey, Path] = {
      Map(dir.register(watcher,
        ENTRY_CREATE,
        ENTRY_DELETE,
        ENTRY_MODIFY) -> dir) ++ {
        for {
          d <- Files.newDirectoryStream(dir)
          if Files.isDirectory(d)
          subDirMap <- register(d)
        } yield subDirMap
      }
    }

    // needs to be var as creating a new 
    // directory will update the map
    var paths = register(root)

    // waits for filesystem events
    @tailrec
    def wait(): Unit = {
      val key = watcher.take()
      paths.get(key) match {
        case None => wait() // got event for something else
        case Some(dir) => {
          for {
            event <- key.pollEvents
            // ignore overflow
            if event.kind != OVERFLOW
          } {
            // if a new directory has been created, register it to
            // be part of watch service monitoring
            if (event.kind == ENTRY_CREATE) {
              val pathEvt = event.asInstanceOf[WatchEvent[Path]]
              val child = dir.resolve(pathEvt.context())
              if (Files.isDirectory(child, NOFOLLOW_LINKS)) {
                paths = paths ++ register(child)
              }
            }
            // run the user supplied handler
            onChange()
          }
          key.reset
        }
      }
    }
    wait()
  }
}
{% endhighlight %}

From this point on, the heavy lifting is done, now the application just needs to supply the root of the source directory and a function that shuts down the actor system on the any change in the source directory.

The standard sbt directory layout places the compiled the classes under the target directory, and the sources under the src directory. Using the tried and true Java classloader methods it is possible to get ahold of the root of the classpath under the target. From that point on, one just needs to walk up the directory hiearchy until a directory containing "src" is encountered. And the required function to shutdown Akka is trivial. In terms of Scala code, the calling application code look like this:

{% highlight scala %}
 def parents(dir: Path): Stream[Path] = dir match {
    case null => Stream.Empty
    case _ => Stream.cons(dir, parents(dir.getParent))
  }

  private val path: String = getClass.getResource(".").getPath

  for {
    parentDir <- parents(Paths.get(path).toRealPath())
    if Files.exists(parentDir.resolve("src"))
  } {
    onFileSystemChange(parentDir.resolve("src"), () => {
      logger.info("source changes, shutting down")
      system.shutdown()
      system.awaitTermination(10 seconds)
    })
  }
{% endhighlight %}

A complete demo is available as a [gist](https://gist.github.com/jrpelkonen/8baac9d89f4172c14472)
