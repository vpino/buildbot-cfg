## Buildbot: Simple Tutorial ![Buildbot logo](http://buildbot.net/favicon.ico "Buildbot")

===================================================

## Install Requirements 

1 - Install the package responsible to manage master

```bash
$ pip install buildbot
```

2 - Install the package responsible to manage slave

```bash
$ pip install buildbot-slave
```

3 - To make it work

```bash
$ pip install -r requirements.txt
```

## Create buildbot master

1 - To create the master execute:

```bash
$ buildbot create-master master
```

2 - Change name of file master.cfg.sample to master.cfg

```bash
$ mv master/master.cfg.sample master/master.cfg
```

3 - Now start it:

```bash
$ buildbot start master
```

## Create buildbot slave

1 - To create the slave execute:

```bash
$ buildslave create-slave BASEDIR MASTERHOST:PORT SLAVENAME PASSWORD

$ buildslave create-slave mirror localhost:9989 mirror 11
```

2 - The file located in BASEDIR/info/admin should contain your name and email address. This is the buildslave admin address, and will be visible from the build status page (so you may wish to munge it a bit if address-harvesting spambots are a concern).

3 - The file located in BASEDIR/info/host should be filled with a brief description of the host: OS, version, memory size, CPU speed, versions of relevant libraries installed, and finally the version of the buildbot code which is running the buildslave.

4 - Init Slave

```bash
cd mirror
buildslave start
```

5 - Restart the buildmaster to take the configuration

```bash
$ buildbot restart basedir
```

6 - Config buildslaves

In the master.cfg config the slaves:

```python
c['slaves'] = [buildslave.BuildSlave("example-slave", "pass")]
```

For the slave created:

```python
c['slaves'] = [buildslave.BuildSlave("mirror", "11")]
```

### Check BUILDBOT

In your browser check the url:

``` 
localhost:8010
```

## CONFIGURACION BASIC 

To basic config created a lists to each component 
which will be explained

```python
c = BuildmasterConfig = {}

c['status'] = []

c['slaves'] = []

c['change_source'] = []

c['schedulers'] = []

c['builders'] = []

c['buildbotURL'] = 'http://localhost:8010/'

c['protocols'] = {'pb': {'port': 9989}}

forced_builders = []
```

## Time to use BUILDBOT


When you config buildbot, should be clear about several concepts:

### Change Sources

Which create a Change object each time something is modified in the VC repository. Most ChangeSources listen for messages from a hook script of some sort. Some sources actively poll the repository on a regular basis. All Changes are fed to the Schedulers.

### List to Change Sources

```python
c['change_source'] = []
```

### Schedulers
    
Which decide when builds should be performed. They collect Changes into BuildRequests, which are then queued for delivery to Builders until a buildslave is available.

### List to Schedulers

```python
c['schedulers'] = []
```

### Builders
    
Which control exactly how each build is performed (with a series of BuildSteps, configured in a BuildFactory). Each Build is run on a single buildslave.

### List to Builders

```python
c['builders'] = []
```


## Create Change Sources

Here we do a git-based change_source, which will verify the changes submitted by the repository. The parameters passed are the following:

- Url.
- Name the change.
- Branch repository.
- Time interval for a poll.

Already with this simple configuration we create a change_source.

```python
c['change_source'].append(
        GitPoller(repourl="http://gitlab.canaima.softwarelibre.gob.ve/canaima-gnu-linux/buildbot-cfg.git"
              project= "buildbot-cfg",
              branches="desarrollo",
              pollinterval="60")
            )
```

## Create Builder

1 - To create a builder we need some things, which are:

- Name to builder.
- factory with all steps.
- slavename.

### First defined the steps to the build:

In this case, the builder have the function of build a debian package

```python
package = 'buildbot-cfg'
CLONE_DIR = 'build'
BUILD_DIR = os.path.join(CLONE_DIR, package)

all_steps = [
        ShellCommand(command = 'rm -rf *'),
        ShellCommand(command = ['git', 'clone', 'http://gitlab.canaima.softwarelibre.gob.ve/canaima-gnu-linux/buildbot-cfg.git'], workdir=CLONE_DIR, haltOnFailure = True),
        ShellCommand(command = ['git', 'checkout', 'desarrollo'], workdir=BUILD_DIR),
        SetPropertyFromCommand(command = "cat debian/changelog", extract_fn=parse_changelog, workdir=BUILD_DIR, haltOnFailure = True),
        SetPropertyFromCommand(command = "cat debian/control", extract_fn=parse_control, workdir=BUILD_DIR, haltOnFailure = True),
        ShellCommand(command = ['gbp', 'buildpackage', '--git-pbuilder', '--git-dist=jessie', '--git-debian-branch='+vcs['dev'],
            '--git-upstream-tree='+vcs['dev'], '--git-arch=i386', '-us', '-uc'], doStepIf=check_i386, workdir=BUILD_DIR, haltOnFailure = True),
        ShellCommand(command = WithProperties("lintian %(package)s_%(version)s_%(architecture)s.changes"), workdir=CLONE_DIR, haltOnFailure = False),
    ]
```

2 - Added the builder

```python
c['builders'].append(BuilderConfig(name=package, factory=BuildFactory(all_steps), slavenames='mirror'))
```

## Create Schedulers

Now we create a Scheduler, he will be responsible the collect changes into Build Requests to after delivery to a Builders until a buildslave is available

The parameters are:

- Name.
- Changefilter with name project and branch.
- treeStableTimer.
- builderNames

```python
sbched = SingleBranchScheduler(name=package,
        change_filter = ChangeFilter(project=package, branch='desarrrollo'),
        treeStableTimer = 10,
        builderNames = [package])
```

Add Scheduler to all schedulers:

```python
c['schedulers'].append(sbched)
```

Also we can add the option to the builder to that can be forced his construction or execution, for this:

"package is the name of the builder"

```python
forced_builders.append(package)
```

And now we created a ForceScheduler that contain all forced builders:

```python
forced_scheduler = ForceScheduler(name='Forzar', builderNames = forced_builders)
```

And we add the force_builders to the schedulers:

```python
c['schedulers'].append(forced_scheduler)
```

## Autentication  

We will create a Object Authz with the configuration to access
the buildbot

```python
authz_cfg = Authz(
    # change any of these to True to enable; see the manual for more
    # options
    auth=BasicAuth([("pyflakes", "pyflakes")]), # user and passwd
    gracefulShutdown = False,
    forceBuild = 'auth', # use this to test your slave once it is set up
    forceAllBuilds = True,
    pingBuilder = False,
    stopBuild = True,
    stopAllBuilds = True,
    cancelPendingBuild = True,
)
```

## Web Status

And to finish the tutorial we add to the WebStatus with the configuration
saved in authz_cfg

```python
c['status'].append(WebStatus(http_port=8010, authz=authz_cfg))
```

And with this we have a simple configuration a of buildbot to construct a package of debian :DD


#### Sources:

[http://docs.buildbot.net/current/manual/index.html]
[http://docs.buildbot.net/current/tutorial/fiveminutes.html]