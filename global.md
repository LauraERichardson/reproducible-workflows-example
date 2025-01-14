---
title: "Example of a reproducible workflow in R"
author: "Ceres Barros"
date: "Last edited: 11 Jul 2023"
output:
  bookdown::html_document2:  
    toc: true
    toc_float: true
    toc_depth: 4
    theme: sandstone
    number_sections: false
    df_print: paged
    keep_md: yes
editor_options:
  chunk_output_type: console
bibliography: references.bib
link-citations: true
always_allow_html: true
---



## Introduction

Repeatable, Reproducible, Reusable and Transparent ($R^3$T) workflows are becoming more and more commonly used in academic and non-academic settings to ensure that analyses and their outputs can be verified and repeated by peers, stakeholders and even the public. As such, ($R^3$T) promote trust within the scientific community and between scientists, stakeholders, end-users and the public. $R^3$T are also fundamental for model benchmarking, conducting meta-analyses and facilitating building-on and improving current scientific methods, especially those involving statistical and non-statistical modelling.

In the ecological domain, $R^3$T are gaining traction, but many of us struggle at the start, not knowing where to begin. This example will take you through setting up an $R^3$T workflow for a common analysis in ecology, species distribution modelling.

The workflow has been kept as simple as possible, while following a few common guidelines that facilitate building $R^3$T workflows:

1.  Scripting;

2.  Minimising no. software/languages used;

3.  Modularising & "functionising";

4.  Centralising the workflow in a single script;

5.  Using a self-contained & project-oriented workflow;

6.  Using version control;

7.  Using (minimal) integrated testing.

It also relies on two important packages that enable reproducibility and speed up subsequent runs (i.e. re-runs) the workflow in the same machine:

