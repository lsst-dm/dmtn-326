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

The more complex bulk service would receive a catalog from the user, possibly supporting a variety of formats (Parquet, CSV, VOTable) containing ICRS RA and Dec coordinates.
They would have to specify a data release and dataset type and the service would inherently be asynchronous and would have to conform to the IVOA Universal Worker Service (UWS) standard {cite:p}`2016ivoa.spec.1024H` to let the user know when their job has completed and where the resulting files will be located.

### Returned Data

The fundamental result from a bulk cutout request consist of, potentially, millions of small FITS files with each containing metadata (including provenance and WCS information), the pixel values and also variance and mask information.[^raws]
This could be considered to be unwieldy in terms of download performance (many small files are not efficient) and in terms of the huge results message to be generated listing every file.

Other options for the results packaging could be Zip archives, data cubes, tables with embedded cutouts, or multi-extension FITS (MEF).
Each of these options reduce the file count but for the largest batch jobs it's likely that we would want to generate more than one output file to balance file size with file count.
Cutouts could be grouped evenly across multiple files or they could be collected together by sky tract (effectively a spatial partition).
The community is also requesting cutouts in Zarr format {cite:p}`10.5281/zenodo.3773449` and both FITS and Zarr are supported by the new Cutana tool {cite:p}`2025arXiv251104429G`.

MEF files have reasonable support in existing tooling but the FITS data model has only limited grouping capabilities and would require the tooling to understand that each `EXTVER` value corresponds to a single cutout.
For large MEF files they are also inefficient to access given the lack of indexing facilities in FITS.
Zip archives would usually be treated as a transport medium with the user unpacking the file as soon as they receive it.

Another approach for light-curves is to return a data cube with a spatial slice and a time axis.
In FITS it's not really possible to have a completely independent absolute-sky-position WCS for each plane in a cube (this is possible using the Starlink AST WCS natively {cite:p}`2016A&C....15...33B` but we have to be FITS standard compliant).
This is not as trivial as it at first appears as the pixel-preserving cutouts from single-epoch images for even completely stationary objects are not a great match, as every image will have different registration and rotational dithering, so there's no conceivable common WCS for the spatial dimensions of the cube as a whole.
Even if, again, for the particular application that didn't matter, and there was no spatial WCS and all the cutouts were just aligned on row/column directions, the source would still be at a different sub-pixel position in every plane, and there's no natural place to stash that information in a FITS cube.
Stationary-object cutouts that are warped to a common pixel map (e.g., `TAN`) with the object at exactly the same pixel position in each epoch do work nicely.
Observation times can be preserved in a lookup-table time-axis WCS, 100% FITS-standard.
Moving-object cutouts for stars with proper motions probably are also OK in that format, as for many use cases it's likely to be interesting to see the target move from cutout to cutout against the background objects.
TAN-warped moving-object cutouts for SSOs are not as good a match, but it is still possible to define a relative WCS for the cube, returning local North/East angular offsets, for instance, reflecting either a differential position from the `DiaSource` position in each individual epoch, or a differential position from the predicted location from the computed orbit; which one to do might best be a user-selectable choice.

A data cube might be useful for returning cutouts from multiple bands from the coadd images.
These all have the same WCS and so there are no complications in returning cutouts in this form.

An alternative approach to MEF is to return each cutout as a row in a binary table using the Greenbank convention {cite}`FITS:GreenBank`.
This has the advantage that the image can be embedded with the associated source information, assuming that the user specified a source/object ID and not a position.
This convention has some issues with SIP WCS with many parameters and we might be required to extend the registered convention.
Multiple files will be needed once the number of cutouts becomes large.

One approach used by IPAC specifically for light curves is to return VOTable data representing the light curve (obtained from the forced source table for example) along with a column corresponding to a DataLinker request for a cutout.
Firefly is able to recognize this form and display a handful of cutouts at a time, updating in near-realtime as the user clicks on specific time stamps.
This works using the existing cutout infrastructure so long as a visit or diffim cutout can be obtained in a fraction of a second.

It's not clear whether we need to integrate bulk cutout requests into Firefly or other VO-capable tooling, and any decision on that may well have an impact on how cutouts are returned.

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

We will initially consider the time-series cutout to be distinct from the catalog-based cutout (if someone wants time-series cutouts at multiple locations that is simply the catalog-based cutout service with visit images but where the resulting packaging of results might require the user to do some book keeping to put things back together for each coordinates).

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
Deep-drilling fields will have far more visits than that.
The requirement that a request for this number of cutouts within 10 seconds is very challenging given that it can currently take a few seconds for an RSP user on Google to retrieve a single image from the Butler.

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
Compute is available at SLAC (but maybe not very much) and at Google (how much can we spend?).

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

#### Use a Third-party Tool

It is conceivable that if we find that small WCS errors caused from using a SIP approximation are acceptable when doing cutouts, we could make use of an existing cutout tool such as Cutana {cite:p}`2025arXiv251104429G`.
There would have to be tooling in front of Cutana to convert the user request to something understandable by Cutana given that the Butler would have to be queried to find the locations of the FITS files and assigning cutout requests to each file.
The source code for Cutana is not yet available to the community so it is not yet clear whether butler could be integrated directly into the software.

For visit requests crossing detector boundaries we would likely have to return multiple cutouts and require the end user deal with any mosaicking.

## Conclusion

To get some form of cutout service to the computer in the shortest time the plan is:

1. Develop a on-demand time-series cutout service.
   This will require that we can get the standard cutout service working with sub-second performance.
2. Develop the time-series service with the cutouts embedded in a FITS binary table.
3. Implement bulk cutouts using an undecided method.

## References

```{bibliography}
```

[^raws]: Cutouts of raw data are likely not useful and so will not be supported (at least initially).
