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