-   [`Require`](https://require.predictiveecology.org/) [@Require] - for self-contained and reproducible `R` package installation;

-   [`reproducible`](https://reproducible.predictiveecology.org/) [@reproducible] - for caching, establishing explicit links to raw data, common GIS operations, and more.

However, please note that there are numerous other `R` packages (as well as other languages and types of software) that facilitate and support $R^3$T workflows. This is simply an example using some of the tools that I use when building my workflows.

## The workflow at a glance

This example workflow aims to project climate-driven range changes of white spruce (*Picea glauca*), a common tree species in Canada. I focused on the province of British Columbia (BC) and, for simplicity sake, only considered the effects of four bioclimatic predictors [obtained from [WorldClim](https://www.worldclim.org/data/bioclim.html); @fick2017]:

-   BIO1 - Annual Mean Temperature;

-   BIO4 - Temperature Seasonality;

-   BIO12 - Annual Precipitation;

-   BIO15 - Precipitation Seasonality.

The baseline white spruce distribution for BC was obtained from @beaudoin2014 maps of percent cover from 2011, available at Canadian National Forest Inventory database, and cropped to the province boundaries. The baseline values for each bioclimatic variable corresponded to projections for the normal climate period of 1970-2010 (hereafter, 2010), while future values where obtained from projections for the periods 2021-2040 (2030), 2041-2060 (2050), 2061-2080 (2070) and 2081-2100 (2090), using the CanESM5 General Circulation Model and the SSP 585 emissions scenario.

In this workflow, I use a single random forest model [@randomForest] to model white spruce range changes as a function of the four bioclimatic variables.

All steps from sourcing and formatting the data, to evaluating and plotting model results, are integrated in the workflow as a sequence of sourced scripts. This method was chosen as it is likely closer to the way most of us learn of to use `R`. However, other approaches could also be used, for instance:

-   converting existing scripts to single or multiple functions called from the controller script (see [Running the workflow]);

-   adapting scripts to `SpaDES` [@chubaty; see the [SpaDES4Dummies guide](https://ceresbarros.github.io/SpaDES4Dummies/index.html)] modules and using `SpaDES` to execute the workflow;

-   using `targets` [see @brousil2023] to execute the workflow;

-   and more...

## Step by step

### Project structure and dependencies

A workflow should start with the initial set-up necessary to execute all operations. In `R` this mostly concerns the R version, setting up project directories and package installation and loading. Other examples are `options()` (set afterwards in this workflow -- [Functions, options and study area]) or any necessary system variables (not used here).

Note that in *self-contained workflows* all packages should be installed at the *project level* to avoid conflicts (or changing) the system and/or user `R` library.


```r
## an assertion:
if (getRversion() < "4.3") {
  warning(paste("This workflow was build using R v4.3.0;",
                "it is recommended you update your R version to v4.3.0 or above"))
}
```


```r
## package installation with Require

## create project folder structure
projPaths <- list("pkgPath" = file.path("packages", version$platform,
                                        paste0(version$major, ".", strsplit(version$minor, "[.]")[[1]][1])),
                  "cachePath" = "cache",
                  "codePath" = "code",
                  "dataPath" = "data",
                  "figPath" = "figures",
                  "outputsPath" = "outputs")
lapply(projPaths, dir.create, recursive = TRUE, showWarnings = FALSE)
```

```
## $pkgPath
## [1] FALSE
## 
## $cachePath
## [1] FALSE
## 
## $codePath
## [1] FALSE
## 
## $dataPath
## [1] FALSE
## 
## $figPath
## [1] FALSE
## 
## $outputsPath
## [1] FALSE
```

```r
## set R main library to project library
.libPaths(projPaths$pkgPath)
```

I use `Require` to install all necessary packages. `Require` can install a mix of CRAN and GitHub packages, and accepts specification of minimum package versions (see `?Require`). For simplicity sake, this example uses the latest versions of each package from CRAN.

Another advantage of using `Require` is that it can save a "package snapshot" (i.e. the state of an `R` library at a particular time) and use it at a later date to install packages using the versions (and sources - CRAN or GitHub) specified in the snapshot file. This ensures that the same library state can be recreated in the future or on another machine.

To demonstrate this, and because it is an essential piece of workflow reproducibility, the code below shows both the initial package installation (ll. 146-151) and saving of the snapshot file (ll. 155-158) -- commented to avoid unintended package/snapshot updates during subsequent runs -- and package installation using the snapshot file.

/!\\ You will need an active internet connection to run (or re-run) the package installation lines /!\\


```r
## install specific versions of packages
installRequire <- !"Require" %in% rownames(installed.packages())
installRequire <- if (!installRequire) packageVersion("Require") < "0.3.0" else installRequire
if (installRequire) {
  install.packages("Require", dependencies = TRUE)
}

library(Require)

## this may take a while.
## Notes for Windows users: failure occurs often because the OS can "hold" on to folders.
##    Don't despair. Restart R session, close all other sessions and run the script up to
##    this point again.
# loadOut <- Require(c("data.table", "dismo",
#                      "ggplot2", "httr", "maps",
#                      "randomForest",
#                      "rasterVis", "reproducible", 
#                      "terra"),
#                    standAlone = TRUE)

## eventually we would save the library state in a package snapshot file
pkgSnapshotFile <- file.path(projPaths$pkgPath, "pkgSnapshot.txt")
# pkgSnapshot(
#   packageVersionFile = file.path(projPaths$pkgPath, "pkgSnapshot.txt"),
#   libPaths = projPaths$pkgPath, standAlone = TRUE
# )
## and replace the above Require() call with
Require(packageVersionFile = pkgSnapshotFile, 
        standAlone = TRUE,
        libPaths = projPaths$pkgPath,
        upgrade = FALSE, require = FALSE)

Require(c("data.table", "dismo",
          "ggplot2", "httr", "maps",
          "randomForest",
          "rasterVis", "reproducible",
          "terra"),
        install = FALSE)
```


```r
loadOut <- Require(c("data.table", "dismo",
                     "ggplot2", "httr", "maps",
                     "randomForest",
                     "rasterVis", "reproducible",
                     "terra"),
                   standAlone = TRUE)
```

In some Windows machines the package installation lines above may have to be run multiple times, with `R` session "refreshes" between them. This is because, Windows often "holds on" to temporary folders created during package installation, preventing `R` from manipulating these folders and causing installation failures. I'm afraid you'll simply have to be patient here.

### Running the workflow

The workflow presented here is relatively simple. It has been broken down into a series of `.R` scripts sourced in sequence. In the past, I found this to be the most intuitive way of learning how to shift from my spaghetti code into something structured and more robust. However, there are many other ways in which the workflow could have been set up. For instance, each of the scripts could be turned into a function and the functions called here; or packages like `SpaDES` and `targets` could be used to set and manage the workflow (see [The workflow at a glance]). The choice will depend on project complexity and on the level of programming proficiency of the developer, among other factors.

<!-- todo: try to execute code below in GH actions with a study SA-->


```r
## workflow functions/options and study area 
source(file.path(projPaths$codePath, "miscFuns.R"))
source(file.path(projPaths$codePath, "setup.R"))

## download data
source(file.path(projPaths$codePath, "climateData.R"))
source(file.path(projPaths$codePath, "speciesData.R"))

## fit SDM
source(file.path(projPaths$codePath, "fitSDM.R"))

## obtain projections
source(file.path(projPaths$codePath, "projections.R"))
```



#### Functions, options and study area

The first script, `miscFuns.R`, contains several functions used across other scripts. It is good practice to document functions -- what they do, what parameters they expect, and what packages they depend on. As in many `R` packages, I use the `roxygen` format, where function documentation is preceded by `#'` and documentation fields are noted by a `@`. Please see <!--roxygen ref/URL--> to learn more about documenting functions using `roxygen`.


```r
#' Plot a `SpatRaster` as a `ggplot`
#'
#'
#' @param ras a SpatRaster layer
#' @param title character. Plot title
#' @param xlab character. X-axis title
#' @param ylab character. Y-axis title
#' @param isDiscrete logical. Should raster data be treated as discrete
#'   or continuous for plotting? If `TRUE` plots will be accompanied with
#'   a colour legend and only existing values. Otherwise, a continuous
#'   colourbar is shown (default).
#'
#' @return a `ggplot`.
#'
#' @importFrom rasterVis gplot
#' @importFrom ggplot2 geom_tile scale_fill_brewer coord_equal theme_bw
plotSpatRaster <- function(ras, plotTitle = "", xlab = "x", ylab = "y", isDiscrete = FALSE) {
  ...
}
```

The second script of the workflow, `setup.R`, prepares the study area and sets some global `options()`. Notice that at the end of this (and other) script(s) I remove objects that are no longer necessary. Although this workflow will not demand a lot of memory, it is good practice to avoid spending memory resources on useless objects and cluttering the environment where the workflow is running -- in this case the `.GlobalEnv`.

#### Data sourcing and preparation

The second and third scripts, `climateData.R` and `speciesData.R`, source and prepare bioclimatic data layers and the white spruce percent cover layer (later converted to presence/absence values) necessary to fit the species distribution model (SDM) and to project changes in species distributions. To ensure that the workflow is reproducible and transparent, I have used FAIR data [@wilkinson2016] explicitly linked to the workflow and prepared on-the-fly by the `prepInputs` function (`reproducible` package). `prepInputs` downloads the data from the specified URL and when the data is a spatial layer it can crop it and re-project it to match another spatial layer (the `studyAreaRas` created in `setup.R`). It will only download the data once and uses internal caching to speed up subsequent runs with the same inputs (e.g. same input data layer and study area). You will also note that I have used outer `Cache` calls (also from the `reproducibility` package) for further speed up subsequent workflow runs.

These data prep. scripts also plot the final climate and species distribution data layers.

#### Analytical steps - model fitting and projections

The scripts `fitSDM.R` and `projections.R` respectively fit and use a random forest (RF) model that relates baseline (from 2011) white spruce presences and absences (derived from the input percent cover raster layer) to the four bioclimatic variables. The model is fit on 80% of the data and tested on the remaining 20% -- note the use of `set.seed()` to ensure that the data partitioning is repeatable. I used tools from the `dismo` package to create the training and testing data partitions, to evaluate the model and to predict changes in white spruce distributions under projected climate conditions for 2030, 2050, 2070 and 2090. I also used `Cache` again to avoid refitting and re-evaluating the model if inputs are not changed.

#### Outputs

The workflow's outputs are the RF model object, its evaluation results and plots of the raster layer projections. Instead of creating a separate script to save outputs to disk, I have embedded these operations within each script. I prefer saving outputs as they are being created, but in this case this is more a matter of personal taste. In very long workflows, however, it is advisable to save outputs as they are created to avoid losing work should there be a failure preventing workflow completion.

<div class="figure">
<img src="figures/SDMprojections_baseline.png" alt="Random forest projections of white spruce distributions in BC, Canada, under baseline conditions of four bioclimatic variables (annual mean temperature and annual precipitation, temperature and precipitation seasonality." width="1500" />
<p class="caption">(\#fig:fig-baselineproj)Random forest projections of white spruce distributions in BC, Canada, under baseline conditions of four bioclimatic variables (annual mean temperature and annual precipitation, temperature and precipitation seasonality.</p>
</div>

<div class="figure">
<img src="figures/SDMprojections_future.png" alt="Random forest projections of white spruce distributions in BC, Canada, for future values of the same four bioclimatic variables -- only one GCM and one emissions scenario were used." width="1500" />
<p class="caption">(\#fig:fig-futureproj)Random forest projections of white spruce distributions in BC, Canada, for future values of the same four bioclimatic variables -- only one GCM and one emissions scenario were used.</p>
</div>

#### And voilà!

That's it! I now invite you to make further changes to the workflow. Some ideas are to make it so that you can control inputs from the main script, turning the existing scripts into functions or using other workflow structures and packages like `SpaDES` and `targets`, which are beyond the scope of this example [but see [SpaDES4Dummies guide](https://ceresbarros.github.io/SpaDES4Dummies/index.html) and @brousil2023].

I hope this minimal example has given useful tools and ideas that will help you create $R^3$T workflows and enhance your existing work with these principles.


## References
