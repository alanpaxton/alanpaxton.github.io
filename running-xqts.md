# Instructions for running XQTS

### Check out my fork of XQTS
```
$ git clone https://github.com/alanpaxton/exist-xqts-runner
```

I just wanted to add some information for you about testing. We have been asked to provide before and after results of the relevant XQTS (XQuery Test Suite) tests for each new feature PR that we open against eXist-db.

You can find the XQTS code here - https://github.com/eXist-db/exist-xqts-runner/

It has to be compiled and run against a specific version of eXist-db, which is controlled by this line here - https://github.com/eXist-db/exist-xqts-runner/blob/main/build.sbt#L41

Running the XQTS against eXist-db 6.0.1 will give you results for before your change.

You then have to `mvn install -DskipTests` your eXist-db branch to get a `6.1.0-SNAPSHOT` version locally in your `~/.m2/repository` , with that you can then test by changing the `build.sbt` to val `existV = "6.1.0-SNAPSHOT"`, and then rebuilding and running the XQTS again to get the after results.

With the XQTS you only need to run the tests that are relevant to your change. So for example, if you were implementing `fn:path` something like the following would be used to run just those tests with XQTS CLI:
`target/scala-2.13/exist-xqts-runner-assembly-1.1.0-SNAPSHOT.jar --xqts-version HEAD --test-set fn-path`

You can find the full set of tests used by XQTS [here](https://github.com/w3c/qt3tests/) . For example for `fn:path` the tests are detailed here - https://github.com/w3c/qt3tests/blob/master/fn/path.xml and we see at the top of that file the name of the `--test-set` value that is passed to exist-xqts-runner:

`<test-set xmlns="http://www.w3.org/2010/09/qt-fots-catalog" name="fn-path" ...`
Results from the exist-xqts-runner should appear in its sub-folder `target/junit` - there is a HTML report for easy consumption in your web-browser in `html/index.html`
