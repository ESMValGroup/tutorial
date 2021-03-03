---
title: "Writing your own diagnostic script"
teaching: TBD
exercises: TBD

questions:
- "How do I write a new diagnostic in ESMValTool?"
- "How do I read preprocessor output in a Python diagnostic?"


objectives:
- "Writing a new Python diagnostic"
- "Understanding the interface between the ESMValCore preprocessor and a diagnostic script."

keypoints:
- "ESMValTool providees helper functions to interface a Python diagnostic script with ESMValCore preprocessor output."
- "Existing diagnostics can be used as templates and modfied to write new diagnostics."
---

## Introduction

The diagnostic script is an important component of ESMValTool where the
scientific analysis or performance metric is implemented. With ESMValTool, you 
can reuse an existing diagnostic, adapt and existing one for your needs or
write your own new diagnostic.  Diagnostics can be written in a number of open 
source languages such as Python, R, Julia and NCL but we will focus on understanding 
and writing Python diagnostics in this lesson. In order to access existing diagnostics or
to write your own, please install ESMValTool in the development mode on your 
machine using the instructions from [this episode](08-development-setup/index.html).

## Understanding an existing Python diagnostic
We revisit a recipe we have seen before, [recipe_python.yml](https://github.com/ESMValGroup/ESMValTool/blob/master/esmvaltool/recipes/examples/recipe_python.yml) and the diagnostic script called by this recipe -- [diag_scripts/examples/diagnostic.py](https://github.com/ESMValGroup/ESMValTool/blob/master/esmvaltool/diag_scripts/examples/diagnostic.py). For reference, we have the diagnostic file in the dropdown box below.


> ## diagnostic.py
>
>~~~
>  1   """Python example diagnostic."""
>  2   import logging
>  3   import os
>  4   from pprint import pformat
>  5   
>  6   import iris
>  7   
>  8   from esmvaltool.diag_scripts.shared import (group_metadata, run_diagnostic,
>  9                                               select_metadata, sorted_metadata)
> 10   from esmvaltool.diag_scripts.shared._base import (
> 11       ProvenanceLogger, get_diagnostic_filename, get_plot_filename)
> 12   from esmvaltool.diag_scripts.shared.plot import quickplot
> 13   
> 14   logger = logging.getLogger(os.path.basename(__file__))
> 15   
> 16   
> 17   def get_provenance_record(attributes, ancestor_files):
> 18       """Create a provenance record describing the diagnostic data and plot."""
> 19       caption = ("Average {long_name} between {start_year} and {end_year} "
> 20                  "according to {dataset}.".format(**attributes))
> 21   
> 22       record = {
> 23           'caption': caption,
> 24           'statistics': ['mean'],
> 25           'domains': ['global'],
> 26           'plot_types': ['zonal'],
> 27           'authors': [
> 28               'andela_bouwe',
> 29               'righi_mattia',
> 30           ],
> 31           'references': [
> 32               'acknow_project',
> 33           ],
> 34           'ancestors': ancestor_files,
> 35       }
> 36       return record
> 37   
> 38   
> 39   def compute_diagnostic(filename):
> 40       """Compute an example diagnostic."""
> 41       logger.debug("Loading %s", filename)
> 42       cube = iris.load_cube(filename)
> 43   
> 44       logger.debug("Running example computation")
> 45       return cube.collapsed('time', iris.analysis.MEAN)
> 46   
> 47   
> 48   def plot_diagnostic(cube, basename, provenance_record, cfg):
> 49       """Create diagnostic data and plot it."""
> 50       diagnostic_file = get_diagnostic_filename(basename, cfg)
> 51   
> 52       logger.info("Saving analysis results to %s", diagnostic_file)
> 53       iris.save(cube, target=diagnostic_file)
> 54   
> 55       if cfg['write_plots'] and cfg.get('quickplot'):
> 56           plot_file = get_plot_filename(basename, cfg)
> 57           logger.info("Plotting analysis results to %s", plot_file)
> 58           provenance_record['plot_file'] = plot_file
> 59           quickplot(cube, filename=plot_file, **cfg['quickplot'])
> 60   
> 61       logger.info("Recording provenance of %s:\n%s", diagnostic_file,
> 62                   pformat(provenance_record))
> 63       with ProvenanceLogger(cfg) as provenance_logger:
> 64           provenance_logger.log(diagnostic_file, provenance_record)
> 65   
> 66   
> 67   def main(cfg):
> 68       """Compute the time average for each input dataset."""
> 69       # Get a description of the preprocessed data that we will use as input.
> 70       input_data = cfg['input_data'].values()
> 71   
> 72       # Demonstrate use of metadata access convenience functions.
> 73       selection = select_metadata(input_data, short_name='pr', project='CMIP5')
> 74       logger.info("Example of how to select only CMIP5 precipitation data:\n%s",
> 75                   pformat(selection))
> 76   
> 77       selection = sorted_metadata(selection, sort='dataset')
> 78       logger.info("Example of how to sort this selection by dataset:\n%s",
> 79                   pformat(selection))
> 80   
> 81       grouped_input_data = group_metadata(
> 82           input_data, 'standard_name', sort='dataset')
> 83       logger.info(
> 84           "Example of how to group and sort input data by standard_name:"
> 85           "\n%s", pformat(grouped_input_data))
> 86   
> 87       # Example of how to loop over variables/datasets in alphabetical order
> 88       for standard_name in grouped_input_data:
> 89           logger.info("Processing variable %s", standard_name)
> 90           for attributes in grouped_input_data[standard_name]:
> 91               logger.info("Processing dataset %s", attributes['dataset'])
> 92               input_file = attributes['filename']
> 93               cube = compute_diagnostic(input_file)
> 94   
> 95               output_basename = os.path.splitext(
> 96                   os.path.basename(input_file))[0] + '_mean'
> 97               provenance_record = get_provenance_record(
> 98                   attributes, ancestor_files=[input_file])
> 99               plot_diagnostic(cube, output_basename, provenance_record, cfg)
>100   
>101   
>102   if __name__ == '__main__':
>103   
>104       with run_diagnostic() as config:
>105           main(config)
>
>~~~
>{: .language-python}
>
{:.solution}

> ## What is the starting point of the diagnostic?
>
> Can you spot a function called *main* in the Python code above? How many times is this
> function mentioned?
>
>
> > ## Answer
> >
> > The main function is defined in the middle of this script on line 67 and is called 
>> near the very end on line 105.  
>>The function *run_diagnostic* just above that is what 
>>is called a context manager provided with ESMValTool and is the 
>>main entry point for most Python diagnostics. The variable *cfg* is a Python dictionary 
>>loaded with all the
>>necessary information needed to run the diagnostic script including location of 
>> input data and various settings.
>>In the *main* function, we will next parse this *cfg* variable and extract 
>>information as needed to do our analyses.
> >
> {: .solution}
{: .challenge}

## What information do I need for my analyses?
The very first thing passed to the diagnostic via the *cfg* dictionary is a path to a file
called *settings.yml*.  It is found at the lowest level of your directory structure under 
the *run* directory. An example path would be */path_to_recipe_output/run/diag_name/script_name/settings.yml*. 

> ## What is in the settings.yml file?
> The ESMValTool documentation page provides a generic overview of what is in the 
>settings.yml file [here](https://docs.esmvaltool.org/projects/esmvalcore/en/latest/interfaces.html).
> 
{: .callout}

> ## Challenge : digging in deeper to understand the preprocesor-diagnostic interface
>
> Can you find one example of the settings.yml file when you run this recipe?
> Take a look at the *input_files* list in your settings.yml file. Do you see a mention of 
> a second yml file called *metadat.yml*? 
> What information do you think is saved in metadata.yml?
>
>
> > ## Answer
> >Congratulations on finding an example each of the *settings.yml* and *metadata.yml* 
>>files! You will have noticed that metadata.yml has information on your preprocessed 
>>data. There is one file for each variable and it has detailed information on your data
>> including project (e.g., CMIP6, OBS), dataset names (e.g., MIROC-6, UKESM-0-1-LL), 
>>variable attributes (e.g., standard_name, units), preprocessor applied and time range
>> of the data. You are now ready to access all of this information for your evaluation!
> > 
> >
> {: .solution}
{: .challenge}

## Extracting information needed for analyses
In the *main* function of the diagnostic, you will see that *input_data* values are  read 
from the *cfg* Python dictionary. Typically, users will now need to group this input data 
according to some criteria such as by model or experiment and select specifics to analyse.
ESMValTool prvides a whole host of convenience functions that can do this for you. 
A list of available functions and their description is provided [here](https://docs.esmvaltool.org/en/latest/api/esmvaltool.diag_scripts.shared.html). In our example, you will see several 
of these imported right at the beginning of the file and used after input data is read.

> ## ESMValTool Diagnostic Interface Functions
>
> Can you spot the functions used for selecting and grouping data in the example?
>After running this example, how can you tell what the functions do?
>
> > ## Answer
> >
> > If you look carefully, you can see that there is a statement after each use of the 
>> select and group functions that starts with *logger.info* (lines 74, 78 and 83). These lines print output to
>> the log files. If you looked at the content of your log files under the run directory, you
>> should see the selected and grouped output. This is how you access preprocessed
>> information within your diagnostic.
> >
> {: .solution}
{: .challenge}

Finally, after grouping we read individual attributes such as the filename which gives us 
the specific name of the preprocessed data file we want to read and analyse. 
Following this, we see the call to a function called *compute_diagnostic*. 
In this example, this is where the analyses on the data is done. 
If you were writing your own diagnostic, this is the function you would write your 
own code in.

## Diagnostic Computation
The *compute_diagnostic* function in this example uses a software called [Iris](https://scitools-iris.readthedocs.io/en/latest/index.html) to read 
data from a *netCDF* file and perform the simple computation of removing any dimension 
of length one. This is just an illustrative example. Iris reads data into data structures called
[cubes](https://scitools-iris.readthedocs.io/en/latest/userguide/iris_cubes.html). The data in
these cubes can be modified, combined with other cubes' data or plotted. Two 
other possible ways of reading netcdf files using [xarrays](http://xarray.pydata.org/en/stable/)  or [SciPy's netCDF library](https://docs.scipy.org/doc/scipy-0.14.0/reference/generated/scipy.io.netcdf.netcdf_file.html) 
for your own diagnostics are given below.

>## Example using xarray 
>
>~~~
>import xarray as xr  #import statement at the beginning of the file
>
>
>def compute_diagnostic(filename):
>    """Compute an example diagnostic."""
>    logger.debug("Loading %s", filename)
>    ds = xr.open_dataset(filename)    
>
>    #do your analyses on the data here
>
>    return ds
>
>~~~
>{: .language-python}
>
{: .solution}


>## Example using Scipy's netCDF library
>
>~~~
>from scipy.io import netcdfx #import statement at the beginning of the file
>
>
>def compute_diagnostic(filename):
>    """Compute an example diagnostic."""
>    logger.debug("Loading %s", filename)
>    f = netcdf.netcdf_file(filename,'r')   
>
>    #do your analyses on the data here
>
>    return f
>
>~~~
>{: .language-python}
>
{: .solution}

## Plotting Diagnostic Output
 

## Another section

and some more text, etc.

## Conclusion


## For development purposes

Here are some of the elements that we can add

> ## Example exercise
>
> This is just a reminder on how to implement exercises
>
> > ## Example answer
> >
> > And this is where to add the answer.
> > This box will be collapsed in the page is first loaded.
> >
> {: .solution}
{: .challenge}

```bash
example command line instruction
```

```
example error message
```
{: .error}

> ## Example callout box
>
> This is how to create a callout box
{: .callout}