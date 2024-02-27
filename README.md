# Helm Char learning path

:link: [Documents](https://helm.sh/docs/)

## Get Charts
Visit :link: [Artifact Hub](https://artifacthub.io/) to explore Helm charts from numerous public repositories.

[Helm Command cheat sheet](https://helm.sh/docs/intro/cheatsheet/)

## Best Practice:

### Chart Names

Chart names must be lower case letters and numbers. Words may be separated with dashes (-):

Examples:

```R
drupal
nginx-lego
aws-cluster-autoscaler
```

Neither uppercase letters nor underscores can be used in chart names. Dots should not be used in chart names.

### Version Numbers
Wherever possible, Helm uses SemVer 2 to represent version numbers. (Note that Docker image tags do not necessarily follow SemVer, and are thus considered an unfortunate exception to the rule.)

When SemVer versions are stored in Kubernetes labels, we conventionally alter the + character to an _ character, as labels do not allow the + sign as a value.

### Formatting YAML

YAML files should be indented using two spaces (and never tabs).

### Usage of the Words Helm and Chart
There are a few conventions for using the words Helm and helm.

Helm refers to the project as a whole
helm refers to the client-side command
The term chart does not need to be capitalized, as it is not a proper noun
However, Chart.yaml does need to be capitalized because the file name is case sensitive
When in doubt, use Helm (with an uppercase 'H').

### Values

#### Naming Conventions
Variable names should begin with a lowercase letter, and words should be separated with camelcase:

Correct:
```R
chicken: true
chickenNoodleSoup: true
```

Incorrect:

```R
Chicken: true  # initial caps may conflict with built-ins
chicken-noodle-soup: true # do not use hyphens in the name
```

:note: Note that all of Helm's built-in variables begin with an uppercase letter to easily distinguish them from user-defined values: `.Release.Name, .Capabilities.KubeVersion`.


#### Flat or Nested Values
YAML is a flexible format, and values may be nested deeply or flattened.

Nested:

```R
server:
  name: nginx
  port: 80
```

Flat:

```R
serverName: nginx
serverPort: 80
```

In most cases, flat should be favored over nested. The reason for this is that it is simpler for template developers and users.

For optimal safety, a nested value must be checked at every level:

```R

{{ if .Values.server }}
  {{ default "none" .Values.server.name }}
{{ end }}

```

For every layer of nesting, an existence check must be done. But for flat configuration, such checks can be skipped, making the template easier to read and use.

```R
{{ default "none" .Values.serverName }}
```

When there are a large number of related variables, and at least one of them is non-optional, nested values may be used to improve readability.

### Make Types Clear

YAML's type coercion rules are sometimes counterintuitive. For example, foo: false is not the same as foo: "false". Large integers like foo: 12345678 will get converted to scientific notation in some cases.

The easiest way to avoid type conversion errors is to be explicit about strings, and implicit about everything else. Or, in short, quote all strings.

Often, to avoid the integer casting issues, it is advantageous to store your integers as strings as well, and use `{{ int $value }}` in the template to convert from a string back to an integer.

In most cases, explicit type tags are respected, so foo: !!string 1234 should treat 1234 as a string. However, the YAML parser consumes tags, so the type data is lost after one parse.

### Consider How Users Will Use Your Values

There are three potential sources of values:

- A chart's values.yaml file
- A values file supplied by helm install `-f` or helm upgrade `-f`
- The values passed to a `--set` or `--set-string` flag on helm install or helm upgrade.

When designing the structure of your values, keep in mind that users of your chart may want to override them via either the `-f` flag or with the `--set` option.

Since `--set` is more limited in expressiveness, the first guidelines for writing your values.yaml file is make it easy to override from `--set`.

For this reason, it's often better to structure your values file using maps.

Difficult to use with `--set`:

```R
servers:
  - name: foo
    port: 80
  - name: bar
    port: 81
```

The above cannot be expressed with --set in Helm <=2.4. In Helm 2.5, accessing the port on foo is `--set servers[0].port=80`. Not only is it harder for the user to figure out, but it is prone to errors if at some later time the order of the servers is changed.

Easy to use:

```R
servers:
  foo:
    port: 80
  bar:
    port: 81
```

Accessing foo's port is much more obvious: `--set servers.foo.port=80`.

### Document : values.yaml
Every defined property in values.yaml should be documented. The documentation string should begin with the name of the property that it describes, and then give at least a one-sentence description.

Incorrect:

```R
# the host name for the webserver
serverHost: example
serverPort: 9191
```

Correct:

```R
 serverHost is the host name for the webserver
serverHost: example
# serverPort is the HTTP listener port for the webserver
serverPort: 9191
```

Beginning each comment with the name of the parameter it documents makes it easy to grep out documentation, and will enable documentation tools to reliably correlate doc strings with the parameters they describe.

### Templates

This part of the Best Practices Guide focuses on templates.

**Structure** of templates/
The templates/ directory should be structured as follows:

- Template files should have the extension .yaml if they produce YAML output. The extension .tpl may be used for template files that produce no formatted content.
- Template file names should use dashed notation (my-example-configmap.yaml), not **camelcase**.
- Each resource definition should be in its own template file.
- Template file names should reflect the resource kind in the name. e.g. foo-pod.yaml, bar-svc.yaml

#### Names of Defined Templates
**Defined templates** (templates created inside a `{{ define }}` directive) are globally accessible. That means that a chart and all of its subcharts will have access to all of the templates created with `{{ define }}`.

For that reason, all defined template names should be namespaced.

Correct:

```R

{{- define "nginx.fullname" }}
{{/* ... */}}
{{ end -}}

```

Incorrect:

```R

{{- define "fullname" -}}
{{/* ... */}}
{{ end -}}

```

It is highly recommended that new charts are created via helm create command as the template names are automatically defined as per this best practice.

Formatting Templates
Templates should be indented using two spaces (never tabs).

Template directives should have whitespace after the opening braces and before the closing braces:

Correct:

```R
{{ .foo }}
{{ print "foo" }}
{{- print "bar" -}}
```

Incorrect:

```R
{{.foo}}
{{print "foo"}}
{{-print "bar"-}}
```

Templates should chomp whitespace where possible:

```R

foo:
  {{- range .Values.items }}
  {{ . }}
  {{ end -}}

```

# Lesson : 1 What is Helm

Managing Kubernetes deployments can be a challenging task for developers. Manually undertaking tasks such as managing application updates, rollbacks, dependencies, and configurations is time-consuming and error-prone. Fortunately, there is a tool that can make this process much easier - Helm.

Helm is a powerful tool that allows developers to package and deploy their applications quickly and easily. It also provides a standardized way of managing dependencies and configurations.

Helm is rather easy to use as a command-line tool. You just tell it to install this, uninstall that, upgrade something, roll back to a previous state, and so on, and it proceeds to do all the heavy lifting behind the scenes. It’s an automation tool where we, the human operators, specify our desired end result, the destination.

With Helm, it doesn’t matter if 5, 10, 20, or 50 actions are necessary to achieve that end result; it will go through all the required steps without bothering us with the details. Helm does its job with the help of charts.

# Lesson : 2 Installation and configuration

:link: [Helm installation link](https://helm.sh/docs/intro/install/)

# Lesson : 3 A quick note about Helm2 vs Helm3

No More Tiller

When **Helm 2** was around, Kubernetes lacked features such as Role-Based Access Control and Custom Resource Definitions. To allow Helm to do its magic, an extra component called Tiller, had to be installed in the Kubernetes cluster. So, whenever you wanted to install a Helm chart, you used the Helm (client) program installed on your local computer. This communicated with Tiller that was running on some server. Tiller, in turn, communicated with Kubernetes and proceeded to take action to make whatever you requested happen. So, Tiller was the middleman

Besides the fact that an extra component sitting between you and Kubernetes adds complexity, there were also some security concerns. By default, Tiller was running in “God mode” or otherwise said, it had the privileges to do anything it wanted. This was good since it allowed it to make whatever changes necessary in your Kubernetes cluster to install your charts. But this was bad since it allowed any user with Tiller access to do whatever they wanted in the cluster.

![helm](/helm-pic/helm2.png)

if there is manual changes perform then Helm2 will not able to create a revision and when apply rollback it won't effect.

![helm](/helm-pic/helm2A.png)

Also, After cool stuff like Role-Based Access Control (RBAC) and Custom Resource Definitions appeared in Kubernetes, the need for Tiller decreased, so it was removed entirely in Helm 3. Now there’s nothing sitting between Helm and the cluster.

That is, when you use the Helm program on your local computer, this connects directly to the cluster (Kubernetes API server) and starts to work its magic.

Furthermore, with RBAC, security is much improved and any user can be limited in what they can do with Helm. Before you had to set these limits in Tiller and that was not the best option. But with RBAC built from the ground up to fine-tune user permissions in Kubernetes, it’s now straightforward to do. As far as Kubernetes is concerned, it doesn’t matter if the user is trying to make changes within the cluster with kubectl or with helm commands. The user requesting the changes has the same RBAC-allowed permissions whatever tool they use.

**Helm 3**, on the other hand, is more intelligent. It compares the chart we want to revert to, the chart currently in use, and also, the live state (how our Kubernetes objects currently look like, their declarations in .yaml form). This is where that fancy “Three-way strategic merge patch” name comes from. By also looking at the live state, it notices that the live state replica count is at 0, but the replica count in revision 1 we want to revert to is at 3, so it makes necessary changes to come back to the original state.

Besides rollbacks, there are also things like upgrades to consider, where Helm 2 was also lacking. For example, say you install a chart. But then you make some changes to some of the Kubernetes objects installed. It all works nicely until you perform an upgrade. Helm 2 looks at the old chart and the new chart you want to upgrade to. All your changes will be lost since they don’t exist in the old chart or the new chart. But Helm 3, as mentioned, looks at the charts and also at the live state. It notices you added some stuff of your own so it performs the upgrade while preserving anything you might have added.

![helm](/helm-pic/helm3.png)

# Lesson 4 : Helm Components

![helm](/helm-pic/helmcomp.png)

**Helm Client**
When it comes to the helm, the main component we will interact with is the helm client, and we can install the helm client in any system.


Once the installation is done, that’s going to facilitate the interaction between the user and the helm components, as well as it’s going to facilitate the communication to the Kubernetes cluster.

**Helm Charts**
The following essential component of the helm is the charts, which will be the building block of the entire helm, making the entire application definition within the charts. The charts provide various features, how can define the applications within the Kubernetes environment.

*Charts have a collection of files*, different structures to define values, build a sub chart, and represent any complex environment within the Kubernetes cluster; it has the options.

Helm configuration files are charts and consist of a few YAML files with metadata and templates rendered into Kubernetes manifest files.

We will be having lots and lots of detailed discussions about every component in upcoming tutorials.

So charts are nothing but how a Kubernetes application should get deployed within the Kubernetes cluster. That’s going to be the definition file, and once the file is created, we need to manage the files.

So that we can share with other team members as well as the next version can be created and that we call repositories.

![helm](/helm-pic/helmcomp2.png)

**Repositories**

Repositories are nothing but a storage environment where various charts of various applications can be stored and shared with other team members or across the teams.

[artifacthub registry](https://artifacthub.io/) : contain thousands of chart.

**Release**
Once the chart is released into the environment, we call it a release, and we can have multiple versions of the release in multiple environments and all that the helm client will manage.

What releases were installed, in what environment, what values were used and any value that I wanted to change and make a next release upgrade or rollback?

Will be handle everything, and release is nothing but the instance of a chart deployed into the environment. So if I had to collaborate all these terminologies as a workflow, we could visualize it this way.

A release is a single installation of an application using Helm chart. Within each release, You can have multiple revision, and each revision is like a snapshot of the application.

Everytime, a chang is made to the application such as an upgrade of the image or change of replicas or configuration objects, a new revision is created. Just like how we can find all kinds of images on Docker Hub.

We can find Helm charts in a public repository. We can easily download publicly available charts for various applications. These are readily available, and we can use them to deploy applications.

On our Cluster. Finally, to kep track of what it did in our cluster, sucj as the releases that it installed, the charts used, revision state and so on...

Helm need a place to save it's data. This data is known as metadata. This is data about data.

![helm](/helm-pic/helmcomp3.png)

You can use same chart and deploy many release as much as you wants.

![helm](/helm-pic/helmcomp4.png)


***It won't be too useful if Helm would save this on our local computer. If another person would need to work with our release through Helm. They would need a copy of this data. Instead helm does a smart thing and save this metadata directly in our Kubernetes cluster as Kubernetes secrets***

![helm](/helm-pic/helmcomp1.png)

This way the data survives and as long as the Kubernetes cluster survives, Everyone from pur team can access it. They can do Helm upgrades or whatever it is that they want to do.

Helm will always know about everything you did in this cluster and will be able t keep track of every action every step of the way, Since it always has its metadata available to perform every action.

# Lesson : 4 Helm charts

Helm uses a packaging format called charts. A chart is a collection of files that describe a related set of Kubernetes resources. A single chart might be used to deploy something simple, like a memcached pod, or something complex, like a full web app stack with HTTP servers, databases, caches, and so on.

Charts are created as files laid out in a particular directory tree. They can be packaged into versioned archives to be deployed.

If you want to download and look at the files for a published chart, without installing it, you can do so with 

```R
helm pull chartrepo/chartname.
```

**The Chart File Structure**
A chart is organized as a collection of files inside of a directory. The directory name is the name of the chart (without versioning information). Thus, a chart describing WordPress would be stored in a wordpress/ directory.

```R
wordpress/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  values.yaml         # The default configuration values for this chart
  values.schema.json  # OPTIONAL: A JSON Schema for imposing a structure on the values.yaml file
  charts/             # A directory containing any charts upon which this chart depends.
  crds/               # Custom Resource Definitions
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

**The Chart.yaml File**
The Chart.yaml file is required for a chart. It contains the following fields:

```R
apiVersion: The chart API version (required)
name: The name of the chart (required)
version: A SemVer 2 version (required)
kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
description: A single-sentence description of this project (optional)
type: The type of the chart (optional)
keywords:
  - A list of keywords about this project (optional)
home: The URL of this projects home page (optional)
sources:
  - A list of URLs to source code for this project (optional)
dependencies: # A list of the chart requirements (optional)
  - name: The name of the chart (nginx)
    version: The version of the chart ("1.2.3")
    repository: (optional) The repository URL ("https://example.com/charts") or alias ("@repo-name")
    condition: (optional) A yaml path that resolves to a boolean, used for enabling/disabling charts (e.g. subchart1.enabled )
    tags: # (optional)
      - Tags can be used to group charts for enabling/disabling together
    import-values: # (optional)
      - ImportValues holds the mapping of source values to parent key to be imported. Each item can be a string or pair of child/parent sublist items.
    alias: (optional) Alias to be used for the chart. Useful when you have to add the same chart multiple times
maintainers: # (optional)
  - name: The maintainers name (required for each maintainer)
    email: The maintainers email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
icon: A URL to an SVG or PNG image to be used as an icon (optional).
appVersion: The version of the app that this contains (optional). Needn't be SemVer. Quotes recommended.
deprecated: Whether this chart is deprecated (optional, boolean)
annotations:
  example: A list of annotations keyed by name (optional).
```

**Charts and Versioning**

Every chart must have a version number. A version must follow the SemVer 2 standard. Unlike Helm Classic, Helm v2 and later uses version numbers as release markers. Packages in repositories are identified by name plus version.

![helm](/helm-pic/Screenshot-00001.png)

![helm](/helm-pic/chart1.png)

Helm charts store their **dependencies** in 'charts/'. For chart developers, it is often easier to manage dependencies in 'Chart.yaml' which declares all dependencies.

The dependency commands operate on that file, making it easy to synchronize between the desired dependencies and the actual dependencies stored in the 'charts/' directory.

For example, this Chart.yaml declares two dependencies:

```R
# Chart.yaml
dependencies:
- name: nginx
  version: "1.2.3"
  repository: "https://example.com/charts"
- name: memcached
  version: "3.2.1"
  repository: "https://another.example.com/charts"
```

The 'name' should be the name of a chart, where that name must match the name in that chart's 'Chart.yaml' file.

The 'version' field should contain a semantic version or version range.

The 'repository' URL should point to a Chart Repository. Helm expects that by appending '/index.yaml' to the URL, it should be able to retrieve the chart repository's index. Note: 'repository' can be an alias. The alias must start with 'alias:' or '@'.

Starting from 2.2.0, repository can be defined as the path to the directory of the dependency charts stored locally. The path should start with a prefix of "file://". For example,

```R
# Chart.yaml
dependencies:
- name: nginx
  version: "1.2.3"
  repository: "file://../dependency_chart/nginx"
```

If the dependency chart is retrieved locally, it is not required to have the repository added to helm by "helm add repo". Version matching is also supported for this case.

Helm chart structure.

![helm](/helm-pic/Screenshot-00002.png)

# Lesson : 5 Working with Helm: basics

**Helm basic command**

[Helm Command cheat sheet](https://helm.sh/docs/intro/cheatsheet/)

![helm](/helm-pic/Screenshot-00003.png)

# Lesson : 6 Customizing chart parameters

![helm](/helm-pic/customize-chart.png)

You can use --set parameter that replace the original value on values.yaml file.

![helm](/helm-pic/customize-chart1.png)

You can create **custom-values** file and pass with **-values** parameter.

![helm](/helm-pic/customize-chart2.png)

If we wish to update existing values file.
 
![helm](/helm-pic/Screenshot-00005.png)

# Lesson : 7 Lifecycle management with Helm

helm history will show all the details as many peoples are working so we does not know the details.

![helm](/helm-pic/Screenshot-00007.png)

Helm rollback allow to revert back to previous version. It actually destroy the current version and create new version ,which is previous version. 

![helm](/helm-pic/rollback.png)

Upgrade / rollback will not impact any database / persistence volume data. If you wish to take any action. We can use helm hook.

![helm](/helm-pic/Screenshot-00008.png)

# Lesson : 8 Writing a Helm chart

![helm](/helm-pic/Screenshot-00009.png)

![helm](/helm-pic/Screenshot-00010.png)

![helm](/helm-pic/Screenshot-00011.png)

Templatize the name of deployment.

![helm](/helm-pic/Screenshot-00012.png)
![helm](/helm-pic/Screenshot-00013.png)

As we can see that Predefine object start with capital letter and user define variable will start with small letter, that you can define with values.yaml file.


![helm](/helm-pic/Screenshot-00014.png)


![helm](/helm-pic/Screenshot-00015.png)
![helm](/helm-pic/Screenshot-00016.png)
![helm](/helm-pic/Screenshot-00017.png)
![helm](/helm-pic/Screenshot-00018.png)



![helm](/helm-pic/Screenshot-00019.png)
![helm](/helm-pic/Screenshot-00020.png)

# Lesson : 9 Testing Chart

**Lint**:
Linting in the context of Helm charts refers to the process of checking a chart for potential issues and errors before deploying it to a Kubernetes cluster. The helm lint command is used for this purpose.

Here's how you can use helm lint:

Open a terminal window.

Navigate to the directory where your Helm chart is located. For example:

```R
cd /path/to/your/chart

Run the helm lint command:

helm lint .

```

**Replace .** with the path to your Helm chart directory if you are not already in that directory.

The helm lint command will analyze the chart files and provide feedback on any potential issues or errors. If everything is correct, you should see a message indicating that the linting passed without errors.

Keep in mind that helm lint checks for common problems in your chart, such as missing or misconfigured values in your values.yaml file, issues with the chart structure, and more.

It's a good practice to run helm lint before deploying a chart to catch any potential problems early in the development process.

![helm](/helm-pic/Screenshot-00021.png)



![helm](/helm-pic/Screenshot-00022.png)
![helm](/helm-pic/Screenshot-00023.png)

**Typo error**

![helm](/helm-pic/Screenshot-00024.png)

**Template testing**:

The helm template command is used to render a Helm chart locally into Kubernetes manifests without actually deploying it to a cluster. This is useful for previewing the rendered Kubernetes YAML files that will be generated based on the Helm chart and its values.

Here's how you can use the helm template command:

Open a terminal window.

Navigate to the directory where your Helm chart is located. For example:

```R

cd /path/to/your/chart
Run the helm template command, specifying the release name and optionally the values file:

helm template myrelease .

```

*Replace myrelease with the desired release name, and . with the path to your Helm chart directory if you are not already in that directory.*

- If you have a separate values file, you can specify it using the -f option:

```R
helm template myrelease -f values.yaml .
```

Replace values.yaml with the path to your custom values file.

The helm template command will generate the Kubernetes manifest files based on the Helm chart and the provided values. The output will be displayed in the terminal.

If you want to save the generated manifests to a file, you can use the > operator to redirect the output:

```R
helm template myrelease . > mymanifests.yaml
```

Now, you can review the contents of mymanifests.yaml to see how your Helm chart would be translated into Kubernetes resources without actually deploying anything to the cluster. This can be useful for debugging, verifying configurations, or inspecting the rendered output before deploying to a Kubernetes cluster.

![helm](/helm-pic/Screenshot-00025.png)
![helm](/helm-pic/Screenshot-00026.png)

Why RELEASE-NAME is in capital letter. Because we did not pass release name with command execution. Once passed it will replace the RELEASE-NAME with passing parameter.

![helm](/helm-pic/Screenshot-00027.png)
![helm](/helm-pic/Screenshot-00028.png)

**Error** : name is not in proper space alignment.
![helm](/helm-pic/Screenshot-00029.png)

**Dry run**

The helm install command has a **--dry-run** option, which allows you to simulate an installation without actually deploying resources to the Kubernetes cluster. This is useful for previewing the changes that would be applied to your cluster based on a Helm chart and its values.

The **--dry-run** option ensures that no changes are made to the cluster, and the **--debug** option provides additional output for debugging purposes.


![helm](/helm-pic/Screenshot-00030.png)


![helm](/helm-pic/Screenshot-00031.png)

# Lesson : 10 Function

Helm allows you to use functions within your Helm charts to perform various operations, manipulate data, and create dynamic configurations. Helm provides a set of built-in functions that you can use in your templates. Here are some common Helm functions:

1] Template Functions:

{{ .Release.Name }}: Returns the name of the release.
{{ .Release.Namespace }}: Returns the namespace of the release.
{{ .Chart.Name }}: Returns the name of the chart.
{{ .Chart.Version }}: Returns the version of the chart.
{{ .Values.someKey }}: Accesses values from the values.yaml file or custom values.

2] Control Flow Functions:

{{ if .Values.enableFeature }} ... {{ else }} ... {{ end }}: Conditional statements.
{{ range .Values.items }} ... {{ end }}: Iterates over a list.

3] String Functions:

{{ printf "Hello, %s!" .Values.name }}: Formats a string.
{{ toYaml .Values.someMap }}: Converts a map to YAML format.

4] Math Functions:

{{ add .Values.number1 .Values.number2 }}: Adds two numbers.
{{ sub .Values.number1 .Values.number2 }}: Subtracts two numbers.

5] Include Function:

{{ include "mychart.mytemplate" . }}: Includes another template file in the chart.

6] Built-in Functions:

env: Reads an environment variable.
tpl: Renders a template string.

Example of using a function in a Helm chart template:

yaml file

```R
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  myKey: {{ .Values.myValue | default "default" | quote }}
```  


In this example, the {{ .Values.myValue | default "default" | quote }} expression does the following:

Accesses the value of myValue from the values.yaml file or custom values.
If the value is not set, it defaults to "default."
The quote function is used to ensure that the value is properly quoted.

Feel free to explore the Helm documentation for a comprehensive list of built-in functions and their usage

[Function](https://helm.sh/docs/chart_template_guide/functions_and_pipelines/)

![helm](/helm-pic/Screenshot-00032.png)





![helm](/helm-pic/Screenshot-00033.png)
![helm](/helm-pic/Screenshot-00034.png)
![helm](/helm-pic/Screenshot-00035.png)
![helm](/helm-pic/Screenshot-00036.png)
![helm](/helm-pic/Screenshot-00037.png)
![helm](/helm-pic/Screenshot-00038.png)
![helm](/helm-pic/Screenshot-00039.png)
![helm](/helm-pic/Screenshot-00040.png)
![helm](/helm-pic/Screenshot-00041.png)
![helm](/helm-pic/Screenshot-00042.png)
![helm](/helm-pic/Screenshot-00043.png)
![helm](/helm-pic/Screenshot-00044.png)
![helm](/helm-pic/Screenshot-00045.png)
![helm](/helm-pic/Screenshot-00046.png)
![helm](/helm-pic/Screenshot-00047.png)
![helm](/helm-pic/Screenshot-00048.png)
![helm](/helm-pic/Screenshot-00049.png)

# Lesson : 11 Conditionals

![helm](/helm-pic/Screenshot-00050.png)
![helm](/helm-pic/Screenshot-00051.png)
![helm](/helm-pic/Screenshot-00052.png)
![helm](/helm-pic/Screenshot-00053.png)
![helm](/helm-pic/Screenshot-00054.png)
![helm](/helm-pic/Screenshot-00055.png)
![helm](/helm-pic/Screenshot-00056.png)
![helm](/helm-pic/Screenshot-00057.png)
![helm](/helm-pic/Screenshot-00058.png)
![helm](/helm-pic/Screenshot-00059.png)
![helm](/helm-pic/Screenshot-00060.png)
![helm](/helm-pic/Screenshot-00061.png)

# Lesson : 12 With Blocks


In Helm charts, the with block is used to set a local scope for a particular variable or a set of variables. It allows you to work with a subset of the data, making the template more readable and concise.

![helm](/helm-pic/Screenshot-00062.png)
![helm](/helm-pic/Screenshot-00063.png)
![helm](/helm-pic/Screenshot-00064.png)
![helm](/helm-pic/Screenshot-00065.png)
![helm](/helm-pic/Screenshot-00066.png)
![helm](/helm-pic/Screenshot-00067.png)
![helm](/helm-pic/Screenshot-00068.png)
![helm](/helm-pic/Screenshot-00069.png)
![helm](/helm-pic/Screenshot-00070.png)
![helm](/helm-pic/Screenshot-00071.png)
![helm](/helm-pic/Screenshot-00072.png)
![helm](/helm-pic/Screenshot-00073.png)
![helm](/helm-pic/Screenshot-00074.png)
![helm](/helm-pic/Screenshot-00075.png)
![helm](/helm-pic/Screenshot-00076.png)
![helm](/helm-pic/Screenshot-00077.png)
![helm](/helm-pic/Screenshot-00078.png)
![helm](/helm-pic/Screenshot-00079.png)
![helm](/helm-pic/Screenshot-00080.png)
![helm](/helm-pic/Screenshot-00081.png)
![helm](/helm-pic/Screenshot-00082.png)
![helm](/helm-pic/Screenshot-00083.png)
![helm](/helm-pic/Screenshot-00084.png)
![helm](/helm-pic/Screenshot-00085.png)
![helm](/helm-pic/Screenshot-00086.png)
![helm](/helm-pic/Screenshot-00087.png)
![helm](/helm-pic/Screenshot-00088.png)
![helm](/helm-pic/Screenshot-00089.png)
![helm](/helm-pic/Screenshot-00090.png)

![helm](/helm-pic/Screenshot-00092.png)
![helm](/helm-pic/Screenshot-00093.png)
![helm](/helm-pic/Screenshot-00094.png)
![helm](/helm-pic/Screenshot-00095.png)
![helm](/helm-pic/Screenshot-00096.png)
![helm](/helm-pic/Screenshot-00097.png)
![helm](/helm-pic/Screenshot-00098.png)
![helm](/helm-pic/Screenshot-00099.png)
![helm](/helm-pic/Screenshot-00100.png)
![helm](/helm-pic/Screenshot-00101.png)
![helm](/helm-pic/Screenshot-00102.png)
![helm](/helm-pic/Screenshot-00103.png)
![helm](/helm-pic/Screenshot-00104.png)
![helm](/helm-pic/Screenshot-00105.png)
![helm](/helm-pic/Screenshot-00106.png)
![helm](/helm-pic/Screenshot-00107.png)
![helm](/helm-pic/Screenshot-00108.png)
![helm](/helm-pic/Screenshot-00109.png)
![helm](/helm-pic/Screenshot-00110.png)
![helm](/helm-pic/Screenshot-00111.png)
![helm](/helm-pic/Screenshot-00112.png)
![helm](/helm-pic/Screenshot-00113.png)
![helm](/helm-pic/Screenshot-00114.png)
![helm](/helm-pic/Screenshot-00115.png)
![helm](/helm-pic/Screenshot-00116.png)
![helm](/helm-pic/Screenshot-00117.png)
![helm](/helm-pic/Screenshot-00118.png)
![helm](/helm-pic/Screenshot-00119.png)
![helm](/helm-pic/Screenshot-00120.png)
![helm](/helm-pic/Screenshot-00121.png)
![helm](/helm-pic/Screenshot-00122.png)
![helm](/helm-pic/Screenshot-00123.png)
![helm](/helm-pic/Screenshot-00124.png)
![helm](/helm-pic/Screenshot-00125.png)
![helm](/helm-pic/Screenshot-00126.png)
![helm](/helm-pic/Screenshot-00127.png)
![helm](/helm-pic/Screenshot-00128.png)
![helm](/helm-pic/Screenshot-00129.png)
![helm](/helm-pic/Screenshot-00130.png)
![helm](/helm-pic/Screenshot-00131.png)
![helm](/helm-pic/Screenshot-00132.png)
![helm](/helm-pic/Screenshot-00133.png)
![helm](/helm-pic/Screenshot-00134.png)
![helm](/helm-pic/Screenshot-00135.png)
![helm](/helm-pic/Screenshot-00136.png)
![helm](/helm-pic/Screenshot-00137.png)
![helm](/helm-pic/Screenshot-00138.png)
![helm](/helm-pic/Screenshot-00139.png)
![helm](/helm-pic/Screenshot-00140.png)
![helm](/helm-pic/Screenshot-00141.png)
![helm](/helm-pic/Screenshot-00142.png)
![helm](/helm-pic/Screenshot-00143.png)
![helm](/helm-pic/Screenshot-00144.png)
![helm](/helm-pic/Screenshot-00145.png)
![helm](/helm-pic/Screenshot-00146.png)
![helm](/helm-pic/Screenshot-00147.png)
![helm](/helm-pic/Screenshot-00148.png)
![helm](/helm-pic/Screenshot-00149.png)

[Pushing chart to repo](https://www.linkedin.com/pulse/creating-publishing-helm-chart-gaurav-sharma/)

[Pushing chart to repo](https://www.devopsschool.com/blog/helm-tutorial-how-to-publish-chart-at-artifacthub/)

[Uploading Chart](https://kodekloud.com/blog/uploading-a-helm-chart/)

# Lesson : 13 Helm Best Practices with Examples

## 1. Chart Names and Versions

**Chart Names**

Chart names should only contain lowercase letters (and, if necessary, numbers). Also, if your name has to have more than one word, separate these words with hyphens '-', for example, my-nginx-app.

**Valid names:**

```R
nginx
wordpress
wordpress-on-nginx
```

**Invalid names:**

```R
Nginx
Wordpress
wordpressOnNginx
wordpress_on_nginx
```

**Version Numbers**
Helm recommends what is called SemVer2 notation (Semantic Versioning 2.0.0). The short explanation is this: version numbers are in the form of 1.2.3 (MAJOR.MINOR.PATCH). The first number represents the major version number, the second the minor, and the third is the patch number.

Let’s say you just created a new application last night. This could be version 0.1.0. You find a security bug, and you patch it. This would be version 0.1.1. Later, you find another bug, and you patch that too. You arrive at version 0.1.2. You make some very small changes to your app - like maybe you change a function so that it executes 2% faster - this could be version 0.2.0.

Notice how when you update the minor version number, the patch number gets reset to 0. And finally, you make a huge update to your app, adding a lot of new features and changing some functionalities. This way, you arrive at version 1.0.0.

## 2. values.yaml file

**Variable Names**
All variable names in the values.yaml file should begin with lowercase letters.

**Valid name:**

```R
enabled: false
```

**Invalid name:**

```R
Enabled: false
```

Often, your variable will need to contain multiple words to describe it better. Use what is called **camelcase**: *the first word starts with a lowercase letter, but the next ones all start with a capital letter.*

**Examples of valid names:**

```R
replicaCount: 3
wordpressUsername: user
wordpressSkipInstall: false
```

**Invalid names:**

```R
ReplicaCount: 3
wordpress-username: user
wordpress_skip_install: false
```

**Flat vs. Nested Values**
We saw in our exercises that we can have nested values like this:

```R
image:
  repository:
  pullPolicy: IfNotPresent
  tag: "1.16.0"
```

This provides some logical grouping. In the snippet above `image` is the **parent** variable with three children values. But this can be rewritten to a **flat** format, like this:

```R
imageRepository:
imagePullPolicy: IfNotPresent
imageTag: "1.16.0"
```

When to use nested format:

**If you have many related variables and at least one of them is always used.**

When to use flat format:

**If you have very few related variables
If all your related variables are optional**

**Quote Strings**
Always wrap your string values between quote signs. For instance:

```enabled: "false"```
The configuration above assigns a string value 'false' to the enabled variable. Consider the value assignment below:

```enabled: false```

This will assign a boolean type to the enabled variable.

**Large Integers**

Assume you have this:

```number: 1234567```

When parsing your template files, this value might get converted to scientific notation and end up as 1.234567e+06. If you run into this issue (which may happen especially with large numbers), you can define a number as a string.

```number: "1234567"```

Then, when you need to use this variable in your template files, prefix it with the type conversion function int (or int64 for very large numbers).

```number: {{ int .Values.number }}"```

This way, the string “1234567” will get converted into the number 1234567 and won’t end up as 1.234567e+06.

You can see other type conversion functions [here](https://helm.sh/docs/chart_template_guide/function_list/?ref=kodekloud.com#type-conversion-functions).

**Documentation for the values.yaml File**
If when a user opens up a values.yaml file all they get is a long list of variables, it can be very hard for them to figure out what each of them is used for. That’s why you should always document each by adding a comment above it. The comment starts with a **#**. Example:

```R
# replicaCount sets the number of replicas to use if autoscaling is enabled
replicaCount: 3
```

Notice how we start the comment with the actual variable name. It’s recommended you do it the same way to make searching easier. It also helps automatic documentation tools correlate these notes with the parameters that they describe.

To learn more about adding comments to your YAML file, check out our blog post: How to Add YAML Comments with Examples.

## 3. Template Files

**Names**
Template file names should all be lowercase letters. If they contain multiple words, these should be separated with dashes, **'-'**.

Examples of correct file names:

```R
deployment.yaml
tls-secrets.yaml
```

Incorrect file names:

```R
Deployment.yaml
tls_secrets.yaml
```

These file names should indicate the kind of Kubernetes resource they define so that users can get an idea of what's in the file from a glance. For long resource names, like a Persistent Volume Claim, you might use their acronym (PVC in this case) and end up with **pvc.yaml**.

**YAML Comments vs. Template Comments**

A template comment follows this syntax:

```R
{{/*
Comment goes here
*/}}
```

You should document anything that is not immediately obvious. Often, your .tpl files (like **_helpers.tpl**) will need to be commented on thoroughly, as it’s not easy to figure out what goes on there. Take a look at this:

```R
{{- define "wordpress.memcached.fullname" -}}
{{- printf "%s-memcached" .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- define "wordpress.image" -}}
{{- include "common.images.image" (dict "imageRoot" .Values.image "global" .Values.global) -}}
{{- end -}}
```

It's pretty hard to figure out what is going on in the file above, and even if we understand what happens, we might wonder, “Why use trunc 63 to truncate to 63 characters?” We can add some comments to clarify it:

```R
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "wordpress.memcached.fullname" -}}
{{- printf "%s-memcached" .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Return the proper WordPress image name
*/}}
{{- define "wordpress.image" -}}
{{- include "common.images.image" (dict "imageRoot" .Values.image "global" .Values.global) -}}
{{- end -}}
```

The file above becomes much easier to understand.

This type of comment (template comment) won’t be displayed if the user enters a command like helm install –debug to see what manifests this chart would generate. If you need these comments to be output in such commands, you can use YAML comments instead. These look like this:

```R
# This comment would be seen in a helm install --debug command.
spec:
  type: ClusterIP
  ports:
    - name: metrics
      port: {{ .Values.metrics.service.port }}
      protocol: TCP
      targetPort: metrics
```

## 4. Dependencies

**Versions**
When your chart depends on other charts, you must declare in Chart.yaml the versions of these dependency charts. Instead of using an exact version like:

```version: 1.8.2```

You should declare an approximate version by prefixing it with the tilde `~` sign.

```version: ~1.8.2```

This is equivalent to saying, “*The version number starts with 1.8, and it can be anything higher than or equal to 1.8.2 but strictly lower than 1.9.0 (it can be 1.8.9 or 1.8.10, but not 1.9.0)*”.

**URLs**
When available, use the more secure **https://** for repository URLs instead of **http://**. HTTP’s encryption and certificate authentication provides extra security for your requests. For instance, they make it harder for someone to perform a man-in-the-middle attack, which can be used to inject malicious code into our dependency charts.


## 5. Recommended Labels

Kubernetes operators find objects by looking at the metadata - more specifically, the labels in the metadata section. There are many objects in the Kubernetes cluster, but only part of them are installed by and managed by Helm. Adding a label such as helm.sh/chart: nginx-0.1.0 makes it easy for operators to find all objects installed by the nginx chart, version 0.1.0.

Your templates should, preferably, generate templates with labels similar to this:

Check out this [page](https://helm.sh/docs/chart_best_practices/labels/?ref=kodekloud.com) to better understand how to use labels. To see an example of how labels are used in templates, use this command:

```R
helm create sample
```

Then explore the **sample/templates/** directory and **sample/templates/_helpers.tpl** file to see how these are defined in a clean, rather simple chart.

## 6. Pods and PodTemplates

Whenever we declare Kubernetes objects such as Deployments, ReplicationControllers, ReplicaSets, DaemonSets, or StatefulSets, we include definitions of PodTemplates.

We tell Kubernetes the kind of pods that it will need to launch, hence a template for these pods. That is because something like a deployment, although just one object, might launch tens of pods. Here’s a list of best practices when we work with such sections.


**Images Versions/Tags**

Do not use an image tag such as latest or any other similar tag that doesn’t point to a specific, fixed image. The latest tag is what we’d call a “floating” tag, meaning the image version it points to is constantly changing; today, it might point to 1.2.3. Tomorrow, when a new version is launched, it might point to 1.2.4 or, even worse, to 2.0.1 (a major update with significant changes).

The components installed by the chart usually need to fit perfectly together, and the latest tag takes out the perfect fit guarantee. If there is a major update to one component, it might not work with the other pieces due to incompatible changes in the latest version.

The recommended practice is to point to a specific image version and use values.yaml as the place where it can be modified. See the sample values.yaml below:

```R
image:
  repository: 
  pullPolicy: IfNotPresent
  tag: "1.16.0"
```

Here is the deployment.yaml that pulls the image name and version (tag) from the values.yaml file:

```R
spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository | default "nginx" }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

If the user did not define a repository name in his values.yaml file, we use a default name for the chart, nginx. We didn’t do this in our example, but we should define a default version in a real production-ready chart. The line:

```R
image: "{{ .Values.image.repository | default "nginx" }}:{{ .Values.image.tag }}"
```

Should be:

```R
image: "{{ .Values.image.repository | default "nginx" }}:{{ .Values.image.tag | default "1.16.0" }}"
```

This way, we achieve the best of both worlds:

The chart will produce predictable results; the same images will be installed in all default installs (when the user doesn’t bother to choose his own settings).
The user still has the power to choose other versions for his images if he has some specific needs. For example, if his desired features only exist in a newer software version).
So we give our users predictable results and the freedom to choose if they so desire.

Include Selectors in Your PodTemplates

Consider the manifest below that would get generated by the default nginx/templates/deployment.yaml file created by the `helm create nginx` command:

```R
apiVersion: apps/v1
kind: Deployment
metadata:
  name: RELEASE-NAME-nginx
  labels:
    helm.sh/chart: nginx-0.1.0
    app.kubernetes.io/name: nginx
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
      app.kubernetes.io/instance: RELEASE-NAME
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
        app.kubernetes.io/instance: RELEASE-NAME
    spec:
      serviceAccountName: RELEASE-NAME-nginx
      securityContext:
        {}
      containers:
        - name: nginx
          securityContext:
            {}
          image: "nginx:1.16.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          resources:
            {}
```

As we mentioned earlier, objects like Kubernetes deployments can create multiple pods. And there will also be pods launched by other objects, resulting in many pods that are mixed together. There needs to be a way to know which pods belong to a group and which belong to another group.

This is exactly what selector labels help in doing.  Your selectors should be the label, or labels (can be one or more) that you know will never change for the entire duration that the object is running. Let us see this in the example below:

```R
  labels:
    helm.sh/chart: nginx-0.1.0
    app.kubernetes.io/name: nginx
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: Helm
```

We chose these two as our selectors:

```R
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
      app.kubernetes.io/instance: RELEASE-NAME
```

We can't choose app.kubernetes.io/version: "1.16.0" as a selector since the version label will definitely change in the future.

## 7. Custom Resource Definitions

Custom Resource Definitions (CRDs) are basically a way to add extra features to Kubernetes. These require some special attention on our part when using them with Helm. To understand why to think about this:

Kubernetes is built to ingest definitions of many objects, even all at once. Say we send it some Deployment manifests for some MySQL pods, PersistentVolume manifests, PersistentVolumeClaims, and so on. We, as the users, don’t care in which order we send these to Kubernetes: But MySQL would not be able to work if it would not have a persistent volume where it can store its database.

Keep in mind that we do not specify anywhere, “Hey, first launch these persistent volumes, and only after that launch the MySQL pods.” But our cluster will know what to do. When we want to install CRDs with Helm, though, the story is different.

Let’s say we define a cool new object in our CRD. Now, we can use this new type of object in Kubernetes by starting our manifests with something like:

```R
apiVersion: "stable.example.com/v1"
kind: CoolObject
```

But Helm needs to send both things to our cluster: the CRD and the manifest of this object. Now, the order matters, or more specifically, the timing. A few seconds might pass until this new CRD is registered. If Kubernetes receives this object before the CRD is registered, it will not know what this is and how to create it since it does not know the CRD. It simply doesn’t know what this CoolObject is.

To tackle this, Helm gives you a special location for CRDs. You need to add a crds directory to the chart we have created. This directory should exist in the same place where we have the templates and charts directory.

```R
nginx
├── charts
├── Chart.yaml
├── crds
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   ├── service.yaml
│   └── tests
│       └── test-nginx.yaml
└── values.yaml
```

Now Helm will install your CRDs first, then your other objects.

CRDs cannot be templated: they should always be written as the exact manifest you want to send to your cluster.
You can’t have a line like this in your CRD.

```R
name: {{ .Chart.Name }}
```

It should be the exact content you want to pass along:

```R
name: someName
```

**Disadvantages of Installing CRDs with Helm**

You cannot upgrade or delete CRDs with Helm.

You also cannot perform a –dry-run install to test if your chart works with Kubernetes without actually installing anything. The dry run would require the CRD to actually get installed so that the other objects in your chart get recognized on a dry run. But that defeats the purpose of not installing anything. Without the CRD, any objects in your chart that want to use this Custom Resource will fail on a dry run test.

