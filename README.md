# CPT Cloud Data Guide

This is a rough guide to one option we have for data sharing within the CPT.

## Background Information

### Open Storage Network

[Open Storage Network](https://www.openstoragenetwork.org/) (OSN) is a distributed data service to support active data sharing and transfer between academic institutions, leveraging existing NSF-funded resources.
It is funded by NSF and the Schmidt Futures Foundation.

Via Pangeo, our CPT has access to an a 10 TB allocation on OSN.
Our allocation is on the OSN pod at [NCSA](http://www.ncsa.illinois.edu/) in Illinois.
This pod has a very high-bandwidth internet connection and should provide good transfer rates to all CPT collaborating institutions.

### Cloud Object Storage

OSN is a cloud object storage service.
Object storage is probably unfamiliar to most CPT members.
It is a technology that allows data to be read / written via HTTP calls.
For a basic primer on object storage, [this document](http://gallery.pangeo.io/repos/earthcube2020/ec20_abernathey_etal/cloud_storage.html) may be a useful reference.

OSN is configured to be compatible with the most common object storage API: Amazon S3.
Therefore, the [Amazon S3 Documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/Introduction.html) is also a useful reference.
To configure your computer to talk to OSN, you need a few key details

| parameter | value |
|-----------|-------|
| `endpoint_url` | `https://ncsa.osn.xsede.org` |
| `bucket` | `Pangeo` |
| `access_key_id` | email Ryan |
| `secret_access_key` | email Ryan |

There are two sets of credentials: full access (read-write) and read-only.
We will use the convention that all CPT data be stored under the prefix `Pangeo/ocean-eddy-cpt`, where `Pangeo` is the bucket name.

In contrast to more user-friendly cloud storage services like Google Drive, Dropbox, etc., there is no pretty website


### Zarr Format

In order to work effectively with object storage, it is desirable to use a "cloud-optimized" data format.
This means transforming our NetCDF data into [Zarr](https://zarr.readthedocs.io/en/latest/).
([This AGU Talk](https://speakerdeck.com/rabernat/cloud-native-data-formats-for-big-scientific-data) provides a simple introduction to Zarr.)
The [Pangeo Guide to Preparing Cloud-Optimized Data](http://pangeo.io/data.html#guide-to-preparing-cloud-optimized-data) is a useful overview of the process of creating Zarr stores from NetCDF data and putting it in the cloud, although some aspects of that guide have to be changed to work with OSN.


## Guide for CPT Usage of OSN

Here we attempt to provide a compact practical guide for CPT users interacting with OSN.
Please feel free to suggest changes / additions where anything is not clear.

### Step 1: Create the Zarr Data

This step is run wherever the original data live (e.g. on a GFDL supercomputer.)
For this part, you can follow the [Pangeo Guide](http://pangeo.io/data.html#guide-to-preparing-cloud-optimized-data) steps 1 and 2 exactly.
We recommend using the python package [Xarray](http://xarray.pydata.org/en/latest) for this step.
(The [Xarray Zarr Documenation](http://xarray.pydata.org/en/latest/io.html#zarr) may also be helpful here.)
The most important parameter you will have to consider is the chunk size / shape on the data variables.
We want to aim for chunks of roughly 100 MB in size.
It is most common to apply chunking along the time dimension.
Chunking in space can also be used for very high-resolution datasets.
When this step is complete, you will have one or more Zarr stores on disk.
_Note that Zarr also makes an excellent on-disk analysis-ready format. Many users prefer to transform all their NetCDF data to Zarr, even if not using the cloud._

### Step 2: Upload to OSN

For this step, we will use the [AWS s3 command line utility](https://docs.aws.amazon.com/cli/latest/reference/s3/).
First, you will have to [install the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).
Then you need to install this pip package:
```
$ pip install awscli-plugin-endpoint
```
Then enable the plugin and create a configuration profile for OSN.
You do this by editing the file `$HOME/.aws/config` and adding a section as follows:

```
[plugins]
endpoint = awscli_plugin_endpoint

[profile osn]
aws_access_key_id=<email Ryan for secrets>
aws_secret_access_key=<email Ryan for secrets>
s3 =
    endpoint_url = https://ncsa.osn.xsede.org
s3api =
    endpoint_url = https://ncsa.osn.xsede.org
```

You can now upload your data to OSN with a command line argument like the following
```
$ aws s3 --profile osn cp --recursive /local/path s3://Pangeo/ocean-eddy-cpt/<dataset name>
```
Where `<dataset name>` is a unique identifier for your dataset.
It can contain `/` characters in order to organize the data in to sub-directories.
We have not yet worked out how to organize the data within the budget, so for now just use your best judgement.

**Note:** there is only one account for the entire bucket.
That means that, if you have the read-write credentials, you can potentially delete / overwrite data created by other people.
Please be very careful!


### Step 3: Verify your upload

First we check that the files are there using the CLI:
```
$ aws s3 --profile osn ls --recursive /local/path s3://Pangeo/ocean-eddy-cpt/
```

Next we check that our uploaded data is readable from python.
For this we need to have the following python packages installed.
- [s3fs](https://github.com/dask/s3fs)
- [Zarr](https://zarr.readthedocs.io/en/latest/)
- [Xarray](http://xarray.pydata.org/)

Here is some code to open a dataset from OSN.

```python
import s3fs
import zarr
import xarray as xr

access_key_id = # email Ryan
secret_access_key = # email Ryan
endpoint_url = 'https://ncsa.osn.xsede.org'
fs = s3fs.S3FileSystem(
    key=access_key_id,
    secret=secret_access_key,
    client_kwargs={'endpoint_url': endpoint_url}
)

mapper = fs.get_mapper('Pangeo/ocean-eddy-cpt/<dataset name>')

# open the data from Zarr (more low-level)
zarr_group = zarr.open_consolidated(mapper)

# open the data from Xarray (recommend)
ds = xr.open_zarr(mapper, consolidated=True)
```

Examine `ds` and ensure it looks right.
Note that not all the data is actually downloaded when you open a dataset.
It is downloaded "lazily", i.e. only when needed for computation or plotting, or when explicitly requested via `.load()`.

### Step 4: Create a catalog entry

Because there is no easy-to-use browser for OSN, we need to maintain our own catalog of data.
For this we will use [intake](https://intake.readthedocs.io/en/latest/) and the [intake-xarray](https://intake-xarray.readthedocs.io/en/latest/) plugin.
Intake is a lightweight package for finding, investigating, loading and disseminating data.
Other CPT members will query and load data using this intake catalog.

Our catalog is stored in this GitHub repo:
<https://github.com/ocean-eddy-cpt/cpt-data/blob/master/catalog.yaml>
It is currently empty.
We need to create a new entry for each dataset.
Some examples can be found here:
https://github.com/pangeo-data/pangeo-datastore/blob/master/intake-catalogs/ocean/MEOM-NEMO.yaml

**Problem:** We don't have an example of an intake catalog pointing to an s3 endpoint other than the standard AWS one.
So we may have to do some trial-and-error to get this to work.
Once we have one working example, it will be easy to duplicate.
