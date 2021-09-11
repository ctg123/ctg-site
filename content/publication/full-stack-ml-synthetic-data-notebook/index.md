---
title: Notebook mapping real-time ISS location
subtitle: This notebook is built in Jupyter notebook in Jupyterlab and conda
  environment. The example will walkthrough interacting with a real time API
  tracking the location of the International Space Station. You can collect and
  save your own data in this example.
publication_types:
  - "0"
  - "4"
authors:
  - ctgadget
draft: false
featured: false
image:
  filename: featured
  focal_point: Smart
  preview_only: false
date: 2021-09-05T17:25:37.275Z
---
# Conda environment with environment.yml

[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/ctg123/iss-api-map/HEAD)

A Binder-compatible repo with an `environment.yml` file.

Access this Binder by clicking the blue badge above or at the following URL:

https://mybinder.org/v2/gh/ctg123/iss-api-map/HEAD

## Notes

The `environment.yml` file should list all Python libraries on which your notebooks
depend, specified as though they were created using the following `conda` commands:

```
conda activate example-environment
conda env export --from-history -f environment.yml
```

Note that the only libraries available to you will be the ones specified in
the `environment.yml`, so be sure to include everything that you need! 

Also, note that if you skip the `--from-history`, conda may include OS-specific
packages in `environment.yml`, which you would have to manually prune from
`environment.yml`.