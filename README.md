# bioanalyzeR

Simple R functions for importing and processing electrophoresis data from an Agilent 2100 Bioanalyzer.

## `read.bioanalyzer`
Reads data from a Bioanalyzer XML file (see below for how to export that) into a molten data frame containing all useful information. Each row is a single measurement at a certain time so there are thousands of rows. The fields are:

* `index` - the loading order of the sample
* `name` - the name of the sample (as a factor, whose levels are in the order observed in the input data)
* `time` - the time this data point was measured
* `fluorescence` - the fluorescence reading at this time
* `length` - the estimated length, in nucleotides, of the molecules being read at this time
* `molarity` - the estimate concentration, in picomolar (pM), of the molecules being read at this time

`length` and `molarity` will be NA if they are outside the range of interpolation from the standard curve. By default the standard curve is fit from the ladder according to linear interpolation between the bands, which seems to match Agilent's approach. Optionally, set `fit = "spline"` to fit the curve more smoothly according to a parametric model.

## `plot.bioanalyzer`
Generates a nice `ggplot` object from a data frame of the type generated by `read.bioanalyzer`.

Each sample is shown in a separate facet, except the ladder is not shown by default (`include.ladder`). `x` and `y` can be set to graph different variables, and `geom` can be any applicable ggplot2 geom function. By default, the vertical axis is scaled independently for each facet; the `scales` argument is passed to `facet_wrap`.

## How to export an XML file
In the 2100 Expert software, open your data file (`.xad`) in the "Data" context. Select "File->Export..." from the top menu. Check the "Export to XML" box and no others. Click "Export" and then save the file wherever you like. This will create an XML file that can be read by the `plot.bioanalyzer` function.

