# Queries for beginners

A plain-language introduction to writing your own Asset Inventory queries. No previous experience needed. When you want the full detail, the [query language reference](query-language.md) and [provider fields](provider-fields.md) pages have it.

## Where you type a query

1. In the Infoblox Portal, go to **Assets → Inventory**.
2. Above the table there is a filter bar. Click **Advanced Mode** in its bottom row. The button is labelled with the mode it switches *to*, so if it already says **Basic Mode**, you are in Advanced Mode.
3. The page starts with a filter of its own, `Managed IS "True"`. Click **Clear** to remove it, or it will quietly combine with your query.
4. Type your query and click **Apply**.

## Part 1: the basics

### A query is a short sentence

Every query has three parts: **what** to look at, **how** to compare it, and **what to compare it with**.

```
asset.Type IN ["Laptop"]
```

| Part | Here | Meaning |
|---|---|---|
| Field | `asset.Type` | the kind of device |
| Comparison | `IN` | is one of |
| Value | `["Laptop"]` | Laptop |

Read it aloud: *"assets whose type is one of: Laptop."*

### Three writing rules

- **Text goes in double quotes:** `"Laptop"`, not `Laptop`.
- **Lists go in square brackets**, with commas between the items: `["Laptop", "Workstation"]`.
- **Dates are month-day-year:** `10-01-2026` is 1 October 2026.

### The comparisons you will use most

| Write | Meaning | Example |
|---|---|---|
| `IN [...]` | is one of these | `asset.OSType IN ["Windows", "macOS"]` |
| `NOTIN [...]` | is none of these | `asset.Providers NOTIN ["ServiceNow"]` |
| `=` or `IS` | is exactly | `asset.Confidence IS "Low"` |
| `CONTAINS` | has this inside it | `asset.Name CONTAINS "lab"` |
| `AFTERDATE` | later than a date | `asset.LastSeen AFTERDATE 10-01-2026` |
| `BEFOREDATE` | earlier than a date | `asset.LastSeen BEFOREDATE 09-01-2026` |
| `IS "EMPTY"` | has no value at all | `asset.SerialNumber IS "EMPTY"` |

### Combining conditions

- `AND`: both must be true. *Laptops running Windows:*

  ```
  asset.Type IN ["Laptop"] AND asset.OSType IN ["Windows"]
  ```

- `OR`: either one is enough. *Zombie assets, or ones UAI has low confidence in:*

  ```
  asset.Classifications IN ["Zombie"] OR asset.Confidence IN ["Low"]
  ```

- **Brackets** group conditions when you mix the two, just as in arithmetic:

  ```
  asset.Type IN ["Laptop"] AND (asset.OSType IN ["Windows"] OR asset.OSType IN ["macOS"])
  ```

### Starter queries to try

All of these have been run against a live tenant.

| Question | Query |
|---|---|
| Which assets have no DNS record? | `asset.MissingRecords IN ["DNS Forward Record"]` |
| Which assets look abandoned? | `asset.Classifications IN ["Zombie"]` |
| Which assets are unencrypted? | `asset.SubClassifications IN ["Unencrypted"]` |
| What could UAI not identify? | `asset.Type IN ["Unclassified"]` |
| What has no serial number? | `asset.SerialNumber IS "EMPTY"` |
| What has not been seen since September? | `asset.LastSeen BEFOREDATE 09-01-2026` |

