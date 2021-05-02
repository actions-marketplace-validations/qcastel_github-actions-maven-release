# github action maven release

The GitHub Action for Maven releases wraps the Maven CLI to enable Maven release.
For example, you can use this action for auto-incrementing your project version and release your java artifacts.

This github action is bot friendly: You can configure the credentials of a bot user, which would be used during the incremental commit.
The commits by the bot can also be signed, giving you the guaranty that only the bot can release in your repo. Additionally,
this give you a clean git history by highlighting nicely which commits where resulting from your CI.

## Supporting this github action

Support this github action by staring this project. Surprisingly, it seems to be the only way for the github market place to highlight popular github actions.

## Sample repository

We created a sample repository that will show you an example of how this github action can be used for releasing a Java application: 
https://github.com/qcastel/github-actions-maven-release-sample

## Features

Obviously, this github actions uses maven release plugin. Although, we did add on top a few features that you may like.

Maven release uses Git behind it, therefore there were a few features related in customising the git configuration:
- Signing the commits (GPG) resulting from the maven release [[GPG](#setup-a-gpg-key)]
- Authenticating to private repository using an SSH key [[SSH](#setup-with-ssh)]
- Configuring the git username and email [[Bot](#customise-the-bot-name)]
- Configuring the jdk version [[JDK](#jdk-version)]

You may want to configure a bit maven too. We added the following features:
- Specify the maven project path. In other words, if your maven project is not at the root of your repo, you can configure a sub path. [[Custom project path](#customise-the-m2-folder-path)] 
- Configure a private maven repository [[Private maven repo](#setup-a-private-maven-repository)]
- Configure a docker registry [[Docker registry](#setup-a-docker-registry)]
- Setup custom maven arguments and/or options to be used when calling maven commands [[Maven options](#maven-options)]
- Configure a custom M2 folder [[Custom M2](#customise-the-m2-folder-path)]
- Print the timestamp for every maven logs. Handy for troubleshooting performance issues in your CI. [[Log timestamp](#log-timestamp)]

For the maven releases, we got also some dedicated functionalities:
- Skip the maven perform [[Skip perform](#skipping-perform)]
- Roll back the maven perform if it failed to perform the release
- Increment the major or minor version (by default, it's the patch version that is increased) [[Major Minor version](#increase-major-or-minor-version)]
- customise the version format completly [[Customize version](#customize-version)]

# Usage

## Setup your pom.xml for maven release

Before you even begin setting up this github action, you would need to set up your pom.xml first to be ready for maven releases.
We recommend you to refer to the maven release plugin documentation for more details: https://maven.apache.org/maven-release/maven-release-plugin/

Nevertheless, we will give you some essential setups

### Configure the SCM

You got two choices here:
- Using SSH URL (Recommended)
```xml
    <scm>
        <connection>scm:git:${project.scm.url}</connection>
        <developerConnection>scm:git:${project.scm.url}</developerConnection>
        <url>git@github.com:idhub-io/idhub-api.git</url>
        <tag>HEAD</tag>
    </scm>
```
- Using HTTPS URL
```xml
	<scm>
        <connection>scm:git:${project.scm.url}</connection>
        <developerConnection>scm:git:${project.scm.url}</developerConnection>
		<url>https://github.com/YOUR_REPO.git</url>
		<tag>HEAD</tag>
	</scm>
```

In the case of SSH, it will use the `ssh-private-key`  to authenticate with the upstream.
In the case of HTTPS, maven releases will use the `access-token` in this github actions to authenticate with the upstream.

Note: SSH is more elegant and usually the easiest one to setup due to the large amount of documents online on this subject.

### maven release plugin

Add the maven release plugin dependency to your project

```xml
    <plugin>
        <artifactId>maven-release-plugin</artifactId>
        <version>XXX</version>
        <configuration>
            <scmCommentPrefix>[ci skip]</scmCommentPrefix>
        </configuration>
    </plugin>
```

Personally, I usually the prefix `[ci skip]` which allows me to skip more easily any commits generated by the bot from the CI.

## Setup the maven release github actions

### Choose your version of this github action

If it's your first time using a github action, I invite you having a quick read to the github official recommendations:
https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/security-hardening-for-github-actions#using-third-party-actions

It's important you understand how the versioning work and the risk/compromise of using master/tags/commit hash

If you are adventurous and like to be always on top of this github action, you can use the reference *master* :

```
 - name: Release
      uses: qcastel/github-actions-maven-release@master
      with:
```

If you are more reserve, you can use a tag instead. You can find the list of the tags for this github action here:

https://github.com/qcastel/github-actions-maven-release/tags

To use a tag:

```
 - name: Release
      uses: qcastel/github-actions-maven-release@TAG_NAME
      with:
```

If you are concerned about the security of this github action, you can also move to a commit hash:

```
 - name: Release
      uses: qcastel/github-actions-maven-release@COMMIT_HASH
      with:
```



### Basic setup
For a simple repo with not much protection and private dependency, you can do:

```yaml
      with:
        access-token: ${{ secrets.GITHUB_ACCESS_TOKEN }}
```

### Setup with SSH

Although you may found better to use a SSH key instead. For this, generate an SSH key with the method of your choice, or use an existing one.
Personally, I like generating an SSH inside a temporary docker image and configure it as a deploy key in my repository:

```bash
docker run -it qcastel/maven-release:latest  bash
```

```bash
ssh-keygen -b 2048 -t rsa -f /tmp/sshkey -q -N ""
export SSH_PRIVATE_KEY=$(base64 /tmp/sshkey)
export SSH_PUBLIC_KEY=$(cat /tmp/sshkey.pub)
echo -n "Copy the following SSH private key and add it to your repo secrets under the name 'SSH_PRIVATE_KEY':"
echo $SSH_PRIVATE_KEY
echo "Copy the encoded SSH public key and add it as one of your repo deploy keys with write access:"
echo $SSH_PUBLIC_KEY

exit 
```

Copy `SSH_PRIVATE_KEY` and add it as a new secret.
![img/Add-ssh-secrets.png](img/Add-ssh-secrets.png?raw=true)

Copy `SSH_PUBLIC_KEY` and add it as a new deployment key with write access.
![img/add-deploy-key.png](img/add-deploy-key.png?raw=true)

Finally, setup the github action with:

```yaml
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

If you want to set up a passphrase for your key:

```yaml
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
          ssh-passphrase: ${{ secrets.SSH_PASSPHRASE }}
```


### log Timestamp

It can be quite difficult to troubleshoot any performance issue on your CI, due to the lack of timestamp from maven by default.
An example of it particular handy, is when you private maven repository is having performance issue that is affecting your CI.

We added the timestamp by default, you don't need to do anything particular to enable this feature.

The logs should look like:

```
14:27:09,491 [INFO] Downloading from spring-snapshots: https://repo.spring.io/snapshot/io/projectreactor/reactor-bom/Dysprosium-SR13/reactor-bom-Dysprosium-SR13.pom
``` 
 
### Maven options

#### Adding maven arguments

You can add some maven arguments, which is handy for skipping tests:

```yaml
        with:
            maven-args: "-Dmaven.javadoc.skip=true -DskipTests -DskipITs -Ddockerfile.skip -DdockerCompose.skip"
```

#### Adding maven options

You can add some maven options. At the difference of the maven arguments, those one are explicitly for the maven release plugin.
See https://maven.apache.org/maven-release/maven-release-plugin/prepare-mojo.html.

```yaml
        with:
            maven-options: "-DbranchName=hotfix"
```

### JDK version

You may want to compile your project with a specific JDK version. You will need to specify the JAVA_HOME variable with the according value.
If you need a specific jdk version that is not in the list, please raise an issue in this github action to request it.

#### JDK 8

```yaml
env:
 JAVA_HOME: /usr/lib/jvm/java-1.8-openjdk/
```

#### JDK 11

```yaml
env:
 JAVA_HOME: /usr/lib/jvm/java-11-openjdk/
```

#### JDK 14

```yaml
env:
 JAVA_HOME: /usr/lib/jvm/java-14-openjdk/
```

#### JDK 15

```yaml
env:
 JAVA_HOME: /usr/lib/jvm/java-15-openjdk/
```
### Customise the bot name

You can simply customise the bot name as follows:


```yaml
        with:
            git-release-bot-name: "release-bot"
            git-release-bot-email: "release-bot@example.com"
```

### Customise the default branch
You may not want to release from your master branch, which is currently the default branch setup by this github action. You can customise the branch name you want to release on, here `release`, as follows:

```yaml
        with:
            release-branch-name: "release"

```

### Skipping perform
If for a reason, you need to skip the maven release perfom, you can disable it as follow:

```yaml
        with:
            skip-perform: false
```


### Increase major or minor version
By default, maven release will increase the patch version. If you are interested to actually increment the major or minor version, you can use
the following options:

#### For major version increment

_1.0.0-SNAPSHOT -> 2.0.0-SNAPSHOT_

```yaml
        with:
            version-major: true
```

#### For minor version increment

_1.0.0-SNAPSHOT -> 1.2.0-SNAPSHOT_

```yaml
        with:
            version-minor: true
```

### Customize version

You may want to fully customize the version number. This option will allow you to fully take control on the version number format.

For Example, you could decide to only have a 2 part version number like 0.2-SNAPSHOT.

```yaml
        with:
            maven-development-version-number: "\${parsedVersion.majorVersion}.\${parsedVersion.nextMinorVersion}-SNAPSHOT"
```



#### Customise the M2 folder path

It's quite common for setting up a caching of your dependencies, that you will be interested to customise the .m2 localisation folder.

```yaml
        with:
            m2-home-folder: '/your-custom-path/.m2'
```


### Setup a GPG key
If you want to set up a GPG key, you can do it by injecting your key via the secrets:

Note: `GITHUB_GPG_KEY` needs to be base64 encoded.
if you haven't setup a GPG key yet, see next section.

```yaml
      with:
        gpg-enabled: "true"
        gpg-key-id: ${{ secrets.GITHUB_GPG_KEY_ID }}
        gpg-key: ${{ secrets.GITHUB_GPG_KEY }}
```

In case you want to skip the GPG step, you can set `gpg-enabled: "false"` or if you prefer to have the same behaviour in your IDE, add this maven plugin in your `pom.xml` to skip GPG step in the release phase:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-gpg-plugin</artifactId>
    <version>1.6</version>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```

### Setup a private maven repository

If you got a private maven repo to set up in the settings.xml, you can do:
Note: we recommend putting those values in your repo secrets.

```yaml
      with:
        maven-repo-server-id: your-maven-repo-id
        maven-repo-server-username: ${{ secrets.MVN_REPO_PRIVATE_REPO_USER }}
        maven-repo-server-password: ${{ secrets.MVN_REPO_PRIVATE_REPO_PASSWORD }}
```


### Setup a docker registry

If you got a private maven repo to set up in the settings.xml, you can do:
Note: we recommend putting those values in your repo secrets.

```yaml
      with:
        docker-registry-id: your-docker-registry-id
        docker-registry-username: ${{ secrets.DOCKER_REGISTRY_USERNAME }}
        docker-registry-password: ${{ secrets.DOCKER_REGISTRY_PASSWORD }}
```

Note: For docker hub, this would look like:
```yaml
      with:
        docker-registry-id: registry.hub.docker.com
        docker-registry-username: ${{ secrets.DOCKER_HUB_USERNAME }}
        docker-registry-password: ${{ secrets.DOCKER_HUB_PASSWORD }}
```

### Configure your maven project

You may also be in the case where you got more than one maven projects inside the repo. We added an option that will make the release job move to the according directly before running the release:

```yaml
    with:
        maven-project-folder: "sub-folder/"
```


## Setup the bot gpg key

Setting up a gpg key for your bot is a good security feature. This way, you can enforce sign commits in your repo,
even for your release bot.

![Screenshot-2019-11-28-at-20-47-06.png](https://i.postimg.cc/9F6cxpqm/Screenshot-2019-11-28-at-20-47-06.png)

- Create dedicate github account for your bot and add him into your team for your git repo.
- Create a new GPG key: https://help.github.com/en/github/authenticating-to-github/generating-a-new-gpg-key

This github action needs the key ID and the key base64 encoded.

```yaml
        with:
            gpg-enabled: true
            gpg-key-id: ${{ secrets.GPG_KEY_ID }}
            gpg-key: ${{ secrets.GPG_KEY }}
```

If you want to set up a passphrase:

```yaml
        with:
            gpg-enabled: true
            gpg-key-id: ${{ secrets.GPG_KEY_ID }}
            gpg-key: ${{ secrets.GPG_KEY }}
            gpg-passphrase: ${{ secrets.GPG_PASSPHRASE }} 
```

### Generate the key

f you like how we created a SSH key pair, here is the same idea using a docker image to generate a GPG key:

```bash
docker run -it qcastel/maven-release:latest  bash
```

```bash
cat >genkey-batch <<EOF
 %no-protection
 Key-Type: default
 Subkey-Type: default
 Name-Real: bot
 Name-Email: bot@idhub.io
 Expire-Date: 0
EOF
gpg --batch --gen-key genkey-batch
```

Note: Don't exit the docker container as we are not done yet.


### Get the KID

You can get the key ID doing the following:

```bash
gpg --list-secret-keys --keyid-format LONG

sec   rsa2048/3EFC3104C0088B08 2019-11-28 [SC]
      CBFD9020DAC388A77C68385C3EFC3104C0088B08
uid                 [ultimate] bot-openbanking4-dev (it's the bot openbanking4.dev key) <bot@openbanking4.dev>
ssb   rsa2048/7D1523C9952204C1 2019-11-28 [E]

```
The key ID for my bot is 3EFC3104C0088B08. Add this value into your github secret for this repo, under `GPG_KEY_ID`
PS: the key id is not really a secret but we found more elegant to store it there than in plain text in the github action yml

### Get the GPG public and private key

Now we need the raw key and base64 encode
```bash

echo 'Public key to add in your bot github account:'
gpg --armor --export FFD651809B1889DF
echo 'Private key to add to the CI secrets under GITHUB_GPG_KEY:'
gpg --export-secret-keys --armor FFD651809B1889DF | base64

exit
```

Copy the public key and import it to the bot account as a GPG key.
Copy the private key and add it in your github repo secrets under `GPG_KEY`.


# License
The Dockerfile and associated scripts and documentation in this project are release under the MIT License.

