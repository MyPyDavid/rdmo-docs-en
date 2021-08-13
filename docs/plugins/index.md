# Plugins

```eval_rst
.. warning::
    This is an advanced feature of RDMO.
```

Plugins can be used to customize or extend specific actions in RDMO using custom Python code outside of the centrally maintained code base. This can be used to perform actions which are specific to a RDMO instance. With the possibility to add code to RDMO comes the danger of introducing additional bugs and security issues. Please be extra careful when using this **advanced** feature.

Plugins are created by implementing Python classes (e.g. in the local `rdmo-app`), and register them in the `config/settings/local.py` file. These classes need to inherit from prepared base classes in the RDMO source code. We also provide a set of plugins [rdmo-plugins](https://github.com/rdmorganiser/rdmo-plugins), which can be used together with the domain model and the other content in [rdmo-content](https://github.com/rdmorganiser/rdmo-content).

As of now the following plugins can be created:

-	Project exports (from `rdmo.projects.export.Export`\)
-	Project imports (from `rdmo.projects.imports.Import`\)
-   Optionset providers (from `rdmo.options.providers.Provider`\)
-   Service providers (from `rdmo.services.providers.Provider`\)

To be usable by RDMO, the plugins need to be available in the virtual environment in which RDMO is running. There are severeal possibilities to archive this, but we suggest to use one of the following two:

1.	If only the plugins provided in [rdmo-plugins](https://github.com/rdmorganiser/rdmo-plugins) should be used and no modifications are desired, they can be directly installed from GitHub using:

	```bash
	pip install git+https://github.com/rdmorganiser/rdmo-plugins
	```

2.	If you intent to write plugins yourself or if you want to modify our plugins, you should create a python module in your `rdmo-app`, which is just a directory containing an empty `__ini__.py` file. Python files in this directory are then automatically available to your RDMO instance.

	```bash
	mkdir my_plugins
	touch my_plugins/__init__.py
	# place, e.g., madmp.py in my_plugins/
	```

As a last step, the plugins need to be registered in `config/settings/local.py`. The setting depends on the class of plugin and is described in detail below. Plugins are always specified by a tuple of `(key, label, class_name)`, therfore plugin settings always look like this:

```python
EXAMPLE_PLUGIN_SETTINGS = [
    ('example', _('Example'), 'module.module.module.Plugin'),
    ...
]
```

## Project export plugins

Custom project exports can be created by implementing a class inheriting from `rdmo.projects.export.Export`. They can be used to create a custom export format to be used by the users to download their project data. The RDMO-XML exports are generated using the `rdmo.projects.exports.RDMOXMLExport`. Examples from [rdmo-plugins](https://github.com/rdmorganiser/rdmo-plugins) are `DataCiteExport` or `MaDMPExport`.

The `Export` plugin class needs to implement a `render()` function which takes no arguments and returns a `django.http.HttpResponse`. The export can be created from `self.project`, `self.snapshot` and `self.values` instance variables.

The export plugin needs to be added to the `PROJECT_EXPORTS` in `config/settings/local.py`. The default settings are:

```python
PROJECT_EXPORTS = [
    ('xml', _('as RDMO XML'), 'rdmo.projects.exports.RDMOXMLExport'),
    ('csvcomma', _('as CSV (comma separated)'), 'rdmo.projects.exports.CSVCommaExport'),
    ('csvsemicolon', _('as CSV (semicolon separated)'), 'rdmo.projects.exports.CSVSemicolonExport'),
]
```

In order to use the plugins in [rdmo-catalog](https://github.com/rdmorganiser/rdmo-plugins), add the following to your `config/settings/local.py`:

```python
PROJECT_EXPORTS = [
    ('xml', _('as RDMO XML'), 'rdmo.projects.exports.RDMOXMLExport'),
    ('csvcomma', _('as CSV (comma separated)'), 'rdmo.projects.exports.CSVCommaExport'),
    ('csvsemicolon', _('as CSV (semicolon separated)'), 'rdmo.projects.exports.CSVSemicolonExport'),
    ('madmp', _('as maDMP JSON'), 'rdmo_plugins.exports.madmp.MaDMPExport'),
    ('datacite', _('as DataCite XML'), 'rdmo_plugins.exports.datacite.DataCiteExport'),
    ('radar', _('as RADAR XML'), 'rdmo_plugins.exports.radar.RadarExport')
]
```

Please refer to <https://github.com/rdmorganiser/rdmo/blob/master/rdmo/projects/exports.py> for the default project export plugins and code examples. The code which uses the plugin is located in <https://github.com/rdmorganiser/rdmo/blob/master/rdmo/projects/views.py> (`ProjectExportView`).

## Project import plugins

Similarly, custom project imports can be created implementing a class inheriting from `rdmo.projects.imports.Import`. They can be used to import project data from files which are uploaded by the user.

The Plugin class needs to implement a `check()` function which takes no arguments and only returns `True` if an uploaded file can be imported by this plugin. The `self.file_name` instance variable can be used for this. In most cases, this will include opening and parsing the file.

In addition, a `process()` needs to be implemented which takes no arguments and returns `None`, but extracts the data from the file and populates the `self.project`, `self.catalog`, `self.values`, `self.snapshots`, `self.tasks` and `self.views` instance variables.

The import plugin needs to be added to the `PROJECT_IMPORTS` in `config/settings/local.py`. The default settings are:

```python
PROJECT_IMPORTS = [
    ('xml', _('from RDMP XML'), 'rdmo.projects.imports.RDMOXMLImport'),
]
```

In order to use the plugins in [rdmo-catalog](https://github.com/rdmorganiser/rdmo-plugins), add the following to your `config/settings/local.py`:

```python
PROJECT_IMPORTS = [
    ('xml', _('from RDMP XML'), 'rdmo.projects.imports.RDMOXMLImport'),
    ('madmp', _('from maDMP'), 'rdmo_plugins.imports.madmp.MaDMPImport'),
    ('datacite', _('from DataCite XML'), 'rdmo_plugins.imports.datacite.DataCiteImport'),
    ('radar', _('from RADAR XML'), 'rdmo_plugins.imports.radar.RadarImport'),
]
```

Please refer to <https://github.com/rdmorganiser/rdmo/blob/master/rdmo/projects/imports.py> for the default project import plugins and code examples. The code which uses the plugin is located in <https://github.com/rdmorganiser/rdmo/blob/master/rdmo/projects/views.py> (`ProjectCreateUploadView`, `ProjectCreateImportView`, `ProjectUpdateUploadView`, `ProjectUpdateImportView`).


## Optionset providers

Optionset providers allow the creation of dynamic option sets. These option sets do not need to have a fixed set of options configured in the database, but instead dynamically determine which options to display, whenever a user accesses the optionset in the interview. This can be done by fetching resources from a webservice and can be based on already given answers. The example implementation of such a plugin is the re3data optionset provider available from [rdmo-plugins](https://github.com/rdmorganiser/rdmo-plugins).

The only function, the plugin class needs to implement, is `get_options(self, project)`. It takes the current project as argument (so it can access the values already entered for the project), and returns the options as a list of the form:

```python
[
    {
        'external_id': 'ID',
        'text': 'TEXT'
    },
    ...
]
```

Both the `external_id` and the `text` are stored, when the user selects the option. The `text` will show up in views and exports.

The plugin needs to be added to the `OPTIONSET_PROVIDERS` in `config/settings/local.py`. In order to use the re3data.org provider from [rdmo-plugins](https://github.com/rdmorganiser/rdmo-plugins), add the following to your `config/settings/local.py`:

```python
OPTIONSET_PROVIDERS = [
    ('re3data', _('Repositories from re3data'), 'rdmo_plugins.optionsets.re3data.Re3DataProvider')
]
```

The re3data optionset will query [re3data.org](https://www.re3data.org/) for repositories that match the research field of the project (as given by the `project/research_field/title` Attribute).


## Service providers

Service providers enable users to add integrations to projects. Using these integrations, project tasks (internally called issues) can be send to external services. This can be done using a simple webhook or, more sophisticated, using the OAuth workflow. As reference implementation, RDMO comes with a GitHub provider, which can be used to push tasks to GitHub issues.

The funtion `send_issue(self, request, issue, integration, subject, message, attachments)` needs to be implemented by the provider. It takes the request object, the issue to be send, the integration (which can have project specific options for the provider), and the subject, message, attachments to be send. In the case of the GitHub provider, this function performs a POST request using the GitHub API in order to create a new issue. The funtion needs to return a `HttpResponse` in the successful case, this will be a redirect. When using the OAuth workflow, a required authorization will result in a redirec to the external service, followed by a redirect to a callback whithin RDMO, and a subsequent second try to perform the POST request. This should happen transparent to the user.

The provider can also implement a `webhook(self, request, integration)` which is used to provide a webhook whithin RDMO to enable the external service to trigger actions, e.g. when an issue is closed.

Furthermore, it needs to implement a property `fields`, which returns the option fields, which users need to enter when adding an integration to a project. For GitHub, this is the repository and a secret string to secure the webhook. Please refer to the [implementation of the GitHubProvider](https://github.com/rdmorganiser/rdmo/blob/master/rdmo/services/providers.py) for more details.

### GitHub service provider

The GitHub provider ships with RDMO, needs to be added to the `SERVICE_PROVIDERS` in `config/settings/local.py` in order to be used:

```python
SERVICE_PROVIDERS = [
    ('github', _('GitHub'), 'rdmo.services.providers.GitHubProvider'),
]
```

In addition, an "App" has to be registered with GitHub, and the `client_id` and the `client_secret` needs to be configured in `config/settings/local.py`:

```python
GITHUB_PROVIDER = {
    'client_id': '',
    'client_secret': ''
}
```

Then users can add an integration to their projects, which requires setting the repo in the form `<user>/<repo>` the tasks from RDMO are send to.

Additionally, but probably only if the project in RDMO is also managed by RDMO savy staff, a secret can be added to enable GitHub to communicate to RDMO when an issue has been closed. For this to work, a webhook has to be added at `https://github.com/<user>/<repo>/settings/hooks`. The webhook has to point to `https://<your rdmo url>/projects/<project_id>/integrations/<integration_id>/webhook/`, the content type is `application/json` and the secret has to be exactly the secret entered in the integration.

## Examples of how to install plugins

As mentioned above, there are two possibilities of how to install [rdmo-plugins](https://github.com/rdmorganiser/rdmo-plugins). Depending on your use case you should pick the one that fits you more. The first one using `pip` does a good job if you are just planning to use the plugins from the repo without modifying these. The other method requires to copy the necessary python files into your local app folder which has advantages if you are planning to modify existing or create your own plugins.

Please note that the imports require Django's translation utils and that we usually use these with an underscore. If you are getting import errors please check if you have the first of the following lines at the beginning of your `local.py`. You also need to import `PROJECT_EXPORTS` and `PROJECT_IMPORTS` as we will later append our imports to these lists. Make sure you also have the second line in your `local.py`.

```python
from django.utils.translation import ugettext_lazy as _
from rdmo.core.settings import PROJECT_EXPORTS, PROJECT_IMPORTS
```

### Use pip

To install directly from github simply run

```shell
pip install git+https://github.com/rdmorganiser/rdmo-plugins
```

Afterwards you need to configure the plugins you are willing to use in your `local.py`. Here is an example of how to add different export formats. The three strings in brackets configure the url under which the export will be available, the name of the entry in the export menu and the path from where to import the plugin. The path resembles the structure in the `rdmo plugins` repository. If you look into it you will find the files containing the imported classes mentioned below. At the beginning of the path you use `rdmo_plugins` because you added `rdmo-plugins` to your installed python libraries. The dash is replaced by an underscore because python is not very fond of dashes when it comes to imports.

```python
PROJECT_EXPORTS.append(('madmp', _('as madmp'), 'rdmo_plugins.exports.madmp.MaDMPExport'))
PROJECT_EXPORTS.append(('datacite', _('as datacite'), 'rdmo_plugins.exports.datacite.DataCiteExport'))
PROJECT_EXPORTS.append(('radar', _('as radar'), 'rdmo_plugins.exports.radar.RadarExport'))
```

### Copy files

The other method we recommend when you are planning to modify the plugins or create your own ones is to simply copy the python files into your local app folder. Let's say you create a subfolder `plugins` containing another folder `exports` in your app directory. Then you copy the `madmp.py` from the `rdmo-plugins` into `exports`. Having done this you can import this file in your local app without specifying a module path. It is found by the importer as it resides in your rdmo app folder. Your import lines in the `local.py` would look like this:

```python
PROJECT_EXPORTS.append(('madmp', _('as madmp'), 'plugins.exports.datacite.DataCiteExport'))
```
