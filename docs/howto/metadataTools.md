## Metadata Tools

TOC
- [Ingest Manifest](#ingest-manifest)
- [Index and ingest manifest](#index-and-ingest-manifest)
  - [Index and ingest packages](#index-and-ingest-packages)
- [Searching Indexd to get GUID for Metadata Ingestion](#searching-indexd-to-get-guid-for-metadata-ingestion)
- [Manifest Merge](#manifest-merge)
  - [Ideal Scenario (Column to Column Match, Indexing:Metadata Manifest Rows)](#ideal-scenario-column-to-column-match-indexingmetadata-manifest-rows)
  - [Non-Ideal Scenario (Partial URL Matching)](#non-ideal-scenario-partial-url-matching)
- [Using Gen3 Jobs](#using-gen3-jobs)
- [Verify Metadata Manifest](#verify-metadata-manifest)


### Ingest Manifest

For populating the metadata service via a file filled with metadata. Uses asynchronous
calls for you.

The file provided must contain a "guid" column (or you can use a different column name or different logic entirely by providing a `guid_for_row` function)

The row contents can contain valid JSON and this script will correctly nest that JSON
in the resulting metadata.

```python
import sys
import logging
import asyncio

from gen3.auth import Gen3Auth
from gen3.tools import metadata
from gen3.tools.metadata.ingest_manifest import manifest_row_parsers
from gen3.utils import get_or_create_event_loop_for_thread

logging.basicConfig(filename="output.log", level=logging.DEBUG)
logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))

# a file containing a "guid" column and additional, arbitrary columns to populate
# into the metadata service
MANIFEST = "dbgap_extract_guid.tsv"

def main():
    loop = get_or_create_event_loop_for_thread()


    auth = Gen3Auth(refresh_file="credentials.json")

    # must provide a str to namespace the metadata from the file in a block in
    # the metadata service
    metadata_source = "dbgap"

    # (optional) override default guid parsing behavior
    def _custom_get_guid_for_row(commons_url, row, lock):
        """
        Given a row from the manifest, return the guid to use for the metadata object.

        Args:
            commons_url (str): root domain for commons where mds lives
            row (dict): column_name:row_value
            lock (asyncio.Semaphore): semaphones used to limit ammount of concurrent http
                connections if making a call to an external service

        Returns:
            str: guid
        """
        return row.get("guid") # OR row.get("some_other_column")

    # (optional) override default guid parsing behavior
    manifest_row_parsers["guid_for_row"] = _custom_get_guid_for_row

    loop.run_until_complete(
        metadata.async_ingest_metadata_manifest(
            auth.endpoint, manifest_file=MANIFEST, metadata_source=metadata_source, auth=auth
        )
    )

if __name__ == "__main__":
    main()

```

Example file:

```
guid    submitted_sample_id biosample_id    dbgap_sample_id sra_sample_id   submitted_subject_id    dbgap_subject_id    consent_code    consent_short_name  sex body_site   analyte_type    sample_use  repository  dbgap_status    sra_data_details    study_accession study_accession_with_consent    study_with_consent  study_subject_id
95a41871-222c-48ae-8004-63f4ed1f0691    NWD680715   SAMN04109058    1784155 SRS1361261  DBG00391    1360750 2   HMB-IRB-MDS female  Blood   DNA ["Seq_DNA_SNP_CNV"] TOPMed_WGS_Amish    Loaded  {"status": "public", "experiments": "1", "runs": "1", "bases": "135458977924", "size_Gb": "25", "experiment_type": "WGS", "platform": "ILLUMINA", "center": "UM-TOPMed"}    phs000956.v3.p1 phs000956.v3.p1.c2  phs000956.c2    phs000956.v3_DBG00391
```

Would result in the following metadata records in the metadata service:

```python
{
    _guid_type: "indexed_file_object",
    dbgap: {
        sex: "female",
        body_site: "Blood",
        repository: "TOPMed_WGS_Amish",
        sample_use: [
            "Seq_DNA_SNP_CNV"
        ],
        analyte_type: "DNA",
        biosample_id: "SAMN04109058",
        consent_code: 2,
        dbgap_status: "Loaded",
        sra_sample_id: "SRS1361261",
        dbgap_sample_id: 1784155,
        study_accession: "phs000956.v3.p1",
        dbgap_subject_id: 1360750,
        sra_data_details: {
            runs: "1",
            bases: "135458977924",
            center: "UM-TOPMed",
            status: "public",
            size_Gb: "25",
            platform: "ILLUMINA",
            experiments: "1",
            experiment_type: "WGS"
        },
        study_subject_id: "phs000956.v3_DBG00391",
        consent_short_name: "HMB-IRB-MDS",
        study_with_consent: "phs000956.c2",
        submitted_sample_id: "NWD680715",
        submitted_subject_id: "DBG00391",
        study_accession_with_consent: "phs000956.v3.p1.c2"
    }
}
```

> NOTE: `_guid_type` is populated automatically, depending on if the provided GUID exists in indexd or not. Either `indexed_file_object` or `metadata_object`.

### Index and ingest manifest

The module for indexing object files in a manifest (against indexd's API) can be configured to submit any additional columns (metadata that are not stored in indexd) to the metadata service. The `submit_additional_metadata_columns` flag should be set to `True` like in the example below.

See the [indexing docs](./diirmIndexing.md#indexing-manifest) for details about other parameters and the input manifest.

The following is an example of tsv manifest. Submitting this manifest would result in the creation of an indexd record containing the `md5`, `size`, `authz` and `url` data, and an MDS record containing the `my_column` data.

```
guid	my_column	md5	size	authz	url
255e396f-f1f8-11e9-9a07-0a80fada099c	my_additional_metadata	473d83400bc1bc9dc635e334faddf33c	363455714	/programs/DEV/project/test	['Open']	[s3://examplebucket/test1.raw]
```

```python
import sys
import logging

from gen3.auth import Gen3Auth
from gen3.tools.indexing.index_manifest import index_object_manifest

logging.basicConfig(filename="output.log", level=logging.DEBUG)
logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))

MANIFEST = "./example_manifest.tsv"


def main():
    auth = Gen3Auth(refresh_file="credentials.json")
    # basic auth for admin privileges in indexd cannot be
    # used when submitting to the metadata service

    index_object_manifest(
        commons_url=auth.endpoint,
        manifest_file=MANIFEST,
        thread_num=8,
        auth=auth,
        replace_urls=False,
        manifest_file_delimiter="\t", # put "," if the manifest is csv file
        submit_additional_metadata_columns=True, # submit additional metadata to the metadata service
    )

if __name__ == "__main__":
    main()

```

#### Index and ingest packages

A package is a set of files. For example, it could be a zip file grouping all individual files for a study.

When `submit_additional_metadata_columns` is set to `True` and additional metadata is submitted to the metadata service, the predefined `record_type` and `package_contents` metadata columns allow the creation of packages.

- `record_type` is an optional column. The allowed values are `object` (default) and `package`. The same manifest can contain both object and package rows.
- If the `record_type` is `package`, the `package_contents` column can be used to describe the individual files in the package. The value is a JSON list of objects. Each object must include a `file_name`, can include a `size` and/or `hashes`, and can include additional arbitrary metadata to describe the files.

The following is an example of tsv manifest describing a package.

```
record_type	guid	md5	size	authz	url	file_name	package_contents
package	255e396f-f1f8-11e9-9a07-0a80fada0901	473d83400bc1bc9dc635e334faddf33c	363455714	['/open/packages']	s3://my-data-bucket/dg.1234/path/package.zip	package.zip	[{"hashes":{"md5sum":"2cd6ee2c70b0bde53fbe6cac3c8b8bb1"},"file_name":"yes.txt","size":35},{"hashes":{"md5sum":"30cf3d7d133b08543cb6c8933c29dfd7"},"file_name":"hi.txt","size":35}]
```

Submitting this manifest would result in the creation of an indexd record containing the `md5`, `size`, `authz`, `url` and `file_name` data, and an MDS record describing the package:

```
{
    "type": "package",
    "package": {
        "version": "0.1",
        "file_name": "package.zip",
        "file_size": 363455714,
        "hashes": {
            "md5sum": "473d83400bc1bc9dc635e334faddf33c"
        },
        "contents": [
            {
                "hashes": {
                    "md5sum": "2cd6ee2c70b0bde53fbe6cac3c8b8bb1"
                },
                "file_name": "yes.txt",
                "file_size": 35,
            },
            {
                "hashes": {
                    "md5sum": "30cf3d7d133b08543cb6c8933c29dfd7"
                },
                "file_name": "hi.txt",
                "file_size": 35,
            }
        ],
        "created_time": "2022-01-07 23:17:02.356981",
        "updated_time": "2022-01-07 23:17:02.356981"
    },
    "_bucket": "s3://my-data-bucket",
    "_filename": "package.zip",
    "_uploader_id": "16",
    "_upload_status": "uploaded",
    "_file_extension": "zip",
    "_resource_paths": [
        "/open/packages"
    ]
}
```

### Searching Indexd to get GUID for Metadata Ingestion

It is possible to try and dynamically retrieve a GUID for a row in the manifest file
provided. However, this is limited by indexd's ability to scale to the queries you
want to run. Indexd's querying capabilities are limited and don't scale well with a
large volume of records (it is meant to be a key:value store much like the metadata service).

> WARNING: This is not recommended to be used at scale. Consider making these associations to metadata before ingestion. See [merging tools](#manifest-merge)

```python
import sys
import logging
import asyncio

from gen3.auth import Gen3Auth
from gen3.tools import metadata
from gen3.tools.metadata.ingest_manifest import manifest_row_parsers

logging.basicConfig(filename="output.log", level=logging.DEBUG)
logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))

MANIFEST = "dbgap_extract.tsv"


def main():
    loop = get_or_create_event_loop_for_thread()


    auth = Gen3Auth(refresh_file="credentials.json")

    # must provide a str to namespace the metadata from the file in a block in
    # the metadata service
    metadata_source = "dbgap"

    # (optional) override default indexd querying (NOTE: must be async)
    async def _custom_query_for_associated_indexd_record_guid(commons_url, row, lock, output_queue):
        """
        Given a row from the manifest, return the guid for the related indexd record.

        WARNING: The query endpoint this uses in indexd is incredibly slow when there are
                 lots of indexd records.

        Args:
            commons_url (str): root domain for commons where mds lives
            row (dict): column_name:row_value
            lock (asyncio.Semaphore): semaphones used to limit ammount of concurrent http
                connections
            output_queue (asyncio.Queue): queue for logging output

        Returns:
            str: guid or None
        """
        mapping = {"urls": "submitted_sample_id"}

        # special query endpoint for matching url patterns
        records = []
        if "urls" in mapping:
            pattern = row.get(mapping["urls"])
            logging.debug(
                f"trying to find matching record matching url pattern: {pattern}"
            )
            records = await metadata.async_query_urls_from_indexd(
                pattern, commons_url, lock
            )

        logging.debug(f"matching record(s): {records}")

        if len(records) > 1:
            msg = (
                "Multiple records were found with the given search criteria, this is assumed "
                "to be unintentional so the metadata will NOT be linked to these records:\n"
                f"{records}"
            )
            logging.warning(msg)
            records = []

        guid = None
        if len(records) == 1:
            guid = records[0].get("did")

        return guid

    # (optional) override default indexd querying
    manifest_row_parsers[
        "indexed_file_object_guid"
    ] = _custom_query_for_associated_indexd_record_guid

    loop.run_until_complete(
        # get_guid_from_file=False tells tool to try and get the guid using
        # the provided custom query function
        metadata.async_ingest_metadata_manifest(
            auth.endpoint,
            manifest_file=MANIFEST,
            metadata_source=metadata_source,
            auth=auth,
            get_guid_from_file=False,
        )
    )


if __name__ == "__main__":
    main()

```

Setting `get_guid_from_file`  to `False` tells tool to try and get the guid using
the provided custom query function instead of relying on a column in the manifest.

> NOTE: By default, the `indexed_file_object_guid` function attempts to query indexd URLs to pattern match
whatever is in the manifest column `submitted_sample_id`.


### Manifest Merge

If you have a manifest full of metadata and a manifest of indexed file objects in Indexd, you can use this script to merge the two into a metadata manifest for ingestion.

For example, a common use case for this is if you have a file full of metadata from dbGaP and want to get associated GUIDs for each row. You can then add the dbGaP metadata to the metadata service for those GUIDs with the file output from this merge script.

The script is also fairly configurable depending on how you need to map between the two files. Even if the "indexing file" doesn't include GUIDs, this script can still be used to combine 2 manifests based on a common column.

The ideal scenario is when you can map column to column between your _metadata manifest_ and _indexing manifest_ (e.g. what's in indexd).

The non-ideal scenario is if you need something for partially matching one column to another. For example: if one of the indexed URLs will contain `submitted_sample_id` somewhere in the filename. In this case, the efficiency of the script becomes O(n^2). If you can reliably parse out the section of the URL to match that could improve this. *tl;dr* Depending on your logic and number of rows in both files, this could be very very slow.

By default this merge can match multiple GUIDs with the same metadata (depending on the configuration). This supports situations where there may exist metadata that applies to multiple files. For example: dbGaP sample metadata applied to both CRAM and CRAI genomic files.

So while this supports metadata matching multiple GUIDs, it does *not* support GUIDs matching multiple sets of metadata.

#### Ideal Scenario (Column to Column Match, Indexing:Metadata Manifest Rows)

Consider the following example files.

*metadata manifest*: dbGaP extract file perhaps by using [this tool](https://github.com/uc-cdis/dbgap-extract):

```
submitted_sample_id, dbgap_subject_id, consent_short_name, body_site, ....
```

*indexing manifest* (perhaps provided by the data owner):

```
guid, sample_id, file_size, md5, md5_hex, aws_uri, gcp_uri
```

The strategy here is to map from the `submitted_sample_id` from the metadata manifest into the `sample_id` and then use the `guid` from the indexing manifest in the final output. That final output will can be used as the ingestion file for metadata ingestion.

```python
import sys
import logging

from gen3.tools.merge import merge_guids_into_metadata
from gen3.tools.merge import manifests_mapping_config


logging.basicConfig(filename="output.log", level=logging.DEBUG)
logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))

def main():
    indexing_manifest = (
        "/path/to/indexing_manifest.csv"
    )
    metadata_manifest = (
        "/path/to/metadata_extract.tsv"
    )

    # what column to use as the final GUID for metadata (this MUST exist in the
    # indexing file)
    manifests_mapping_config["guid_column_name"] = "guid"

    # what column from the "metadata file" to use for mapping
    manifests_mapping_config["row_column_name"] = "submitted_sample_id"

    # this configuration tells the function to use the "sample_id" column
    # from the "indexing file" to map to the metadata column configured above
    # (and these should match EXACTLY, 1:1)
    manifests_mapping_config["indexing_manifest_column_name"] = "sample_id"

    output_filename = "metadata-manifest.tsv"

    merge_guids_into_metadata(
        indexing_manifest, metadata_manifest, output_filename=output_filename,
        manifests_mapping_config=manifests_mapping_config,
        include_all_indexing_cols_in_output=False,
    )

if __name__ == "__main__":
    main()

```

The final output file will contain all the columns from the metadata manifest in addition to a new GUID column which maps to indexed records.

> NOTE: If you want all the indexing manifest columns as well, make sure to set `include_all_indexing_cols_in_output=True`

*output manifest* (to be used in metadata ingestion):

```
guid, submitted_sample_id, dbgap_subject_id, consent_short_name, body_site, ....
```

#### Non-Ideal Scenario (Partial URL Matching)

Consider the following example files.

*metadata manifest*: dbGaP extract file perhaps by using [this tool](https://github.com/uc-cdis/dbgap-extract):

```
submitted_sample_id, dbgap_subject_id, consent_short_name, body_site, ....
```

*indexing manifest* (perhaps by using the [download manifest tool](#download-manifest)):

```
guid, urls, authz, acl, md5, file_size, file_name
```

> NOTE: The indexing manifest contains no exact column match to the metadata manifest.

The strategy here is to look for partial matches of the metadata manifest's `submitted_sample_id` in the indexing manifest's `urls` field.

```python
import sys
import logging

from gen3.tools.merge import (
    merge_guids_into_metadata,
    manifest_row_parsers,
    manifests_mapping_config,
    get_guids_for_manifest_row_partial_match,
)


logging.basicConfig(filename="output.log", level=logging.DEBUG)
logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))


def main():
    indexing_manifest = (
        "/path/to/indexing_manifest.csv"
    )
    metadata_manifest = (
        "/path/to/metadata_extract.tsv"
    )
    # what column to use as the final GUID for metadata (this MUST exist in the
    # indexing file)
    manifests_mapping_config["guid_column_name"] = "guid"

    # what column from the "metadata file" to use for mapping
    manifests_mapping_config["row_column_name"] = "submitted_sample_id"

    # this configuration tells the function to use the "gcp_uri" column
    # from the "indexing file" to map to the metadata column configured above
    # (for partial matching the metdata data column to this column )
    manifests_mapping_config["indexing_manifest_column_name"] = "urls"

    # by default, the functions for parsing the manifests and rows assumes a 1:1
    # mapping. There is an additional function provided for partial string matching
    # which we can use here.
    manifest_row_parsers["guids_for_manifest_row"] = get_guids_for_manifest_row_partial_match

    output_filename = "metadata-manifest-partial.tsv"

    merge_guids_into_metadata(
        indexing_manifest=indexing_manifest,
        metadata_manifest=metadata_manifest,
        output_filename=output_filename,
        manifests_mapping_config=manifests_mapping_config,
        manifest_row_parsers=manifest_row_parsers,
        include_all_indexing_cols_in_output=False,
    )


if __name__ == "__main__":
    main()
```

> WARNING: The efficiency here is O(n2) so this does not scale well with large files.

The final output file will contain all the columns from the metadata manifest in addition to a new GUID column which maps to indexed records.

> NOTE: If you want all the indexing manifest columns as well, make sure to set `include_all_indexing_cols_in_output=True`

*output manifest* (to be used in metadata ingestion):

```
guid, submitted_sample_id, dbgap_subject_id, consent_short_name, body_site, ....
```

### Using Gen3 Jobs

There are some Gen3 jobs that were tailored for metadata ingestions and getting metadata from a public dbGaP API. The following are some example scripts that could be useful for utilizing those new jobs:

> NOTE: All of these jobs require specific permissions in the Gen3 environment

```python
import sys
import logging
import asyncio

from gen3.index import Gen3Index
from gen3.auth import Gen3Auth
from gen3.jobs import Gen3Jobs, DBGAP_METADATA_JOB, INGEST_METADATA_JOB

# An API Key downloaded from the above commons' "Profile" page
API_KEY_FILEPATH = "credentials.json"

logging.basicConfig(filename="output.log", level=logging.INFO)
logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))

def get_dbgap_merged_metadata_manifest():
    auth = Gen3Auth(refresh_file=API_KEY_FILEPATH)
    jobs = Gen3Jobs(auth_provider=auth)

    # this configuration tells the job to pull sample information from the public dbgap
    # api for the list of dbgap phsids (AKA study accession numbers) provided.
    #
    # The indexing_manifest_url is a publically available indexing manifest with at
    # a minimum columns to represent the GUID and some other field we can map to
    # a field from dbgap, in this example, we're doing a partial string match of
    # "submitted_sample_id" from dbgap to the indexing manifest's "urls" column
    #
    # If there is an exact match available, you can set "partial_match_or_exact_match"
    # to "exact_match" and this will perform the merging MUCH faster
    job_input = {
        "phsid_list": "phs000920 phs000921 phs000946 phs000951 phs000954 phs000956 phs000964 phs000972 phs000974 phs000988 phs000993 phs000997 phs001024 phs001032 phs001040 phs001062 phs001143 phs001189 phs001207 phs001211 phs001215 phs001217 phs001218 phs001237 phs001293 phs001345 phs001359 phs001368 phs001387 phs001402 phs001412 phs001416",
        "indexing_manifest_url": "https://example-test-bucket.s3.amazonaws.com/indexing_manifest_with_guids.csv",
        "manifests_mapping_config": {
            "guid_column_name": "guid",
            "row_column_name": "submitted_sample_id",
            "indexing_manifest_column_name": "urls",
        },
        "partial_match_or_exact_match": "partial_match",
    }

    loop = get_or_create_event_loop_for_thread()


    job_output = loop.run_until_complete(
        jobs.async_run_job_and_wait(job_name=DBGAP_METADATA_JOB, job_input=job_input)
    )

    # output contains signed URLs to download the new merged metadata manifest
    print(job_output)


def metadata_ingest():
    auth = Gen3Auth(refresh_file=API_KEY_FILEPATH)
    jobs = Gen3Jobs(auth_provider=auth)

    # provide a URL for a manifest that contains a GUID column along with arbitrary
    # other columns to add to the metadata service. The "metadata_source" namespaces
    # this data in the metadata service to support multiple different sources of metadata
    #
    # For example, this will create a metadata blob like:
    # {"dbgap": {"colA": "valA", "colB": valB}}
    job_input = {
        "URL": "https://example-bucket/merged_metadata_manifest.tsv",
        "metadata_source": "dbgap",
    }

    loop = get_or_create_event_loop_for_thread()


    job_output = loop.run_until_complete(
        jobs.async_run_job_and_wait(job_name=INGEST_METADATA_JOB, job_input=job_input)
    )
    print(job_output)


if __name__ == "__main__":
    get_dbgap_merged_metadata_manifest()

    # TODO: QA the manifest from the above step, make it available to the next job for
    #       actual ingestion into the metadat service

    metadata_ingest()

```

### Verify Metadata Manifest

How to verify the metadata objects in metadata service against a "source of truth" manifest.

In the example below we assume a manifest named `dbgap-metadata-manifest.tsv` already exists with info of what's expected in the metadata service per guid. The headers in this example `dbgap-metadata-manifest.tsv` are `guid, metadata_field_0, metadata_field_1, ...`.

> NOTE: The metadata fields in the mds will be nested under a metadata *source* name. You must specify the expected source name in the verify function. In this case, we expect all this metadata to exist in a `dbgap` block in the mds.


```python
import sys
import logging
import asyncio

from gen3.tools import metadata

logging.basicConfig(filename="output.log", level=logging.INFO)
logging.getLogger().addHandler(logging.StreamHandler(sys.stdout))

COMMONS = "https://{{insert-commons-here}}/"


def main():
    loop = get_or_create_event_loop_for_thread()


    loop.run_until_complete(
        metadata.async_verify_metadata_manifest(
            COMMONS, manifest_file="dbgap-metadata-manifest.tsv", metadata_source="dbgap"
        )
    )


if __name__ == "__main__":
    main()
```
