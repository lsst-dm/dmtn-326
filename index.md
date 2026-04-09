# Bulk Cutout Service Implementation Options

```{abstract}
The bulk cutout service is a deliverable required to support Data Preview 2.
This document explains our implementation options regarding how to perform the cutouts at scale and in what form the resulting cutouts should be returned.
It distinguishes bulk cutout of objects or sources specified in a catalog, from bulk cutout of light curves for a single object.
```

## Introduction

The existing one-at-a-time cutout service {cite:p}`DMTN-208` was deployed for Data Preview 1 {cite:p}`RTN-095` and uses the IVOA SODA (Server-side Operations for Data Access) standard {cite:p}`2017ivoa.spec.0517B`.
This service is generally called via DataLinker with results from SIA or ObsTAP queries, to perform cutouts from specific images at a single location.

The community are very keen on being able to retrieve millions of small image cutouts from LSST images and we have had explicit inquiries from the LINCC team over the years asking us how they would obtain billions of cutouts.
In {cite:t}`DMTN-207` an initial discussion was begun concerning how the EPO team would generate their Zooniverse cutouts at scale but that discussion faltered, due to other priorities, and the EPO team started to try to use the Butler directly in a loop (which has serious performance issues).
Additionally we have seen demand from Data Preview 1 {cite:p}`RTN-095` users where people have been calling into the existing one-at-a-time cutout service over many days to obtain cutouts for ML training data.
Formally we have agreed to release some form of bulk cutout service in time for Data Preview 2.

## Requirements

There is one formal requirement listed in LSE-61 {cite:p}`LSE-61` relating to bulk cutouts:

* DMS-REQ-0375 _Retrieval of postage stamp light curve images_.

  > Postage stamp cutouts, of size **postageStampSize** \[51 pixels] square, of all observations of a single Object shall be retrievable within **postageStampRetrievalTime** \[10 seconds], with **postageStampRetrievalUsers** \[10] simultaneous requests of distinct Objects.

## Existing Cutout Services

There are some existing services that support requests for cutouts of multiple targets:

