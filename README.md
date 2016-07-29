# pummeler

This is a set of utilities for analyzing the American Community Survey's Public Use Microdata Sample files (ACS PUMS), mostly following

> Flaxman, Wang, and Smola. Who Supported Obama in 2012? Ecological Inference through Distribution Regression. KDD 2015.
> ([official](http://dx.doi.org/10.1145/2783258.2783300), [from author](http://sethrf.com/files/ecological.pdf))

The package currently only supports the 2006-10 5-year estimates, but more will be added.


## Usage

### Installation

`pip install pummeler` should work soon, once it's done enough to have an actual release; it will then be available via `import pummler` and install a `pummel` script.

If you prefer, you can also check out the source directory, which should work as long as you put the `pummel` directory on your `sys.path` (or start Python from the root of the checkout). In that case you should use the `pummel` script at the top level of the checkout.


### Getting the census data

First, download the data from the Census site. You probably want the "csv_pus.zip" file, [here](http://www2.census.gov/programs-surveys/acs/data/pums/2010/5-Year/csv_pus.zip) for the 2006-10 data (2.1 GB).


### Picking regions of analysis

Election results are generally reported by counties; PUMS data are in census blockgroups, which are related but not the same. This module ships with regions that merge all overlapping blockgroups / counties, found with [the MABLE/Geocorr tool](http://mcdc2.missouri.edu/websas/geocorr12.html), in Pandas dataframes stored in `pummeler/data/regions.h5`.

Regions are named like `AL_00_01`, which means Alabama's region number 01 in the 2000 geography. If you use 2010 geographies (i.e. you're using ACS data of vintage after 2011 – not yet supported, but it will be), those regions are named `AL_10_01`.

**Note:** Alaska electoral districts are weird. For now, I just lumped all of Alaska into one region.

This was done in the Jupyter notebook [`notebooks/get regions.ipynb`](notebooks/get regions.ipynb). Centroids are calculated in [`notebooks/region centroids.ipynb`](notebooks/region centroids.ipynb), using a shapefile for counties [from here](https://geonet.esri.com/thread/24614).


### Preprocessing

First, we need to sort the features by region and collect statistics about them so we can do the featurization.

Run `pummel sort -z csv_pus.zip SORT_DIR`. (A few extra options are shown if you pass `--help`.) This will:

- Make a bunch of files in `SORT_DIR` like `feats_AL_00_01.h5`, which contain basically the original features (except with the `ADJINC` adjustment applied to fields that need it to account for inflation) grouped by region. These are stored in HDF5 format with pandas, because it's much faster and takes less disk space than CSVs.

- Makes a file `SORT_DIR/stats.h5` containing means and standard deviations of the real-valued features, counts of the different values for the categorical features, and a random sample of all the features.

This will take a while (~15 minutes on my fast-ish laptop) and produce about 4GB of temp data (for the 2006-10 files). Luckily you should only need to do it once per ACS file.


### Featurization

Run `pummel featurize SORT_DIR`. (Again, you have a couple of options shown by `--help`.) This will get both linear embeddings (i.e. means) and random Fourier feature embeddings for each region, saving the output in `SORT_DIR/embeddings.npz`.

On my laptop (with a quad-core Haswell i7), this takes about an hour; it scales reasonably well with more cores. Make sure you're using a numpy linked to a fast multithreaded BLAS (like MKL or OpenBLAS; the easiest way to do this is to use the [Anaconda](https://www.continuum.io/downloads) Python distribution, which includes MKL by default); otherwise, this step will be much slower.

If it's using too much memory, decrease `--chunksize`.

The original paper used Fastfood transforms instead of the default random Fourier features used here, which with a good implementation will be faster. I'm not currently aware of a high-quality, easily-available Python-friendly implementation. A GPU implementation of regular random Fourier features would also help.

`SORT_DIR/embeddings.npz`, which you can load with `np.load`, will then have:

 - `emb_lin`: the `n_regions x n_feats` array of feature means.
 - `emb_rff`: the `n_regions x (2 * n_freq)` array of random Fourier feature embeddings.
 - `region_names`: the names corresponding to the first axis of the embeddings.
 - `feature_names`: the names for each used feature.
 - `freqs`: the `n_feats x n_freq` array of random frequencies for the random Fourier features.
 - `bandwidths`: the bandwidth used for selecting the `freqs`.


### Getting the election data

This package includes results derived from [`huffpostdata/election-2012-results`](https://github.com/huffpostdata/election-2012-results), in `pummeler/data/2012-by-region.csv.gz`. That data was created in [`notebooks/election data by region.ipynb`](notebooks/election data by region.ipynb).

There doesn't seem to be a good publicly-available county-level election results resource for years prior to 2012. If you get some, follow that notebook to get results in a similar format. (Your might have an institutional subscription to CQ Press's election data, for example. That source, though, doesn't use FIPS codes, so it'll be a little more annoying to line up; I might do that at some point.)


### Analysis

**TODO**:

 - Make a notebook replicating the analysis from the paper, with helpers in the package
