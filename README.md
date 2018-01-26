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
It actually was a bit more tricky than expected:

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
