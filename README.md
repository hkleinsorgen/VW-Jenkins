# Building images with Jenkins CI

You need

*   Jenkins ([Jenkins project page](http://www.jenkins-ci.org))
*   Apache Tomcat (or any other servlet engine)
*   The Jenkins automation parcels / packages  (JenkinsAutomation, JenkinsImage, JenkinsUnitTester, JenkinsRuntimePackager), either from Github or from the public store repository
*   A Store repository

This instruction assumes that Jenkins is installed on a machine running Windows. It should be possible to adapt it to other OS, but I have not done that yet.

## Install Jenkins

Download and install Jenkins.war as usual. You might want to set HUDSON_HOME in web.xml to specify the configuration/build directory used by Jenkins. After installation, open the Jenkins page and go to `Manage Jenkins > Configure System`. Add three global properties:

*   `VisualWorksConfigurationDirectory` - A directory containing configuration files. Export your Store respositories into this directory (as respositories.xml)
*   `VisualWorksRepository` - The name of the profile of the Store repository that contains your packages.
*   `VisualWorksStoreParcel` - The name of the parcel of the Store connection (e.g. "StoreForPostgreSQL").

You might want to define additional variables, for example

*   `VisualWorksHome` - The VisualWorks home directory

## Create a job

For each image, create a new job as `free-style software project`.

You now need to put some files into the `workspace`. Jenkins does not create a workspace directory automatically, so build the non-configured job once. Jenkins will create a workspace directory `%HUDSON_HOME%\jobs\%JOB_NAME%\workspace`.

Put the following files into the workspace directory:

*   Parcel JenkinsAutomation
*   Parcel JenkinsImage
*   Parcel JenkinsUnitTester (optional)
*   Image and VM (optional)

## Write a batch file/script

You need a batch file (Windows) / shell script (Unix) to build the image. Basically, it should contain the following:

````
{virtualMachine} {image} -headless -pcl JenkinsImage {other parcels} -loadBundles {some bundles} -loadPackages {some packages} -saveAs {targetImage}
````

*   `-loadBundles` specifies the bundles that should be loaded from the Store repository. Each bundle name can contain an optional version pattern: Name(VersionPattern), e.g. AutoComplete(7.9 *). Remember to use quotes when necessary.
*   `-loadPackages` specifies the packages that should be loaded from the Store repository. May contain a version pattern, too
*   `-saveAs` specifies the name of the image (without file extension). It's convenient to use the environment variable `JOB_NAME` of Jenkins.
*   It is probably a good idea to copy the source image to a temporary image, to avoid inflating the changes files of the source image

### Settings

JenkinsImage automatically loads settings of all available settings domains if the workspace directory contains an exported XML file , e.g. `VisualWorksSettings.xml`.

### Unit tests

To run test cases and see the test results in Jenkins, you need to load the package `JenkinsUnitTester`. Simply add it to the list of packages to load (if you have published it in your repository), or load it as parcel via -pcl (it must appear before "JenkinsImage").

JenkinsUnitTester performs all SUnitToo tests that are loaded in the image. The test results will be saved as a JUnit compatible file named {imageName}.xml in the workspace directory.

Another package in the public repository, `CodeCriticTest`, performs code critic (aka Lint) checks as a test case. It will check all 'Bugs' rules.

### Example

````batch
REM clean up files of previous builds
del %JOB_NAME%.*
del *.log
REM create a temporary image
copy %VisualWorksHome%\image\visual.im temp.im
REM build
set parcels=RBCodeHighlighting RBTabbedToolsets JenkinsImage
set bundles=InfoVis 
set packages=JenkinsUnitTester "AutoComplete(7.9 *)" Windows7LookPolicy((7.9).*) Windows7LookPolicyAutoSelect
set vm=%VisualWorksHome%\bin\win\vwntconsole.exe
%vm% temp.im -headless -pcl %parcels% -loadBundles %bundles% -loadPackages %packages% -saveAs %JOB_NAME%
REM clean up temporary stuff
del temp.*
````

## Configure the job

At least the following elements of the job must be configured:

*   `Build` - Add the Windows batch/shell command desribed above as build step. It will be executed in the workspace directory
*   Post build actions - `Archive the artifacts` - specifiy the name of the image and the changes file, separated by a comma
*   Post build actions - `Publish JUnit test result report` - specify the name of the JUnit results file {imageName.xml}

It's useful to add a build trigger - `Build periodically` and to `Discard Old Builds`

You can now start the job to build an image and run all tests - enjoy your fresh, tested images!

## Create a runtime image

`JenkinsRuntimePackager` allows to automate building images with the runtime packager.

*   Create a new job as `free-style software project`
*   Put the parcel JenkinsRuntimePackager and an RTP options file into the workspace directory
*   Define a batch/shell build step. Basically you need to load JenkinsRuntimePackager and specify the RTP file via the command line option `-rtp`.

    ````
    {virtualMachine} {image} -headless -pcl JenkinsRuntimePackager -rtp {optionsFile.rtp}
    ````
    
*   The runtime image can be tested, too, by loading JenkinsUnitTester and specifying a test result filename: -test {resultFilename}
*   Add a post build action - `Archive the artifacts` - specifiy the name of the runtime image
*   Add a post build action - `Publish JUnit test result report` - specify {testResultFilename.xml}

### Example

````batch
REM clean up files of previous builds
del *.log
del *.im
del *.cha
REM copy the development image
set source=..\..\dev-image\workspace\dev-image
copy "%source%.im" temp.im
copy "%source%.cha" temp.cha
REM build
%VisualWorksHome%\bin\win\vwntconsole.exe -noherald temp.im -headless -pcl JenkinsRuntimePackager -rtp seaside-server.rtp
REM clean up temporary stuff
del temp.*
REM test
%VisualWorksHome%\bin\win\vwntconsole.exe -noherald seaside-server.im -headless -pcl JenkinsUnitTester -test seaside-server.xml
````
