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

- Get GPG4Win working
    - Clean up PATH s.t. that installer can be executed successfully
    - See .appveyor/setup.ps1
