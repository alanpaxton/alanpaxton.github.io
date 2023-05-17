# Debug eXistDB in-situ

### Check out eXistDB
* fork the repo if you haven't already
* git clone 
* `./quick-build.sh`

```
set -e

mvn -V -T2C clean package -DskipTests -Ddependency-check.skip=true -Ddocker=false -P \!mac-dmg-on-mac,\!codesign-mac-dmg,\!mac-dmg-on-unix,\!installer,\!concurrency-stress-tests,\!micro-benchmarks,\!build-dist-archives
```
#### Install the resulting installation jar.
For `7.0.0` it will be something like this
```
java -jar ./exist-installer/target/exist-installer-7.0.0-SNAPSHOT.jar
```
* the installer will prompt for a location, which you can change
* I have created `~/exist-installations` directory, and for V7.0 snapshot a subdirectory in `~/exist-installations/v70`

### Make the installation debuggable
* Open the installation directory in VSCode
```
code ~/exist-installations/v70
```
* Edit ./bin/luancher.sh - add the line
```
-Dexist.debug.launcher=true \
```
to the `exec` call at the end, so that it waits for a debugger
* Run it, from a shell
```
./bin/launcher.sh
```
It should announce that it is waiting for a debugger

### Open the project in IntelliJ
* That's the git directory, this time
* You can probably do it in VSCode, too
* Import the maven project if you need to
* Run-->"Attach to process"
The waiting launcher should now start up
* You can access the dashboard at `http://localhost:8080/
* From there you can access _eXide_ and save files, run _xQuery_ etc

### Reload an updated eXist core
This process doesn't quite allow hot reload (to be investigated) but when you need to make changes
* Quit the running _eXist_ via the X menu in the toolbar
* Make your changes in the IntelliJ editor (if you haven't already)
* Rebuild using `./quick-build.sh`
* Copy the new jar in
    * `cp ./exist-core/target/exist-core-7.0.0-SNAPSHOT.jar ~/exist-installations/v70/lib`
* Start again
    * `./bin/launcher.sh`
* Re-attach the debugger

