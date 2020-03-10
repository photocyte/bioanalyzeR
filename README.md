Simple R functions for importing and analyzing electrophoresis data from an Agilent 2100 Bioanalyzer or 2200/4150/4200 TapeStation.

# Reading input data
Simple R functions for importing and graphing electrophoresis data from an Agilent 2100 Bioanalyzer or 2200/4150/4200 TapeStation.

## `read.bioanalyzer`
Reads data from a Bioanalyzer XML files (see below for how to export those) into an S3 object of class `electrophoresis` containing the raw data and varous metadata. 

## `read.tapestation`
Reads data from a TapeStation run, requiring both an XML metadata file and a PNG gel-image file (see below for how to export those), into an S3 object of class `electrophoresis` containing the raw data and varous metadata.

## The `electrophoresis` class
This is a generalized data structure for the data and metadata of one or more Bioanalyzer or TapeStation runs. The members are:

* `data` - a tall data frame of the run data, specifically:
	* `time` (Bioanalyzer) - time when this data point was measured
	* `aligned.time` (Bioanalyzer) - measurement time aligned between the expected times of the marker peaks
	* `distance` (TapeStation) - migration distance of the measurement from the top of the gel area
	* `relative.distance` (TapeStation) - migration distance normalized relative to the marker peaks
	* `fluorescence` - fluorescence reading at this point
	* `length` - estimated molecule length at this point
	* `molarity` - estimated molarity of the area under the curve between this point and the previous one
	* `peak` - which peak, if any, this measurement is inside
* `samples` - a data frame of metadata for each sample (also annotated to `data`, `peaks`, and `regions` with the same factor levels), specifically:
	* `batch` - the batch (instrument run) of the sample, from the file name
	* `well.number` - the well number in which the sample was loaded
	* `sample.name` - the name of the sample
	* `reagent.id` (TapeStation) - the name of the ScreenTape used for this sample
	* `is.ladder` - a Boolean indicating whether this sample is a ladder of standards for calibration
	* `sample.observations` - notes about this sample supplied by the user or the Agilent software
* `wells.by.ladder` - a list of which wells in each batch correspond to a given ladder (the TapeStation may run a separate ladder on each tape)
* `peaks` - a data frame of peaks reported by the Agilent software, annotated with their lower and upper boundaries in various scales
* `regions` - a data frame of regions of interest reported by the TapeStation software, annotated with their lower and upper boundaries in varous scales **(currently does not read the smear analysis from Bioanalyzer data)**
* `mobility functions` - a list of estimated functions, one per ladder used for calibration, to convert migration speed measurements (aligned time or relative distance) into estimated molecule lengths
* `mass.coefficients` - a list of coefficients, one per sample (Bioanalyzer) or one per ladder (TapeStation), to convert area under the curve to estimated mass

The calculated measurements (`length` and `molarity`) will be NA if they are outside the range of interpolation from the standard curve. By default the standard curve is fit from the ladder according to a spline function. This is similar to Agilent's approach for DNA samples, which you can approximate by setting `fit = "interpolation"`, but the spline produces smoother molarity estimates. You can also set `fit = "regression"` to fit a log-linear regression, but that does not appear to fit these data very well.


# Graphing the data

## `qplot.electrophoresis`
Given an `electrophoresis` object generated by `read.bioanalyzer` and `read.tapestation`, this function generates a `ggplot` object, which can be printed directly or customized by adding more features. Each sample is shown in a separate facet, except the ladder is not shown by default (`include.ladder`). `x` and `y` can be set to graph different variables, and `geom` can be any applicable ggplot2 geom function. By default, the vertical axis is scaled independently for each facet; the `scales` argument is passed to `facet_wrap`. You can override these settings by adding another `facet_wrap` etc.

Arguments:

* `x` - which variable to show on the x-axis (e.g. `"length"`, `"time"`, `"distance"`)
* `y` - which variable to show on the y-axis (e.g. `"fluorescence"`, `"molarity"`)
* `log` - which axis or axes to log-scale; recognizes `"x"`, `"y"`, `"xy"`, or set to `NA` for no log-scaling; defaults to `"x"` if the x-value is length or `NA` otherwise
* `facet` - if true, plot each sample in a different facet; if false; overlay them all in one graph and color-code the data
* `scales` - scaling rule for facets, passed to `facet_wrap()`
* `geom` - geom to use for graphs, either `"line"` for simple lines or `"area"` to fill the area under the lines
* `include.ladder` - if false, show only the regular samples and not the ladder
* `between.markers` - if true, show only the data between the reported ladder peaks
* `peak.fill` - color to fill under the curves indicating where peaks are, or set to `NA` to hide peaks
* `region.alpha` - alpha transparency for shading indicating where regions are, or set to `NA` to hide regions
* `area.alpha` - alpha transparency for the filled areas if `geom = "area"`

# Checking the accuracy of the calculations

## qc.stdcrv
Check the fit of the mobility model (molecule length vs. migration speed) by plotting the standard curve.

Arguments:

* `n.simulate` - number of points to simulate with the mobility function
* `line.color` - color to draw the standard curve

## qc.mobility
Check the fit of the mobility model by comparing the estimated molecule lengths of reported peaks with the lengths reported by the Agilent software.

Arguments:

* `log` - if true, log-scale both axes (typically fits the data better)


## qc.molarity
Check the accuracy of the molarity estimates by comparing the estimated molarities of reported peaks with the molarities reported by the Agilent software.

Arguments:

* `log` - if true, log-scale both axes (typically fits the data better)


# Manipulating and analyzing the data

## Combining data: `rbind.electrophoresis`
Combines multiples `electrophoresis` objects into one. Different runs will still be distinguished by `batch`. `data$peak` is renumbered so it corresponds to the new set of `peaks`.

## Summing data

### `integrate.rawdata`
Computes the sum of some variable under the electrophoresis curve between specified boundaries, or within an annotated peak, or in the intersection of both. To calculate the results separately by sample you must provide this function a subset of the data, or see `integrate.custom` below.

Arguments:

* `lower.bound`, `upper.bound` - boundaries of your target region in the desired unit
* `bound.unit` - unit of the boundaries, e.g. `"length"`, `"distance"`
* `peak` - which peak's boundaries to use
* `sum.unit` - what value to sum, e.g. `"molarity"`, `"fluorescence"`

### `integrate.peaks`
Computes the sum of some variable (specified by `sum.unit`) in each peak in the peak table.

### `integrate.regions`
Computes the sum of some variable (specified by `sum.unit`) in each region in the region table.

### `integrate.custom`
Computes the sum of some variable (specified by `sum.unit` and additional arguments are passed to `integrate.rawdata`) within a single set of boundaries, individually for each sample.

## Comparing regions

### `region.ratio`
Computes the ratio of sums within two or more regions in each sample. Regions are specified as a list of boundary pairs, e.g. `list(c(100, 200), c(200, 700))`. Passes other arguments to `integrate.custom`.

### `illumina.library.ratio`
Shortcut for Illumina sequencing libraries: Computes the molar ratio of fragments between 200-700 bases (good inserts) to fragments between 100-200 bases (adapter dimers) for each sample.


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
1. Save the gel image as a file in PNG format, preferably with the same name as the XML file (but the `.png` extension).

### Why does `read.tapestation` need a gel image?
The TapeStation software cannot export the raw fluorescence data in any other format; the XML only contains metadata (sample ID, peak list, etc.). The data would be easily accessible inside the `.D1000`, `.HSD1000`, etc. file formats except they are encrypted. I have asked Agilent about this and they stated that they do not want the data to be accessible to users.

