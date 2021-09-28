---
title: Troubleshoot issues writing data
seotitle: Troubleshoot issues writing data
list_title: Troubleshoot issues writing data
weight: 105
description: >
  Troubleshoot issues writing data. Discover how writes fail, including rate limit failures, timeouts, size of write payload, not conforming to an explicit bucket schema, and partial writes. Find response codes for failed writes.
menu:
  influxdb_cloud:
    name: Troubleshoot issues
    parent: Write data
nfluxdb/cloud/tags: [write, line protocol, errors]
related:
  - /influxdb/cloud/api/#tag/Write, InfluxDB API /write endpoint
  - /influxdb/cloud/reference/internals
  - /influxdb/cloud/reference/cli/influx/write
---

Learn how to handle and recover from errors when writing to InfluxDB.

- [Discover common failure scenarios](#common-failure-scenarios)
- [HTTP status codes](#http-status-codes)
- [Troubleshoot failures](#troubleshoot-failures)

## Common failure scenarios

InfluxDB write requests may fail for a number of reasons.
Common failure scenarios that return an HTTP `4xx` or `5xx` error status code include the following:

- Exceeded a rate limit.
- API token was invalid.
- Client or server reached a timeout threshold.
- Data was not formatted correctly.
- Data did not conform to the [explicit bucket schema](/influxdb/cloud/organizations/buckets/bucket-schema/).
  See how to troubleshoot specific [bucket schema errors](/influxdb/cloud/organizations/buckets/bucket-schema/#troubleshoot-errors).
- Size of the data payload was too large.

Writes may fail partially or completely even though InfluxDB returns an HTTP `2xx` status code for a valid request.
For example, a partial write may occur when InfluxDB writes points that conform to the schema of existing data, but rejects points that have the wrong data type in a field.

## HTTP status codes

InfluxDB uses conventional HTTP status codes to indicate the success or failure of a request.
Write requests return the following status codes:

- `204` **Success**: InfluxDB validated the request data format and accepted the data for writing to the bucket.
    {{% note %}}
`204` doesn't indicate a successful write operation since writes are asynchronous. If some of your data did not write to the bucket, see how to [check for rejected points](#review-rejected-points).
    {{% /note %}}

- `400` **Bad request**: The line protocol data in the request was malformed. The response body contains the first malformed line in the data. All request data is rejected and not written.
- `401` **Unauthorized**: May indicate one of the following:
  - `Authorization: Token` header is missing or malformed.
  - API token value is missing from the header.
  - API token does not have sufficient permissions to write to the organization and bucket.
- `404` **Not found**: A requested resource (e.g. an organization or bucket) was not found. The response body contains the requested resource type, e.g. "organization", and resource name.
- `413` **Request entity too large**: The payload exceeded the 50MB limit. All request data was rejected and not written.
- `429` **Too many requests**: API token is temporarily over quota. The `Retry-After` header describes when to try the write request again.
- `500` **Internal server error**: Default HTTP status for an error.
- `503` **Service unavailable**: Server is temporarily unavailable to accept writes. The `Retry-After` header describes when to try the write again.

The `message` property of the response body may contain additional details about the error.

## Troubleshoot failures

If you notice data is missing in your bucket, check the following:

- Does the `message` property in the response body contain details about the error?
- Do all lines contain valid syntax, e.g. [line protocol](/influxdb/cloud/reference/syntax/line-protocol/) or [CSV](/influxdb/cloud/reference/syntax/annotated-csv/)?
- Do the data types match the [bucket schema](/influxdb/cloud/organizations/buckets/bucket-schema/)?
  For example, did you attempt to write `string` data to an `int` field?
- Do the timestamps match the precision parameter?

### Review rejected points

InfluxDB may have rejected points even if the HTTP request returned "Success".
To get a log of rejected data points, query the [`rejected_points` measurement](/influxdb/cloud/reference/internals/system-buckets/#_monitoring-bucket-schema) in your organization's `_monitoring` bucket.

```js
from(bucket: "_monitoring")
  |> range(start:-1h)
  |> filter(fn:(r) =>
    r._measurement == "rejected_points"
  )
```

InfluxDB returns `rejected_points` log entries.
Each entry contains the `bucket` where the rejection occurred and a `reason` for the rejection.

```sh
rejected_points,bucket=YOUR_BUCKET,field=YOUR_FIELD,gotType=Float,
  measurement=YOUR_MEASUREMENT,reason=type\ conflict\ in\ batch\ write,
  wantType=Integer count=1i 1627906197091972750
```

#### Troubleshoot field type conflicts

Field type conflicts occur if you attempt to write different data types to a field.
For example, you attempt to write `string` data to an `int` field.

{{% note %}}
If you want to reject batches that contain points with field type conflicts, use an [explicit bucket schema](/influxdb/cloud/organizations/buckets/bucket-schema/). Explicit bucket schemas reject the write request and return an HTTP `400 Bad Request` status code and a JSON response body like the following:
```json
{
  "code": "invalid",
  "message": "partial write error (2 written): unable to parse 'air_sensor,service=S1,sensor=L1 temperature=\"90.5\",humidity=70.0 1632850122': schema: field type for field \"temperature\" not permitted by schema; got String but expected Float"
}
```
{{% /note %}}

If InfluxDB rejects a point due to a [field type](/influxdb/cloud/reference/key-concepts/data-elements/#field-value) conflict, the rejected point log entry contains the following data elements:

| Name          | Value                                                                                                                                        |
|:------        |:-----                                                                                                                                        |
| `bucket`      | ID of the bucket that rejected the point.                                                                                                    |
| `measurement` | Measurement name of the point.                                                                                                               |
| `field`       | Name of the field that caused the rejection.                                                                                                 |
| `reason`      | Brief description of the problem. See [reasons for rejected points](#reasons-for-rejected-points)                                            |
| `gotType`     | Received [field](/influxdb/cloud/reference/key-concepts/data-elements/#field-value) type: `Boolean`, `Float`, `Integer`, or `UnsignedInteger` |
| `wantType`    | Expected [field](/influxdb/cloud/reference/key-concepts/data-elements/#field-value) type: `Boolean`, `Float`, `Integer`, or `UnsignedInteger` |
| `error`       | Additional error detail.                                                                                                                     |
| `count`       | `1`                                                                                                                                          |
| `<timestamp>` | Time the rejected point was logged.                                                                                                          |

##### Reasons for rejected points

Field type conflicts occur for the following reasons:

| Reason                             | Meaning                                                                                                       |
|:------                             |:-------                                                                                                       |
| `type conflict in batch write`     | The **batch** contains another point with the same series, but one of the fields has a different value type.  |
| `type conflict with existing data` | The **bucket** contains another point with the same series, but one of the fields has a different value type. |
