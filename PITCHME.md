---

### PDAL Algorithm<br>Development Deep Dive
#### FOSS4G North America 2018, 16 May 2018
Bradley J Chambers, Radiant Solutions

---

### Overview

* PDAL Refresher
* Conda
* Jupyter
* Differential Morphological Profiles Notebook
* DMP Filter

---

### PDAL Refresher

* Pipeline
* Translate
* Python Filter

+++

### PDAL Pipeline

+++

### PDAL Translate

+++

### PDAL Python Filter

---

### Conda

* Why Conda?

+++

### New Conda Packages

* hexer
* laz-perf
* laszip
* nitro
* pdal

+++

### Environment

Create environment file `pdalenv.yml` containing the following

```yaml
name: pdalenv
channels:
  - defaults
  - conda-forge
dependencies:
  - python=2
  - numpy
  - scipy
  - pandas
  - matplotlib
  - jupyter
  - scikit-learn
  - scikit-image
  # The following packages should not be needed after pdal is available (they will be installed as needed).
  - hexer=1.4.0
  - laz-perf=1.1.0
  - laszip=3.2.2
  - nitro=2.7.dev2
```
@[4](All of our packages are available in the conda-forge channel)
@[6-13](This list will vary depending on your needs)
@[14-18](This will collapse to `- pdal` once the package is available)

+++

### Install

```bash
conda env create -f pdalenv.yml
source activate pdalenv
```

---

### Jupyter

+++

### Start

```bash
jupyter notebook
```

---

### Differential Morphological Profiles Notebook

---

### DMP Filter

```json
{
    "pipeline":[
        "/data/bare_earth_eval/isprs/converted/laz/samp11-utm.laz",
        {
            "type":"filters.python",
            "module":"anything",
            "function":"filter",
            "source":"from scipy import ndimage, signal, spatial
from scipy.ndimage import morphology

import numpy as np
import pandas as pd

S = 30
k = 0.2
n = 0.3
b = 0.2

def idw(data):
    # Find indices of the ground returns, i.e., anything that is not a nan, and create a KD-tree.
    # We will search this tree when looking for nearest neighbors to perform the interpolation.
    valid = np.argwhere(~np.isnan(data))
    tree = spatial.cKDTree(valid)

    # Now find indices of the non-ground returns, as indicated by nan values. We will interpolate
    # at these locations.
    nans = np.argwhere(np.isnan(data))    
    for row in nans:
        d, idx = tree.query(row, k=12)
        d = np.power(d, -2)
        v = data[valid[idx, 0], valid[idx, 1]]
        data[row[0], row[1]] = np.inner(v, d)/np.sum(d)

    return data

def filter(ins, outs):
    # Use the X, Y, and Z dimension data to classify points as ground or
    # non-ground.
    X = ins['X']
    Y = ins['Y']
    Z = ins['Z']

    # Begin by getting the Classification dimension and setting all values to 1
    # for unclassified.
    cls = ins['Classification']
    cls.fill(1)

    df3D = pd.DataFrame({'X': X, 'Y': Y, 'Z': Z})
    density = len(X) / (Y.ptp() * X.ptp())
    hres = 1. / density

    # Initilize the process by selecting points with lowest elevation at the target resolution.
    # Replace those points that are more than 1.0m below the estimated surface (as determined by
    # morphological opening/closing.

    # Next, we compute bin edges ranging between XY min and max, spaced at a fixed resolution.
    xi = np.ogrid[X.min():X.max():hres]
    yi = np.ogrid[Y.min():Y.max():hres]

    # We can effectively group all incoming points by their XY bin.
    bins = df3D.groupby([np.digitize(X, xi), np.digitize(Y, yi)])

    # Record coords of point with minimum Z for each XY bin.
    cz = np.empty((yi.size, xi.size))
    cz.fill(np.nan)
    czfoo = bins.Z.min()
    for name, val in czfoo.iteritems():
        cz[name[1]-1, name[0]-1] = val

    # For any remaining undefined cells, estimate via IDW
    cz = idw(cz)

    # Open the control elevations using a diamond structuring element with radius 11.
    struct = ndimage.iterate_structure(ndimage.generate_binary_structure(2, 1), 11).astype(int)
    opened = morphology.grey_opening(cz, structure=struct)

    # Close the control elevations using a diamond structuring element with radius 9.
    struct = ndimage.iterate_structure(ndimage.generate_binary_structure(2, 1), 9).astype(int)
    closed = morphology.grey_closing(opened, structure=struct)

    # Record the locations where there appear to be low outliers.
    lowx, lowy = np.where((closed - cz) >= 1.0)

    # Replace low outliers with the morphologically altered surface.
    cz[lowx, lowy] = closed[lowx, lowy]
    stdev = 14
    G = np.outer(signal.gaussian(113,stdev), signal.gaussian(113,stdev))
    low = signal.fftconvolve(np.pad(cz,2*stdev,'edge'), G, mode='same')[2*stdev:-2*stdev,2*stdev:-2*stdev]/1000.
    high = cz - low

    # Multiscale decomposition achieved by using a granulometry of morphological openings.
    erosions = []
    granulometry = []
    erosions.append(morphology.grey_erosion(high, size=3))
    for scale in xrange(1, S):
        erosions.append(morphology.grey_erosion(erosions[scale-1], size=3))
    for scale in xrange(1, S+1):
        granulometry.append(morphology.grey_dilation(erosions[scale-1], size=2*scale+1))

    # A top-hat scale-space, known as DMPs
    out = []
    for i in xrange(1, len(granulometry)):
        out.append(granulometry[i-1]-granulometry[i])

    # gprime registers the largest responses from the DMP
    # gstar describes the smallest scales at which the largest response is registered
    # gplus is the sum of all responses registered before and including the largest response
    xs, ys = out[0].shape
    gprime = np.maximum.reduce(out)
    gstar = np.zeros((xs,ys))
    gplus = np.zeros((xs,ys))
    for ii in xrange(0,xs):
        for jj in xrange(0,ys):
            for kk in xrange(0,len(out)):
                if out[kk][ii,jj] < gprime[ii,jj]:
                    gplus[ii,jj] += out[kk][ii,jj]
                if out[kk][ii,jj] == gprime[ii,jj]:
                    gplus[ii,jj] += out[kk][ii,jj]
                    gstar[ii,jj] = kk
                    break

    T = k * gstar + n
    Sg = gprime < T
    F=cz.copy()
    F[np.where(Sg==0)] = np.nan
    G=idw(F)
    struct = ndimage.iterate_structure(ndimage.generate_binary_structure(2, 1), 1).astype(int)
    gradDTM = morphology.grey_dilation(G, structure=struct)
    xbins = np.digitize(df3D.X, xi)
    ybins = np.digitize(df3D.Y, yi)
    ground = np.where(df3D.Z < gradDTM[ybins-1, xbins-1]+b)

    cls[ground] = 2

    # Set the output dimension and return.
    outs['Classification'] = cls
    return True"
        },
        "/data/bare_earth_eval/isprs/converted/laz/samp11-utm-dmp.las"
    ]
}
```
@[8-17](Imports and variable initialization)
@[19-34](Inverse Distance Weighting)
