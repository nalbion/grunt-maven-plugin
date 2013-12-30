# grunt-maven-plugin

**grunt-maven-plugin** plugin allows to integrate **Grunt** tasks into **Maven** build process. [**Grunt**](http://gruntjs.com/) is the JavaScript task runner utility. **grunt-maven-plugin** works on both Windows and \*nix systems.

## Prerequisites

The only required dependency is [**nodejs**](http://nodejs.org/) with **npm**.
Globally installed [**grunt-cli**](http://gruntjs.com/getting-started) is optional and preferred, but not necessary, as installing custom node modules can be problematic in some environments (ex. CI servers). Additional configuration is needed when using local **grunt-cli**.

## Usage

Add **grunt-maven-plugin** to application build process in your *pom.xml*:

```xml
<plugin>
    <groupId>pl.allegro</groupId>
    <artifactId>grunt-maven-plugin</artifactId>
    <version>1.1.0</version>
    <configuration>
        <!-- relative to src/main/webapp/, default: static -->
        <jsSourceDirectory>path_to_js_project</jsSourceDirectory>

        <!-- example options usage to get verbose output in logs -->
        <gruntOptions>
            <gruntOption>--verbose</gruntOption>
        </gruntOptions>

        <!-- example options usage to filter variables in given resource -->
        <filteredResources>
            <filteredResource>maven-properties.json</filteredResource>
        </filteredResources>

    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>create-resources</goal>
                <goal>npm</goal>
                <goal>grunt</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### Usage with local grunt-cli

In order to use local **grunt-cli**, add

```xml
<gruntExecutable>node_modules/grunt-cli/bin/grunt</gruntExecutable>
<runGruntWithNode>true</runGruntWithNode>
```

options to plugin configuration and add **grunt-cli** to JS project **package.json**:

```javascript
{
    "devDepenencies": {
        "grunt-cli": "~0.1.7",
        "grunt": "~0.4.1"
        /*...*/
    }
}
```

### Usage with preinstalled node_modules

When you want to use tool like [frontend-maven-plugin](https://github.com/eirslett/frontend-maven-plugin)
to install node and npm locally or when you have commited-in **node_modules** directory
(not really recommended, see [npm shronkwrap](https://npmjs.org/doc/shrinkwrap.html) to
freeze module versions), you can make **grunt-maven-plugin** use preinstalled
node_modules. Just replace **npm** goal with **link-node-modules** and add **nodeModulesPath**
to configuration options. Example:

```xml
    <configuration>
        <nodeModulesPath>${basedir}/preinstalled_modules_dir/</nodeModulesPath>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>create-resources</goal>
                <goal>link-node-modules</goal>
                <goal>grunt</goal>
            </goals>
        </execution>
    </executions>
```

Under the hood it uses [maven-junction-plugin](http://pyx4j.com/snapshot/pyx4j/pyx4j-maven-plugins/maven-junction-plugin/introduction.html)
to create a link between provided path and *gruntBuildDirectory*. Grunt thinks it has
its modules in place and executes flawlessly.

## How it works?

1. JavaScript project sources from

        sourceDirectory/jsSourceDirectory (default: src/main/webapp/static)

    are copied to

        gruntBuildDirectory (default: target-grunt)

    Copied directory has to contain **package.json** and **Gruntfile.js** files

1. `npm install` is called, fetching all dependencies

1. `grunt` is run to complete build process

Since plugin creates own target dir, it should be added to ignored resources in SCM configuration (like .gitignore).

## grunt-maven-plugin options

Plugin options available in `<configuration>...</configuration>` are:

#### misc

* **showColors** : should Grunt and npm use color output; defaults to *false*
* **filteredResources** : list of files (or expressions) that will be filtered using **maven-resources-plugin** when creating resources,
remember to exclude those files from integrated workflow config, as else Grunt will override filtered values

#### environment

* **gruntBuildDirectory** : path to Grunt build directory (target for Grunt); defaults to *${basedir}/target-grunt*
* **sourceDirectory** : path to directory containing source JavaScript project directory; defaults to *${basedir}/src/main/webapp*
* **jsSourceDirectory** : name of directory relative to *sourceDirectory*, which contents are going to be copied to *jsTargetDirectory*; defaults to *static*

#### node

* **nodeExecutable** : name of globally available **node** executable; defaults to *node*
* **npmExecutable** : name of globally available **npm** executable; defaults to *npm*
* **nodeModulesPath** : path where preinstalled **node_modules** are stored

#### grunt options

* **target** : name of Grunt target to run (defaults to null, default Grunt target is run)
* **gruntOptions** : list of custom options passed to grunt (defaults to empty)
* **gruntExecutable** : name of **grunt** executable; defaults to *grunt*
* **runGruntWithNode** : if Grunt executable is a js script, it needs to be run using node, ex: `node path/to/grunt`; defaults to *false*

## Execution goals

* **create-resources** : copies resources from *sourceDirectory/jsSourceDirectory* to *gruntBuildDirectory*
* **npm** : executes `npm install` in target directory
* **grunt** : executes Grunt in target directory
* **clean** : deletes *gruntBuildDirectory*

## Integrated workflow

Using grunt-maven-plugin is convenient, but there is still a huge gap between frontend and backend development workflow. Various IDEs (Netbeans, IntelliJ Idea, Eclipse)
try to ease webapp development by synchronizing changes made in *src/webapp/* to exploded WAR directory in *target/*. This naive approach works as long as there is no
need to minify JavaScript sources, compile less files or project does not follow "single WAR" rule and can afford using Maven profiles. Rebuilding project each time a
change was made in *.js is a horrible thing. Fortunately grunt-maven-plugin is a great tool to solve this problem.

Idea is to ignore IDE mechanisms and run Grunt build each time a change in static files was made. Traditional workflow looks like:

1. user changes *src/webapp/static/hello.js*
1. IDE detects change
1. IDE copies *hello.js* to *target/_war_name_/static/hello.js*

This gives little room to integrate other processes in between. Workflow utilizing **grunt-maven-plugin**:

1. run [Grunt watch process](https://github.com/gruntjs/grunt-contrib-watch)
1. user changes *src/webapp/static/hello.js*
1. Grunt detects changes, copies *hello.js* to *target-grunt/hello.js*
1. run Grunt tasks, produce *target-grunt/dist/hello.min.js* with source map *target-grunt/dist/hello.map.js*
1. Grunt copies results to *target/_war_name_/static*

Now what happens inside *target-grunt* is for us to decide, it can be virtually anything - minification, less compilation, running
JS tests, JS linting. Anything Grunt can do, just like during *heavy* build process.

### Configuring Maven

Since we want grunt-maven-plugin to take control of what ends up in WAR, we need to tell Maven WAR plugin to ignore our statics dir when creating WAR:

```xml
<build>
    <plugins>
	<plugin>
	    <artifactId>maven-war-plugin</artifactId>
	    <version>2.3</version>
	    <configuration>
		<warSourceExcludes>jsSourceDirectory/**</warSourceExcludes>
	    </configuration>
	</plugin>
    </plugins>
</build>
```

### Configuring Grunt

There are two Grunt modules that need to be imported: **grunt-contrib-watch** and **grunt-contrib-copy**. Integrated
workflow tasks are emebedded in separate Grunt module, which should be loaded by adding:

```javascript
grunt.loadTasks('maven-tasks');
```

Now there are two additional tasks registered: `maven` and `maven-watch`. `maven` task should be added to default task
execution after tasks creating deliverables, for example:

```javascript
grunt.registerTask('default', ['jshint', 'karma', 'less', 'uglify' 'maven']);
```

`maven-watch` should be used as a task to run in watch process, it copies files from maven source to Grunt working directory.
Integrated workflow registers `watch.maven` multitask with files specified, but you need to add tasks:

```javascript
watch: {
    maven: {
        tasks: ['maven-watch', 'default']
    }
}
```

Integrated workflow tasks have own properties, some of them derived straight from **pom.xml** settings to eliminate
duplication of config, everything gets configured in **pom.xml**.

### Configuring workflow

After first build, file named **maven-workflow-properties.json** will appear in **target-grunt**. This file is read by
workflow Grunt tasks. It should be added to application sources and configured according to project needs. This file
is filtered by Maven, so it is possible to use project properties like ${project.version} inside. Available options:

* **gruntTargetFilePatterns** : array or string that adheres to [Grunt src files](http://gruntjs.com/configuring-tasks#files) specification, defines which
    files will be copied from Maven src to **target-grunt**
* **distDirectory** : name of directory in **target-grunt** where deliverables reside, content of this directory is later copied to WAR
* **distFilePatterns** : array or string that adheres to [Grunt src files](http://gruntjs.com/configuring-tasks#files) specification, defines which
    files will be copied from **target-grunt** to **distDirectory**
* **warName** name of WAR file in Maven target

### Deep customization of workflow

It is possible to override **any** workflow configuration during runtime. After reading properties JSON file, workflow
Grunt task seeks for **maven-custom-\* ** file in **target-grunt** and overrides original properties with custom ones.
Files to customize are:

* **maven-workflow-properties.json** with **maven-custom-workflow-properties.json**
* **maven-inner-properties.json** with **maven-custom-inner-properties.json** (maven-inner-properties.json resides in **target-grunt/maven-tasks**)

### Configuring IDE

Depending on IDE, synchronization processes behave differently. Having JSP files synchronized is a nice feature, so it is up to you
to decide what should be turned off. In Netbeans and IntelliJ no configuration is needed, they play nicely with new workflow.

You still have to remember to run Grunt watch process in background so it can monitor changes. It can be run from IDE via grunt-maven-plugin.
Define custom runner that will run:

    mvn grunt:grunt -Dtarget=watch:maven

You should see process output each time static sources change.

#### Configuring Eclipse

Eclipse is a special case. Unfortunately it does not read WAR from Maven target, instead it keeps own file hierarchy.
Eclipse developers on the team should use deep workflow customization to override properties used by others. Proposed
way is to create **maven-custom-inner-properties.json** in Maven sources and add it to **.gitignore**, so it will not pollute
source repository. Since i have little experience with Eclipse, i would welcome a contribution of sample configuration.


## Changelog

* **1.1.0** (30.12.2013)
  * integrated workflow as a separate, auto-configured Grunt task
* **1.0.4** (8.12.2013)
  * explicit declaration of resources filtered on create-resources goal
* **1.0.3** (24.11.2013)
  * passing custom options to grunt executable ( #8 )
  * ability to use external or preinstalled node_modules ( #6 )
* **1.0.2** (15.10.2013)
  * option to disable npm and grunt color output, by default no colors are shown as it looks bad in Maven logs ( #3 )
* **1.0.1** (13.09.2013)
  * compatibility with Maven 3.1.x ( #1 )


## License

**grunt-maven-plugin** is published under [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).
