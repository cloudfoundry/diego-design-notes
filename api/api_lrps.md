# Long Running Processes API Reference

This reference does not cover the JSON payload supplied to each endpoint.  That is documented in detail in [Understanding LRPs](lrps.md).

We recommend using the [Receptor http client](https://github.com/cloudfoundry-incubator/receptor) to communicate with Diego's API.  The methods on the client are self-explanatory.

## Creating DesiredLRPs

To create a DesiredLRP submit a valid [`DesiredLRPCreateRequest`](lrps.md#describing-desiredlrps) via:

```
POST /v1/desired_lrps
```

Diego responds by spinning up ActualLRPs

## Modifying DesiredLRPs

To modify an existing DesiredLRP, submit a valid [`DesiredLRPUpdateRequest`](lrps.md#updating-desiredlrps) via:

```
PUT /v1/desired_lrps/:process_guid
```

Diego responds by immediately taking actions to attain consistency between ActualLRPs and DesiredLRPs.

> Alternatively, you can `POST` an updated `DesiredLRPCreateRequest`, though it is an error to modify fields that are not in `DesiredLRPUpdateRequest`.

## Deleting DesiredLRPs

To delete an existing DesiredLRP (thereby shutting down all associated ActualLRPs):

```
DELETE /v1/desired_lrps/:process_guid
```

## Fetching DesiredLRPs

### Fetching all DesiredLRPs

To fetch *all* DesiredLRPs:

```
GET /v1/desired_lrps
```

This returns an array of [`DesiredLRPResponse`](lrps.md#fetching-desiredlrps) objects


### Fetching DesiredLRPs by Domain

To fetch all DesiredLRPs in a given [`domain`](lrps.md#domain):

```
GET /v1/domain/:domain/desired_lrps
```

This returns an array of [`DesiredLRPResponse`](lrps.md#fetching-desiredlrps) objects

### Fetching a Specific DesiredLRP

To fetch a DesiredLRP by [`process_guid`](lrps.md#process-guid):

```
GET /v1/desired_lrps/:process_guid
```

This returns a single [`DesiredLRPResponse`](lrps.md#fetching-desiredlrps) object or `404` if none is found.

## Fetching ActualLRPs

### Fetching all ActualLRPs

To fecth all ActualLRPs:

```
GET /v1/actual_lrps
```

This returns an array of [`ActualLRPResponse`](lrps.md#fetching-actuallrps) response objects.

### Fetching all ActualLRPs in a Domain

To fetch all ActualLRPs in a given domain:

```
GET /v1/domains/:domain/actual_lrps
```
This returns an array of [`ActualLRPResponse`](lrps.md#fetching-actuallrps) response objects.

### Fetching all ActualLRPs for a given `process_guid`

To fetch all ActualLRPs associated with a given DesiredLRP (by `process_guid`):

```
GET /v1/desired_lrps/:process_guid/actual_lrps
```

This returns an array of [`ActualLRPResponse`](lrps.md#fetching-actuallrps) response objects.

To fetch all `actual_lrps` at a *particular index*:

```
GET /v1/desired_lrps/:process_guid/actual_lrps?index=N
```

This returns an array of [`ActualLRPResponse`](lrps.md#fetching-actuallrps) response objects.  Note that this is an *array* of `ActualLRPResponse`: it is possible for multiple instances to inhabit the same index (though this should only ever be a transient state).

## Killing ActualLRPs

Diego supports killing the ActualLRPs for a given `process_guid` at a given `index`.  This will shut down the specific ActualLRPs but does not modify the desired state - thus the missing instance will automatically restart eventually.

```
DELETE /v1/desired_lrps/:process_guid/actual_lrps?index=N
```

## Freshness

As described [here](lrps.md#freshness) Diego must be told when desired state is up to date before it wil take potentially destructive actions.

### Bumping Freshness

To mark a domain as fresh:

Post a `FreshDomainBumpRequest` of the form:

```
{
    "domain": "the-domain-to-bump",
    "ttl_in_seconds": N
}
```

to:

```
POST /v1/fresh_domains
```

You must repeat the post before the `ttl` expires.  To opt out of this, you may specify a `ttl` of 0.

### Fetching all Fresh Domains

To fetch all fresh domains:

```
GET /v1/fresh_domains
```

This returns an array of `FreshDomainResponse` objects (which are identical to `FreshDomainBumpRequest` objects).

[back](README.md)
