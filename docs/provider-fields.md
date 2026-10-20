# Provider fields

Asset Inventory exposes two layers of data, and Advanced Mode can filter on both:

| Layer | Prefix | What it is |
|---|---|---|
| Normalised | `asset.` | One reconciled record per asset, the same shape in every tenant. Covered in [the query language reference](query-language.md#asset-fields). |
| Provider | `<namespace>.<source_table>.` | The record each integration reported, kept verbatim as JSON. Hundreds of fields, different for every provider. |

This page covers the provider layer: how a field path is built, how to find out which fields exist, how to write filters against them, and a reference of every field seen in one demo tenant.

Like the rest of `docs/`, none of this is official Infoblox documentation. It comes from inspecting live tenants.

## How a path is built

A provider field is addressed as

```
<namespace>.<source_table>.<path inside the provider's JSON>
```

where the JSON path joins nested object keys with dots, keeping each key exactly as the provider spells it, case included. Take this fragment of an Azure VM record:

```json
{
  "instance_view": {
    "patchStatus": {
      "availablePatchSummary": {
        "criticalAndSecurityPatchCount": 4,
        "rebootPending": false,
        "status": "Succeeded"
      }
    }
  }
}
```

It lives in the source table `azure_compute_virtual_machines`, so `rebootPending` is

```
azure.azure_compute_virtual_machines.instance_view.patchStatus.availablePatchSummary.rebootPending
```

The namespace is the provider's own prefix on the table name. The catalog's counted queries confirm the rule for four tables:

| Namespace | Source table | Catalog query |
|---|---|---|
| `servicenow` | `servicenow_computers` | `retired-in-cmdb-still-on-network` |
| `crowdstrike_falcon` | `crowdstrike_falcon_devices` | `crowdstrike-firewall-and-prevention-off` |
| `intune` | `intune_managed_devices` | `laptops-disk-encryption-off` |
| `jamf` | `jamf_computers_inventory` | `laptops-disk-encryption-off` |

The namespaces in the [reference](#reference) below (`aws`, `azure`, `gcp`, `meraki`, `bep`, `dhcp`, `ngc`, `ni`) follow the same rule but have not been run in the filter bar. That matters most for the Infoblox-internal tables (`bep_devices`, `dhcp_devices`, `ngc_devices`, `ni_devices`), whose namespace is the least obvious. Type the namespace and a dot, and check that autocomplete offers the table, before relying on one.

## Finding fields

**Autocomplete.** The editor offers namespaces, tables and keys that exist in the tenant you are signed in to. It is the quickest check that a path exists, but it only covers what that tenant has, and on some tenants it offers nothing below the root (see [What differs between tenants](query-language.md#what-differs-between-tenants-and-what-does-not)).

**The raw record.** Every provider record is stored as a JSON string in the Cube dimension `pd_raw_json` of the `Asset_Insight_ProviderData_Expanded` cube, alongside `pd_provider` and `pd_source_table`. This cube is reachable through the Infoblox MCP server or the Cube API. To find which assets carry a given key, and in which table, filter on the key name:

```json
{
  "dimensions": [
    "Asset_Insight_ProviderData_Expanded.name",
    "Asset_Insight_ProviderData_Expanded.pd_source_table"
  ],
  "filters": [
    { "member": "Asset_Insight_ProviderData_Expanded.pd_raw_json",
      "operator": "contains", "values": ["rebootPending"] }
  ]
}
```

Then pull `pd_raw_json` for one of those assets and read the path off the JSON.

<details>
<summary>Listing every path in a tenant</summary>

The reference below was produced by fetching `cqid`, `pd_source_table` and `pd_raw_json` for every asset, then walking the JSON in DuckDB:

```sql
WITH RECURSIVE t(src, cqid, path, j, depth) AS (
  SELECT pd_source_table, cqid, '', CAST(pd_raw_json AS JSON), 0 FROM raw
  UNION ALL
  SELECT src, cqid,
         CASE WHEN json_type(j) = 'OBJECT'
              THEN path || CASE WHEN path = '' THEN '' ELSE '.' END || k
              ELSE path || '[]' END,
         CASE WHEN json_type(j) = 'OBJECT'
              THEN json_extract(j, '$."' || k || '"')
              ELSE json_extract(j, '$[' || k || ']') END,
         depth + 1
  FROM (
    SELECT src, cqid, path, j, depth, UNNEST(json_keys(j)) AS k
      FROM t WHERE json_type(j) = 'OBJECT' AND depth < 8
    UNION ALL
    SELECT src, cqid, path, j, depth,
           CAST(UNNEST(range(CAST(LEAST(json_array_length(j), 3) AS BIGINT))) AS VARCHAR)
      FROM t WHERE json_type(j) = 'ARRAY' AND depth < 8
  )
)
SELECT src, path, json_type(j) AS type, COUNT(DISTINCT cqid) AS assets
FROM t
WHERE path <> '' AND json_type(j) NOT IN ('OBJECT', 'ARRAY')
GROUP BY ALL ORDER BY src, path;
```

Paths whose keys are data rather than schema (the IP-keyed `ip_ipspace_id` map, Azure's resource-ID-keyed `userAssignedIdentities`) come out once per key and need collapsing by hand.

</details>

## Using them in filters

**Quote every value, including booleans and numbers.** Provider values are compared as strings. The counted CrowdStrike query matches a JSON boolean with `applied != "true"`, and the same form applies to `rebootPending`:

```
azure.azure_compute_virtual_machines.instance_view.patchStatus.availablePatchSummary.rebootPending = "false"
```

**Numbers support equality only.** The language has no greater-than or less-than operators, so `criticalAndSecurityPatchCount = "0"` is expressible and "more than 5" is not.

**Operators.** `=` and `!=` are the ones proven on provider fields. The string operators from the [operator table](query-language.md#operators) (`IS`, `CONTAINS`, `IN` and the rest) are presumed to work, but no counted query uses them on a provider field yet.

**A missing field is not `false`.** Many fields appear on only some records: `patchStatus` is present only once Azure Update Manager has assessed the VM, which in the sampled tenant was 1 of 86 VMs. Check the coverage column in the reference before building a query on a sparse field. Whether `!=` matches records that lack the field entirely is untested.

**Arrays are untested.** Paths shown with `[]` below (`instance_view.statuses[].displayStatus`, for example) come from JSON arrays. How `filterel` addresses array elements, if it does at all, has not been tested.

**Some tables carry two schema generations.** `dhcp_devices` and `ngc_devices` hold a mix of PascalCase records (`Os`, `FirstSeen`, `Interfaces`) and snake_case ones (`os`, `first_seen`, `interfaces`), and `meraki_network_clients` has both `OS` and `Os`. Keys are case-sensitive, so cover both spellings with `OR`.

**Tags are tenant-defined.** Every `tags` map below holds whatever keys the customer uses, addressed as `tags.<Key>`. Look the keys up in the tenant rather than assuming them.

**Combine freely with `asset.` fields:**

```
azure.azure_compute_virtual_machines.instance_view.patchStatus.availablePatchSummary.rebootPending = "true" AND asset.LastSeen AFTERDATE 10-01-2026
```

## Reference

Generated on 2026-10-20 from the `pd_raw_json` of every asset in one demo tenant: 9 providers, 26 source tables, 1,072 field paths. Other tenants will have other integrations (the ServiceNow, CrowdStrike, Intune and Jamf tables above are not in it) and other fields, so treat this as a map of what exists rather than a guarantee.

- **Coverage** is the share of that table's assets whose record carries the field, in the sampled tenant.
- **Example values** are shown only for generic enumerations such as states, modes and SKUs. Names, identifiers, addresses, serials and timestamps never are, because they would identify the tenant.
- Paths that were `null` on every record are left out.
- `<ip>`, `<resource-id>` and `<key>` mark object keys that are data, not field names.

- **AWS**: [`aws_ec2_customer_gateways`](#awsaws_ec2_customer_gateways), [`aws_ec2_ebs_volumes`](#awsaws_ec2_ebs_volumes), [`aws_ec2_eips`](#awsaws_ec2_eips), [`aws_ec2_instances`](#awsaws_ec2_instances), [`aws_ec2_internet_gateways`](#awsaws_ec2_internet_gateways), [`aws_ec2_nat_gateways`](#awsaws_ec2_nat_gateways), [`aws_ec2_security_groups`](#awsaws_ec2_security_groups), [`aws_ec2_vpn_gateways`](#awsaws_ec2_vpn_gateways), [`aws_s3_buckets`](#awsaws_s3_buckets)
- **Azure**: [`azure_compute_disks`](#azureazure_compute_disks), [`azure_compute_virtual_machines`](#azureazure_compute_virtual_machines), [`azure_network_public_ip_addresses`](#azureazure_network_public_ip_addresses), [`azure_network_security_groups`](#azureazure_network_security_groups), [`azure_network_virtual_network_gateways`](#azureazure_network_virtual_network_gateways), [`azure_storage_containers`](#azureazure_storage_containers)
- **GCP**: [`gcp_compute_addresses`](#gcpgcp_compute_addresses), [`gcp_compute_disks`](#gcpgcp_compute_disks), [`gcp_compute_firewalls`](#gcpgcp_compute_firewalls), [`gcp_compute_instances`](#gcpgcp_compute_instances), [`gcp_storage_buckets`](#gcpgcp_storage_buckets)
- **Cisco Meraki**: [`meraki_network_clients`](#merakimeraki_network_clients), [`meraki_organization_devices`](#merakimeraki_organization_devices)
- **Infoblox Endpoint**: [`bep_devices`](#bepbep_devices)
- **DHCP Logs**: [`dhcp_devices`](#dhcpdhcp_devices)
- **NIOS Network Insight**: [`ngc_devices`](#ngcngc_devices)
- **Network Insight**: [`ni_devices`](#nini_devices)

### `aws.aws_ec2_customer_gateways`

Provider **AWS**, 5 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `account_id` | string | 100% |  |
| `arn` | string | 100% |  |
| `bgp_asn` | string | 100% |  |
| `customer_gateway_id` | string | 100% |  |
| `device_name` | string | 100% |  |
| `ip_address` | string | 100% |  |
| `region` | string | 100% |  |
| `state` | string | 100% | `"available"` |
| `tags` | key/value map | 100% |  |
| `type` | string | 100% | `"ipsec.1"` |

### `aws.aws_ec2_ebs_volumes`

Provider **AWS**, 45 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `account_id` | string | 100% |  |
| `arn` | string | 100% |  |
| `attachments[].AttachTime` | string | 89% |  |
| `attachments[].DeleteOnTermination` | boolean | 89% | `true` |
| `attachments[].Device` | string | 89% |  |
| `attachments[].InstanceId` | string | 89% |  |
| `attachments[].State` | string | 89% | `"attached"` |
| `attachments[].VolumeId` | string | 89% |  |
| `availability_zone` | string | 100% |  |
| `availability_zone_id` | string | 100% |  |
| `create_time` | string | 100% |  |
| `encrypted` | boolean | 100% | `false`, `true` |
| `iops` | string | 100% |  |
| `kms_key_id` | string | 47% |  |
| `multi_attach_enabled` | boolean | 100% | `false` |
| `operator.Managed` | boolean | 100% | `false` |
| `region` | string | 100% |  |
| `size` | string | 100% |  |
| `snapshot_id` | string | 100% |  |
| `sse_type` | string | 100% |  |
| `state` | string | 100% | `"available"`, `"in-use"` |
| `tags` | key/value map | 100% |  |
| `throughput` | string | 89% |  |
| `volume_id` | string | 100% |  |
| `volume_type` | string | 100% | `"gp2"`, `"gp3"` |

### `aws.aws_ec2_eips`

Provider **AWS**, 12 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `account_id` | string | 100% |  |
| `allocation_id` | string | 100% |  |
| `association_id` | string | 83% |  |
| `domain` | string | 100% |  |
| `instance_id` | string | 42% |  |
| `network_border_group` | string | 100% |  |
| `network_interface_id` | string | 58% |  |
| `network_interface_owner_id` | string | 58% |  |
| `private_ip_address` | string | 83% |  |
| `public_ip` | string | 100% |  |
| `public_ipv4_pool` | string | 100% |  |
| `region` | string | 100% |  |
| `service_managed` | string | 100% | `"rnat"` |
| `tags` | key/value map | 100% |  |

### `aws.aws_ec2_instances`

Provider **AWS**, 39 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `ami_launch_index` | number | 100% |  |
| `architecture` | string | 100% | `"x86_64"` |
| `arn` | string | 100% |  |
| `block_device_mappings[].DeviceName` | string | 100% |  |
| `block_device_mappings[].Ebs.AttachTime` | string | 100% |  |
| `block_device_mappings[].Ebs.DeleteOnTermination` | boolean | 100% | `true` |
| `block_device_mappings[].Ebs.Status` | string | 100% | `"attached"` |
| `block_device_mappings[].Ebs.VolumeId` | string | 100% |  |
| `boot_mode` | string | 100% | `"uefi"`, `"uefi-preferred"` |
| `capacity_reservation_specification.CapacityReservationPreference` | string | 100% |  |
| `client_token` | string | 100% |  |
| `cpu_options.AmdSevSnp` | string | 100% |  |
| `cpu_options.CoreCount` | number | 100% |  |
| `cpu_options.ThreadsPerCore` | number | 100% |  |
| `current_instance_boot_mode` | string | 100% | `"legacy-bios"`, `"uefi"` |
| `ebs_optimized` | boolean | 100% | `false`, `true` |
| `ena_support` | boolean | 100% | `true` |
| `enclave_options.Enabled` | boolean | 100% | `false` |
| `hibernation_options.Configured` | boolean | 100% | `false` |
| `hypervisor` | string | 100% | `"xen"` |
| `iam_instance_profile.Arn` | string | 3% |  |
| `iam_instance_profile.Id` | string | 3% |  |
| `image_id` | string | 100% |  |
| `instance_id` | string | 100% |  |
| `instance_lifecycle` | string | 100% |  |
| `instance_type` | string | 100% |  |
| `key_name` | string | 100% |  |
| `launch_time.seconds` | number | 100% |  |
| `maintenance_options.AutoRecovery` | string | 100% | `"default"` |
| `maintenance_options.RebootMigration` | string | 100% | `"default"` |
| `metadata_options.HttpEndpoint` | string | 100% | `"enabled"` |
| `metadata_options.HttpProtocolIpv6` | string | 100% |  |
| `metadata_options.HttpPutResponseHopLimit` | number | 100% |  |
| `metadata_options.HttpTokens` | string | 100% | `"optional"`, `"required"` |
| `metadata_options.InstanceMetadataTags` | string | 100% |  |
| `metadata_options.State` | string | 100% | `"applied"` |
| `model` | string | 100% |  |
| `monitoring.State` | string | 100% | `"disabled"` |
| `network_interfaces[].Association.IpOwnerId` | string | 13% |  |
| `network_interfaces[].Association.PublicDnsName` | string | 13% |  |
| `network_interfaces[].Association.PublicIp` | string | 13% |  |
| `network_interfaces[].Attachment.AttachTime` | string | 100% |  |
| `network_interfaces[].Attachment.AttachmentId` | string | 100% |  |
| `network_interfaces[].Attachment.DeleteOnTermination` | boolean | 100% | `true` |
| `network_interfaces[].Attachment.DeviceIndex` | number | 100% |  |
| `network_interfaces[].Attachment.NetworkCardIndex` | number | 100% |  |
| `network_interfaces[].Attachment.Status` | string | 100% | `"attached"` |
| `network_interfaces[].Description` | string | 100% |  |
| `network_interfaces[].Groups[].GroupId` | string | 100% |  |
| `network_interfaces[].Groups[].GroupName` | string | 100% |  |
| `network_interfaces[].InterfaceType` | string | 100% | `"interface"` |
| `network_interfaces[].MacAddress` | string | 100% |  |
| `network_interfaces[].NetworkInterfaceId` | string | 100% |  |
| `network_interfaces[].Operator.Managed` | boolean | 100% | `false` |
| `network_interfaces[].OwnerId` | string | 100% |  |
| `network_interfaces[].PrivateDnsName` | string | 100% |  |
| `network_interfaces[].PrivateIpAddress` | string | 100% |  |
| `network_interfaces[].PrivateIpAddresses[].Association.IpOwnerId` | string | 13% |  |
| `network_interfaces[].PrivateIpAddresses[].Association.PublicDnsName` | string | 13% |  |
| `network_interfaces[].PrivateIpAddresses[].Association.PublicIp` | string | 13% |  |
| `network_interfaces[].PrivateIpAddresses[].Primary` | boolean | 100% | `true` |
| `network_interfaces[].PrivateIpAddresses[].PrivateDnsName` | string | 100% |  |
| `network_interfaces[].PrivateIpAddresses[].PrivateIpAddress` | string | 100% |  |
| `network_interfaces[].SourceDestCheck` | boolean | 100% | `true` |
| `network_interfaces[].Status` | string | 100% | `"in-use"` |
| `network_interfaces[].SubnetId` | string | 100% |  |
| `network_interfaces[].VpcId` | string | 100% |  |
| `os` | string | 100% | `"Linux/UNIX"`, `"Windows"` |
| `placement.AvailabilityZone` | string | 100% |  |
| `placement.AvailabilityZoneId` | string | 100% |  |
| `placement.GroupName` | string | 100% |  |
| `placement.Tenancy` | string | 100% | `"default"` |
| `platform` | string | 100% | `"windows"` |
| `platform_details` | string | 100% | `"Linux/UNIX"`, `"Windows"` |
| `private_dns_name` | string | 100% |  |
| `private_dns_name_options.EnableResourceNameDnsAAAARecord` | boolean | 100% | `false` |
| `private_dns_name_options.EnableResourceNameDnsARecord` | boolean | 100% | `false`, `true` |
| `private_dns_name_options.HostnameType` | string | 100% |  |
| `private_ip_address` | string | 100% |  |
| `product_codes[].ProductCodeId` | string | 44% |  |
| `product_codes[].ProductCodeType` | string | 44% | `"marketplace"` |
| `public_dns_name` | string | 100% |  |
| `public_ip_address` | string | 100% |  |
| `region` | string | 100% |  |
| `root_device_name` | string | 100% |  |
| `root_device_type` | string | 100% | `"ebs"` |
| `security_groups[].GroupId` | string | 100% |  |
| `security_groups[].GroupName` | string | 100% |  |
| `signature.dhcp_fingerprint` | string | 100% |  |
| `source_dest_check` | boolean | 100% | `true` |
| `state.Code` | number | 100% |  |
| `state.Name` | string | 100% |  |
| `state_reason.Code` | string | 100% |  |
| `state_reason.Message` | string | 100% |  |
| `state_transition_reason` | string | 100% |  |
| `state_transition_reason_time.seconds` | number | 87% |  |
| `subnet_id` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `types.aws.source_assigned` | string | 100% | `"virtual_machine"` |
| `updated_at` | string | 100% |  |
| `usage_operation` | string | 100% |  |
| `usage_operation_update_time.seconds` | number | 100% |  |
| `vendor` | string | 100% | `"aws"` |
| `virtualization_type` | string | 100% | `"hvm"` |
| `vpc_id` | string | 100% |  |

### `aws.aws_ec2_internet_gateways`

Provider **AWS**, 143 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `account_id` | string | 100% |  |
| `arn` | string | 100% |  |
| `attachments[].State` | string | 96% | `"available"` |
| `attachments[].VpcId` | string | 96% |  |
| `internet_gateway_id` | string | 100% |  |
| `owner_id` | string | 100% |  |
| `region` | string | 100% |  |
| `tags` | key/value map | 100% |  |

### `aws.aws_ec2_nat_gateways`

Provider **AWS**, 3 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `account_id` | string | 100% |  |
| `arn` | string | 100% |  |
| `connectivity_type` | string | 100% | `"public"` |
| `create_time` | string | 100% |  |
| `nat_gateway_addresses[].AllocationId` | string | 100% |  |
| `nat_gateway_addresses[].AssociationId` | string | 100% |  |
| `nat_gateway_addresses[].IsPrimary` | boolean | 100% | `true` |
| `nat_gateway_addresses[].NetworkInterfaceId` | string | 100% |  |
| `nat_gateway_addresses[].PrivateIp` | string | 100% |  |
| `nat_gateway_addresses[].PublicIp` | string | 100% |  |
| `nat_gateway_addresses[].Status` | string | 100% | `"succeeded"` |
| `nat_gateway_id` | string | 100% |  |
| `region` | string | 100% |  |
| `state` | string | 100% | `"available"` |
| `subnet_id` | string | 33% |  |
| `tags` | key/value map | 100% |  |
| `vpc_id` | string | 100% |  |

### `aws.aws_ec2_security_groups`

Provider **AWS**, 266 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `account_id` | string | 100% |  |
| `arn` | string | 100% |  |
| `description` | string | 100% |  |
| `group_id` | string | 100% |  |
| `group_name` | string | 100% |  |
| `ip_permissions[].FromPort` | number | 100% |  |
| `ip_permissions[].IpProtocol` | string | 100% |  |
| `ip_permissions[].IpRanges[].CidrIp` | string | 43% |  |
| `ip_permissions[].IpRanges[].Description` | string | 43% |  |
| `ip_permissions[].ToPort` | number | 100% |  |
| `ip_permissions[].UserIdGroupPairs[].Description` | string | 57% |  |
| `ip_permissions[].UserIdGroupPairs[].GroupId` | string | 57% |  |
| `ip_permissions[].UserIdGroupPairs[].UserId` | string | 57% |  |
| `ip_permissions_egress[].FromPort` | number | 100% |  |
| `ip_permissions_egress[].IpProtocol` | string | 100% |  |
| `ip_permissions_egress[].IpRanges[].CidrIp` | string | 100% |  |
| `ip_permissions_egress[].IpRanges[].Description` | string | 100% |  |
| `ip_permissions_egress[].Ipv6Ranges[].CidrIpv6` | string | <1% |  |
| `ip_permissions_egress[].ToPort` | number | 100% |  |
| `ip_permissions_egress[].UserIdGroupPairs[].Description` | string | <1% |  |
| `ip_permissions_egress[].UserIdGroupPairs[].GroupId` | string | <1% |  |
| `ip_permissions_egress[].UserIdGroupPairs[].UserId` | string | <1% |  |
| `owner_id` | string | 100% |  |
| `region` | string | 100% |  |
| `security_group_arn` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `vpc_id` | string | 100% |  |

### `aws.aws_ec2_vpn_gateways`

Provider **AWS**, 5 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `account_id` | string | 100% |  |
| `amazon_side_asn` | string | 100% |  |
| `arn` | string | 100% |  |
| `region` | string | 100% |  |
| `state` | string | 100% | `"available"` |
| `tags` | key/value map | 100% |  |
| `type` | string | 100% | `"ipsec.1"` |
| `vpc_attachments[].State` | string | 60% | `"attached"` |
| `vpc_attachments[].VpcId` | string | 60% |  |
| `vpn_gateway_id` | string | 100% |  |

### `aws.aws_s3_buckets`

Provider **AWS**, 30 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `arn` | string | 100% |  |
| `creation_date.seconds` | number | 100% |  |
| `name` | string | 100% |  |
| `policy_status.IsPublic` | boolean | 60% | `false` |
| `region` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `updated_at` | string | 100% |  |

### `azure.azure_compute_disks`

Provider **Azure**, 90 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `id` | string | 100% |  |
| `location` | string | 100% |  |
| `managed_by` | string | 96% |  |
| `name` | string | 100% |  |
| `properties.creationData.createOption` | string | 100% | `"FromImage"`, `"Import"` |
| `properties.creationData.imageReference.id` | string | 94% |  |
| `properties.creationData.sourceUri` | string | 6% |  |
| `properties.creationData.storageAccountId` | string | 6% |  |
| `properties.diskIOPSReadWrite` | number | 100% |  |
| `properties.diskMBpsReadWrite` | number | 100% |  |
| `properties.diskSizeBytes` | number | 100% |  |
| `properties.diskSizeGB` | number | 100% |  |
| `properties.diskState` | string | 100% | `"Attached"`, `"Reserved"`, `"Unattached"` |
| `properties.encryption.type` | string | 100% | `"EncryptionAtRestWithPlatformKey"` |
| `properties.hyperVGeneration` | string | 94% | `"V1"`, `"V2"` |
| `properties.networkAccessPolicy` | string | 100% |  |
| `properties.osType` | string | 100% | `"Linux"`, `"Windows"` |
| `properties.provisioningState` | string | 100% | `"Succeeded"` |
| `properties.publicNetworkAccess` | string | 100% |  |
| `properties.purchasePlan.name` | string | 13% |  |
| `properties.purchasePlan.product` | string | 13% |  |
| `properties.purchasePlan.publisher` | string | 13% | `"infoblox"` |
| `properties.securityProfile.securityType` | string | 13% |  |
| `properties.supportedCapabilities.acceleratedNetwork` | boolean | 94% | `false`, `true` |
| `properties.supportedCapabilities.architecture` | string | 94% | `"x64"` |
| `properties.supportedCapabilities.diskControllerTypes` | string | 86% |  |
| `properties.supportsHibernation` | boolean | 94% | `false`, `true` |
| `properties.tier` | string | 91% | `"P10"`, `"P15"`, `"P20"`, `"P4"`, `"P6"` |
| `properties.timeCreated` | string | 100% |  |
| `properties.uniqueId` | string | 100% |  |
| `sku.name` | string | 100% |  |
| `sku.tier` | string | 100% | `"Premium"`, `"Standard"` |
| `subscription_id` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `type` | string | 100% | `"Microsoft.Compute/disks"` |
| `zones[]` | string | 78% |  |

### `azure.azure_compute_virtual_machines`

Provider **Azure**, 86 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `id` | string | 100% |  |
| `identity.principalId` | string | 3% |  |
| `identity.tenantId` | string | 3% |  |
| `identity.type` | string | 6% |  |
| `identity.userAssignedIdentities.<resource-id>.clientId` | string | 2% |  |
| `identity.userAssignedIdentities.<resource-id>.principalId` | string | 2% |  |
| `instance_view.bootDiagnostics.consoleScreenshotBlobUri` | string | 7% |  |
| `instance_view.bootDiagnostics.serialConsoleLogBlobUri` | string | 7% |  |
| `instance_view.computerName` | string | 23% |  |
| `instance_view.disks[].name` | string | 100% |  |
| `instance_view.disks[].statuses[].code` | string | 100% | `"ProvisioningState/succeeded"` |
| `instance_view.disks[].statuses[].displayStatus` | string | 100% | `"Provisioning succeeded"` |
| `instance_view.disks[].statuses[].level` | string | 100% | `"Info"` |
| `instance_view.disks[].statuses[].time` | string | 100% |  |
| `instance_view.hyperVGeneration` | string | 100% | `"V1"`, `"V2"` |
| `instance_view.osName` | string | 23% |  |
| `instance_view.osVersion` | string | 23% |  |
| `instance_view.patchStatus.availablePatchSummary.assessmentActivityId` | string | 1% |  |
| `instance_view.patchStatus.availablePatchSummary.criticalAndSecurityPatchCount` | number | 1% |  |
| `instance_view.patchStatus.availablePatchSummary.error.code` | string | 1% |  |
| `instance_view.patchStatus.availablePatchSummary.error.message` | string | 1% |  |
| `instance_view.patchStatus.availablePatchSummary.lastModifiedTime` | string | 1% |  |
| `instance_view.patchStatus.availablePatchSummary.otherPatchCount` | number | 1% |  |
| `instance_view.patchStatus.availablePatchSummary.rebootPending` | boolean | 1% | `false` |
| `instance_view.patchStatus.availablePatchSummary.startTime` | string | 1% |  |
| `instance_view.patchStatus.availablePatchSummary.status` | string | 1% | `"Succeeded"` |
| `instance_view.patchStatus.configurationStatuses[].code` | string | 1% |  |
| `instance_view.patchStatus.configurationStatuses[].displayStatus` | string | 1% | `"Ready"`, `"Supported"` |
| `instance_view.patchStatus.configurationStatuses[].level` | string | 1% | `"Info"` |
| `instance_view.patchStatus.configurationStatuses[].time` | string | 1% |  |
| `instance_view.platformFaultDomain` | number | 2% |  |
| `instance_view.platformUpdateDomain` | number | 2% |  |
| `instance_view.statuses[].code` | string | 100% |  |
| `instance_view.statuses[].displayStatus` | string | 100% | `"Provisioning succeeded"`, `"VM deallocated"`, `"VM running"`, `"VM stopped"` |
| `instance_view.statuses[].level` | string | 100% | `"Info"` |
| `instance_view.statuses[].time` | string | 100% |  |
| `instance_view.vmAgent.statuses[].code` | string | 29% | `"ProvisioningState/Unavailable"`, `"ProvisioningState/succeeded"` |
| `instance_view.vmAgent.statuses[].displayStatus` | string | 29% | `"Not Ready"`, `"Ready"` |
| `instance_view.vmAgent.statuses[].level` | string | 29% | `"Info"`, `"Warning"` |
| `instance_view.vmAgent.statuses[].message` | string | 29% |  |
| `instance_view.vmAgent.statuses[].time` | string | 29% |  |
| `instance_view.vmAgent.vmAgentVersion` | string | 29% |  |
| `interfaces[].ip_addresses[]` | string | 100% |  |
| `interfaces[].mac_address` | string | 100% |  |
| `interfaces[].subnets[]` | string | 100% |  |
| `location` | string | 100% |  |
| `model` | string | 100% |  |
| `name` | string | 100% |  |
| `os` | string | 100% |  |
| `plan.name` | string | 10% |  |
| `plan.product` | string | 10% |  |
| `plan.publisher` | string | 10% | `"infoblox"` |
| `properties.additionalCapabilities.hibernationEnabled` | boolean | 84% | `false` |
| `properties.availabilitySet.id` | string | 2% |  |
| `properties.diagnosticsProfile.bootDiagnostics.enabled` | boolean | 99% | `true` |
| `properties.diagnosticsProfile.bootDiagnostics.storageUri` | string | 7% |  |
| `properties.hardwareProfile.vmSize` | string | 100% |  |
| `properties.licenseType` | string | 2% | `"Windows_Client"` |
| `properties.networkProfile.networkInterfaces[].id` | string | 100% |  |
| `properties.networkProfile.networkInterfaces[].properties.deleteOption` | string | 84% | `"Delete"`, `"Detach"` |
| `properties.networkProfile.networkInterfaces[].properties.primary` | boolean | 16% | `false`, `true` |
| `properties.osProfile.adminUsername` | string | 94% |  |
| `properties.osProfile.allowExtensionOperations` | boolean | 94% | `true` |
| `properties.osProfile.computerName` | string | 94% |  |
| `properties.osProfile.linuxConfiguration.disablePasswordAuthentication` | boolean | 92% | `false`, `true` |
| `properties.osProfile.linuxConfiguration.patchSettings.assessmentMode` | string | 92% | `"ImageDefault"` |
| `properties.osProfile.linuxConfiguration.patchSettings.automaticByPlatformSettings.rebootSetting` | string | 1% | `"IfRequired"` |
| `properties.osProfile.linuxConfiguration.patchSettings.patchMode` | string | 92% | `"AutomaticByPlatform"`, `"ImageDefault"` |
| `properties.osProfile.linuxConfiguration.provisionVMAgent` | boolean | 92% | `true` |
| `properties.osProfile.linuxConfiguration.ssh.publicKeys[].keyData` | string | 80% |  |
| `properties.osProfile.linuxConfiguration.ssh.publicKeys[].path` | string | 80% |  |
| `properties.osProfile.requireGuestProvisionSignal` | boolean | 94% | `true` |
| `properties.osProfile.windowsConfiguration.enableAutomaticUpdates` | boolean | 2% | `true` |
| `properties.osProfile.windowsConfiguration.enableVMAgentPlatformUpdates` | boolean | 2% | `true` |
| `properties.osProfile.windowsConfiguration.patchSettings.assessmentMode` | string | 2% | `"ImageDefault"` |
| `properties.osProfile.windowsConfiguration.patchSettings.enableHotpatching` | boolean | 2% | `false` |
| `properties.osProfile.windowsConfiguration.patchSettings.patchMode` | string | 2% | `"AutomaticByOS"` |
| `properties.osProfile.windowsConfiguration.provisionVMAgent` | boolean | 2% | `true` |
| `properties.provisioningState` | string | 100% | `"Succeeded"` |
| `properties.securityProfile.securityType` | string | 14% |  |
| `properties.securityProfile.uefiSettings.secureBootEnabled` | boolean | 14% | `true` |
| `properties.securityProfile.uefiSettings.vTpmEnabled` | boolean | 14% | `true` |
| `properties.storageProfile.diskControllerType` | string | 87% | `"SCSI"` |
| `properties.storageProfile.imageReference.exactVersion` | string | 94% |  |
| `properties.storageProfile.imageReference.offer` | string | 94% |  |
| `properties.storageProfile.imageReference.publisher` | string | 94% | `"canonical"`, `"infoblox"`, `"microsoftwindowsdesktop"` |
| `properties.storageProfile.imageReference.sku` | string | 94% |  |
| `properties.storageProfile.imageReference.version` | string | 94% |  |
| `properties.storageProfile.osDisk.caching` | string | 100% | `"ReadOnly"`, `"ReadWrite"` |
| `properties.storageProfile.osDisk.createOption` | string | 100% | `"Attach"`, `"FromImage"` |
| `properties.storageProfile.osDisk.deleteOption` | string | 100% | `"Delete"`, `"Detach"` |
| `properties.storageProfile.osDisk.diskSizeGB` | number | 29% |  |
| `properties.storageProfile.osDisk.managedDisk.id` | string | 100% |  |
| `properties.storageProfile.osDisk.managedDisk.storageAccountType` | string | 29% |  |
| `properties.storageProfile.osDisk.name` | string | 100% |  |
| `properties.storageProfile.osDisk.osType` | string | 100% | `"Linux"`, `"Windows"` |
| `properties.timeCreated` | string | 100% |  |
| `properties.vmId` | string | 100% |  |
| `subscription_id` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `type` | string | 100% |  |
| `types.azure.source_assigned` | string | 100% | `"virtual_machine"` |
| `updated_at` | string | 100% |  |
| `vendor` | string | 100% | `"azure"` |
| `zones[]` | string | 81% |  |

### `azure.azure_network_public_ip_addresses`

Provider **Azure**, 108 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `etag` | string | 100% |  |
| `id` | string | 100% |  |
| `location` | string | 100% |  |
| `name` | string | 100% |  |
| `properties.ddosSettings.protectionMode` | string | 28% | `"Disabled"`, `"VirtualNetworkInherited"` |
| `properties.dnsSettings.domainNameLabel` | string | 6% |  |
| `properties.dnsSettings.fqdn` | string | 6% |  |
| `properties.idleTimeoutInMinutes` | number | 100% |  |
| `properties.ipAddress` | string | 94% |  |
| `properties.ipConfiguration.id` | string | 87% |  |
| `properties.ipTags[].ipTagType` | string | 1% |  |
| `properties.ipTags[].tag` | string | 1% |  |
| `properties.provisioningState` | string | 100% | `"Succeeded"` |
| `properties.publicIPAddressVersion` | string | 100% |  |
| `properties.publicIPAllocationMethod` | string | 100% |  |
| `properties.resourceGuid` | string | 100% |  |
| `sku.name` | string | 100% |  |
| `sku.tier` | string | 100% | `"Regional"` |
| `subscription_id` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `type` | string | 100% |  |
| `zones[]` | string | 73% |  |

### `azure.azure_network_security_groups`

Provider **Azure**, 104 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `etag` | string | 100% |  |
| `id` | string | 100% |  |
| `location` | string | 100% |  |
| `name` | string | 100% |  |
| `properties.defaultSecurityRules[].etag` | string | 100% |  |
| `properties.defaultSecurityRules[].id` | string | 100% |  |
| `properties.defaultSecurityRules[].name` | string | 100% |  |
| `properties.defaultSecurityRules[].properties.access` | string | 100% | `"Allow"`, `"Deny"` |
| `properties.defaultSecurityRules[].properties.description` | string | 100% |  |
| `properties.defaultSecurityRules[].properties.destinationAddressPrefix` | string | 100% |  |
| `properties.defaultSecurityRules[].properties.destinationPortRange` | string | 100% |  |
| `properties.defaultSecurityRules[].properties.direction` | string | 100% | `"Inbound"` |
| `properties.defaultSecurityRules[].properties.priority` | number | 100% |  |
| `properties.defaultSecurityRules[].properties.protocol` | string | 100% |  |
| `properties.defaultSecurityRules[].properties.provisioningState` | string | 100% | `"Succeeded"` |
| `properties.defaultSecurityRules[].properties.sourceAddressPrefix` | string | 100% |  |
| `properties.defaultSecurityRules[].properties.sourcePortRange` | string | 100% |  |
| `properties.defaultSecurityRules[].type` | string | 100% |  |
| `properties.networkInterfaces[].id` | string | 88% |  |
| `properties.provisioningState` | string | 100% | `"Succeeded"` |
| `properties.resourceGuid` | string | 100% |  |
| `properties.securityRules[].etag` | string | 99% |  |
| `properties.securityRules[].id` | string | 99% |  |
| `properties.securityRules[].name` | string | 99% |  |
| `properties.securityRules[].properties.access` | string | 99% | `"Allow"` |
| `properties.securityRules[].properties.description` | string | 13% |  |
| `properties.securityRules[].properties.destinationAddressPrefix` | string | 99% |  |
| `properties.securityRules[].properties.destinationPortRange` | string | 99% |  |
| `properties.securityRules[].properties.destinationPortRanges[]` | string | 1% |  |
| `properties.securityRules[].properties.direction` | string | 99% | `"Inbound"`, `"Outbound"` |
| `properties.securityRules[].properties.priority` | number | 99% |  |
| `properties.securityRules[].properties.protocol` | string | 99% |  |
| `properties.securityRules[].properties.provisioningState` | string | 99% | `"Succeeded"` |
| `properties.securityRules[].properties.sourceAddressPrefix` | string | 99% |  |
| `properties.securityRules[].properties.sourcePortRange` | string | 99% |  |
| `properties.securityRules[].type` | string | 99% |  |
| `properties.subnets[].id` | string | 5% |  |
| `subscription_id` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `type` | string | 100% |  |

### `azure.azure_network_virtual_network_gateways`

Provider **Azure**, 2 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `etag` | string | 100% |  |
| `id` | string | 100% |  |
| `location` | string | 100% |  |
| `name` | string | 100% |  |
| `properties.activeActive` | boolean | 100% | `false` |
| `properties.allowRemoteVnetTraffic` | boolean | 100% | `false` |
| `properties.allowVirtualWanTraffic` | boolean | 100% | `false` |
| `properties.disableIPSecReplayProtection` | boolean | 100% | `false` |
| `properties.enableBgp` | boolean | 100% | `false` |
| `properties.enableBgpRouteTranslationForNat` | boolean | 100% | `false` |
| `properties.enablePrivateIpAddress` | boolean | 100% | `false` |
| `properties.gatewayType` | string | 100% | `"Vpn"` |
| `properties.ipConfigurations[].etag` | string | 100% |  |
| `properties.ipConfigurations[].id` | string | 100% |  |
| `properties.ipConfigurations[].name` | string | 100% |  |
| `properties.ipConfigurations[].properties.privateIPAllocationMethod` | string | 100% |  |
| `properties.ipConfigurations[].properties.provisioningState` | string | 100% | `"Succeeded"` |
| `properties.ipConfigurations[].properties.publicIPAddress.id` | string | 100% |  |
| `properties.ipConfigurations[].properties.subnet.id` | string | 100% |  |
| `properties.provisioningState` | string | 100% | `"Succeeded"` |
| `properties.resourceGuid` | string | 100% |  |
| `properties.sku.capacity` | number | 100% |  |
| `properties.sku.name` | string | 100% |  |
| `properties.sku.tier` | string | 100% | `"Basic"` |
| `properties.vpnClientConfiguration.vpnClientProtocols[]` | string | 100% | `"IkeV2"` |
| `properties.vpnGatewayGeneration` | string | 100% | `"Generation1"`, `"Generation2"` |
| `properties.vpnType` | string | 100% | `"RouteBased"` |
| `subscription_id` | string | 100% |  |
| `tags` | key/value map | 100% |  |
| `type` | string | 100% |  |

### `azure.azure_storage_containers`

Provider **Azure**, 9 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `etag` | string | 100% |  |
| `id` | string | 100% |  |
| `name` | string | 100% |  |
| `properties.defaultEncryptionScope` | string | 100% |  |
| `properties.deleted` | boolean | 100% | `false` |
| `properties.denyEncryptionScopeOverride` | boolean | 100% | `false` |
| `properties.hasImmutabilityPolicy` | boolean | 100% | `false` |
| `properties.hasLegalHold` | boolean | 100% | `false` |
| `properties.immutableStorageWithVersioning.enabled` | boolean | 100% | `false` |
| `properties.lastModifiedTime` | string | 100% |  |
| `properties.leaseState` | string | 100% | `"Available"` |
| `properties.leaseStatus` | string | 100% | `"Unlocked"` |
| `properties.publicAccess` | string | 100% | `"None"` |
| `properties.remainingRetentionDays` | number | 100% |  |
| `subscription_id` | string | 100% |  |
| `type` | string | 100% |  |

### `bep.bep_devices`

Provider **Infoblox Endpoint**, 8 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `alternative_id` | string | 100% |  |
| `contact` | string | 100% |  |
| `created_at` | string | 100% |  |
| `description` | string | 100% |  |
| `domain_id` | string | 100% |  |
| `first_seen` | string | 100% |  |
| `id` | string | 100% |  |
| `interfaces[].ip_addresses[]` | string | 100% |  |
| `interfaces[].mac_address` | string | 75% |  |
| `is_cluster` | boolean | 100% | `false` |
| `is_shared` | boolean | 100% | `false` |
| `is_virtual` | boolean | 100% | `false` |
| `last_seen` | string | 100% |  |
| `location` | string | 100% |  |
| `location_details.city` | string | 100% |  |
| `location_details.country` | string | 100% |  |
| `location_details.region` | string | 100% |  |
| `management_address` | string | 100% |  |
| `model` | string | 100% |  |
| `names.discovered_name` | string | 100% |  |
| `os` | string | 100% | `"Windows 11 Home"`, `"iOS 27.0.1"`, `"iPadOS 27.0.1"`, `"macOS 26.6"`, `"macOS 26.6.2"`, `"macOS 26.7.1"` |
| `parent_cluster_id` | string | 100% |  |
| `parent_host_id` | string | 100% |  |
| `public_ip` | string | 100% |  |
| `serial_number` | string | 100% |  |
| `specs.system_uuid` | string | 100% |  |
| `tenant` | string | 100% |  |
| `types.source_assigned` | string | 100% | `"mobile_device"`, `"workstation"` |
| `types.type` | string | 100% | `"mobile_device"`, `"workstation"` |
| `updated_at` | string | 100% |  |
| `vendor` | string | 100% |  |

### `dhcp.dhcp_devices`

Provider **DHCP Logs**, 98 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `AlternativeId` | string | 60% |  |
| `Contact` | string | 60% |  |
| `CreatedAt` | string | 60% |  |
| `Description` | string | 60% |  |
| `DomainId` | string | 60% |  |
| `FirstSeen` | string | 60% |  |
| `Id` | string | 60% |  |
| `Interfaces[].ip_addresses[]` | string | 60% |  |
| `Interfaces[].mac_address` | string | 60% |  |
| `IsCluster` | boolean | 60% | `false` |
| `IsShared` | boolean | 60% | `false` |
| `IsVirtual` | boolean | 60% | `false` |
| `LastSeen` | string | 60% |  |
| `Location` | string | 60% |  |
| `ManagementAddress` | string | 60% |  |
| `Model` | string | 60% |  |
| `Names.discovered_name` | string | 29% |  |
| `Os` | string | 60% | `"Android"`, `"OS X"`, `"Windows"`, `"iOS"` |
| `ParentClusterId` | string | 60% |  |
| `ParentHostId` | string | 60% |  |
| `SerialNumber` | string | 60% |  |
| `Signature.device_make` | string | 59% |  |
| `Signature.device_model` | string | 59% |  |
| `Signature.device_osname` | string | 59% |  |
| `Signature.device_osvers` | string | 59% |  |
| `Signature.device_rank` | number | 59% |  |
| `Signature.device_type` | string | 59% |  |
| `Signature.dhcp_fingerprint` | string | 54% |  |
| `Signature.source` | string | 59% | `"DHCP"`, `"MACCLUSTER"` |
| `Tenant` | string | 60% |  |
| `Types.source_assigned` | string | 1% | `"COMPUTER"` |
| `Types.type` | string | 1% | `"COMPUTER"` |
| `UpdatedAt` | string | 60% |  |
| `Vendor` | string | 60% |  |
| `alternative_id` | string | 40% |  |
| `contact` | string | 40% |  |
| `created_at` | string | 40% |  |
| `description` | string | 40% |  |
| `domain_id` | string | 40% |  |
| `first_seen` | string | 40% |  |
| `id` | string | 40% |  |
| `interfaces[].ip_addresses[]` | string | 40% |  |
| `interfaces[].ip_ipspace_id.<ip>.dhcp` | string | 3% |  |
| `interfaces[].mac_address` | string | 40% |  |
| `is_cluster` | boolean | 40% | `false` |
| `is_shared` | boolean | 40% | `false` |
| `is_virtual` | boolean | 40% | `false` |
| `last_seen` | string | 40% |  |
| `location` | string | 40% |  |
| `management_address` | string | 40% |  |
| `model` | string | 40% |  |
| `names.discovered_name` | string | 32% |  |
| `os` | string | 40% | `"Android"`, `"Tizen"`, `"Windows"` |
| `parent_cluster_id` | string | 40% |  |
| `parent_host_id` | string | 40% |  |
| `serial_number` | string | 40% |  |
| `signature.device_make` | string | 40% |  |
| `signature.device_model` | string | 40% |  |
| `signature.device_osname` | string | 40% |  |
| `signature.device_osvers` | string | 40% |  |
| `signature.device_rank` | number | 40% |  |
| `signature.device_type` | string | 40% |  |
| `signature.dhcp_fingerprint` | string | 32% |  |
| `signature.source` | string | 40% | `"DHCP"`, `"MACCLUSTER"` |
| `tenant` | string | 40% |  |
| `types.derived_from_dhcp_fingerprint` | string | 31% |  |
| `types.source_assigned` | string | 35% |  |
| `types.type` | string | 35% |  |
| `updated_at` | string | 40% |  |
| `vendor` | string | 40% |  |

### `gcp.gcp_compute_addresses`

Provider **GCP**, 1 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `address` | string | 100% |  |
| `address_type` | string | 100% |  |
| `creation_timestamp` | string | 100% |  |
| `description` | string | 100% |  |
| `id` | string | 100% |  |
| `kind` | string | 100% |  |
| `label_fingerprint` | string | 100% |  |
| `labels` | key/value map | 100% |  |
| `name` | string | 100% |  |
| `network_tier` | string | 100% |  |
| `project_id` | string | 100% |  |
| `region` | string | 100% |  |
| `self_link` | string | 100% |  |
| `status` | string | 100% | `"IN_USE"` |
| `users[]` | string | 100% |  |

### `gcp.gcp_compute_disks`

Provider **GCP**, 101 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `architecture` | string | 99% | `"X86_64"` |
| `creation_timestamp` | string | 100% |  |
| `enable_confidential_compute` | boolean | 100% | `false` |
| `guest_os_features[].type` | string | 99% | `"GVNIC"`, `"MULTI_IP_SUBNET"`, `"UEFI_COMPATIBLE"`, `"VIRTIO_SCSI_MULTIQUEUE"` |
| `id` | string | 100% |  |
| `kind` | string | 100% |  |
| `label_fingerprint` | string | 100% |  |
| `labels` | key/value map | 100% |  |
| `last_attach_timestamp` | string | 100% |  |
| `license_codes[]` | string | 99% |  |
| `licenses[]` | string | 99% |  |
| `name` | string | 100% |  |
| `physical_block_size_bytes` | string | 100% |  |
| `project_id` | string | 100% |  |
| `resource_policies[]` | string | 83% |  |
| `satisfies_pzi` | boolean | 82% | `false`, `true` |
| `self_link` | string | 100% |  |
| `size_gb` | string | 100% |  |
| `source_image` | string | 100% |  |
| `source_image_id` | string | 100% |  |
| `status` | string | 100% | `"READY"` |
| `type` | string | 100% |  |
| `users[]` | string | 100% |  |
| `zone` | string | 100% |  |

### `gcp.gcp_compute_firewalls`

Provider **GCP**, 142 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `allowed[].I_p_protocol` | string | 100% | `"all"`, `"icmp"`, `"tcp"`, `"udp"` |
| `allowed[].ports[]` | string | 92% |  |
| `creation_timestamp` | string | 100% |  |
| `description` | string | 100% |  |
| `destination_ranges[]` | string | 4% |  |
| `direction` | string | 100% | `"EGRESS"`, `"INGRESS"` |
| `disabled` | boolean | 100% | `false` |
| `id` | string | 100% |  |
| `kind` | string | 100% |  |
| `log_config.enable` | boolean | 100% | `false`, `true` |
| `log_config.metadata` | string | 2% | `"INCLUDE_ALL_METADATA"` |
| `name` | string | 100% |  |
| `network` | string | 100% |  |
| `priority` | string | 100% |  |
| `project_id` | string | 100% |  |
| `self_link` | string | 100% |  |
| `source_ranges[]` | string | 99% |  |
| `target_tags[]` | string | 9% |  |

### `gcp.gcp_compute_instances`

Provider **GCP**, 101 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `can_ip_forward` | boolean | 100% | `false` |
| `confidential_instance_config.enable_confidential_compute` | boolean | 99% | `false` |
| `cpu_platform` | string | 100% | `"Intel Broadwell"`, `"Unknown CPU Platform"` |
| `creation_timestamp` | string | 100% |  |
| `deletion_protection` | boolean | 100% | `false` |
| `description` | string | 100% |  |
| `disks[].architecture` | string | 99% | `"X86_64"` |
| `disks[].auto_delete` | boolean | 100% | `true` |
| `disks[].boot` | boolean | 100% | `true` |
| `disks[].device_name` | string | 100% |  |
| `disks[].disk_size_gb` | number | 100% |  |
| `disks[].guest_os_features[].type` | string | 99% | `"GVNIC"`, `"MULTI_IP_SUBNET"`, `"UEFI_COMPATIBLE"`, `"VIRTIO_SCSI_MULTIQUEUE"` |
| `disks[].index` | number | 100% |  |
| `disks[].interface` | string | 100% | `"SCSI"` |
| `disks[].kind` | string | 100% |  |
| `disks[].licenses[]` | string | 99% |  |
| `disks[].mode` | string | 100% | `"READ_WRITE"` |
| `disks[].source` | string | 100% |  |
| `disks[].type` | string | 100% | `"PERSISTENT"` |
| `display_device.enable_display` | boolean | 99% | `false` |
| `fingerprint` | string | 100% |  |
| `hostname` | string | 100% |  |
| `id` | number | 100% |  |
| `key_revocation_action_type` | string | 100% |  |
| `kind` | string | 100% |  |
| `label_fingerprint` | string | 100% |  |
| `labels` | key/value map | 100% |  |
| `last_start_timestamp` | string | 100% |  |
| `last_stop_timestamp` | string | 100% |  |
| `machine_type` | string | 100% |  |
| `metadata.fingerprint` | string | 100% |  |
| `metadata.items[].key` | string | 100% |  |
| `metadata.items[].value` | string | 100% |  |
| `metadata.kind` | string | 100% |  |
| `model` | string | 100% | `"e2-medium"`, `"e2-micro"`, `"e2-standard-4"` |
| `name` | string | 100% |  |
| `network_interfaces[].access_configs[].kind` | string | 100% |  |
| `network_interfaces[].access_configs[].name` | string | 100% |  |
| `network_interfaces[].access_configs[].nat_i_p` | string | 2% |  |
| `network_interfaces[].access_configs[].network_tier` | string | 100% |  |
| `network_interfaces[].access_configs[].set_public_ptr` | boolean | 1% | `false` |
| `network_interfaces[].access_configs[].type` | string | 100% | `"ONE_TO_ONE_NAT"` |
| `network_interfaces[].alias_ip_ranges[].ip_cidr_range` | string | 1% |  |
| `network_interfaces[].fingerprint` | string | 100% |  |
| `network_interfaces[].kind` | string | 100% |  |
| `network_interfaces[].name` | string | 100% |  |
| `network_interfaces[].network` | string | 100% |  |
| `network_interfaces[].network_i_p` | string | 100% |  |
| `network_interfaces[].nic_type` | string | 3% | `"GVNIC"`, `"VIRTIO_NET"` |
| `network_interfaces[].stack_type` | string | 100% | `"IPV4_ONLY"` |
| `network_interfaces[].subnetwork` | string | 100% |  |
| `project_id` | string | 100% |  |
| `reservation_affinity.consume_reservation_type` | string | 100% | `"ANY_RESERVATION"` |
| `resource_status.effective_instance_metadata.enable_osconfig_metadata_value` | boolean | 65% | `true` |
| `resource_status.effective_instance_metadata.enable_oslogin_metadata_value` | boolean | 100% | `true` |
| `resource_status.effective_instance_metadata.vm_dns_setting_metadata_value` | string | 100% |  |
| `scheduling.automatic_restart` | boolean | 100% | `true` |
| `scheduling.on_host_maintenance` | string | 100% | `"MIGRATE"` |
| `scheduling.preemptible` | boolean | 100% | `false` |
| `scheduling.provisioning_model` | string | 100% | `"STANDARD"` |
| `self_link` | string | 100% |  |
| `service_accounts[].email` | string | 100% |  |
| `service_accounts[].scopes[]` | string | 100% |  |
| `shielded_instance_config.enable_integrity_monitoring` | boolean | 99% | `true` |
| `shielded_instance_config.enable_secure_boot` | boolean | 99% | `false` |
| `shielded_instance_config.enable_vtpm` | boolean | 99% | `true` |
| `shielded_instance_integrity_policy.update_auto_learn_policy` | boolean | 99% | `true` |
| `start_restricted` | boolean | 100% | `false` |
| `status` | string | 100% | `"RUNNING"`, `"TERMINATED"` |
| `tags` | key/value map | 100% |  |
| `types.gcp.source_assigned` | string | 100% | `"virtual_machine"` |
| `updated_at` | string | 100% |  |
| `vendor` | string | 100% | `"gcp"` |
| `zone` | string | 100% |  |

### `gcp.gcp_storage_buckets`

Provider **GCP**, 2 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `bucket_policy_only.Enabled` | boolean | 100% | `true` |
| `bucket_policy_only.LockedTime` | string | 100% |  |
| `created.nanos` | number | 100% |  |
| `created.seconds` | number | 100% |  |
| `default_event_based_hold` | boolean | 100% | `false` |
| `etag` | string | 100% |  |
| `labels` | key/value map | 100% |  |
| `location` | string | 100% |  |
| `location_type` | string | 100% |  |
| `meta_generation` | number | 100% |  |
| `model` | string | 100% | `"STANDARD"` |
| `name` | string | 100% |  |
| `predefined_acl` | string | 100% |  |
| `predefined_default_object_acl` | string | 100% |  |
| `project_id` | string | 100% |  |
| `project_number` | number | 100% |  |
| `public_access_prevention` | number | 100% |  |
| `requester_pays` | boolean | 100% | `false` |
| `rpo` | number | 100% |  |
| `storage_class` | string | 100% | `"STANDARD"` |
| `uniform_bucket_level_access.Enabled` | boolean | 100% | `true` |
| `uniform_bucket_level_access.LockedTime` | string | 100% |  |
| `updated_at` | string | 100% |  |
| `versioning_enabled` | boolean | 100% | `false` |

### `meraki.meraki_network_clients`

Provider **Cisco Meraki**, 10 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `AdaptivePolicyGroup` | string | 100% |  |
| `AlternativeId` | string | 100% |  |
| `ClientID` | string | 100% |  |
| `Contact` | string | 100% |  |
| `CreatedAt` | string | 100% |  |
| `Description` | string | 100% |  |
| `DeviceTypePrediction` | string | 100% |  |
| `DomainId` | string | 100% |  |
| `FirstSeen` | string | 100% |  |
| `GroupPolicy8021x` | string | 100% |  |
| `IP6` | string | 100% |  |
| `IP6Local` | string | 100% |  |
| `Id` | string | 100% |  |
| `Interfaces[].ip_addresses[]` | string | 100% |  |
| `Interfaces[].mac_address` | string | 100% |  |
| `IsCluster` | boolean | 100% | `false` |
| `IsShared` | boolean | 100% | `false` |
| `IsVirtual` | boolean | 100% | `false` |
| `LastSeen` | string | 100% |  |
| `Location` | string | 100% |  |
| `ManagementAddress` | string | 100% |  |
| `Manufacturer` | string | 100% | `"Amazon Technologies"`, `"Commscope"`, `"Intel"`, `"Sony Interactive..."`, `"Universal Global..."` |
| `Model` | string | 100% |  |
| `NamedVlan` | string | 100% |  |
| `Names.discovered_name` | string | 100% |  |
| `NetworkId` | string | 100% |  |
| `Notes` | string | 100% |  |
| `OS` | string | 100% | `"Android"`, `"Debian-based Linux"`, `"PlayStation Portable"`, `"Windows 10"` |
| `OrganizationId` | string | 100% |  |
| `Os` | string | 100% | `"Android"`, `"Debian-based Linux"`, `"PlayStation Portable"`, `"Windows 10"` |
| `PSKGroup` | string | 100% |  |
| `ParentClusterId` | string | 100% |  |
| `ParentHostId` | string | 100% |  |
| `RecentDeviceConnection` | string | 100% | `"Wired"`, `"Wireless"` |
| `RecentDeviceMac` | string | 100% |  |
| `RecentDeviceName` | string | 100% |  |
| `RecentDeviceSerial` | string | 100% |  |
| `SMInstalled` | boolean | 100% | `false` |
| `SSID` | string | 100% |  |
| `SerialNumber` | string | 100% |  |
| `Status` | string | 100% | `"Online"` |
| `Switchport` | string | 100% |  |
| `Tenant` | string | 100% |  |
| `UpdatedAt` | string | 100% |  |
| `Usage.recv` | number | 100% |  |
| `Usage.sent` | number | 100% |  |
| `User` | string | 100% |  |
| `VLAN` | string | 100% |  |
| `Vendor` | string | 100% | `"Amazon Technologies"`, `"Commscope"`, `"Intel"`, `"Sony Interactive..."`, `"Universal Global..."` |
| `WirelessCapabilities` | string | 100% |  |

### `meraki.meraki_organization_devices`

Provider **Cisco Meraki**, 4 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `AlternativeId` | string | 100% |  |
| `Contact` | string | 100% |  |
| `CreatedAt` | string | 100% |  |
| `Description` | string | 100% |  |
| `Details[].name` | string | 100% |  |
| `Details[].value` | string | 100% |  |
| `DomainId` | string | 100% |  |
| `Firmware` | string | 100% | `"switch-17-2-2"`, `"wireless-26-8-3"`, `"wireless-30-7-1"`, `"wireless-32-1-7"` |
| `Id` | string | 100% |  |
| `Imei` | string | 100% |  |
| `Interfaces[].ip_addresses[]` | string | 100% |  |
| `Interfaces[].mac_address` | string | 100% |  |
| `IsCluster` | boolean | 100% | `false` |
| `IsShared` | boolean | 100% | `false` |
| `IsVirtual` | boolean | 100% | `false` |
| `Location` | string | 100% |  |
| `LocationDetails.address` | string | 100% |  |
| `LocationDetails.latitude` | number | 100% |  |
| `LocationDetails.longitude` | number | 100% |  |
| `ManagementAddress` | string | 100% |  |
| `Model` | string | 100% | `"MR32"`, `"MR33"`, `"MR36"`, `"MS220-8P"` |
| `Names.discovered_name` | string | 100% |  |
| `NetworkId` | string | 100% |  |
| `Notes` | string | 100% |  |
| `OrganizationId` | string | 100% |  |
| `Os` | string | 100% |  |
| `ParentClusterId` | string | 100% |  |
| `ParentHostId` | string | 100% |  |
| `ProductType` | string | 100% | `"switch"`, `"wireless"` |
| `SerialNumber` | string | 100% |  |
| `Tags[]` | string | 75% |  |
| `Tenant` | string | 100% |  |
| `Types.source_assigned` | string | 100% | `"access_point"`, `"switch"` |
| `UpdatedAt` | string | 100% |  |
| `Vendor` | string | 100% | `"Cisco Meraki"` |

### `ngc.ngc_devices`

Provider **NIOS Network Insight**, 85 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `AlternativeId` | string | 14% |  |
| `Contact` | string | 14% |  |
| `CreatedAt` | string | 14% |  |
| `Description` | string | 14% |  |
| `DeviceMac` | string | 14% |  |
| `DomainId` | string | 14% |  |
| `FirstSeen` | string | 14% |  |
| `Id` | string | 14% |  |
| `Interfaces[].cidrs[]` | string | 1% |  |
| `Interfaces[].dhcp_lease_info.end` | string | 4% |  |
| `Interfaces[].dhcp_lease_info.start` | string | 4% |  |
| `Interfaces[].dhcp_lease_info.state` | string | 4% | `"active"`, `"static"` |
| `Interfaces[].ip_addresses[]` | string | 14% |  |
| `Interfaces[].ip_ipspace_id.<ip>.ngc` | string | 2% |  |
| `Interfaces[].mac_address` | string | 5% |  |
| `Interfaces[].port_name` | string | 1% |  |
| `Interfaces[].switch_ip_address` | string | 1% |  |
| `Interfaces[].switch_name` | string | 1% |  |
| `Interfaces[].vlan_numbers[]` | string | 1% |  |
| `IsCluster` | boolean | 14% | `false` |
| `IsEndHost` | boolean | 14% | `false`, `true` |
| `IsShared` | boolean | 14% | `false` |
| `IsVirtual` | boolean | 14% | `false` |
| `LastSeen` | string | 14% |  |
| `Location` | string | 14% |  |
| `ManagementAddress` | string | 14% |  |
| `Model` | string | 14% |  |
| `Names.discovered_name` | string | 13% |  |
| `Os` | string | 14% | `"Windows OS"` |
| `ParentClusterId` | string | 14% |  |
| `ParentHostId` | string | 14% |  |
| `SerialNumber` | string | 14% |  |
| `Signature.dhcp_fingerprint` | string | 1% |  |
| `Signature.dhcp_params` | string | 1% |  |
| `Tenant` | string | 14% |  |
| `Types.derived_from_dhcp_fingerprint` | string | 1% |  |
| `Types.source_assigned` | string | 1% | `"unknown"` |
| `Types.type` | string | 1% | `"unknown"` |
| `UpdatedAt` | string | 14% |  |
| `Vendor` | string | 14% |  |
| `alternative_id` | string | 86% |  |
| `contact` | string | 86% |  |
| `created_at` | string | 86% |  |
| `description` | string | 86% |  |
| `device_mac` | string | 86% |  |
| `domain_id` | string | 86% |  |
| `first_seen` | string | 86% |  |
| `id` | string | 86% |  |
| `interfaces[].cidrs[]` | string | 53% |  |
| `interfaces[].description` | string | 1% |  |
| `interfaces[].dhcp_lease_info.end` | string | 24% |  |
| `interfaces[].dhcp_lease_info.start` | string | 24% |  |
| `interfaces[].dhcp_lease_info.state` | string | 24% | `"abandoned"`, `"active"`, `"static"` |
| `interfaces[].ip_addresses[]` | string | 86% |  |
| `interfaces[].ip_ipspace_id.<ip>.ngc` | string | 2% |  |
| `interfaces[].mac_address` | string | 33% |  |
| `interfaces[].name` | string | 6% |  |
| `interfaces[].port_name` | string | 13% |  |
| `interfaces[].switch_ip_address` | string | 12% |  |
| `interfaces[].switch_name` | string | 12% |  |
| `interfaces[].type` | string | 6% | `"ethernet-csmacd"`, `"softwareLoopback"` |
| `interfaces[].vlan_numbers[]` | string | 14% |  |
| `interfaces[].vlans[].vlan_id` | number | 1% |  |
| `is_cluster` | boolean | 86% | `false` |
| `is_end_host` | boolean | 86% | `false`, `true` |
| `is_shared` | boolean | 86% | `false` |
| `is_virtual` | boolean | 86% | `false` |
| `last_seen` | string | 86% |  |
| `location` | string | 86% |  |
| `location_details.region` | string | 9% |  |
| `management_address` | string | 86% |  |
| `model` | string | 86% |  |
| `names.discovered_name` | string | 27% |  |
| `names.name` | string | 1% |  |
| `names.netbios_name` | string | 1% |  |
| `os` | string | 86% |  |
| `parent_cluster_id` | string | 86% |  |
| `parent_host_id` | string | 86% |  |
| `serial_number` | string | 86% |  |
| `signature.dhcp_fingerprint` | string | 7% |  |
| `signature.dhcp_params` | string | 19% |  |
| `signature.dhcp_vendor` | string | 9% | `"android-dhcp-16"`, `"udhcp 1.31.1"`, `"udhcp 1.33.1"`, `"udhcp 1.37.0"` |
| `tenant` | string | 86% |  |
| `types.derived_from_dhcp_fingerprint` | string | 8% |  |
| `types.source_assigned` | string | 51% | `"Linux"`, `"Router"`, `"Switch"`, `"X-Terminal"`, `"unknown"`, `"vNIOS"` |
| `types.type` | string | 51% | `"Linux"`, `"Router"`, `"Switch"`, `"X-Terminal"`, `"unknown"`, `"vNIOS"` |
| `updated_at` | string | 86% |  |
| `vendor` | string | 86% | `"Cisco"`, `"Infoblox"` |

### `ni.ni_devices`

Provider **Network Insight**, 31 assets in the sampled tenant.

| Path | Type | Coverage | Example values |
|---|---|---|---|
| `AlternativeId` | string | 3% |  |
| `Contact` | string | 3% |  |
| `CreatedAt` | string | 3% |  |
| `Description` | string | 3% |  |
| `DeviceAssurance` | number | 3% |  |
| `DeviceMac` | string | 3% |  |
| `DeviceUniqueKey` | string | 3% |  |
| `DiscoveryStatuses.app_infra_service_id` | string | 3% |  |
| `DiscoveryStatuses.cli_collection_enabled` | boolean | 3% | `true` |
| `DiscoveryStatuses.config_collection_enabled` | string | 3% |  |
| `DiscoveryStatuses.created_time` | string | 3% |  |
| `DiscoveryStatuses.device.device_assurance` | string | 3% |  |
| `DiscoveryStatuses.device_first_occurrence_time` | string | 3% |  |
| `DiscoveryStatuses.device_group_message` | string | 3% |  |
| `DiscoveryStatuses.device_group_status` | string | 3% | `"OK"` |
| `DiscoveryStatuses.device_group_timestamp` | string | 3% |  |
| `DiscoveryStatuses.device_id` | string | 3% |  |
| `DiscoveryStatuses.device_ip_dotted` | string | 3% |  |
| `DiscoveryStatuses.device_ip_numeric` | string | 3% |  |
| `DiscoveryStatuses.device_name` | string | 3% |  |
| `DiscoveryStatuses.device_type` | string | 3% | `"Bridge"` |
| `DiscoveryStatuses.exists_message` | string | 3% |  |
| `DiscoveryStatuses.exists_timestamp` | string | 3% |  |
| `DiscoveryStatuses.finger_print_enabled` | string | 3% |  |
| `DiscoveryStatuses.finger_print_message` | string | 3% |  |
| `DiscoveryStatuses.finger_print_status` | string | 3% | `"Error"` |
| `DiscoveryStatuses.finger_print_timestamp` | string | 3% |  |
| `DiscoveryStatuses.global_config_collection_enabled` | string | 3% |  |
| `DiscoveryStatuses.global_config_collection_message` | string | 3% |  |
| `DiscoveryStatuses.last_action` | string | 3% |  |
| `DiscoveryStatuses.last_seen` | string | 3% |  |
| `DiscoveryStatuses.last_timestamp` | string | 3% |  |
| `DiscoveryStatuses.license_override` | string | 3% |  |
| `DiscoveryStatuses.overall_status` | string | 3% | `"ERROR"` |
| `DiscoveryStatuses.reachable_message` | string | 3% |  |
| `DiscoveryStatuses.reachable_status` | string | 3% | `"Error"` |
| `DiscoveryStatuses.reachable_timestamp` | string | 3% |  |
| `DiscoveryStatuses.snmp_collection_enabled` | boolean | 3% | `true` |
| `DiscoveryStatuses.snmp_credential_message` | string | 3% |  |
| `DiscoveryStatuses.snmp_credential_status` | string | 3% | `"Error"` |
| `DiscoveryStatuses.snmp_credential_timestamp` | string | 3% |  |
| `DiscoveryStatuses.status_detail` | string | 3% | `"Unlicensed"` |
| `DiscoveryStatuses.unit_id` | string | 3% |  |
| `DiscoveryStatuses.updated_time` | string | 3% |  |
| `DomainId` | string | 3% |  |
| `FirstSeen` | string | 3% |  |
| `Id` | string | 3% |  |
| `Interfaces[].ip_addresses[]` | string | 3% |  |
| `Interfaces[].ip_ipspace_id.<ip>.ni` | string | 3% |  |
| `Interfaces[].mac_address` | string | 3% |  |
| `Interfaces[].port_name` | string | 3% |  |
| `Interfaces[].switch_ip_address` | string | 3% |  |
| `Interfaces[].switch_name` | string | 3% |  |
| `IsCluster` | boolean | 3% | `false` |
| `IsInfraDevice` | boolean | 3% | `false` |
| `IsShared` | boolean | 3% | `false` |
| `IsVirtual` | boolean | 3% | `false` |
| `LastSeen` | string | 3% |  |
| `Location` | string | 3% |  |
| `ManagementAddress` | string | 3% |  |
| `Model` | string | 3% |  |
| `Names.discovered_name` | string | 3% |  |
| `OphId` | string | 3% |  |
| `Os` | string | 3% |  |
| `ParentClusterId` | string | 3% |  |
| `ParentHostId` | string | 3% |  |
| `SerialNumber` | string | 3% |  |
| `Tenant` | string | 3% |  |
| `Types.source_assigned` | string | 3% | `"Bridge"` |
| `UpdatedAt` | string | 35% |  |
| `Vendor` | string | 3% |  |
| `created_at` | string | 32% |  |
| `created_time` | string | 97% |  |
| `device_additional_info` | string | 97% |  |
| `device_assurance` | string | 97% |  |
| `device_dns_name` | string | 29% |  |
| `device_first_occurrence_time` | string | 97% |  |
| `device_id` | string | 97% |  |
| `device_ip_dotted` | string | 97% |  |
| `device_mac` | string | 68% |  |
| `device_model` | string | 3% | `"mx68"` |
| `device_name` | string | 97% |  |
| `device_net_bios_name` | string | 3% |  |
| `device_oui` | string | 55% |  |
| `device_sys_contact` | string | 32% |  |
| `device_sys_descr` | string | 35% |  |
| `device_sys_location` | string | 32% |  |
| `device_sys_name` | string | 35% |  |
| `device_timestamp` | string | 97% |  |
| `device_type` | string | 97% |  |
| `device_unique_key` | string | 35% |  |
| `device_vendor` | string | 6% | `"HP"`, `"Meraki"` |
| `discovery_statuses.app_infra_service_id` | string | 97% |  |
| `discovery_statuses.cli_collection_enabled` | boolean | 97% | `true` |
| `discovery_statuses.cli_credential_message` | string | 3% |  |
| `discovery_statuses.cli_credential_status` | string | 3% | `"Error"` |
| `discovery_statuses.cli_credential_timestamp` | string | 3% |  |
| `discovery_statuses.config_collection_enabled` | string | 97% |  |
| `discovery_statuses.created_time` | string | 97% |  |
| `discovery_statuses.device.device_assurance` | string | 90% |  |
| `discovery_statuses.device_first_occurrence_time` | string | 97% |  |
| `discovery_statuses.device_group_message` | string | 97% |  |
| `discovery_statuses.device_group_status` | string | 97% | `"OK"` |
| `discovery_statuses.device_group_timestamp` | string | 97% |  |
| `discovery_statuses.device_id` | string | 97% |  |
| `discovery_statuses.device_ip_dotted` | string | 97% |  |
| `discovery_statuses.device_ip_numeric` | string | 97% |  |
| `discovery_statuses.device_name` | string | 97% |  |
| `discovery_statuses.device_support` | string | 19% | `"Advanced"`, `"Basic"` |
| `discovery_statuses.device_type` | string | 97% |  |
| `discovery_statuses.exists_message` | string | 97% |  |
| `discovery_statuses.exists_timestamp` | string | 97% |  |
| `discovery_statuses.finger_print_enabled` | string | 97% |  |
| `discovery_statuses.finger_print_message` | string | 90% |  |
| `discovery_statuses.finger_print_status` | string | 90% | `"Error"`, `"OK"` |
| `discovery_statuses.finger_print_timestamp` | string | 90% |  |
| `discovery_statuses.global_config_collection_enabled` | string | 97% |  |
| `discovery_statuses.global_config_collection_message` | string | 97% |  |
| `discovery_statuses.is_access` | string | 35% | `"N/A"` |
| `discovery_statuses.is_forwarding` | string | 35% | `"N/A"` |
| `discovery_statuses.last_action` | string | 94% |  |
| `discovery_statuses.last_seen` | string | 97% |  |
| `discovery_statuses.last_timestamp` | string | 94% |  |
| `discovery_statuses.license_override` | string | 97% |  |
| `discovery_statuses.overall_status` | string | 97% | `"ERROR"`, `"OK"` |
| `discovery_statuses.reachable_message` | string | 94% |  |
| `discovery_statuses.reachable_status` | string | 94% | `"Error"`, `"OK"` |
| `discovery_statuses.reachable_timestamp` | string | 94% |  |
| `discovery_statuses.snmp_collection_enabled` | boolean | 97% | `true` |
| `discovery_statuses.snmp_collection_message` | string | 35% |  |
| `discovery_statuses.snmp_collection_status` | string | 35% | `"Error"`, `"OK"`, `"Running"` |
| `discovery_statuses.snmp_collection_timestamp` | string | 35% |  |
| `discovery_statuses.snmp_credential_message` | string | 94% |  |
| `discovery_statuses.snmp_credential_status` | string | 94% | `"Error"`, `"OK"` |
| `discovery_statuses.snmp_credential_timestamp` | string | 94% |  |
| `discovery_statuses.status_detail` | string | 65% | `"Unlicensed"` |
| `discovery_statuses.unit_id` | string | 97% |  |
| `discovery_statuses.updated_time` | string | 97% |  |
| `interfaces[].admin_status` | string | 32% | `"enabled"` |
| `interfaces[].attached_devices[].cq_id` | string | 3% |  |
| `interfaces[].attached_devices[].device_name` | string | 3% |  |
| `interfaces[].attached_devices[].device_type` | string | 3% | `"Switch"` |
| `interfaces[].attached_devices[].interface_name` | string | 3% |  |
| `interfaces[].attached_devices[].ip_address` | string | 3% |  |
| `interfaces[].attached_devices[].mac_address` | string | 3% |  |
| `interfaces[].attached_devices[].metadata.device_id` | string | 3% |  |
| `interfaces[].attached_devices[].metadata.interface_id` | string | 3% |  |
| `interfaces[].attached_devices[].metadata.neighbor_device_id` | string | 3% |  |
| `interfaces[].attached_devices[].metadata.neighbor_id` | string | 3% |  |
| `interfaces[].attached_devices[].metadata.neighbor_timestamp` | string | 3% |  |
| `interfaces[].attached_devices[].metadata.sources[]` | string | 3% | `"switch_fwd"` |
| `interfaces[].cidrs[]` | string | 29% |  |
| `interfaces[].description` | string | 32% |  |
| `interfaces[].interface_classification` | string | 32% | `"layer2"`, `"layer3"` |
| `interfaces[].ip_addresses[]` | string | 94% |  |
| `interfaces[].ip_ipspace_id.<ip>.ni` | string | 32% |  |
| `interfaces[].mac_address` | string | 68% |  |
| `interfaces[].name` | string | 32% |  |
| `interfaces[].oper_status` | string | 32% | `"down"`, `"up"` |
| `interfaces[].port_name` | string | 29% |  |
| `interfaces[].speed_mbps` | number | 3% |  |
| `interfaces[].switch_ip_address` | string | 29% |  |
| `interfaces[].switch_name` | string | 29% |  |
| `interfaces[].type` | string | 32% | `"ethernet-csmacd"` |
| `is_infra_device` | boolean | 97% | `false`, `true` |
| `is_network_device` | boolean | 97% | `false`, `true` |
| `is_virtual` | boolean | 97% | `false` |
| `ophid` | string | 97% |  |
| `updated_at` | string | 65% |  |
| `updated_time` | string | 97% |  |
| `virtual_network_id` | string | 90% |  |
