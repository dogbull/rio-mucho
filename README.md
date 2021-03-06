# rio-mucho

Windowed parallel processing wrapper for rasterio

 [![PyPI](https://img.shields.io/pypi/v/rio-mucho.svg?maxAge=2592000?style=plastic)]() [![Build Status](https://travis-ci.org/mapbox/rio-mucho.svg?branch=master)](https://travis-ci.org/mapbox/rio-mucho) [![Coverage Status](https://coveralls.io/repos/mapbox/rio-mucho/badge.svg?branch=master&service=github)](https://coveralls.io/github/mapbox/rio-mucho?branch=master)

## Install

From pypi:

`pip install rio-mucho`

From github:

`pip install pip install git+ssh://git@github.com/mapbox/rio-mucho.git@<branch>`

Development:

```
git clone git@github.com:mapbox/rio-mucho.git
cd rio-mucho
pip install -e .
```

## Usage

```python
with riomucho.RioMucho([{inputs}], {output}, {run function},
    windows={windows},
    global_args={global arguments}, 
    options={options to write}) as rios:

    rios.run({processes})
```

### Arguments

#### `inputs`

An list of file paths to open and read.

#### `output`

What file to write to.

#### `run_function`

A function to be applied to each window chunk. This should have input arguments of:

1. A data input, which can be one of:
 - A list of numpy arrays of shape (x,y,z), one for each file as specified in input file list `mode="simple_read" [default]`
 - A numpy array of shape ({_n_ input files x _n_ band count}, {window rows}, {window cols}) `mode=array_read"`
 - A list of open sources for reading `mode="manual_read"`
2. A `rasterio` window tuple
3. A `rasterio` window index (`ij`)
4. A global arguments object that you can use to pass in global arguments

This should return:

1. An output array of ({depth|count}, {window rows}, {window cols}) shape, and of the correct data type for writing

```python
def basic_run({data}, {window}, {ij}, {global args}):
    ## do something
    return {out}
```

### Keyword arguments

#### `windows={windows}`

A list of `rasterio` (window, ij) tuples to operate on. `[Default = src[0].block_windows()]`

#### `global_args={global arguments}`

Since this is working in parallel, any other objects / values that you want to be accessible in the `run_function`. `[Default = {}]`

```python
global_args = {
    'divide_value': 2
}
```

#### `options={keyword args}`

The options to pass to the writing output. `[Default = srcs[0].meta]`

## Example

```python
import riomucho, rasterio, numpy

def basic_run(data, window, ij, g_args):
    ## do something
    out = np.array(
        [d /= global_args['divide'] for d in data]
        )
    return out

# get windows from an input
with rasterio.open('/tmp/test_1.tif') as src:
    ## grabbing the windows as an example. Default behavior is identical.
    windows = [[window, ij] for ij, window in src.block_windows()]
    options = src.meta
    # since we are only writing to 2 bands
    options.update(count=2)

global_args = {
    'divide': 2
}

processes = 4

# run it
with riomucho.RioMucho(['input1.tif','input2.tif'], 'output.tif', basic_run,
    windows=windows,
    global_args=global_args, 
    options=options) as rm:

    rm.run(processes)

```

## Utility functions

### `riomucho.utils.array_stack([array, array, array,...])`

Given a list of ({depth}, {rows}, {cols}) numpy arrays, stack into a single ({list length * each image depth}, {rows}, {cols}) array. This is useful for handling variation between `rgb` inputs of a single file, or separate files for each.

#### One RGB file

```python
files = ['rgb.tif']
open_files = [rasterio.open(f) for f in files]
rgb =riomucho.utils.array_stack([src.read() for src in open_files])
```

#### Separate RGB files

```python
files = ['r.tif', 'g.tif', 'b.tif']
open_files = [rasterio.open(f) for f in files]
rgb = riomucho.utils.array_stack([src.read() for src in open_files])
```
