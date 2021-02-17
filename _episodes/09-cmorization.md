---
title: "CMORization: Using observational datasets"
teaching: 15
exercises: 45

questions:
- "What is so challenging about observational data?"
- "How do I use observational datasets in ESMValTool?"
- "How add support for new (observational) datasets?"

objectives:
- "Understand what CMORization is and why it is necessary."
- "Learn how to write a new CMORizer script."

keypoints:
- "CMORizers are dataset-specific scripts that can be run once to generate CMOR-compliant data."
- "ESMValTool comes with a set of CMORizers readily available, but you can also add your own."
---

## Introduction

This episode deals with "CMORization". ESMValTool was designed to work with data
that follow the CMOR standards. Unfortunately, not all datasets follow these
standards. In order to use such datasets in ESMValTool we first need to reformat
the data. This process is called "CMORization".

> ## What are the CMOR standards?
>
> The name "CMOR" originates from a tool: [the Climate Model Output
> Rewriter](https://cmor.llnl.gov/). This tool is used to create "CF-Compliant
> netCDF files for use in the CMIP projects". So CMOR extends the
> [CF-standard](https://cfconventions.org/) with additional requirements for
> the Coupled Model Intercomparison Projects (see e.g.
> [here](https://pcmdi.llnl.gov/CMIP6/Guide/modelers.html#5-model-output-requirements)).
>
>
> Concretely, the CMOR standards dictate e.g. the variable names and units,
coordinate information, how the data should be structured (e.g. 1 variable per
file), additional metadata requirements, but also file naming conventions a.k.a.
the data reference syntax (DRS). All this information is stored in so-called
CMOR tables. As example, the CMOR tables for the CMIP6 project can be found
[here](https://github.com/PCMDI/cmip6-cmor-tables).
{: .callout}

The Earth System Grid Federation (ESGF) is home to all CMIP data. The data
hosted there typically (mostly) follow the CMIP standards, and therefore
ESMValTool should work with these data without problems.

Datasets that are *not* part of one of the CMIP projects often don't follow the
CMOR standards. In this case, a reformatting script can be used to create a
CMOR-compliant copy of these datasets. CMORizer scripts for several popular
datasets are included in ESMValTool, and ESMValTool also provides a convenient
way to execute them.

Occasionally it happens that there are still minor issue with CMIP datasets. In
those cases, it is possible to fix those issues in ESMValCore before any further
processing is done. The same can be done for non-CMIP data. The advantage is
that you don't need to store an additional, reformatted copy of the data. The
disadvantage is that these fixes should be implemented inside ESMValCore.
Writing a CMORizer script is technically is simpler.

The concepts discussed so far are illustrated in the figure below.
![Data flow with ESMValTool](../fig/data_flow.png)
*Illustration of the data flow in ESMValTool.*

In this lesson, we will re-implement a CMORizer script for the FLUXCOM dataset
that contains observations of the Gross Primary Production (GPP), a variable
that is important for calculating components of the global carbon cycle. We will
go through all the steps and explain relevant topics as we go. If you prefer to
implement CMOR fixes, please read the documentation
[here](https://docs.esmvaltool.org/projects/esmvalcore/en/latest/develop/fixing_data.html#fixing-data).
While fixes are implemented slightly differently, conceptually the process is
the same and the concepts explained in this episode are still useful.

## Obtaining the data

> ## Get the data
> The data for this episode is available via the [FluxCom Data
> Portal](http://www.bgc-jena.mpg.de/geodb/BGI/Home). First you'll need to
> register. After registration, in the dropdown boxes, select FLUXCOM as the data
> choice and click download. Three files will be displayed. Click the download
> button on the "FLUXCOM (RS+METEO) Global Land Carbon Fluxes using CRUNCEP
> climate data". You'll be send an FTP address to access the server. Connect
> to the server, follow the path in your email, and look for the file
> `raw/monthly/GPP.ANN.CRUNCEPv6.monthly.2000.nc`. Download that file and save in
> in a folder called `/RAWOBS/Tier3/`.
>
> Note: you'll need a user-friendly ftp client. On Linux, `ncftp` works okay.
{: .challenge}

> ## What is the deal with those "tiers"?
>
> Many datasets come with access restrictions. In this way the data providers
> can keep track of how their data is used. In many cases "restricted access"
> just means that one has to register with an email address and accept the terms
> of use, which typically ask that you acknowledge the data providers.
>
> There are also datasets available that do not need a registration. The
> "obs4MIPs" or "ana4MIPs" datasets, for example, are specifically produced to
> facilitate comparisons with model simulations.
>
> To reflect these different levels of access restriction, the ESMValTool team
> has created a tier-system. The definition of the different tiers are
> as follows:
>
> - **Tier1**: obs4MIPs and ana4MIPS datasets (can be used directly with the ESMValTool)
> - **Tier2**: other freely available datasets (most of them will need some kind of
>   cmorization)
> - **Tier3**: datasets with access restrictions (most of these datasets will also
>   need some kind of cmorization)
>
> These access restrictions are also the reason why the ESMValTool developers
> cannot distribute copies or automate downloading of all observations and
> reanalysis data used in the recipes. As a compromise we provide the
> CMORization scripts so that each user can CMORize their own copy of the access
> restricted datasets if they need them.
>
{: .callout}

## Make a test recipe

Now that we have downloaded the data, our end goal is to run a recipe that can
load it and work with it. So we will make this recipe and try to run it. It will
probably fail, but that's okay. As you will see, the error messages will come in
handy. Moreover, the recipe will serve as a test case. Once we get it to run, we
know we have completed our task.

> ## Create a test recipe
>
> Create a simple recipe that loads the "FLUXCOM" data. It should include a
> datasets section with a single entry for the "FLUXCOM" dataset with the correct
> dataset keys, and a diagnostics section with two variables: gpp and gppStderr.
> We won't need any preprocessors or scripts (set `scripts: null`), but you will
> have to add a documentation section with a description, authors and
> maintainer, otherwise the recipe will fail.
>
> Use the following dataset keys:
>
> - project: OBS6
> - dataset: FLUXCOM
> - type: reanaly
> - version: ANN-v1
> - mip: Lmon
> - start_year: 2000
> - end_year: 2000
> - tier: 3
>
> Some of these dataset keys are further explained in the callout boxes in this
> episode.
>
> > ## Answer
> >
> > Here's an example recipe
> >
> > ```yaml
> > documentation:
> >
> >   description: Test recipe for FLUXCOM data
> >
> >   authors:
> >     - kalverla_peter
> >
> >   maintainer:
> >     - kalverla_peter
> >
> > datasets:
> >   - {project: OBS6, dataset: FLUXCOM, mip: Lmon, tier: 3, start_year: 2000, end_year: 2000, type: reanaly, version: ANN-v1}
> >
> > diagnostics:
> >   FLUXCOM:
> >     description: Check that ESMValTool can load the cmorized FLUXCOM data without errors.
> >     variables:
> >       gpp:
> >       gppStderr:
> >     scripts: null
> >
> > ```
> >
> > Note: a recipe similar to this one is available under
> > `~/path/to/ESMValTool/esmvaltool/recipes/examples/recipe_check_obs.yml`.
> > That recipe includes checks all datasets for which CMORizers are available.
> >
> {: .solution}
{: .challenge}

Try to run the example recipe with

```bash
esmvaltool run recipe_check_fluxnet.yml --log_level debug
```

The `log_level` flag ensures that all relevant information is included in the
output. We will need this information to find out what needs to be done. Look
carefully through the log messages. You'll probably find something like

```
DEBUG   Retrieving OBS6 configuration
DEBUG   Skipping non-existent /home/peter/default_inputpath/Tier3/FLUXCOM
DEBUG   Looking for files matching ['OBS6_FLUXCOM_reanaly_ANN-v1_Lmon_gppStderr[_.]*nc'] in []
ERROR   No input files found for variable {'variable_group': 'gppStderr' ...
...
ERROR   Missing data for preprocessor FLUXCOM/gppStderr
...
ERROR   Program terminated abnormally, see stack trace below for more information:
...
```
{: .error}

So the output tells us that it cannot find the data, and also where it has been
looking. Let's try to understand what's going on.

## Location, location, location

The error messages above tell you that ESMValTool cannot find the data. That
makes total sense, because we have not yet created our CMORized copy of the
data. Our data is located in the `RAWOBS` folder, but ESMValTool is looking in
`OBS6`. So the first thing our CMORizer will need to do is copy the data over
from one folder to the other. To do end, we need to tell ESMValTool where our
data may be found.

> ## Set the correct paths in your user configuration file:
>
> This information is set in `config-user.yml`. Modify your
> configuration file so that it has the correct paths
>
> ```yaml
> rootpath:
>   OBS6: /path/to/my/obs6/data
>   RAWOBS: /path/to/my/rawobs/data
> ```
{: .challenge}

> ## RAWOBS, OBS, OBS6!?
>
> ESMValTool uses project IDs to find the data on your hard drive, and also to
> find more information about the data. The `RAWOBS` and `OBS` projects were
> created for external data before and after CMORization, respectively. These
> names can be misleading, though, since not all external datasets are
> observations.
>
> Then, in going from CMIP5 to CMIP6, the CMOR standards changed a bit. Some
> variables are named differently in CMIP5 and CMIP6. This posed a dilemma:
> should CMORization reformat to the CMIP5 or CMIP6 definition? To solve this,
> the `OBS6` project was created. So data in the `OBS6` follow the CMIP6
> standards.
>
{: .callout}

Looking closer at the path that ESMValTool told us it went looking for the data,
we see that it has a very specific structure:

`OBS6_FLUXCOM_reanaly_ANN-v1_Lmon_gppStderr[_.]*nc`

Note that all components in the filename correspond to one of the dataset keys
defined in our test recipe. By replacing them with the original tags, we can
see the general structure for the OBS6 data:

`[project]_[dataset]_[type]_[version]_[mip]_[short_name]_*.nc`

This is the Data Reference Syntax (DRS) for OBS6 data. Each project has its own
DRS, and some project IDs even have multiple different options. These settings
can be found in the file
[config-developer.yml](https://github.com/ESMValGroup/ESMValCore/blob/master/esmvalcore/config-developer.yml).
Luckily, we don't need to worry about that. This file name will be automatically
correctly created when we run our CMORizer script.


## Check if your variable is following the CMOR standard / Check if it's in a CMOR table

The very first step we have to do is to check if your data file follows the CMOR standard.
Only data files that fully follow this standard can be read by the ESMValTool.

Most variables that we would want to use with the ESMValTool are defined in the Coupled
Model Intercomparison Project (CMIP) data request and can be found in the
CMOR tables in the folder
[cmip6/Tables](https://github.com/ESMValGroup/ESMValCore/tree/master/esmvalcore/cmor/tables/cmip6/Tables),
differentiated according to the "MIP" they belong to. The tables are a
copy of the [PCMDI](https://github.com/PCMDI) guidelines.

> ## Find the variable "gpp" in a CMOR table
>
> Check the available CMOR tables to find the variable "gpp" with the following characteristics:
> - standard_name: ``gross_primary_productivity_of_biomass_expressed_as_carbon``
> - frequency: ``mon``
> - modeling_realm: ``land``
>
> > ## Answers
> >
> > The variable "gpp" belongs to the land variables. The temporal resolution that we are looking
> > for is "monthly". This information points to the "Lmon" CMIP table. And indeed, the variable
> > "gpp" can be found in the file
> > [here](https://github.com/ESMValGroup/ESMValCore/blob/master/esmvalcore/cmor/tables/cmip6/Tables/CMIP6_Lmon.json).
> >
> {: .solution}
{: .challenge}


If the variable you are interested in is not available in the standard CMOR tables,
you need to write a custom CMOR table entry for the variable. Don't worry! It sounds
more complicated than it is! Examples of custom CMOR table entries are for example
the standard error of a specific variable.
For our variable "gpp" there is indeed no CMOR definition for the standard error,
therefore "gppStderr" was defined in the custom CMOR table
[here](https://github.com/ESMValGroup/ESMValCore/tree/master/esmvalcore/cmor/tables/custom),
as ``CMOR_gppStderr.dat``.

To create a new custom CMOR table you need to follow these
guidelines:

- Provide the ``variable_entry``;
- Provide the ``modeling_realm``;
- Provide the variable attributes, but leave ``standard_name`` blank. Necessary
  variable attributes are: ``units``, ``cell_methods``, ``cell_measures``,
  ``long_name``, ``comment``.
- Provide some additional variable attributes. Necessary additional variable
  attributes are: ``dimensions``, ``out_name``, ``type``. There are also
  additional variable attributes that can be defined here (see the already
  available cmorizers).

It is easiest to start a new custom CMOR table by using an existing custom table
as a template.
You can then edit the content and save it as ``CMOR_<short_name>.dat``.

> ## Does the variable "cVegStderr" need a costum CMOR table?
>
> Check the available CMOR tables to find the variable "cVegStderr" with the
> following characteristics:
> - standard_name: ``vegetation_carbon_content``
> - frequency: ``mon``
> - modeling_realm: ``land``
>
> If it is not available, create a custom CMOR table following the template of
> the CMIP6 CMOR table of "cVeg" and custom CMOR table of "gppStderr".
>
> > ## Answers
> >
> > The first step here is to check if the variable "cVegStderr" is already
> > listed in a CMOR table. We have the information that the modeling_realm
> > of the variable is ``land`` and that the frequency is ``mon``. This
> > means we have to check the CMOR table "Lmon" for an entry. We focus
> > our search on the CMIP6 tables since these are the most recent tables
> > available. You can find the lists here:
> > `<https://github.com/ESMValGroup/ESMValCore/tree/master/esmvalcore/cmor/tables/cmip6/Tables>`
> >
> > We do not find the variable "cVegStderr" in the "Lmon" table, which
> > means we will have to write our own custom table for this.
> > There were two examples given which we could use as templates for the
> > new custom table that we have to create. These two examples can be
> > found here: [cVeg](https://github.com/ESMValGroup/ESMValCore/blob/master/esmvalcore/cmor/tables/cmip6/Tables/CMIP6_Lmon.json)
> > and [gppStderr](https://github.com/ESMValGroup/ESMValCore/blob/master/esmvalcore/cmor/tables/custom/CMOR_gppStderr.dat)
> >
> > We have to create a new file with the name ``CMOR_cVegStrerr.dat`` in
> > the custom CMOR table folder (https://github.com/ESMValGroup/ESMValCore/blob/master/esmvalcore/cmor/tables/custom/).
> > The content of the file should then look like this:
> >
> > ```
> > SOURCE: CMIP6
> > !============
> > variable_entry:    cVegStderr
> > !============
> > modeling_realm:    land
> > !----------------------------------
> > ! Variable attributes:
> > !----------------------------------
> > standard_name:
> > units:             kg m-2
> > cell_methods:      area: mean where land time: mean
> > cell_measures:     area: areacella
> > long_name:         Carbon Mass in Vegetation Error
> > !----------------------------------
> > ! Additional variable information:
> > !----------------------------------
> > dimensions:        longitude latitude time
> > out_name:          cVegStderr
> > type:              real
> > !----------------------------------
> > !
> > ```
> >
> > Note that there is no entry for ``standard_name``. This is on purpose.
> > It is a sign for the ESMValTool to not crash although the variable that
> > we are looking for is ok to have no official CMIP6 ``standard_name``.
> >
> {: .solution}
{: .challenge}

## 5. Create a CMORizer for the new dataset

Now that we have the data ready, have told the ESMValTool where to look
for it, and have made sure that our variable of choice is listed either
on a pre-existing or custom CMOR table, let's test if the data is actually 
following the necessary CMOR standard already, or if we have to do some
reformatting for the dataset.

> ## Run the test recipe again
>
> Run the test recipe again from earlier where you wanted to check if the
> ESMValTool can already find and read the freshly downloaded FLUXCOM data.
>
> > ## Answers
> >
> > ```bash
> > esmvaltool run recipe_check_fluxnet.yml --log_level debug
> > ``` 
> > 
> > The ESMValTool should now find your FLUXCOM datafile, and the error
> > message that the ESMValTool can't find the data should now not be present
> > anymore. However, there are plenty of other error messages around. 
> > Somewhere within the many messages, you should see this:
> > 
> > ```bash
> > ...
> > esmvalcore.cmor.check.CMORCheckError: There were errors in variable GPP:
> > Variable GPP units unknown can not be converted to kg m-2 s-1
> >  lon: standard_name should be longitude, not None
> >  lat: standard_name should be latitude, not None
> >  lon longitude coordinate has values < -360 degrees
> >  lon longitude coordinate has values > 720 degrees
> >  lat: has values < valid_min = -90.0
> >  lat: has values > valid_max = 90.0
> >  GPP: does not match coordinate rank
> > in cube:
> > gross_primary_productivity_of_carbon / (unknown) (time: 12; lat: 360; lon: 720)
> >      Dimension coordinates:
> >           time                                        x        -         -
> >           lat                                         -        x         -
> >           lon                                         -        -         x
> >      Attributes:
> >           created_by: Fabian Gans [fgans@bgc-jena.mpg.de], Ulrich Weber [uweber@bgc-jena.mpg...
> >           flux: GPP
> >           forcing: CRUNCEPv6
> >           institution: MPI-BGC-BGI
> >           invalid_units: gC m-2 day-1
> >           method: Artificial Neural Networks
> >           provided_by: Martin Jung [mjung@bgc-jena.mpg.de] on behalf of FLUXCOM team
> >           reference: Jung et al. 2016, Nature; Tramontana et al. 2016, Biogeosciences
> >           source_file: /mnt/lustre02/work/bd0854/b309143/Data/Tier3/FLUXCOM/OBS_FLUXCOM_reana...
> >           temporal_resolution: monthly
> >           title: GPP based on FLUXCOM RS+METEO with CRUNCEPv6 climate
> >           version: v1
> > ...
> > ``` 

> {: .solution}
{: .challenge}

The error messages that we see tell us that we do have to reformat the dataset
slightly to have it follow the CMOR standard. It seems like the variable "gpp"
does not have the correct unit that is given in the CMOR table where it is 
[defined](https://github.com/ESMValGroup/ESMValCore/tree/master/esmvalcore/cmor/tables/cmip6/Tables).
It also seems like the "latitude" and "longitude" coordinates have some 
problems. So let's start writing a short python script that will fix these 
problems.

The first step now is to create a file in the right folder that will contain
the short python script. The home of all CMORizer scripts for observations 
and reanalysis datasets is 
[here](https://github.com/ESMValGroup/ESMValTool/tree/master/esmvaltool/cmorizers/obs). 
Add a file with the name ``cmorize_obs_fluxcom.py`` to this folder.

> ## Note
>
> Always, always, when modifying or creating new code for the ESMValTool 
> repositories, work on your *own, local* branch of the ESMValTool. Optimally,
> you have forked that branch directly from the most up-to-date version of 
> the "master" branch to avoid conflicts later when you want to merge your
> code with the "master" branch of the ESMValTool. 
>
{: .callout}

Within our python script we have to make sure that we cover three main 
aspects of the reformatting:
- We need to be able to locate and read the data to be CMORized
- We have to CMORize the data (fix the CMOR problems that the ESMValTool
had so nicely pointed out for us)
- We need to store the data with the correct filename so that the ESMValTool
will be able to identify it later

But the very first part of the CMORizing script is a header. The header 
contains information about where to obtain the data, when it was accessed
the last time, which ESMValTool "tier" it is associated with, and more 
detailed information about the necessary downloading and processing steps.

> ## Fill out the header for the "FLUXCOM" dataset
>
> Fill out the information that is necessary in the header of a CMORizing
> script for the dataset "FLUXCOM". The different parts that need to be
> present in the header are the following:
> - caption: " """ESMValTool CMORizer for FLUXCOM GPP data."
> - Tier
> - Source
> - Last access
> - Download and processing instructions
>
> > ## Answers
> >
> > The header for the "FLUXCOM" dataset could look something like this:
> >
> > ```python
> > """ESMValTool CMORizer for FLUXCOM GPP data.
> > 
> > Tier
> >     Tier 3: restricted dataset.
> > 
> > Source
> >     http://www.bgc-jena.mpg.de/geodb/BGI/Home
> > 
> > Last access
> >     20190727
> > 
> > Download and processing instructions
> >     From the website, select FLUXCOM as the data choice and click download.
> >     Two files will be displayed. One for Land Carbon Fluxes and one for
> >     Land Energy fluxes. The Land Carbon Flux file (RS + METEO) using
> >     CRUNCEP data file has several data files for different variables.
> >     The data for GPP generated using the
> >     Artificial Neural Network Method will be in files with name:
> >     GPP.ANN.CRUNCEPv6.monthly.\*.nc
> >     A registration is required for downloading the data.
> >     Users in the UK with a CEDA-JASMIN account may request access to the jules
> >     workspace and access the data.
> >     Note : This data may require rechunking of the netcdf files.
> >     This constraint will not exist once iris is updated to
> >     version 2.3.0 Aug 2019
> > """
> > ```
> > 
> > This is the header of the "FLUXCOM" CMORizer that is available with the
> > ESMValTool already. It is therefore entirely possible that your entries for
> > the section "Last access" and "Download and processing instructions" 
> > differ from the example here since the entries for these sections are
> > somewhat subjective.
> > 
> {: .solution}
{: .challenge}

Now that we have made sure, that it is clear where our dataset comes from, and
how we can obtain it, the next step is to add the python code to actually read
the data.

As we learned earlier, the DRS of the ESMValTool needs some very specific
pieces of information to build the name for each data file. Since we are now
trying to introduce a new dataset to the ESMValTool, we will have to specify
some things to make sure the dataset name can be correctly created, and
ultimately the ESMValTool knows how to look for the new dataset.

Therefore it is necessary to create a configuration file for the new dataset.
This configutation file needs to be stored in the 
following folder: 

```bash
ESMValTool/esmvaltool/cmorizers/obs/cmor_config/
```
It is imporant to note that the name of the configuration file has to be 
identical to the name of the dataset. For our example the configuration file,
traditionally written in the ``yaml`` format, therefore must be called 
``FLUXCOM.yml``.

Let's have a closer look what that configuration file needs to contain:
- information about the filenames of the raw observation files (stored in the
"RAWOBS" folder)
- information about "global attributes" for the netCDF file that will be
created (both for the name of the file and the metadata that the file needs to
contain)
- information about the variables that need to be cmorized

> ## Let's create the configuration file for the "FLUXCOM" dataset
>
> Here is the skeleton of the "FLUXCOM" configuration file as it exists in 
> the ESMValTool framework. Try to fill in all missing pieces of information
> for this configuration file that are marked with ``???``.
>
> ```yaml
> ---
> # Filename 
> filename: ???
> 
> # Common global attributes for Cmorizer output
> attributes:
>   dataset_id: ???
>   version: ???
>   tier: ???
>   modeling_realm: ???
>   project_id: ???
>   source: ???
>   reference: ???
>   comment: ???
> 
> # Variables to cmorize
> variables:
>   ???:
>     mip: ???
> ```
>
> > ## Answers
> >
> > The configuration file for the "FLUXCOM" dataset with all necessary pieces
> > of information looks  like this:
> >
> > ```yaml
> > ---
> > # Filename 
> > filename: 'GPP.ANN.CRUNCEPv6.monthly.*.nc'
> > 
> > # Common global attributes for Cmorizer output
> > attributes:
> >   dataset_id: FLUXCOM
> >   version: 'ANN-v1'
> >   tier: 3
> >   modeling_realm: reanaly
> >   project_id: OBS
> >   source: 'http://www.bgc-jena.mpg.de/geodb/BGI/Home'
> >   reference: 'fluxcom'
> >   comment: ''
> > 
> > # Variables to cmorize
> > variables:
> >   gpp:
> >     mip: Lmon
> > ```
> > 
> > The original configuration file for the "FLUXCOM" dataset can be
> > found here:
> > [FLUXCOM.yml]
> > (https://github.com/ESMValGroup/ESMValTool/blob/master/esmvaltool/cmorizers/obs/cmor_config/FLUXCOM.yml)
> > 
> > Note the attribute "reference" here: it should include a ``doi`` related 
> > to the dataset. For more information on how to add references to the 
> > ``reference`` section of the configuration file, see the section in the
> > documentation about this: [adding references]
> > (https://docs.esmvaltool.org/en/latest/community/diagnostic.html#adding-references)
> > 
> > If a single dataset has more than one reference, it is possible to add 
> > tags as a list e.g. ``reference: ['tag1', 'tag2']``.
> {: .solution}
{: .challenge}
 
Now that we have defined the configuration file for our "FLUXCOM" data, we can
finally start writing the actual code for the CMORizer script. The main body 
of the CMORizer script must contain a function called

```python
def cmorization(in_dir, out_dir, cfg, config_user):
```

with this exact call signature. Here, ``in_dir`` corresponds to the input
directory of the raw files, ``out_dir`` to the output directory of final
reformatted data set and ``cfg`` to the configuration dictionary given by
the  ``.yml`` configuration file. The return value of this function is ignored.
All the work, i.e. loading of the raw files, processing them and saving the 
final output, has to be performed inside its body. To simplify this process, 
ESMValTool provides a set of predefined utilities.py_, which can be imported 
into your CMORizer by

```python
from . import utilities as utils
```

Apart from a function to easily save data, this module contains different 
kinds of small fixes to the data attributes, coordinates, and metadata which 
are necessary for the data field to be CMOR-compliant. We will come back to 
these functionalities in a bit. 

Note that this specific CMORizer script contains several subroutines in order
to make the code clearer and more readable (we strongly recommend to follow
that code style). For example, the function ``_get_filepath`` converts the raw
filepath to the correct one and the function ``_extract_variable`` extracts and
saves a single variable from the raw data.

After all that theory, let's have a look at the actualy python code of the 
existing "FLUXCOM" CMORizer script. For now, we only want to read in the data
and then store it in a new file.

```python
"""ESMValTool CMORizer for FLUXCOM GPP data.

Tier
    Tier 3: restricted dataset.

Source
    http://www.bgc-jena.mpg.de/geodb/BGI/Home

Last access
    20190727

Download and processing instructions
    From the website, select FLUXCOM as the data choice and click download.
    Two files will be displayed. One for Land Carbon Fluxes and one for
    Land Energy fluxes. The Land Carbon Flux file (RS + METEO) using
    CRUNCEP data file has several data files for different variables.
    The data for GPP generated using the
    Artificial Neural Network Method will be in files with name:
    GPP.ANN.CRUNCEPv6.monthly.*.nc
    A registration is required for downloading the data.
    Users in the UK with a CEDA-JASMIN account may request access to the jules
    workspace and access the data.
    Note : This data may require rechunking of the netcdf files.
    This constraint will not exist once iris is updated to
    version 2.3.0 Aug 2019
"""
import logging
import os
import re
import numpy as np
import iris
from . import utilities as utils

logger = logging.getLogger(__name__)


def _get_filepath(in_dir, basename):
    """Find correct name of file (extend basename with timestamp)."""
    regex = re.compile(basename)

    all_files = [
        f for f in os.listdir(in_dir)
        if os.path.isfile(os.path.join(in_dir, f))
    ]
    for filename in all_files:
        if regex.match(filename):
            return os.path.join(in_dir, basename)
    raise OSError(
        f"Cannot find input file matching pattern  '{basename}' in '{in_dir}'")


def _extract_variable(cmor_info, attrs, filepath, out_dir):
    """Extract variable."""
    var = cmor_info.short_name
    logger.info("Var is %s", var)
    cubes = iris.load(filepath)
    for cube in cubes:
        logger.info("Saving file")
        utils.save_variable(cube,
                            var,
                            out_dir,
                            attrs,
                            unlimited_dimensions=['time'])


def cmorization(in_dir, out_dir, cfg, _):
    """Cmorization func call."""
    glob_attrs = cfg['attributes']
    cmor_table = cfg['cmor_table']
    filepath = _get_filepath(in_dir, cfg['filename'])
    logger.info("Found input file '%s'", filepath)

    # Run the cmorization
    for (var, var_info) in cfg['variables'].items():
        logger.info("CMORizing variable '%s'", var)
        glob_attrs['mip'] = var_info['mip']
        logger.info(var_info['mip'])
        cmor_info = cmor_table.get_variable(var_info['mip'], var)
        _extract_variable(cmor_info, glob_attrs, filepath, out_dir)
```

Let's run this CMORizing script to see if the dataset is read correctly, and
what kind of file is written out. Tere is a specific command available in the
ESMValTool to run the CMORizing scripts:

```bash
cmorize_obs -c <config-user.yml> -o <dataset-name>
```

The ``config-user-yml`` is the file in which we define the different data 
paths, e.g. where the ESMValTool would find the "RAWOBS" folder. The 
``dataset-name`` needs to be idential to the folder name that was created
to store the raw observation data files, in our case this would be "FLUXCOM".
The ESMValTool will create a folder with the correct tier information in your
defined output directory if that tier folder is not already available, and 
then a folder named after the data set. In this folder the cmorized data set 
will be stored as a netCDF file. If your run was successful, one or more 
NetCDF files are produced in your output directory.

> ## Was the CMORization successful so far?!
>
> If you check the folders in your output path, you should see the following
> folder structure:
> ```bash
> /Tier3/FLUXCOM/
> ```
>
> Within the "FLUXCOM" folder there should be a NetCDF file with the name:
> ```bash
> OBS_FLUXCOM_reanaly_ANN-v1_Lmon_gpp_xxxx01-xxxx12.nc
> ``` 
>
> The "xxxx" represents the start year of the data period you wanted to 
> CMORize, and the "yyyy" represents the end year.
> 
{: .callout}

Great! 
So we have produced a NetCDF file with the CMORizer that follows the naming
convention for ESMValTool datasets. Let's have a look at the NetCDF file as 
it was written with the very basic CMORizer from above (note, we are only 
looking at the year 1980 in our example).
 
```bash
netcdf OBS_FLUXCOM_reanaly_ANN-v1_Lmon_gpp_198001-198012 {
dimensions:
        time = UNLIMITED ; // (12 currently)
        lat = 360 ;
        lon = 720 ;
variables:
        float GPP(time, lat, lon) ;
                GPP:_FillValue = 1.e+20f ;
                GPP:long_name = "GPP" ;
        double time(time) ;
                time:axis = "T" ;
                time:units = "days since 1582-10-15 00:00:00" ;
                time:standard_name = "time" ;
                time:calendar = "gregorian" ;
        double lat(lat) ;
        double lon(lon) ;

// global attributes:
                :_NCProperties = "version=2,netcdf=4.7.4,hdf5=1.10.6" ;
                :created_by = "Fabian Gans [fgans@bgc-jena.mpg.de], Ulrich Weber [uweber@bgc-jena.mpg.de]" ;
                :flux = "GPP" ;
                :forcing = "CRUNCEPv6" ;
                :institution = "MPI-BGC-BGI" ;
                :invalid_units = "gC m-2 day-1" ;
                :method = "Artificial Neural Networks" ;
                :provided_by = "Martin Jung [mjung@bgc-jena.mpg.de] on behalf of FLUXCOM team" ;
                :reference = "Jung et al. 2016, Nature; Tramontana et al. 2016, Biogeosciences" ;
                :temporal_resolution = "monthly" ;
                :title = "GPP based on FLUXCOM RS+METEO with CRUNCEPv6 climate " ;
                :version = "v1" ;
                :Conventions = "CF-1.7" ;
}
```

The file contains a variable named "GPP" that contains three dimensions:
"time", "lat", "lon". The units for this variable are not defined yet.
The ESMValTool did not know how to convert from the
original units to the units that are defined in the CMOR table for the 
variable "gpp". The original units are therefore listed in the "global
attributes" section as ``invalid_units``. The ESMValTool recognized that the
units given in the original file did not match the units given in the CMOR
table and therefore put the information about the units in the metadata.

If we test this NetCDF file now with our CMOR checker
```bash
esmvaltool run recipe_check_fluxcom.yml --log_level debug
```
we will encounter an error again. The dataset is not correctly CMORized, and
the first thing that the ESMValTool complains about is:
```bash
iris.exceptions.UnitConversionError: Cannot convert from unknown units. The 
"units" attribute may be set directly.
```

Ok, so let's fix the units of the "GPP" variable in the CMORizer. 


.. utilities.py: https://github.com/ESMValGroup/ESMValTool/blob/master/esmvaltool/cmorizers/obs/utilities.py


## 6. Run the CMORizer script

The cmorizing script for the given dataset can be run with:

```bash
cmorize_obs -c <config-user.yml> -o <dataset-name>
```

> ## Note
>
> The output path given in the configuration file is the path where
> your cmorized dataset will be stored. The ESMValTool will create a folder
> with the correct tier information (see Section `2. Edit your configuration file`) if that tier folder is not
> already available, and then a folder named after the data set. In this
> folder the cmorized data set will be stored as a netCDF file.
{: .callout}

If your run was successful, one or more NetCDF files are produced in your
output directory.

> ## Check that your cmoriziation was successful
>
> Check your output directory for the freshly produced NetCDF file.
>
> > ## Answers
> >
> > Write the answer here.
> >
> {: .solution}
{: .challenge}



## Conclusion


[documentation](https://docs.esmvaltool.org/en/latest/input.html#observations)

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