The full list of `asset.` fields and their allowed values is in the [query language reference](query-language.md#asset-fields). In the editor, typing `asset.` also brings up suggestions.

### Did it work?

The filter bar never shows an error. A typo simply returns the wrong number of results, often zero. So:

- **Wait a moment before reading the count.** The "Displaying N entries" number updates a little after the table does, and for a second or two it still shows the previous query's figure.
- **If you get zero, check the spelling first,** including capitals and quotes, before concluding there is nothing to find.

## Part 2: provider fields

### Two kinds of information

Every asset in the inventory has two layers of information:

- **The summary.** UAI combines what every source reported into one tidy record: name, type, operating system, IP addresses, last seen. These are the `asset.` fields from Part 1, and they look the same in every tenant.
- **The original reports.** Each source (Azure, AWS, a Meraki controller, ServiceNow, and so on) also keeps its own report exactly as it sent it. These *provider fields* hold far more detail: whether an Azure VM needs a reboot, whether an AWS disk is encrypted, which policies CrowdStrike applied.

Think of the summary as the index card and the provider fields as the original paperwork behind it.

### How to find the data

Start with one asset you know has the information you are after.

1. Go to **Assets → Inventory** and click that asset's row. A panel opens on the right. Its **Overview** tab shows the summary fields.
2. In that panel:
   - **View Details** opens the full asset view. **Raw Attributes** there shows each provider's original report, and **Provider Comparison** puts what each source said side by side.
   - **Export** downloads the same raw attributes as a CSV file, which is handy for searching through a long report.
3. Find the value you care about in the raw attributes, and note every name on the way down to it.

In the editor, autocomplete helps too: type a provider's name and a dot, for example `azure.`, and it suggests what comes next. It only suggests what exists in your tenant, so if nothing appears, that source may not be connected.

To browse without opening an asset, the [provider fields reference](provider-fields.md#reference) lists over a thousand fields seen in a demo tenant, grouped by source.

### Turning what you found into a query

Say an Azure VM's raw attributes contain this:

```
instance_view
  └ patchStatus
      └ availablePatchSummary
          └ rebootPending: false
```

A provider field is written as three pieces joined by dots:

| Piece | Here | Where it comes from |
|---|---|---|
| Provider | `azure` | the source |
| Table | `azure_compute_virtual_machines` | the kind of record (see below) |
| Path | `instance_view.patchStatus.availablePatchSummary.rebootPending` | the names on the way down to the value, in order |

Put together, that gives *"Azure VMs with no reboot pending"*:

```
azure.azure_compute_virtual_machines.instance_view.patchStatus.availablePatchSummary.rebootPending = "false"
```

Common tables:

| Kind of asset | Table |
|---|---|
| Azure virtual machine | `azure.azure_compute_virtual_machines` |
| Azure disk | `azure.azure_compute_disks` |
| AWS EC2 instance | `aws.aws_ec2_instances` |
| AWS EBS volume | `aws.aws_ec2_ebs_volumes` |
| GCP VM instance | `gcp.gcp_compute_instances` |
| Meraki network device | `meraki.meraki_organization_devices` |
| ServiceNow computer | `servicenow.servicenow_computers` |
| CrowdStrike device | `crowdstrike_falcon.crowdstrike_falcon_devices` |
| Intune device | `intune.intune_managed_devices` |
| Jamf computer | `jamf.jamf_computers_inventory` |

The [reference](provider-fields.md#reference) has the rest.

### Four rules for provider fields

1. **Put quotes around everything, even `true`, `false` and numbers:** `= "false"`, `= "0"`.
2. **Copy the capitals exactly.** `rebootPending` and `rebootpending` are different fields.
3. **Many fields exist on only some assets.** Azure only reports `patchStatus` once Update Manager has checked a VM. An asset without the field never matches, so a small result may be correct.
4. **Skip anything inside a list for now.** The reference marks list items with `[]`, as in `statuses[].displayStatus`. How to filter on those has not been worked out yet.

### Provider examples to try

Built from fields seen in a demo tenant. They follow the rules above but have not yet been run in the filter bar, so check the count makes sense.

| Question | Query |
|---|---|
| Azure VMs waiting for a reboot | `azure.azure_compute_virtual_machines.instance_view.patchStatus.availablePatchSummary.rebootPending = "true"` |
| Unencrypted AWS disks | `aws.aws_ec2_ebs_volumes.encrypted = "false"` |
| Azure disks not attached to anything | `azure.azure_compute_disks.properties.diskState = "Unattached"` |
| Stopped GCP VMs | `gcp.gcp_compute_instances.status = "TERMINATED"` |

You can mix provider fields with `asset.` fields freely:

```
azure.azure_compute_disks.properties.diskState = "Unattached" AND asset.LastSeen AFTERDATE 10-01-2026
```

## Next steps

- [Query language reference](query-language.md): every operator, field and value, and the quirks of the filter bar.
- [Provider fields](provider-fields.md): the detail behind Part 2, and the full field reference.
- [Contributing](../CONTRIBUTING.md): how to add a query you have written to this library.
