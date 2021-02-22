  

### Introduction

Previously, two kinds of recipes were available to bootstrap a cloud developer workspace and to make it portable: [Chefile](https://www.eclipse.org/che/docs/chefile.html) and [Factories](https://www.eclipse.org/che/docs/factories-getting-started.html#try-a-factory).

As a continuation of this, the brand new `devfile` format was introduced, which combines simplicity and support for a wide variety of different components available to develop a container-based application.

### What the devfile consists of

The minimal devfile sufficient to run a workspace from it, consists of the following parts:
 - Specification version
 - Name

Without any further configuration, a workspace with default editor launched along with its default plugins, which configures on Che Server. By default, `Che Theia` is configured as a default one along with the `Che Machine Exec` plugin.

This functional workspace provides the following parts:

 - A list of components: the development components and user runtimes
 - A list of projects: the source code repositories
 - A list of commands: actions to manage the workspace components like running the dev tools, starting the runtime environments, etc.

Example of the minimal devfile with the project and standard plugins set (Theia editor + exec plugin):

```yaml
---
apiVersion: 1.0.0
metadata:
  name: petclinic-dev-environment
projects:
  - name: petclinic
    source:
      type: git
      location: 'https://github.com/che-samples/web-java-spring-petclinic.git'
components:
  - alias: theia-editor
    type: cheEditor
    id: eclipse/che-theia/next
  - alias: exec-plugin
    type: chePlugin
    id: eclipse/che-machine-exec-plugin/0.0.1
```

For the detailed explanation of all devfile components assignment and possible values, please see the following resources:

 - [Specification repository](https://github.com/redhat-developer/devfile)  
 - [detailed json-schema documentation](https://redhat-developer.github.io/devfile/devfile).

### Getting Started

The simplest way to use devfile is to have it deployed into the GitHub source repository and then create a factory from this repo. Here is a simple `devfile.yaml` file in the root of your GH repo, and then execute the factory:

```html
https://<your-che-host>/f?url=https://github.com/mygroup/myrepo
```

Also, it is possible to execute devfile by constructing the factory with the URL to it's raw content, for example,

```html
https://<your-che-host>/f?url=https://pastebin.com/raw/ux6iCGaW
```
or sending a devfile to a dedicated REST API using curl/swagger, which will create new workspace and return it's configuration:

```bash
curl -X POST  -H "Authorization: <TOKEN>" -H "Content-Type: application/yaml" -d <devlile_content> https://<your-che-host>/api/devfile
```

If you're a user of `chectl` tool, it is also possible to execute workspace from devfile, using `workspace:start` command parameter as follows:

```bash
chectl workspace:start --devfile=devfile.yaml
```

Please note that at the moment, this way only works for the local (same machine) devfiles - URL not permitted here atm.

### Project details

A single devfile can specify several projects. For each project, specify the type of the source repository, its location, and optionally the directory the project should clone.

As an example, consider this devfile:

```yaml
apiVersion: 1.0.0
metadata:
  name: example-devfile
projects:
- name: frontend
  source:
    type: git
    location: https://github.com/acmecorp/frontend.git
- name: backend
  clonePath: src/github.com/acmecorp/backend
  source:
    type: git
    location: https://github.com/acmecorp/backend.git
```

In the example above, we see a devfile with 2 projects, `frontend` and `backend,` each located in
its repository on Github. `backend` has a specific requirement to be cloned into the `src/github.com/acmecorp/backend` directory under the source root (implicitly defined by the Che runtime) while frontend cloned into the `frontend` directory under the source root.

### Supported component types

There are currently four types of components supported. There are two more straightforward types, such as `cheEditor` and `chePlugin` and two more complex - `kubernetes` (or `openshift`) and `dockerimage`. Please note that all components inside single devfile must have unique names.

Detailed component types explanation below:

#### cheEditor
Describes the editor used in the workspace by defining its id.
Devfile can only contain one component with the `cheEditor` type.

```yaml
...
components:
  - alias: theia-editor
    type: cheEditor
    id: eclipse/che-theia/next
```

If missing, then a default editor provided along with its default plugins.

The default plugins also provided for an explicitly defined editor with the same`ID` as the default one (even if in a different version). By default, `Che Theia` is configured as default editor along with the `Che Machine Exec` plugin. You're able to put `editorFree:true` attribute into Devfile attributes in case you do not need any editor in your workspace.

#### chePlugin
Describes the plugin used in the workspace by defining its id.
It is allowed to have several `chePlugin` components.

```
...
  components:
   - alias: exec-plugin
     type: chePlugin
     id: eclipse/che-machine-exec-plugin/0.0.1
```

Both types above are using id, which is a slash-separated publisher, name, and version of the plugin from the Che Plugin registry. List of available Che plugins and more information about the registry found on https://github.com/eclipse/che-plugin-registry.

It is also possible to specify own registry for the cheEditor and chePlugin types, by using
`registryUrl` parameter as follows:

```yaml
...
  components:
   - alias: exec-plugin
     type: chePlugin
     registryUrl: https://my-customregistry.com
     id: eclipse/che-machine-exec-plugin/0.0.1
```

As an alternative way of specifying an editor or plugin, instead of using plugin id (+ optional registry),
it is possible to provide a direct link to the plugin descriptor (typically, named `meta.yaml`) by using
the reference field:

```yaml
...
  components:
   - alias: exec-plugin
     type: chePlugin
     reference: https://raw.githubusercontent.com.../plugin/1.0.1/meta.yaml
```

Please note it's not possible to mix id and reference in single plugin definition, they are mutually exclusive. 

For each of the types above, it is also possible to specify the container(s) memory limit as follows:

```yaml
...
  components:
   - alias: exec-plugin
     type: chePlugin
     id: eclipse/che-machine-exec-plugin/0.0.1
     memoryLimit: 256M
```

This limit will be applied to each container of given plugin.

A plugin may need to be precisely tuned, and in such cases, plugin preferences used.
The example shows how JVM  configured with plugin's preferences.
```
...
-
  id: redhat/java/0.38.0
  type: chePlugin
  preferences:
     java.jdt.ls.vmargs: '-noverify -Xmx1G -XX:+UseG1GC -XX:+UseStringDeduplication'
```

#### kubernetes/openshift

More complex component type, which allows applying configuration from kubernetes/openshift lists. Content of the component may be provided either via the `reference` attribute, which points to the file with component content.

```yaml
...
  components:
    - alias: mysql
      type: kubernetes
      reference: petclinic.yaml
      selector:
        app.kubernetes.io/name: mysql
        app.kubernetes.io/component: database
        app.kubernetes.io/part-of: petclinic
```

Alternatively, if you need to post devfile with such components to REST API, contents of K8S/Openshift list can be embedded into devfile using `referenceContent` field:

```yaml
...
  components:
    - alias: mysql
      type: kubernetes
      reference: petclinic.yaml
      referenceContent: |
           kind: List
           items:
            -
             apiVersion: v1
             kind: Pod
             metadata:
              name: ws
             spec:
              containers:
              ... etc
```

As with `dockerimage` component described below, it is possible to override the entry point of the containers contained in the Kubernetes/Openshift list using the `command` and `args` properties (as
[understood](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#notes) by Kubernetes). Of course, there can be more containers in the list (contained in pods or pod templates of deployments), and so there needs to be a way of selecting which containers to apply the entry point changes.

The entry points can be defined, for example, like this:

```yaml
...
  components:
    - alias: appDeployment
      type: kubernetes
      reference: app-deployment.yaml
      entrypoints:
      - parentName: mysqlServer
        command: ['sleep']
        args: ['infinity']
      - parentSelector:
          app: prometheus
        args: ['-f', '/opt/app/prometheus-config.yaml']
```

You can see that the `entrypoints` list contains constraints for picking the containers along with
the command/args to apply to them. In the example above, the constraint is `parentName: mysqlServer` which cause the command to apply to all containers defined in any parent object called
`mysqlServer`. The parent object is assumed to be a top-level object in the list defined in the
referenced file, e.g. `app-deployment.yaml` in the example above.

Other types of constraints (and their combinations) are possible:

* `containerName` - the name of the container
* `parentName` - the name of the parent object that (indirectly) contains the containers to override
* `parentSelector` - the set of labels the parent object needs to have

A combination of these constraints used to precisely locate the containers inside the referenced the Kubernetes list.

#### docker image

The component type which allows defining docker image based configuration of the container in a workspace. Devfile can only contain one component with the `dockerimage` type.

```yaml
 ...
 components:
   - alias: maven
     type: dockerimage
     image: eclipe/maven-jdk8:latest
     volumes:
       - name: mavenrepo
         containerPath: /root/.m2
     env:
       - name: ENV_VAR
         value: value
     endpoints:
       - name: maven-server
         port: 3101
         attributes:
           protocol: http
           secure: 'true'
           public: 'true'
           discoverable: 'false'
     memoryLimit: 1536M
     command: ['tail']
     args: ['-f', '/dev/null']
```

### Commands expanded

Devfile allows specifying commands set to be available for execution in the workspace. Each command may contain a subset of actions, which are related to a specific component, in whose container is executed.

```yaml
 ...
 commands:
   - name: build
     actions:
       - type: exec
         component: mysql
         command: mvn clean
         workdir: /projects/spring-petclinic
```

### Devfile attributes

Devfile attributes used to configure some features.

#### Editor free
If the editor is not specified, Devfile then defaults one provided. In the case of no editor is needed, then use `editorFree` attribute. The default value is `false` and means Devfile needs the default editor provisioned if no one is defined.

Example of Devfile without the editor:
```yaml
apiVersion: 1.0.0
metadata:
  name: petclinic-dev-environment
components:
  - alias: myApp
    type: kubernetes
    local: my-app.yaml
attributes:
  editorFree: true
```

#### Ephemeral mode

By default, volumes and PVCs specified in Devfile are bound to host folder to persist data even after a container restart. Sometimes it may be needed to disable data persistence for some reason, like when volume backend is incredibly slow, and it is needed to make workspace faster.

To achieve a `persistVolumes` devfile attribute should be used. The default value is `true`, and in case of `false` `emptyDir` volumes used for configured volumes and PVC.

Example of Devfile with ephemeral mode enabled:

```yaml
apiVersion: 1.0.0
metadata:
  name: petclinic-dev-environment
projects:
  - name: petclinic
    source:
      type: git
      location: 'https://github.com/che-samples/web-java-spring-petclinic.git'
attributes:
  persistVolumes: false
```

### Live working examples

  - [NodeJS simple "Hello World" example](https://che.openshift.io/f?url=https://raw.githubusercontent.com/redhat-developer/devfile/master/samples/web-nodejs-sample/devfile.yaml)
  - [NodeJS Application with Mongo DB example](https://che.openshift.io/f?url=https://raw.githubusercontent.com/redhat-developer/devfile/master/samples/web-nodejs-with-db-sample/devfile.yaml)
  - [Java Spring-Petclinic example](https://che.openshift.io/f?url=https://raw.githubusercontent.com/redhat-developer/devfile/master/samples/web-java-spring-petclinic/devfile.yaml)
  - [Theia frontend plugin example](https://che.openshift.io/f?url=https://raw.githubusercontent.com/redhat-developer/devfile/master/samples/theia-hello-world-frontend-plugin/devfile.yaml)
