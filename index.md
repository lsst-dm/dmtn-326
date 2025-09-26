# Bulk Cutout Service Implementation Options

```{abstract}
The bulk cutout service is a deliverable required to support Data Preview 2.
This document explains our implementation options regarding how to perform the cutouts at scale and in what form the resulting cutouts should be returned.
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


## Interface

There are two variants of what a user might request in terms of bulk cutouts.

1. A time-series of a single coordinate for a specific `visit`-dimensioned dataset type.
2. A catalog containing many target coordinates with a request for cutouts of a specified size for each target for a specific dataset type.

It is conceivable that these two variants are distinct services and they could also have completely different implementations.

A time-series request where a single coordinate can return a single image, might be possible using a standard SODA interface much like the existing cutout service, with the addition of a dataset type parameter.
The data release could either be specified using an additional parameter or there could be different endpoints for each data release.
A key difference from the existing cutout service being that the service itself would have to find the datasets rather than being sent an explicit UUID.

The more complex bulk service would receive a catalog from the user, possibly supporting a variety of formats (Parquet, CSV, VOTable) containing ICRS RA and Dec coordinates.
They would have to specify a data release and dataset type and the service would inherently be asynchronous and would have to conform to the IVOA Universal Worker Service (UWS) standard {cite:p}`2016ivoa.spec.1024H` to let the user know when their job has completed and where the resulting files will be located.

### Returned Data

The fundamental result from a bulk cutout request consist of, potentially, millions of small FITS files with each containing metadata (including provenance and WCS information), the pixel values and also variance and mask information.[^raws]
This could be considered to be unwieldy in terms of download performance (many small files are not efficient) and in terms of the huge results message to be generated listing every file.

Other options for the results packaging could be Zip archives or multi-extension FITS (MEF).
Each of these options reduce the file count but for the largest batch jobs it's likely that we would want to generate more than one output file to balance file size with file count.
Cutouts could be grouped evenly across multiple files or they could be collected together by sky tract (effectively a spatial partition).

MEF files have reasonable support in existing tooling but the FITS data model has only limited grouping capabilities and would require the tooling to understand that each `EXTVER` value corresponds to a single cutout.
A MEF may well be the best option for the specific case of a time-series request at a single coordinate.
Zip archives would usually be treated as a transport medium with the user unpacking the file as soon as they receive it.

It's not clear whether we need to integrate bulk cutout requests into Firefly or other VO-capable tooling, and any decision on that may well have an impact on how cutouts are returned.

## Constraints

For co-added data which is warped onto a standard sky map, the pipeline processing ensures that there is a 200 pixel overlap between patches and tracts.
This implies that so long as cutout requests for co-adds are smaller than 200 pixels there is never a need to read in multiple input images to generate a cutout.

For visit images the key question is whether there is an expectation for the cutout to cross a detector boundary and include data from adjacent detectors.
It may be prudent to disallow this in the first version and return individual cutouts from each detector.
There is an implicit assumption that the visit/detector regions defined in the butler have been recalculated to reflect the final astrometric calibration as part of a data release.
Prompt products might not be able to have this correction applied.

## Implementation

We will initially consider the time-series cutout to be distinct from the catalog-based cutout (if someone wants time-series cutouts at multiple locations that is simply the catalog-based cutout service with visit images but where the resulting packaging of results might require the user to do some book keeping to put things back together for each coordinates).

### Time-Series cutout

At the end of Data Release 10 the plan is for there to be more than 800 visits to each part of the survey region.
The requirement that a request for this number of cutouts within 10 seconds is very challenging given that it can currently take a few seconds for an RSP user on Google to retrieve a single image from the Butler.

A naive version of this service along the lines of:

* make a Butler query of the `visit` dataset type for a the circular cutout region;
* call the existing cutout service in parallel for each result using hundreds of workers;
* collect the resulting FITS images and combines them into a MEF;
* return the MEF to the caller;

could work, but presumably not within the required time constraints, at least not once a significant number of visits have been acquired.

### Catalog-based cutouts

A request for a million cutouts is going to require significant compute and may require a large number of files to be read from the Butler datastore.
This is inherently an asynchronous request and there is no expectation for the results to be available instantly.
Compute is available at SLAC (but maybe not very much) and at Google (how much can we spend?).

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

An alternative proposal has been to use Google compute.
This has the advantage that we can scale to as many nodes as needed to deal with many users requesting billions of cutouts, something that SLAC can not support.
It does, though, have the downside that it will require that all the image files are copied into Google, which will impact the throughput.
It may be that we can still utilize the existing pipeline infrastructure given that we have previously run BPS with PanDA on Google as part of Data Preview 0.2 {cite:p}`RTN-041`.

## Conclusion

TBD.

## References

```{bibliography}
```

[^raws]: Cutouts of raw data are likely not useful and so will not be supported (at least initially).
