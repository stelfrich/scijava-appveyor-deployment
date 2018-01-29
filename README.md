# Multi-platform deployment with Travis CI and AppVeyor

Some of you might have noticed our roll-out of Travis CI for projects in the SciJava / ImageJ ecosystem. In parallel, @ctrueden and @stelfrich have worked on providing Windows-based CI infrastructure as well. The most prominent beneficiary of this change is the imagej-launcher project, which wasn't properly update for about a year due to the failing of Windows build worker.

## Deploy multi-platform SNAPSHOT artifacts to Nexus
Do not use `maven-exists-plugin` but let the `maven-deploy-plugin` handle everything for you. The reason why it's that easy are unique suffixes added to uploaded artifacts. That means, that two builds triggered by different CI systems (in our case Travis CI and AppVeyor) will deploy all their artifacts with unique suffixes.

[ Insert screenshot ]

In Nexus, however, those unique artifacts will be gathered under one GAV (GroupId, ArtifactId, Version).

- Does a complete deploy due to the unique suffices (no errors when trying to upload an already existing artifact)

## Deploy multi-platform, non-SNAPSHOT artifacts (ie. releases) to Nexus

- `deploy:deploy-file` as additional step
- POM has to contain `maven-exists-plugin` s.t. a "regular deploy" that contains all attached artifacts is skipped
- Attached artifacts are all `*-sources.jar`, `*-tests.jar`, `*-executable.nar`, `*-noarch.nar`, etc
- Maybe your permission system is prohibiting uploads from specific users to a release repository?

## Deploy multi-platform, non-SNAPSHOT artifacts (ie. releases) to OSS Sonatype

- Artifacts have to be signed
- Staging repository ID has to be synced between Travis CI and AppVeyor

## Signing artifacts

### Cleaning up PATH for GPG4Win installation
After downloading the installer I have tried to install GPG4Win several times before almost quitting on the subject. The issue was really mysterious, until I logged into the AppVeyor build worker via RDP to be greeted by a "PATH too long" popup. Figuring out which parts of the original `PATH` had to remain was actually a bit more tricky than expected.

