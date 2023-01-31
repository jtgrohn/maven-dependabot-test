# maven-dependabot-test

The [PR](https://github.com/dependabot/dependabot-core/pull/6491) broke Maven `pom.xml` updates for dependabot.

It causes duplicate PRs to be sent. Previously, PRs only seemed to be sent for top level parent poms with the versions specified and other poms that relied on the version pins in those parents wouldn't get PRs. Now, all the poms are getting their own dependabot PRs updating the top level parent poms that contain the versions - whether they use the dependencies or not.

## Reproduction

1. Setup dev environment

    ```shell
    git clone git@github.com:dependabot/dependabot-core.git
    cd dependabot-core
    docker pull ghcr.io/dependabot/dependabot-core-development:latest
    docker tag ghcr.io/dependabot/dependabot-core-development dependabot/dependabot-core-development
    ```

1. test with latest dependabot code

    ```shell
    bin/docker-dev-shell maven
    bin/dry-run.rb maven jtgrohn/maven-dependabot-test
    bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /foo
    bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /bar
    ```

    note that it would have created 3 PRs with identical file changes (output below)

    changes `pom.xml` for root directory (expected as this is where the dependency version is set)
    ```shell
    [dependabot-core-dev] ~/dependabot-core $ bin/dry-run.rb maven jtgrohn/maven-dependabot-test
    Resolving dependencies.....
    => fetching dependency files
    => dumping fetched dependency files: ./dry-run/jtgrohn/maven-dependabot-test/
    => parsing dependency files
    => updating 1 dependencies: com.amazonaws:aws-java-sdk-core

    === com.amazonaws:aws-java-sdk-core (1.12.389)
    => checking for updates 1/1
    🌍 --> GET https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 --> HEAD https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    => latest available version is 1.12.396
    => latest allowed version is 1.12.396
    => requirements to unlock: own
    => requirements update strategy: 
    => updating com.amazonaws:aws-java-sdk-core from 1.12.389 to 1.12.396

        ± pom.xml
        ~~~
        12c12
        <                 <version>1.12.389</version>
        ---
        >                 <version>1.12.396</version>
        ~~~
    🌍 Total requests made: '2'
    ```

    changes `pom.xml` for `/foo` directory (not expected as the dependency version is set in the root)

    ```shell
    [dependabot-core-dev] ~/dependabot-core $ bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /foo
    => fetching dependency files
    => dumping fetched dependency files: ./dry-run/jtgrohn/maven-dependabot-test/foo
    => parsing dependency files
    => updating 1 dependencies: com.amazonaws:aws-java-sdk-core

    === com.amazonaws:aws-java-sdk-core (1.12.389)
    => checking for updates 1/1
    🌍 --> GET https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 --> HEAD https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    => latest available version is 1.12.396
    => latest allowed version is 1.12.396
    => requirements to unlock: own
    => requirements update strategy: 
    => updating com.amazonaws:aws-java-sdk-core from 1.12.389 to 1.12.396

        ± ../pom.xml
        ~~~
        12c12
        <                 <version>1.12.389</version>
        ---
        >                 <version>1.12.396</version>
        ~~~
    🌍 Total requests made: '2'
    ```

    changes `pom.xml` for `/bar` directory (not expected as `/bar/pom.xml` doesn't even use _any_ dependencies from `pom.xml`) 

    ```shell
    [dependabot-core-dev] ~/dependabot-core $ bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /bar
    => fetching dependency files
    => dumping fetched dependency files: ./dry-run/jtgrohn/maven-dependabot-test/bar
    => parsing dependency files
    => updating 1 dependencies: com.amazonaws:aws-java-sdk-core

    === com.amazonaws:aws-java-sdk-core (1.12.389)
    => checking for updates 1/1
    🌍 --> GET https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 --> HEAD https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    => latest available version is 1.12.396
    => latest allowed version is 1.12.396
    => requirements to unlock: own
    => requirements update strategy: 
    => updating com.amazonaws:aws-java-sdk-core from 1.12.389 to 1.12.396

        ± ../pom.xml
        ~~~
        12c12
        <                 <version>1.12.389</version>
        ---
        >                 <version>1.12.396</version>
        ~~~
    🌍 Total requests made: '2'
    ```

1. revert [each](https://github.com/dependabot/dependabot-core/pull/6248) [PR](https://github.com/dependabot/dependabot-core/pull/6491) and test again (need to revert [6248](https://github.com/dependabot/dependabot-core/pull/6248) to get a clean revert, though this didn't cause the issue)

    ```shell
    exit
    git revert 30bcb99b064d2d7d595f7d79fe8982e054cf7b65 -m 1
    git revert 0f87a35d9e195ee22ae88ca0f281606b5cdf2523 -m 1
    bin/docker-dev-shell maven
    bin/dry-run.rb maven jtgrohn/maven-dependabot-test
    bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /foo
    bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /bar
    ```

    note that it only creates 1 pr (output below)

    changes `pom.xml` for root directory (expected as this is where the dependency version is set)

    ```shell
    [dependabot-core-dev] ~/dependabot-core $ bin/dry-run.rb maven jtgrohn/maven-dependabot-test
    Resolving dependencies.....
    => fetching dependency files
    => dumping fetched dependency files: ./dry-run/jtgrohn/maven-dependabot-test/
    => parsing dependency files
    => updating 1 dependencies: com.amazonaws:aws-java-sdk-core

    === com.amazonaws:aws-java-sdk-core (1.12.389)
    => checking for updates 1/1
    🌍 --> GET https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 --> HEAD https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    => latest available version is 1.12.396
    => latest allowed version is 1.12.396
    => requirements to unlock: own
    => requirements update strategy: 
    => updating com.amazonaws:aws-java-sdk-core from 1.12.389 to 1.12.396

        ± pom.xml
        ~~~
        12c12
        <                 <version>1.12.389</version>
        ---
        >                 <version>1.12.396</version>
        ~~~
    🌍 Total requests made: '2'
    ```

    makes no changes for `/foo` directory as its dependencies are tracked in the root directory `pom.xml`

    ```
    [dependabot-core-dev] ~/dependabot-core $ bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /foo
    => fetching dependency files
    => dumping fetched dependency files: ./dry-run/jtgrohn/maven-dependabot-test/foo
    => parsing dependency files
    => updating 1 dependencies: com.amazonaws:aws-java-sdk-core

    === com.amazonaws:aws-java-sdk-core ()
    => checking for updates 1/1
    🌍 --> GET https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/maven-metadata.xml
    🌍 --> HEAD https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    🌍 <-- 200 https://repo.maven.apache.org/maven2/com/amazonaws/aws-java-sdk-core/1.12.396/aws-java-sdk-core-1.12.396.jar
    => latest available version is 1.12.396
        (no update needed as it's already up-to-date)
    🌍 Total requests made: '2'
    ```

    makes no changes for `/bar` directory as it has no dependencies

    ```
    [dependabot-core-dev] ~/dependabot-core $ bin/dry-run.rb maven jtgrohn/maven-dependabot-test --dir /bar
    => fetching dependency files
    => dumping fetched dependency files: ./dry-run/jtgrohn/maven-dependabot-test/bar
    => parsing dependency files
    => updating 0 dependencies: 
    🌍 Total requests made: '0'
    ```
