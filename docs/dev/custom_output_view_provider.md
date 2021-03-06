# Custom Output View Provider

A custom _output view provider_ generates visualizations of experiment outputs.
Output view providers are implemented as a Python function and packaged as a
Python package, with the requisite metadata (more on this below). An output view
provider is associated with the output of an application in the Application
Catalog.

There are several different output view display types, such as: image, link, or
html. If configured an output view will be displayed for an output file and the
Airavata Django Portal will invoke the custom output view provider to get the
data to display. For example, if the output view display type is image, then the
output view provider will be invoked and it should return image data.

## Getting started

See the
[Gateways 2019 tutorial](../tutorial/gateways2019_tutorial.md#tutorial-exercise-create-a-custom-output-viewer-for-an-output-file)
for help on setting up a development environment and implementing a simple
output view provider.

You can use this as a starting point to create your own custom output view
provider. Here is what you would need to change:

1. First add your custom output view provider implementation to
   `output_views.py`.
2. Rename the Python package name in `setup.py`.
3. Update the `install_requires` list of dependencies based on what your custom
   output view provider requires.
4. Rename the Python module folder from `./gateways2019_tutorial` to whatever
   you want to call it.
5. Rename the output view provider in the `entry_points` metadata in `setup.py`.
   For example, if you wanted to name your output view provider
   `earthquake-sites-visualization` and you renamed your Python module folder
   from `./gateways2019_tutorial` to `./earthquake_gateway`, then you could have
   the following in the `entry_points`:

```
...
    entry_points="""
[airavata.output_view_providers]
earthquake-sites-visualization = earthquake_gateway.output_views:EarthquakeSitesViewProvider
""",
```

6. If you don't need a Django app, you can remove the `[airavata.djangoapp]`
   section from `entry_points`.

Please note, if you update `setup.py` and you're doing local development, you'll
need to reinstall the package into your local Django instance's virtual
environment using:

```bash
python setup.py develop
```

## Reference

### Output View Provider interface

Output view providers should be defined as a Python class. They should define
the following attributes:

-   `display_type`: this should be one of _link_, _image_ or _html_.
-   `name`: this is the name of the output view provider displayed to the user.
-   `test_output_file`: (optional) the path to a file to use for testing
    purposes. This file will be passed to the `generate_data` function as the
    `output_file` parameter when the output file isn't available and the Django
    server is running in DEBUG mode. This is helpful when developing a custom
    output view provider in a local Django instance that doesn't have access to
    the output files.

The output view provider class should define the following method:

```python
def generate_data(self, request, experiment_output, experiment, output_file=None, **kwargs):

    # Return a dictionary
    return {
        #...
    }
```

The required contents of the dictionary varies based on the _display type_.

#### Display type link

The returned dictionary should include the following entries:

-   url
-   label

The _label_ is the text of the link. Generally speaking this will be rendered
as:

```html
<a href="{{ url }}">{{ label }}</a>
```

**Examples**

-   [SimCCS Maptool - SolutionLinkProvider](https://github.com/SciGaP/simccs-maptool/blob/master/simccs_maptool/output_views.py#L5)

#### Display type image

The returned dictionary should include the following entries:

-   image: a stream of bytes, i.e., either the result of `open(file, 'rb')` or
    something equivalent like `io.BytesIO`.
-   mime-type: the mime-type of the image, for example, `image/png`.

**Examples**

-   [AMP Gateway - TRexXPlotViewProvider](https://github.com/SciGaP/amp-gateway-django-app/blob/master/amp_gateway/plot.py#L115)

#### Display type html

The returned dictionary should include the following entries:

-   output: a raw HTML string
-   js: a static URL to a JavaScript file, for example,
    `/static/earthquake_gateway/custom-leaflet-script.js`.

**Examples**

-   [dREG - DregGenomeBrowserViewProvider](https://github.com/SciGaP/dreg-djangoapp/blob/master/dreg_djangoapp/output_views.py#L4)

### Entry Point registration

Custom output view providers are packaged as Python packages in order to be
deployed into an instance of the Airavata Django Portal. The Python package must
have metadata that indicates that it contains a custom output view provider.
This metadata is specified as an _entry point_ in the package's `setup.py` file
under the named parameter `entry_points`.

The entry point must be added to an entry point group called
`[airavata.output_view_providers]`. The entry point format is:

```
label = module:class
```

The _label_ is the identifier you will use when associating an output view
provider with an output file in the Application Catalog. As such, you can name
it whatever you want. The _module_ must be the Python module in which exists
your output view provider. The _class_ must be the name of your output view
provider class.

See the **Getting Started** section for an example of how to format the entry
point in `setup.py`.

### Associating an output view provider with an output file

In the Application Catalog, you can add JSON metadata to associate an output
view provider with an output file.

1. In the top navigation menu in the Airavata Django Portal, go to **Settings**.
2. If not already selected, select the **Application Catalog** from the left
   hand side navigation.
3. Click on the application.
4. Click on the **Interface** tab.
5. Scroll down to the _Output Fields_ and find the output file with which you
   want to associate the output view provider.
6. In the _Metadata_ field, add or update the `output-view-providers` key. The
   value should be an array (beginning and ending with square brackets). The
   name of your output view provider is the label you gave it when you created
   the entry point.

The _Metadata_ field will have a value like this:

```json
{
    "output-view-providers": ["gaussian-eigenvalues-plot"]
}
```

Where instead of `gaussian-eigenvalues-plot` you would put or add the label of
your custom output view provider.

There's a special `default` output view provider that provides the default
interface for output files, namely by providing a download link for the output
file. This `default` output view provider will be shown initially to the user
and the user can then select a custom output view provider from a drop down
menu. If, instead, you would like your custom output view provider to be
displayed initially, you can add the `default` view provider in the list of
output-view-providers and place it second. For example:

```json
{
    "output-view-providers": ["gaussian-eigenvalues-plot", "default"]
}
```

would make the `gaussian-eigenvalues-plot` the initial output view provider. The
user can access the default output view provider from the drop down menu.

### Interactive parameters

You can add some interactivity to your custom output view provider by adding one
or more interactive parameters. An interactive parameter is a parameter that
your custom output view provider declares, with a name and current value. The
Airavata Django Portal will display all interactive parameters in a form and
allow the user to manipulate them. When an interactive parameter is updated by
the user, your custom output view provider will be again invoked with the new
value of the parameter.

To add an interactive parameter, you first need to add a keyword parameter to
your `generate_data` function. For example, let's say you want to add a boolean
`show_grid` parameter that the user can toggle on and off. You would change the
signature of the `generate_data` function to:

```python
def generate_data(self, request, experiment_output, experiment, output_file=None, show_grid=False, **kwargs):

    # Return a dictionary
    return {
        #...
    }
```

In this example, the default value of `show_grid` is `False`, but you can make
it `True` instead. The default value of the interactive parameter will be its
value when it is initially invoked. It's recommended that you supply a default
value but the default value can be `None` if there is no appropriate default
value.

Next, you need to declare the interactive parameter in the returned dictionary
along with its current value in a special key called `interactive`. For example:

```python
def generate_data(self, request, experiment_output, experiment, output_file=None, show_grid=False, **kwargs):

    # Return a dictionary
    return {
        #...
        'interactive': [
            {'name': 'show_grid', 'value': show_grid}
        ]
    }
```

declares the interactive parameter named `show_grid` and its current value.

The output view display will render a form showing the value of `show_grid` (in
this case, since it is boolean, as a checkbox).

#### Supported parameter types

Besides boolean, the following additional parameter types are supported:

| Type    | UI Control              | Additional options                                                                                          |
| ------- | ----------------------- | ----------------------------------------------------------------------------------------------------------- |
| Boolean | Checkbox                |                                                                                                             |
| String  | Text input              |                                                                                                             |
| Integer | Stepper or Range slider | `min`, `max` and `step` - if `min` and `max` are supplied, renders as a range slider. `step` defaults to 1. |
| Float   | Stepper or Range slider | `min`, `max` and `step` - if `min` and `max` are supplied, renders as a range slider.                       |

Further, if the interactive parameter defines an `options` list, this will
render as a drop-down select. The `options` list can either be a list of
strings, for example:

```python
def generate_data(self, request, experiment_output, experiment, output_file=None, color='red', **kwargs):

    # Return a dictionary
    return {
        #...
        'interactive': [
            {'name': 'color', 'value': color, 'options': ['red', 'green', 'blue']}
        ]
    }
```

Or, the `options` list can be a list of `(text, value)` tuples:

```python
def generate_data(self, request, experiment_output, experiment, output_file=None, color='red', **kwargs):

    # Return a dictionary
    return {
        #...
        'interactive': [
            {'name': 'color', 'value': color, 'options': [('Red', 'red'), ('Blue', 'blue'), ('Green', 'green')]}
        ]
    }
```

The `text` is what is displayed to the user as the value's label in the
drop-down. The `value` is what will be passed to the output view provider when
selected by the user.

#### Additional configuration

The following additional properties are supported:

-   **label** - by default the name of the interactive parameter is its label in
    the interactive form. You can customize the label with the `label` property.
-   **help** - you can also display help text below the parameter in the
    interactive form with the `help` property.

For example:

```python
def generate_data(self, request, experiment_output, experiment, output_file=None, color='red', **kwargs):

    # Return a dictionary
    return {
        #...
        'interactive': [
            {'name': 'color',
            'value': color,
            'options': [('Red', 'red'), ('Blue', 'blue'), ('Green', 'green')],
            'label': 'Bar chart color',
            'help': 'Change the primary color of the bar chart.'}
        ]
    }
```
