This page describes the steps required to publish a new final release of the Apache Shiro plugin for Stormpath.

All of Stormpath's JVM-based open-source projects are released to Maven Central according to the [Sonatype OSS Maven Repository Usage Guide](https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide).

### Prerequisites
- JDK 6 is installed, `$JAVA_HOME` is set, and `$JAVA_HOME/bin` is added to your `$PATH`
- Git 1.8+ is installed (e.g. `brew install git`) 
- Maven 3.0.4+ is installed and `mvn` is available in your `$PATH`
- The `gpg2` encryption client is installed and in your path (e.g. `brew install gpg2`).
- You have [created your GPG keys](https://docs.sonatype.org/display/Repository/How+To+Generate+PGP+Signatures+With+Maven#HowToGeneratePGPSignaturesWithMaven-GenerateaKeyPair) and [distributed your public key](https://docs.sonatype.org/display/Repository/How+To+Generate+PGP+Signatures+With+Maven#HowToGeneratePGPSignaturesWithMaven-DistributeYourPublicKey) (note that if you used `brew install gpg2` as recommended, the executable for these commands is `gpg2` and _not_ `gpg`).
- You have [created a Sonatype OSS issues account](https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide#SonatypeOSSMavenRepositoryUsageGuide-2.Signup) and that account has been granted release permissions to the Sonatype OSS repository for Stormpath.  If you do not yet have permissions and you believe you should, [open a GitHub Issue and request it](https://github.com/stormpath/stormpath-shiro/issues) and we'll work with Sonatype to grant you those permissions if you should have them.
- You have configured `$HOME/.m2/settings.xml` and added the following profiles:
```xml
<profile>
    <id>stormpath-signature</id>
    <properties>
        <gpg.executable>gpg2</gpg.executable>
        <gpg.keyname>YOUR_GPG_KEY_NAME</gpg.keyname>
        <gpg.passphrase>YOUR_GPG_PASSPHRASE</gpg.passphrase>
    </properties>
</profile>
<profile>
    <id>sonatype-oss-release</id>
    <properties>
        <gpg.executable>gpg2</gpg.executable>
        <gpg.keyname>YOUR_GPG_KEY_NAME</gpg.keyname>
        <gpg.passphrase>YOUR_GPG_PASSPHRASE</gpg.passphrase>
    </properties>
</profile>
```
where `YOUR_GPG_KEY_NAME` is the key name you gave the key when you created it and `YOUR_GPG_PASSPHRASE` is the passphrase you used when you created the key.

    NOTE: adding these lines is a security risk if your `$HOME/.m2/settings.xml` file is readable by anyone other than you! Ensure you've protected that file, e.g.
```bash
chmod go-rwx $HOME/.m2/settings.xml
```
    Again, the `gpg.executable` name of `gpg2` assumes you used `brew install gpg2`.
- You have enabled both of these profiles under the `<activeProfiles>` element:
```xml
<activeProfiles>
    <activeProfile>sonatype-oss-release</activeProfile>
    <activeProfile>stormpath-signature</activeProfile>
    <!-- any others: -->
</activeProfiles>
```

## Pre-release Verification

1. Ensure all changes for the version being released have been represented in the project's top-level `Readme.md` file's Change Log in a section named after the version (e.g. *0.7.1*), listing the changes associated with the release.  No release should be published without appropriate change log entries.

    The new section should be written in a feature branch and then those changes should be merged into the `master` branch right before you perform the release.

2. Ensure the [wiki documentation](https://github.com/stormpath/stormpath-shiro/wiki) has been updated to document all new features and/or fixes.

3. Ensure all changes that are to be released (including change log edits) have been committed to the `master` branch and that the [Travis CI project status](https://travis-ci.org/stormpath/stormpath-shiro) representing those changes reports no errors whatsoever.

## Perform the Release

1. After the above prerequisites have been satisfied and you have performed the pre-release verification, run the following on the command line:
    ```bash
    git checkout master
    # ensure git status reports no changes:
    git status
    # assuming no reported changes:
    mvn clean
    mvn release:clean
    # assuming no build errors:
    mvn release:prepare
    ```
    
    Enter the release version and the SCM release tag / label.  Often retaining the defaults and hitting enter is a viable option *IF* you know for sure you do not need to change them: 

    If you're introducing a publicly-facing API change that end-users will invoke and compile code against, you _must_ increment the minor version (e.g. 0.4 -> 0.5 or 1.3 -> 1.4).

    If you're introducing an internal change that end-users *cannot* compile against (e.g. introducing a private class or method, or changing an existing method implementation with no signature changes) you must increment the point revision (e.g. 0.4.0 -> 0.4.1 or 1.3.1 -> 1.3.2).

    If in doubt, ask Les.

    When asked for the next development version, you can specify the next minor version and not the point revision change.  It will prompt you for a point revision change, e.g. `0.5.1-SNAPSHOT`.  Change this to the next minor version snapshot, e.g. `0.6.0-SNAPSHOT`.

    Ok, continuing on.  

2. Assuming the previous build commands did not result in errors:
    ```bash
    mvn release:perform
    ```
    This will build the final versioned artifacts and upload them to the Sonatype OSS repository server in a _staging_ repository.  This is called a _[staged release](https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide#SonatypeOSSMavenRepositoryUsageGuide-7a.3.StageaRelease)_ - it is not yet available to the world.

    A staged release is fully 'done' as far as the build process is concerned, but the staged artifacts will not be released to the world in Maven Central until we log in to the repository user interface and manually execute this behavior.

    This is a really nice safety net: if there is any error at all the final versioned artifacts do not 'leak' into Maven Central, where they could be used by end-users.  Only after you've ensured the release was 'clean' and that the artifacts were built as expected, and all is well, then we release the artifacts to the world.

3.  Release the artifacts to the world by [using the Sonatype Nexus UI](https://docs.sonatype.org/display/Repository/Sonatype+OSS+Maven+Repository+Usage+Guide#SonatypeOSSMavenRepositoryUsageGuide-8a.ReleaseIt)

    In practice this really means the following:

    1. Visit [https://oss.sonatype.org](https://oss.sonatype.org/index.html) and login (upper right) using the account you created as a prerequisite.
    2. Access the 'Staging Repositories' menu item on the left
    3. Find the `comstormpath-###` entry - usually at the bottom - and check it.
    4. After checking the entry, at the top of the list, you will see a `Close` icon enabled.  Click `Close` and type in a reason (e.g. 'closing for the 0.5.0 release') and then click the `Confirm` button.  'Closing' a repository prevents other artifacts from being added to it.  We've never needed to do that, so closing it immediately is fine.
    5. Wait a little bit, and click the `Refresh` button at the top of the list.  The `comstormpath-###` entry should still be checked.
    6. Click the `Release` button and type in a message, e.g 'releasing 0.5.0'.  Ensure the `Automatically Drop` option *is* checked and then click the `Confirm` button.

    After you `Confirm` the `Release`, that's it - the artifacts will be propagated to Maven central.  The sync time usually takes about 2 to 3 hours to make the artifacts available to the world.

## Post release

- Email the Stormpath marketing team about the release.  The email should contain the following:
    1. A link to the specific section in the specific _tagged release_'s `Readme.md` file, e.g. [https://github.com/stormpath/stormpath-shiro/tree/stormpath-shiro-root-0.5.0#050](https://github.com/stormpath/stormpath-shiro/tree/stormpath-shiro-root-0.5.0#050).  Notice that this URL points to a specific section of a specific file in a tagged release.  This guarantees that this link will always work because the tag will never be changed/removed - we keep tags permanent.  If you were to link to the `HEAD` version of the doc, it is possible that the link would break in the future.  This way, we ensure that whatever link marketing uses to notify Stormpath end-users will always work in the future (e.g. in blog articles, etc).
    2. A link to the previously-verified project documentation, e.g. [https://github.com/stormpath/stormpath-shiro/wiki](https://github.com/stormpath/stormpath-shiro/wiki).
    3. A link to the GitHub milestone containing all issues that were resolved for the release.

Example email template:

    Subject: Apache Shiro plugin for Stormpath 0.5.0 Released!
    Body:
    
    Hi everyone,
    
    I am pleased to announce that the version 0.5.0 of the Apache Shiro plugin for Stormpath has been released!

    This is a bugfix point release that resolves 1 issue: https://github.com/stormpath/stormpath-shiro/issues?milestone=1    
    Project documentation is here: https://github.com/stormpath/stormpath-shiro/wiki
    Change log for 0.5.0 is here: https://github.com/stormpath/stormpath-shiro/tree/stormpath-shiro-root-0.5.0#050
    
    Please allow 3 to 4 hours for the release artifacts to appear in Maven Central before announcing the release to the Stormpath user community.
    
    Cheers,
    
    Your Name

- Close the [milestone](https://github.com/stormpath/stormpath-shiro/issues/milestones) that represents the just-released version.

- Add a new [milestone](https://github.com/stormpath/stormpath-shiro/issues/milestones) that represents the next version to be released.  Newly created issues can then have easy access to a milestone.
    
    
    