* [PANSTARRS](https://outerspace.stsci.edu/spaces/PANSTARRS/pages/298812251/PS1+Image+Cutout+Service) talks about user uploads of 1,000 catalog positions taking less than 2 seconds and implies one file per cutout.
* [TESSCut](https://mast.stsci.edu/tesscut/) supports object and moving target cutout requests with a maximum cutout area of 10,000 pixels for time-series data, returning the results in a Zip of FITS files.
* The [IRSA Cutout Service](https://irsa.ipac.caltech.edu/applications/Cutouts/) implies that multiple cutouts are returned in a tar file.
* The [ESO Archive Science Portal](https://support.eso.org/en-US/kb/articles/asp-cut-out-service) cutout service has an example where two coordinates are uploaded and then two files are downloaded individually or in a Zip.
* [Cirada](http://cutouts.cirada.ca/help/#batch-example) has a batch mode that downloads results as PNG files in a gzipped tar file.
* The [CADC MegaPipe](https://www4.cadc.hia.nrc.gc.ca/en/megapipe/access/cut.html?) service returns the cutouts from each band for a single object individually.
* The [DESI Legacy Imaging Survey](https://www.legacysurvey.org) cutout service can return images for a single coordinate from multiple bands in a Multi-Extension FITS file.

There is also the Cutana {cite:p}`2025arXiv251104429G` application although that system processes one file at a time as provided to it by some other service.

## Interface

There are two variants of what a user might request in terms of bulk cutouts.

1. A time-series of a single coordinate for a specific `visit`-dimensioned dataset type.
2. A catalog containing many target coordinates with a request for cutouts of a specified size for each target for a specific dataset type.

It is also possible to consider a special case of requesting a cutout of a single location from a co-add in all the available wavebands.
For a single coordinate, this is closer to the time-series interface than the catalog interface.

In this context solar system objects are, from a user perspective, closer to a time-series request than a bulk cutout catalog request.
Internally there is added complication due to having to match up the solar system object to specific butler datasets.

There is no reason why the two variants have to be the same implementation.

A time-series request where a single coordinate can return a single file, might be possible using a standard SODA interface much like the existing cutout service, with the addition of a dataset type parameter.
The data release could either be specified using an additional parameter or there could be different endpoints for each data release.
A key difference from the existing cutout service being that the service itself would have to find the datasets rather than being sent an explicit UUID.
An alternate approach is to return a table, similar to how SIA works, with the time series (or co-add waveband) parameters along with data link columns referring to the existing single-cutout service.
This would allow for on-demand cutouts as usually displayed by a GUI tool.

The more complex bulk object cutout service would receive a catalog from the user, possibly supporting a variety of formats (Parquet, CSV, VOTable) containing ICRS RA and Dec coordinates.
They would have to specify a data release and dataset type and the service would inherently be asynchronous and would have to conform to the IVOA Universal Worker Service (UWS) standard {cite:p}`2016ivoa.spec.1024H` to let the user know when their job has completed and where the resulting files will be located.

### Returned Data

It is likely that the best file format for returning light-curve cutouts is not the same format that would be best for bulk object cutouts.

#### Light-Curve Data

Cutouts for a light curve of a single Object can be returned in a single file since even for a deep-drilling-field object after the 10 years of operations we are only talking about tens of thousands of cutouts.
Without resampling the cutouts the spatial WCS of each cutout is not the same because each cutout comes from a different part of the focal plane and is subject to different distortions with the object in question not being centered in the same place in every cutout pixel.
The simplest possible representation is for each image (with variance and mask) to be stored as distinct entities in the output file with their own metadata obtained from each visit.

A more compact option, which may be easier for Machine Learning systems, would be for the image, variance, and mask data to be stored in cubes and then having some table data that describes the WCS, timing, and band information for each cutout.
This would likely require we calculated an approximate linear spatial WCS for each cutout and dropped some of the more Rubin-specific FITS metadata from the output files.
It is also possible to have an entirely table-based format.

One option that should be considered is to support an on-demand cutout retrieval option for light curves.
For visualization of light curve images it is not certain that a user will want or need to display every single cutout.
In this scenario the light curve request would receive a VOTable response containing information about every source (for example the data from the `ForcedSource` table) along with a column corresponding to a DataLinker request for a cutout using the existing SODA cutout service.
Firefly is able to recognize this form and display a handful of cutouts at a time, updating in near-realtime as the user clicks on specific time stamps.
This works using the existing cutout infrastructure so long as a cutout can be obtained in a fraction of a second.

When the light-curve cutouts need to be persisted the three baseline options for light-curve file formats are therefore:

1. Multi-Extension FITS (MEF).
2. Greenbank convention binary tables in FITS {cite}`FITS:GreenBank`.
3. Extend the [Multimodal Universe](https://github.com/MultimodalUniverse/MultimodalUniverse) {cite}`2024RNAAS...8..301A` HDF5 data model to allow for light-curve cutouts and consider providing a Zarr {cite:p}`10.5281/zenodo.3773449` variant.

```{warning}
Note that in this document when we mention the Multimodal Universe file formats, we are not suggesting we send all our cutouts there for open access model training.
We are saying that there is prior art for how to store astronomy data in a form suitable for Machine Learning and providing our data in a form that is compatible with this tooling is presumed to be useful for that tooling and the community that needs to use it for model training more widely.
```

##### Multi-Extension FITS (MEF)

MEF files are the default file format that most astronomers would think of.
We use this format for writing out guider data from the camera and in our `lsst.meas.algorithms.Stamps` Python class.
MEF is well supported in display tools for looking at individual images and you have full fidelity for WCS and visit metadata.
File access of multiple cutouts is not efficient and tooling has to understand how to group each extension based on the `EXTVER` and `EXTNAME` FITS headers.
For small numbers of cutouts this format is acceptable.

##### Greenbank Convention Binary Tables

The Greenbank convention {cite}`FITS:GreenBank` stores cutouts in FITS binary tables.
This has the advantage that the image can be embedded with the associated source information, assuming that the user specified a source/object ID and not a position.
This convention has some issues with SIP WCS with many parameters and we might be required to extend the registered convention unless we recalculated the WCS for the smaller area.
We need to investigate how many rows can be stored in one of these files before they become to large to be used efficiently, and we need to understand the current situation with tooling that understands how to read and display these files.

##### Multimodal Universe HDF5

The [Multimodal Universe](https://github.com/MultimodalUniverse/MultimodalUniverse) project (MMU) is collecting datasets from different instruments in a form that makes it easy to use them when training Machine Learning models.
Data files in the MMU are stored in HDF5 format and each instrument is expected to provide some Python tooling to be able to read those files into the system in a standard way.
They currently have datasets from HSC which provides some guidance for how we should layout our files but they do not have any light-curve examples that include cutouts (the light-curve examples are using catalog photometry data).
Nevertheless, it seems like there is a straightforward way to combine the HSC approach (for a Rubin proposed format see [the Appendix](#multimodal-universe-file-format)) with the DES/PS1 light-curve approach (one HDF5 file per light-curve).

The HDF5 approach requires that cutouts be stored in data cubes with tabular data describing each cutout (such as the time and the band and any WCS approximations).
We would have to clarify whether there is an expectation that each cutout would be resampled into the same WCS grid.

Given that the data model used for writing to MMU HDF5 would be very similar to what we would be using for Zarr {cite:p}`10.5281/zenodo.3773449`, it would be straightforward to support both Zarr and HDF5.

#### Catalog Cutout Data

At its simplest the outputs from a bulk object catalog cutout request would be FITS files for every catalog position and every band with each file containing the image pixels, the variance, the mask, and a PSF image as well as provenance and inherited metadata.
We are not expecting to support raw data cutouts.
For millions of cutouts this number of files are difficult to manage at USDF and essentially impossible for an end user to download and manage.

We therefore need to come up with a scheme where multiple cutouts are combined into files using some kind of partitioning.
Given the potential

For deep coadd data we grid the data into a fixed sky map that consists of "tracts" that are split into "patches" with some overlap.
The batched output products would naturally be written at the tract level, potentially splitting into chunks if there are too many cutouts, or at the patch level, although for the patch level there is a potential to end up with very unbalanced file sizes if one patch only has a couple of cutouts but another crowded area had tens of thousands.
Downstream users may well want the outputs partitioned into HEALPix, necessitating a post-processing after cutout extraction, something that we might want to avoid.

For visit-based data the WCS varies for each cutout and if multiple detectors cover the cutout area some resampling will be required.
Additionally there is no guarantee of a matching number of cutouts for each band so the band and timestamp would have to be specified for every cutout.

For catalog cutouts the sheer scale of the potential number of cutouts does drive us towards a file format solution where the data can be compressed efficiently with minimal overhead for metadata, and where we can generate files of a reasonable size whilst trying to minimize the file count.

A Multi-extension FITS file containing four or five HDUs per cutout seems fundamentally unsuitable for storing tens of thousands of cutouts even if we added the HDU index extension that we are developing for the new visit image FITS format.
The Greenbank table extension may also not be suitable.
These FITS options *might* work if we explicitly capped the number of cutouts we store per file, at the expense of creating many more files.

The MMU approach of storing the cutouts in N-dimensional arrays seems like the best solution and we can build on the basic data model outlined in [the Appendix](#multimodal-universe-file-format) for HDF5, Zarr, and FITS variants, storing the important WCS and associated information in tables.
We can also include the Greenbank columns in tabular form (or as discrete 1-D arrays) such that we are able to specify band and WCS in a form that is already documented even if the pixel data is no longer in the table.
For visit-based FITS cubes it is acceptable to include the time of observation in the WCS as a lookup table, and for coadd cubes we can store the nominal wavelength of the filters in the WCS.
Most FITS libraries should be able to read FITS HDUs with more than 2 billion pixels (32-bit signed int), although I'm not sure if we have tested `lsst.afw.image` with that scale of data.

#### Tabular Data vs Image Data

Conceptually, from a data modeling perspective the file options described above correspond to two distinct data models.

1. A collection of discrete images.
2. Tabular data where a row corresponds to a cutout.

When N becomes high the only efficient approach is to think of the data as tabular, whether that is a single FITS binary tables containing everything or multiple hypercubes representing the pixel data indexed by cutout number along with associated metadata tables representing the cutout coordinates or time.
The actual serialized format (FITS, Zarr, HDF5, Parquet) does not matter as much as deciding to adopt tabular form as the baseline.

## Constraints

For co-added data which is warped onto a standard sky map, the pipeline processing ensures that there is a 200 pixel overlap between patches and tracts.
This implies that so long as cutout requests for co-adds are smaller than 200 pixels there is never a need to read in multiple input images to generate a cutout.

For visit images the key question is whether there is an expectation for the cutout to cross a detector boundary and include data from adjacent detectors.
It may be prudent to disallow this in the first version and return individual cutouts from each detector --- a cutout service that can cross boundaries can no longer be given an explicit IVOID {cite:p}`DMTN-302` for a Butler dataset (as used by the current cutout service) but must query the Butler itself, do the cutout from each (up to four) image, and resample and combine the small cutouts.
There is an implicit assumption that the visit/detector regions defined in the butler have been recalculated to reflect the final astrometric calibration as part of a data release.
Prompt products might not be able to have this correction applied.

Lossy-compressed visit images are now expected to be part of a formal data release but difference images are expected to be created on demand.
This makes them effectively unusable in a fast cutout service retrieving light curve images but would not necessarily be a problem for catalog-driven bulk cutouts.

## Implementation

The constraints described in the previous section result in some core requirements for any system that we use to process cutouts:

* We need a discovery layer that can convert a user request to Butler datasets.
* We potentially need to be able to combine multiple datasets with or without resampling to generate a single cutout (either because of detector boundaries or because the cutout size exceeds the overlap size).
* We likely will have to eventually support cutouts on "virtual" datasets that need to be generated on demand.
* We need to be able to realize when multiple cutout requests (from a single catalog) correspond to a single data file so that we can minimize how many times a file is read.
* In theory we would like to understand the actual WCS for visits when deciding on pixel bounds instead of the approximated FITS-compliant WCS.

We will initially consider the time-series cutout service to be distinct from the catalog-based cutout (if someone wants time-series cutouts at multiple locations that is simply the catalog-based cutout service with visit images but where the resulting packaging of results might require the user to do some book keeping to put things back together for each coordinates).

### Time-series on-demand

A service to return the results in a table with deferred cutouts would need to:

* Take an Object ID as parameter, and optional cutout size.
* Use the TAP service to return the forced source (or multi-band Object data).
* Determine the Butler data IDs corresponding to each result.
* Retrieve the UUIDs from the Butler for those data IDs.
* Form the IVOIDs for those datasets.
* Construct a VOTable with the TAP results and the data linker URL for the cutout service.
* Return the table.

For time-series the difference images and visit images could be returned as different columns or the user could specify the dataset type explicitly.
This assumes that these dataset types are part of a data release.

### Time-Series cutout

At the end of Data Release 10 the plan is for there to be more than 800 visits to each part of the survey region.
Deep-drilling fields will have far more visits than that but no more than 50,000.
The requirement that a request for this number of cutouts within 10 seconds is very challenging given that it can currently take up to a few seconds for an RSP user on Google to retrieve a single image from the Butler, longer if the file is at USDF and is not in the hot storage layer.

A naive version of this service along the lines of:

* make a Butler query of the `visit` dataset type for the circular cutout region;
* call the existing cutout service in parallel for each result using hundreds of workers;
* collect the resulting FITS images and combines them into a MEF;
* return the MEF to the caller;

could work, but presumably not within the required time constraints, at least not once a significant number of visits have been acquired.
Currently the fastest we can do a 100x100 pixel cutout of a co-add (using GCS) is about 0.5s, with visits taking slightly longer (since the WCS is not fixed) but with the caveat that visits will never be hosted at Google after DP1.

If it is also necessary to include source parameters for light-curve plotting in tabular form there will have to be an additional query to Qserv, as detailed in the previous section, along with explicit data ID driven Butler queries.

### Catalog-based cutouts

A request for a million cutouts is going to require many cutout jobs and may require a large number of files to be read from the Butler datastore.
This is inherently an asynchronous request and there is no expectation for the results to be available instantly.
Compute is available at SLAC (but maybe not very much) and at Google (how much can we spend and can SLAC networking to Google support the requests at scale?).

A key question here is whether we try to reuse the pipeline middleware infrastructure that we already use for data releases (involving a bespoke graph builder for BPS {cite:p}`2025ASPC..538..325G` and an after burner that packages the files) or we use a Kubernetes native system as a scaled up implementation of the existing cutout service.

#### Using BPS

A baseline implementation option discussed previously is to effectively convert the request into an HTCondor BPS submission running on the SLAC SLURM cluster.
This would look something like:

* Receive the catalog from the user (the service is running at SLAC and not Google).
* Analyze the catalog to determine which Butler datasets are relevant.
* Convert the catalog to parquet format.
* Make a quantum graph with the known dataset IDs as inputs and with a placeholder for the user-supplied catalog, modifying the graph after creation so it can point at the user catalog location.
* Submit a BPS job for that graph.
* Run a special PipelineTask that can read the catalog and select all the positions that are on the current image.
* Write the cutouts to a single MEF (Butler can do this using the `Stamps` class from `meas_algorithms`).
* In `finalJob` collect all the output MEF files and repackage them as desired.
* Send job completion message back to the server so that the user can retrieve the files.

The bulk service can, in theory, query the job status itself using the normal BPS tooling.

#### Using Google with Queues

An alternative proposal has been to use Google compute with a queue system.
This has the advantage that we can scale to as many nodes as needed to deal with many users requesting billions of cutouts, something that SLAC can not support.
It does, though, have the downside that it will require that all the image files are copied into Google, which will impact the throughput.
It may be that we can still utilize the existing pipeline infrastructure given that we have previously run BPS with PanDA on Google as part of Data Preview 0.2 {cite:p}`RTN-041`.

Additionally, we need to decide if we are putting a single job on the queue per catalog row (and so potentially requesting the same file multiple times) or if the queue manager is going to query the Butler and pass a single dataset ref plus subset of rows to the compute job.
Multiple cutouts per file would change the analysis on whether to try to use Astropy for remote S3 reads vs downloading the entire file.
Even if the caller is only requesting image data (no mask or variance) once more than 2 or 3 cutouts are requested for a dataset it is more efficient to download the file and cutout from the file than it is is to do remote reads.

One possibility for the large-scale cutout requests is that we do not run the job instantly but wait for other requests to come in.
It is possible, especially for co-adds, that we might get multiple requests that use the same Butler dataset.
Combining requests from multiple users into a single cutout job would reduce load on the system at the expense of potential delays and extra book-keeping to make sure each person gets the cutouts they asked for.

Any Google-based system would need to gather the results from the workers and collect them into the desired response format and write them to the output bucket.

## Conclusion

To get some form of cutout service to the community in the shortest time the plan is:

1. Develop a on-demand time-series cutout service.
   This will require that we can get the standard cutout service working with sub-second performance.
2. Develop the time-series service with the cutouts embedded in a FITS binary table.
3. Implement bulk cutouts using an undecided method.

## References

```{bibliography}
```

## Appendix

(multimodal-universe-file-format)=
### MultiModal Universe File Format

Version: 1.0 (draft) \
Scope: Multiband coadd postage stamps with PSF, variance, and masks \
Compatibility target: MultiModalUniverse (HSC-style datasets)

The default design follows what is currently being used for HSC datasets.
Time-series datasets in MMU are currently single catalog value light-curves and are not image based.
We could design our own extension to this format to support a time axis with the one constraint being that light-curve MMU datasets have one file per object and are named after the object ID.

#### Overview

This specification defines an HDF5 file format for storing multiband image cutouts extracted from Rubin Observatory coadd images.

Each file contains multiple astronomical objects.
Each object includes:

- Aligned cutouts in multiple bands
- Variance and mask images
- PSF models for each cutout
- Catalog metadata giving at least the RA/Dec position of each cutout.

The format is designed to:

- Support efficient array-based access
- Allow deterministic export to MultiModalUniverse (MMU)

#### File Organization

##### Directory Layout

MMU requires that files SHOULD be partitioned by HEALPix:

```
rubin_deep_coadds/
  healpix=XXXX/
    part-0000.hdf5
    part-0001.hdf5
```

##### Top-Level Structure

Each HDF5 file MUST contain:

```
/
├── meta/
├── catalog/
└── images/
```

MMU itself does not care directly about the file layouts, only requiring that each instrument provides some loader code that can read the contents in using a standard API.

#### Metadata Group (/meta)

##### Required Datasets

```
/meta/survey                "LSST"
/meta/instrument            "LSSTCam"
/meta/release               string
/meta/product               "deep_coadd_cutouts"
/meta/schema_version        "1.0"
/meta/healpix_nside_catalog int
/meta/healpix_ordering      "nested"
```

##### Band and Geometry

```
/meta/band_order            ["u","g","r","i","z","y"]
/meta/image_shape           [Ny, Nx]
/meta/psf_shape             [Py, Px]
```

The shapes are duplicated here (they are known from the HDF5 structure) to provide a quick way to read the information and to provide internal conformance.

#### Catalog Group (/catalog)

##### Required Fields

```
/catalog/object_id
/catalog/ra
/catalog/dec
/catalog/healpix
```

where the HEALPix NSIDE is specified in the `/meta/` group.
In theory we could also store MOCs of the cutouts, although that can become quite complicated.
HSC includes additional information obtained from the catalog products.
Including that information would require additional queries to Qserv or reads of the equivalent Butler parquet files.

#### Image Data Group (/images)

For N objects cut out from Nb bands:

```
/images/band                (N, Nb)
/images/flux                (N, Nb, Ny, Nx)
/images/variance            (N, Nb, Ny, Nx)
/images/mask                (N, Nb, Ny, Nx)
/images/psf                 (N, Nb, Py, Px)
/images/psf_fwhm            (N, Nb)
/images/pixel_scale         (N, Nb)
```

The `/images/band` and `/images/pixel_scale` are there for consistency with the underlying HSC example for the MMU files even though we can assume the band order for each object and can assume the pixel scale is constant.
We can drop them from our format and move them to constants in the `/meta` section.

MMU prefers inverse variance so we do have to decide if we store that directly or have to write a mapping for MMU import.

| Rubin | MMU |
|------|-----|
| flux | image_array |
| variance | image_ivar |
| band | image_band |
