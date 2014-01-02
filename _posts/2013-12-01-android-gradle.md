# A day with Android and Gradle

Update 2014-Jan-02: Some corrections after feedback from @Vampire.

Once in a while I have an issue with my Android phone which is painful enough
that I decide to fix it. Since I am not a regular Android developer, every time
I do this I find that I have lots of new stuff to learn. This time I get to
learn about Android, Gradle, and Groovy.

* `git clone https://github.com/dschuermann/offline-calendar.git`. I am
  using offline-calendar as an example. Any recent Android project using gradle
  would behave the same.
* `cd offline-calendar`
* In the November 30 version (commit e7d1324b) the `README.md` tells me to
  install gradle. For me on Arch Linux this amounts to `aurget gradle; cd
  gradle; makepkg -s -i`, which installs gradle 1.9.
* `gradle wrapper` or `gradle build` or just `gradle`. It does not really
  matter which task we run, since it will bail out before it gets to that part:

<pre>

FAILURE: Build failed with an exception.

* Where:
Build file '/home/drizzd/src/offline-calendar/Offline-Calendar/build.gradle' line: 1

* What went wrong:
A problem occurred evaluating project ':Offline-Calendar'.
> Gradle version 1.8 is required. Current version is 1.9. If using the gradle wrapper, try editing the distributionUrl in /home/drizzd/src/offline-calendar/gradle/wrapper/gradle-wrapper.properties to gradle-1.8-all.zip

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED

Total time: 12.569 secs
</pre>

Long story short, the android plugin selected in build.gradle checks that the
gradle version is exactly 1.8 and refuses to function otherwise. If you
actually want to build offline-calender successfully, downgrade to gradle 1.8
or use `./gradlew` from the [latest git](https://github.com/dschuermann/offline-calendar.git)
version.

That fixes it, but we are as clueless as before. Why does this happen? From the
message it looks as if gradle refuses to run its own version.

## What's going on?

Passing `--stacktrace` to gradle gives us (cut to the interesting parts):

<pre>
* Exception is:
org.gradle.api.GradleScriptException: A problem occurred evaluating project ':Offline-Calendar'.
	at org.gradle.groovy.scripts.internal.DefaultScriptRunnerFactory$ScriptRunnerImpl.run(DefaultScriptRunnerFactory.java:54)
[... many more ...]
Caused by: org.gradle.tooling.BuildException: Gradle version 1.8 is required. Current version is 1.9. If using the gradle wrapper, try editing the distributionUrl in /home/drizzd/src/offline-calendar/gradle/wrapper/gradle-wrapper.properties to gradle-1.8-all.zip
	at com.android.build.gradle.BasePlugin.checkGradleVersion(BasePlugin.groovy:246)
	at com.android.build.gradle.BasePlugin.apply(BasePlugin.groovy:184)
	at com.android.build.gradle.AppPlugin.super$2$apply(AppPlugin.groovy)
	at com.android.build.gradle.AppPlugin.apply(AppPlugin.groovy:80)
	at com.android.build.gradle.AppPlugin.apply(AppPlugin.groovy)
	at org.gradle.api.internal.plugins.DefaultPluginContainer.providePlugin(DefaultPluginContainer.java:104)
[...]
</pre>

This was not obvious to me, but @Vampire tells me on
[#gradle](irc://chat.freenode.net/gradle) that `BasePlugin.groovy:246`
is where things go awry. But where does `com.android.build.gradle.BasePlugin`
come from? It is not part of gradle, it is not part of offline-calendar, and it
is not part of anything else installed on the system either. And why do we
decide to run it in the first place?

Here is what gradle is does (not necessarily in exactly this order due to lazy
evaluation):

* Process `settings.gradle`. The command `include ':Offline-Calendar'`
  registers `/Offline-Calendar/build.gradle` in addition to `/build.gradle`.
* Enter
  [build.gradle](https://github.com/dschuermann/offline-calendar/blob/e7d1324bf5d1760089f5ad0497f242a5f29074c0/build.gradle).
* Enter `buildscript { ... }`.
* `repositories { mavenCentral() }` registers Maven central repository as a
  source to look for external dependencies.
* `dependencies { classpath 'com.android.tools.build:gradle:0.6.+' }` adds the
  specified [artifact](http://stackoverflow.com/questions/2487485/what-is-maven-artifact)
  to the class path of the gradle build script (not to the application we are
  trying to build). The syntax is explained in
  [8.4 External dependencies](http://www.gradle.org/docs/current/userguide/artifact_dependencies_tutorial.html#N105C3).
  I could not find documentation on the plus sign, but @Vampire tells me that
  "0.6.+ means anything starting with 0.6. and thereof the newest one."
  Update: See also [50.7 How dependency resolution works](http://www.gradle.org/docs/current/userguide/userguide_single.html#sec:dependency_resolution).
* Download `gradle-0.6.3.jar` from [Maven central](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.android.tools.build%22%20AND%20a%3A%22gradle%22).
  I wonder if those signatures are actually verified.
* Exit `build.gradle`.
* Enter `Offline-Calendar/build.gradle`.
* The command `apply plugin: 'android'` eventually gets to gradle's
  [DefaultPluginRegistry](https://github.com/gradle/gradle/blob/REL_1.8/subprojects/core/src/main/groovy/org/gradle/api/internal/plugins/DefaultPluginRegistry.java#L80),
  which reads `implementation-class=com.android.build.gradle.AppPlugin` from
  the previously downloaded
  `gradle-0.6.3.jar:META-INF/gradle-plugins/android.properties`. Interesting...

* This in turn leads to
  `gradle-0.6.3.jar:com/android/build/gradle/AppPlugin.groovy`. It is
  derived from `BasePlugin.groovy`, where we eventually run this code:

<pre>
public static final String GRADLE_MIN_VERSION = "1.8";

private void checkGradleVersion() {
    [...]
    throw new BuildException(
        String.format(
            "Gradle version %s is required. Current version is %s. " +
            "If using the gradle wrapper, try editing the distributionUrl in %s " +
            "to gradle-%s-all.zip",
            GRADLE_MIN_VERSION, project.getGradle().gradleVersion, file.getAbsolutePath(),
            GRADLE_MIN_VERSION), null);
}
</pre>

I extracted this snippet from
[gradle-0.6.3-source.jar](http://search.maven.org/remotecontent?filepath=com/android/tools/build/gradle/0.6.3/gradle-0.6.3-sources.jar). Do not let `GRADLE_MIN_VERSION` fool you. It is the only accepted version.

The latest git version of
[com.android.tools.build](git://android.googlesource.com/platform/tools/build.git)
already sets `GRADLE_MIN_VERSION` to 1.19. Now I wonder what I would have to
do to actually use it. I suppose I would have to compile it locally and point
the buildscript repositories and dependencies clauses at it.

drizzd, 2013-Dec-01
