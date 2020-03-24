Simple R functions for importing and analyzing electrophoresis data from an Agilent 2100 Bioanalyzer or 2200/4150/4200 TapeStation.

# Why is this useful?

1. **More flexible and professional-looking graphs.** The graphing options in the Agilent software are limited, especially when you combine data from multiple batches. Importing the data into R, especially with the help of `ggplot2`, allows you to customize the presentation in any way you can imagine.
1. **Less misleading graphs.** The electropherograms from the Agilent software show only fluorescence on the y-axis. Fluorescence is proportional to mass, but often the real variable of interest is molarity: how many molecules? A 200 bp peak a 100 bp peak with twice as much molarity look the same on a fluorescence graph but very different on a molarity graph. Or if your only interest is the size distribution within each sample and not the total amount, you can normalize them all to the same scale for easy comparison.
1. **Automated analysis.** If you just want to perform a simple QC analysis like DV200 or library insert:dimer ratio, you can script that for an unlimited number of batches, or use the provided command-line script, and collect the results directly into a table—which you can also graph.


# Installation

    > library(devtools)
    > install_github("jwfoley/bioanalyzeR")

# Reading input data

## `read.electrophoresis`
Read data from one or more Bioanalyzer or Tapestation runs, given the path of the exported XML file(s) (and corresponding PNG files for TapeStation) and combine into one data set of class `electrophoresis`.

## The `electrophoresis` class
This is a generalized data structure for the data and metadata of one or more Bioanalyzer or TapeStation runs. The members are:

* `data` - a tall data frame of the run data, specifically:
	* `time` (Bioanalyzer) - time when this data point was measured
	* `aligned.time` (Bioanalyzer) - measurement time aligned between the expected times of the marker peaks
	* `distance` (TapeStation) - migration distance of the measurement from the top of the gel area
	* `relative.distance` (TapeStation) - migration distance normalized relative to the marker peaks
	* `fluorescence` - fluorescence reading at this point
	* `length` - estimated molecule length at this point
	* `concentration` - estimated concentration of the area under the curve between this point and the previous one
	* `molarity` - estimated molarity of the area under the curve between this point and the previous one
* `assay.info` - a list of metadata about each batch and the assay kit used
* `samples` - a data frame of metadata for each sample (also annotated to `data`, `peaks`, and `regions` with the same factor levels), specifically:
	* `batch` - the batch (instrument run) of the sample, from the file name
	* `well.number` - the well number in which the sample was loaded
	* `sample.name` - the name of the sample
	* `sample.observations` - notes about this sample supplied by the user or the Agilent software
	* `sample.comment` - notes about this sample supplied by the user
	* `reagent.id` (TapeStation) - the name of the ScreenTape used for this sample
	* `ladder.well` - which well contains the ladder that calibrates this sample
* `peaks` - a data frame of peaks reported by the Agilent software, annotated with their lower and upper boundaries in various scales
* `regions` - a data frame of regions of interest reported by the Agilent software, annotated with their lower and upper boundaries in varous scales
* `mobility functions` - a list of model functions, one per ladder used for calibration, to convert migration speed measurements (aligned time or relative distance) into estimated molecule lengths
* `mass.coefficients` - a list of coefficients, one per sample, to convert area under the curve to estimated mass


# Graphing the data

## `qplot.electrophoresis`
Given an `electrophoresis` object, this function graphs the data as a `ggplot` object, which can be drawn directly or customized by adding more features. `x` and `y` can be set to graph different variables such as fluorescence, concentration, or molarity.

# Checking the accuracy of the calculations

## stdcrv.mobility
Check the fit of the mobility model (molecule length vs. migration speed) by plotting the standard curve.

## qc.electrophoresis
Compare the estimated molecule lengths, concentrations, or molarities from this package with the values reported by the Agilent software.


# Manipulating the data

## `rbind.electrophoresis`
Combine multiple `electrophoresis` objects into one.

## `subset.electrophoresis`
Subset an `electrophoresis` object by some variable of the samples.


# Analyzing the data

## `integrate.peaks`, `integrate.regions`
Computes the sum of some variable in each peak in the peak table or each region in the region table.

## `integrate.custom`
Computes the sum of some variable within a single pair of boundaries, individually for each sample.

## `region.ratio`
Computes the ratio of sums within two or more regions in each sample.

## `dv200`
Shortcut for RNA: Computes the proportion of mass above 200 nt (DV200).

## `illumina.library.ratio`
Shortcut for Illumina sequencing libraries: Computes the molar ratio of good inserts to adapter dimers for each sample.


# How to export data

## Bioanalyzer
In the 2100 Expert software, open your data file (`.xad`) in the "Data" context. Select "File->Export..." from the top menu. Check the "Export to XML" box and no others. Click "Export" and then save the file wherever you like. This will create an XML file that can be read by the `read.bioanalyzer` function.

## TapeStation
In the TapeStation Analysis Software, open your data file (`.D1000`, `.HSD1000`, `.RNA`., `.gDNA`, etc.). You need to export both the metadata (in XML format) and gel image (PNG).

### Metadata XML
Select "File->Export Data->Export to XML". You do not need to export the gel image or individual EPG images at this point. Select your destination and then click "Export" to save the file.

### Gel image PNG
It is important to follow these unusual directions carefully!

1. In the "Home" tab (top), verify that there is a ladder lane ("Electronic Ladder" is okay) and that the markers are correctly identified in every sample (except failed lanes, which are okay).
1. Select the "Gel" context (top left button).
1. Select "Show All Lanes" if the button is not grayed out.
1. Unselect "Aligned", "Scale to Sample", and "Scale to MW Range" if it is not grayed out.
1. Leave the contrast slider in the middle.
1. Maximize the window and drag the lower end of the gel image area to make it as tall as possible. At this point you should see all lanes from the run, with the marker bands present but unaligned.
1. Right-click on a lane near the left end of the gel (it doesn't matter which) and you will see a context menu.
1. Move your cursor over "Snapshot" but **also over a lane to the right of the one you right-clicked on**.
1. Left-click "Snapshot". This will copy the gel image to your system clipboard, but you should see also see the newly selected lane become highlighted in light blue.
1. Open any image editor (e.g. Paint) and paste the image from your clipboard. You should see one lane highlighted in light blue. If not, try taking the snapshot again.
1. Save the gel image as a file in PNG format, preferably with the same name as the XML file and the `.png` extension.

### Why does `read.tapestation` need a gel image?
The TapeStation software cannot export the raw fluorescence data in any other format; the XML only contains metadata (sample ID, peak list, etc.). The data would be easily accessible inside the `.D1000`, `.HSD1000`, etc. file formats except they are encrypted. I have asked Agilent about this and they stated that they do not want the data to be accessible to users.