As of writing of this post, this is what the default `PATH` looks like (if you are reading this post, it might be worth your time to connect to a worker via RDP (https://www.appveyor.com/docs/how-to/rdp-to-build-worker/) and confirm this before cleaning up):

```
PS C:\Users\appveyor> $Env:Path
C:\Perl\site\bin;C:\Perl\bin;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Program Files\7-Zip;C:\Program Files\Microsoft\Web Platform Installer\;C:\Tools\GitVersion;C:\Tools\PsTools;C:\Program Files\Git LFS;C:\Program Files (x86)\Subversion\bin;C:\Program Files\Microsoft SQL Server\120\Tools\Binn\;C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\110\Tools\Binn\;C:\Program Files (x86)\Microsoft SQL Server\120\Tools\Binn\;C:\Program Files\Microsoft SQL Server\120\DTS\Binn\;C:\Program Files (x86)\Microsoft SQL Server\120\Tools\Binn\ManagementStudio\;C:\Tools\WebDriver;C:\Program Files (x86)\Microsoft SDKs\TypeScript\1.4\;C:\Program Files (x86)\Microsoft Visual Studio 12.0\Common7\IDE\PrivateAssemblies\;C:\Program Files (x86)\Microsoft SDKs\Azure\CLI\wbin;C:\Ruby193\bin;C:\Tools\NUnit\bin;C:\Tools\xUnit;C:\Tools\MSpec;C:\Tools\Coverity\bin;C:\Program Files (x86)\CMake\bin;C:\go\bin;C:\Program Files\Java\jdk1.8.0\bin;C:\Python27;C:\Program Files\nodejs;C:\Program Files (x86)\iojs;C:\Program Files\iojs;C:\Users\appveyor\AppData\Roaming\npm;C:\Program Files\Microsoft SQL Server\130\Tools\Binn\;C:\Program Files (x86)\MSBuild\14.0\Bin;C:\Tools\NuGet;C:\Program Files (x86)\Microsoft Visual Studio 14.0\Common7\IDE\CommonExtensions\Microsoft\TestWindow;C:\Program Files\Microsoft DNX\Dnvm;C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\130\Tools\Binn\;C:\Program Files (x86)\Microsoft SQL Server\130\Tools\Binn\;C:\Program Files (x86)\Microsoft SQL Server\130\DTS\Binn\;C:\Program Files\Microsoft SQL Server\130\DTS\Binn\;C:\Program Files (x86)\Microsoft SQL Server\110\DTS\Binn\;C:\Program Files (x86)\Microsoft SQL Server\120\DTS\Binn\;C:\Program Files (x86)\Apache\Maven\bin;C:\Python27\Scripts;C:\Tools\NUnit3;C:\Program Files\Mercurial\;C:\Program Files\LLVM\bin;C:\Program Files\dotnet\;C:\Tools\curl\bin;C:\Program Files\Amazon\AWSCLI\;C:\Program Files (x86)\Microsoft SQL Server\140\DTS\Binn\;C:\Program Files (x86)\Microsoft Visual Studio 14.0\Common7\IDE\Extensions\Microsoft\SQLDB\DAC\140;C:\Program Files\Git\cmd;C:\Program Files\Git\usr\bin;C:\Tools\vcpkg;C:\Program Files\Microsoft Service Fabric\bin\Fabric\Fabric.Code;C:\Program Files\Microsoft SDKs\Service Fabric\Tools\ServiceFabricLocalClusterManager;C:\Program Files (x86)\Yarn\bin;C:\Program Files (x86)\Microsoft SQL Server\140\Tools\Binn\;C:\Program Files\Microsoft SQL Server\140\Tools\Binn\;C:\Program Files\Microsoft SQL Server\140\DTS\Binn\;C:\Program Files (x86)\nodejs\;C:\ProgramData\chocolatey\bin;C:\Program Files\PowerShell\6.0.0\;C:\Program Files\erl9.2\bin;C:\Users\appveyor\AppData\Local\Yarn\bin;C:\Users\appveyor\AppData\Roaming\npm
```

From my experience, the AppVeyor build agent (`C:\Program Files\AppVeyor\BuildAgent\`) has to be on the PATH for the whole build orchestration to work. Since we are trying to build a project that contains both native code as well as Java code, we are keeping Java and MINGW in the mix. In addition to that, we are using Maven for the compilation of the project, which will in turn use GPG for signing the artefacts before uploading them to Nexus:

```powershell
# Clean up the machine-wide Path environment variable to a minimum (to enable GPG4Win installation)
[Environment]::SetEnvironmentVariable("Path", "C:\Program Files\AppVeyor\BuildAgent\", "Machine");

# Set up this process' Path environment variable
$Env:Path = "$Env:GIT_BIN;$Env:MAVEN_BIN;${Env:JAVA_HOME}bin;$Env:GNUPG_HOME;$Env:MINGW64_BIN;$Env:MINGW32_BIN;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;"

# Set up the machine-wide Path environment variable
[Environment]::SetEnvironmentVariable("Path", $Env:Path, "Machine");
```

See [.appveyor/setup.ps1](https://github.com/imagej/imagej-launcher/blob/master/.appveyor/setup.ps1) for more context.

### Installing GPG4Win
Download installer if it's not cached locally and start silent installation.

Since the `Invoke-Expression` returns directly without waiting for the installation to finish

```powershell
Invoke-Expression ("" + $setupFilename + " /S /D=" + $installFolder)
```

we have added a **very** simple polling mechanism that checks for the existance of the `gpg2` binary:

```powershell
$pollCounter = 0;
# Wait for gpg2.exe to exists
while (!(Test-Path ("" + $installFolder + "\gpg2.exe")) -and ($pollCounter -le 10)) {
    Start-Sleep 5
    $pollCounter++
    Write-Output ("Waiting for installer to finish (#" + $pollCounter + ")")
}
```

See [.appveyor/install-gpg4win.ps1](https://github.com/imagej/imagej-launcher/blob/master/.appveyor/install-gpg4win.ps1) for more context.
