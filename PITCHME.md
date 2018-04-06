---

### PDAL Algorithm Development Deep Dive
#### FOSS4G North America 2018, 16 May 2018
Bradley J Chambers, Radiant Solutions

---

### Overview

In this talk, we will outline the steps to develop a point cloud filtering
algorithm using PDAL's Python extension. We will show how PDAL can be installed
via Conda, and in a Jupyter notebook will work through the algorithm
development process, showing how the finished algorithm can be distributed and
executed using the PDAL command line interface.

* Conda
* Jupyter

---

### Conda

* hexer
* laz-perf
* laszip
* nitro

---

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

---

### Install

```bash
conda env create -f pdalenv.yml
source activate pdalenv
```
@[1](Create environment according to YAML)
@[2](Activate the environment)

---

### Start

```bash
jupyter notebook
```

---

### Instead of...

```bash
docker pull chambbj/pdal-notebook
docker run -it --init --rm -p 8888:8888 -v $(pwd)/notebooks:/notebooks chambbj/pdal-notebook
```
