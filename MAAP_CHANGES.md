# Changes from upstream

Upstream: https://github.com/ECOSTRESS-Collection-3/ECOv003-L3T-L4T-JET, tag `v1.18.0` (branch `main`) at commit 3c0ad65c0d7bc8af6a85145951ed9ab134814775 (2026-10-12), by Gregory H. Halverson, Kerry Cawse-Nicholson, Madeleine Pascolini-Campbell, Evan Davis and Claire Villanueva-Weeks (NASA Jet Propulsion Laboratory), Apache License 2.0.

This copy starts from upstream's file tree at 3c0ad65 without its git history. It is unchanged except for the items below. The science code, the run-config template, the tests and the LICENSE are untouched, so the products do not change. `.github/` was left out: it holds the upstream CI workflow and the workflow that publishes releases to PyPI, which belong to the upstream repository.

The package version is set statically in `pyproject.toml` (`version = "1.18.0"`, read at run time through `importlib.metadata` and written to each product's `PGEVersion`); it was not changed, so products from this copy still say 1.18.0. This copy is tagged `v1.18.0.post1`.

## 2026-10-25: the run-config generator reads Collection 3 inputs

`generate_L3T_L4T_JET_runconfig` parsed the L2T LSTE granule name with the Collection 2 class (`ECOv002_granules.L2TLSTE`). Collection 2 names carry a build ID (`..._0713_01`) and Collection 3 names do not (`..._20261002T173304_02`), so for every Collection 3 input it read the orbit as `LSTE` and stopped with `ValueError: invalid literal for int() with base 10: 'LSTE'`. It only worked with the Collection 2 inputs that `examples/example_L3T_L4T_JET_run_with_download.py` downloads.

The generator now picks the class from the collection in the file name, as `read_ECOv003_configuration.py` and `read_ECOv003_inputs.py` already do: `ECOv003` names use `ECOv003_granules.L2TLSTE`, `ECOv002` names keep the Collection 2 class, and other names raise a clear error. Only the generator changed; the PGE and its products do not.

## 2026-10-25: the GDAL command-line tools are declared

The PGE runs GDAL's command-line programs: `gedi-canopy-height` calls `gdalinfo` and `gdalbuildvrt` on the canopy height mosaic, and `rasters` writes every output layer as a COG with `gdal_translate` (through `os.system`, without checking the return code). pip does not install these programs, and neither `make environment` nor the Dockerfile added them. They cannot be declared in `pyproject.toml`, so `make environment` now creates the conda environment with conda-forge `gdal`, and the Dockerfile installs `gdal-bin`. Results do not change.

## 2026-10-25: the Dockerfile uses Python 3.13

The Dockerfile started from `python:3.10.16-bookworm`, but `pyproject.toml` requires Python 3.11 or newer, so `pip install -e .[dev]` in the image stops on the Python version. It now starts from `python:3.13.2-bookworm`, the image its own comment names and the Python version `make environment` uses. Results do not change. (Not built here.)

## 2026-10-25: README

A short Installation section: `pip install .`, the GDAL tools, and how to run the PGE from a generated run-config.

## Notes for running it

- Inputs: one ECOv003 L2T LSTE granule and the matching ECOv003 L2T STARS granule from LP DAAC (Earthdata login), as unzipped folders of COGs.
- Static data: left alone, the dependencies download whole-globe files for the GEDI canopy height, SoilGrids field capacity and wilting point, and the MODIS clumping index (about 40 GB). Placing small windows of them under the run-config's static folder, with the file names the code looks for, avoids this.
- GEOS-5 FP: the PGE downloads the files it needs from NCCS itself (public, about 900 MB per scene).
- The MODIS clumping index download in `MODISCI` passes the Earthdata password on a `wget` command line. It only runs when `MODISCI_download/global_clumping_index.tif` is missing from the static folder.

## Check

On 2026-10-25, in a fresh conda environment made the way `make environment` now makes it (conda-forge Python 3.13.17 and GDAL 3.13.3), from a clone of this repository:

```sh
pip install ".[dev]"
pytest -q                     # 20 passed (the tests import the package and its dependencies)
python -c "from ECOv003_L3T_L4T_JET import generate_L3T_L4T_JET_runconfig as g; print(g(
    L2T_LSTE_filename='work/ECOv003_L2T_LSTE_46739_001_11SPS_20261002T173304_02',
    L2T_STARS_filename='work/ECOv003_L2T_STARS_11SPS_20261002_02',
    working_directory='work', static_directory='static'))"
ECOv003-L3T-L4T-JET work/runconfig/ECOv003_L3T_JET_46739_001_11SPS_20261002T173304_0700_01.xml
```

The scene is orbit 46739, scene 001, tile 11SPS (2026-10-02, Imperial Valley and Salton Sea): the V003 L2T LSTE and STARS granules (113 MB) from LP DAAC, and small windows of the static files. The generator wrote the run-config directly from the Collection 3 names. The PGE exited with code 0 in 9 min 39 s on 4 cores (most of it the GEOS-5 FP download from NCCS), with 5.6 GiB peak memory, and wrote the four products (L3T JET, L3T ETAUX, L4T ESI, L4T WUE; 170 MB of zips).

Against the official LP DAAC V003 granules for the same scene, all 29 layers have the same grid and the same no-data mask. Ta, RH, SM and the water and cloud masks are bit-identical; ETdaily has r = 0.999997 and a mean absolute difference of 0.0011 mm/day. The small differences come from the shortwave radiation (Rg, 0.3 W/m2 on average); the official granules were made with PGE 1.13.0. Every per-layer statistic is the same as from the unchanged 1.18.0 code with the same inputs.
