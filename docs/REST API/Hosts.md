# Hosts

- [On the different timestamps in the host data structure](#on-the-different-timestamps-in-the-host-data-structure)
- [List hosts](#list-hosts)
- [Count hosts](#count-hosts)
- [Get hosts summary](#get-hosts-summary)
- [Get host](#get-host)
- [Get host by identifier](#get-host-by-identifier)
- [Delete host](#delete-host)
- [Refetch host](#refetch-host)
- [Transfer hosts to a team](#transfer-hosts-to-a-team)
- [Transfer hosts to a team by filter](#transfer-hosts-to-a-team-by-filter)
- [Bulk delete hosts by filter or ids](#bulk-delete-hosts-by-filter-or-ids)
- [Get host's Google Chrome profiles](#get-hosts-google-chrome-profiles)
- [Get host's mobile device management (MDM) information](#get-hosts-mobile-device-management-mdm-information)
- [Get mobile device management (MDM) summary](#get-mobile-device-management-mdm-summary)
- [Get host's macadmin mobile device management (MDM) and Munki information](#get-hosts-macadmin-mobile-device-management-mdm-and-munki-information)
- [Get aggregated host's mobile device management (MDM) and Munki information](#get-aggregated-hosts-macadmin-mobile-device-management-mdm-and-munki-information)
- [Get host OS versions](#get-host-os-versions)
- [Get hosts report in CSV](#get-hosts-report-in-csv)
- [Get host's disk encryption key](#get-hosts-disk-encryption-key)

## On the different timestamps in the host data structure

Hosts have a set of timestamps usually named with an "_at" suffix, such as created_at, enrolled_at, etc. Before we go
through each of them and what they mean, we need to understand a bit more about how the host data structure is
represented in the database.

The table `hosts` is the main one. It holds the core data for a host. A host doesn't exist if there is no row for it in
this table. This table also holds most of the timestamps, but it doesn't hold all of the host data. This is an important
detail as we'll see below.

There's adjacent tables to this one that usually follow the name convention `host_<extra data descriptor>`. Examples of
this are: `host_additional` that holds additional query results, `host_software` that links a host with many rows from
the `software` table.

- `created_at`: the time the row in the database was created, which usually corresponds to the first enrollment of the host.
- `updated_at`: the last time the row in the database for the `hosts` table was updated.
- `detail_updated_at`: the last time Fleet updated host data, based on the results from the detail queries (this includes updates to host associated tables, e.g. `host_users`).
- `label_updated_at`: the last time Fleet updated the label membership for the host based on the results from the queries ran.
- `last_enrolled_at`: the last time the host enrolled to Fleet.
- `policy_updated_at`: the last time we updated the policy results for the host based on the queries ran.
- `seen_time`: the last time the host contacted the fleet server, regardless of what operation it was for.
- `software_updated_at`: the last time software changed for the host in any way.

## List hosts

`GET /api/v1/fleet/hosts`

#### Parameters

| Name                    | Type    | In    | Description                                                                                                                                                                                                                                                                                                                                 |
| ----------------------- | ------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| page                    | integer | query | Page number of the results to fetch.                                                                                                                                                                                                                                                                                                        |
| per_page                | integer | query | Results per page.                                                                                                                                                                                                                                                                                                                           |
| order_key               | string  | query | What to order results by. Can be any column in the hosts table.                                                                                                                                                                                                                                                                             |
| after                   | string  | query | The value to get results after. This needs `order_key` defined, as that's the column that would be used.                                                                                                                                                                                                                                     |
| order_direction         | string  | query | **Requires `order_key`**. The direction of the order given the order key. Options include `asc` and `desc`. Default is `asc`.                                                                                                                                                                                                               |
| status                  | string  | query | Indicates the status of the hosts to return. Can either be `new`, `online`, `offline`, `mia` or `missing`.                                                                                                                                                                                                                                  |
| query                   | string  | query | Search query keywords. Searchable fields include `hostname`, `machine_serial`, `uuid`, `ipv4` and the hosts' email addresses (only searched if the query looks like an email address, i.e. contains an `@`, no space, etc.).                                                                                                                |
| additional_info_filters | string  | query | A comma-delimited list of fields to include in each host's additional information object. See [Fleet Configuration Options](https://fleetdm.com/docs/using-fleet/fleetctl-cli#fleet-configuration-options) for an example configuration with hosts' additional information. Use `*` to get all stored fields.                                                  |
| team_id                 | integer | query | _Available in Fleet Premium_ Filters the hosts to only include hosts in the specified team.                                                                                                                                                                                                                                                 |
| policy_id               | integer | query | The ID of the policy to filter hosts by.                                                                                                                                                                                                                                                                                                    |
| policy_response         | string  | query | Valid options are `passing` or `failing`.  `policy_id` must also be specified with `policy_response`.                                                                                                                                                                                                                                       |
| software_id             | integer | query | The ID of the software to filter hosts by.                                                                                                                                                                                                                                                                                                  |
| os_id                   | integer | query | The ID of the operating system to filter hosts by.                                                                                                                                                                                                                                                                                          |
| os_name                 | string  | query | The name of the operating system to filter hosts by. `os_version` must also be specified with `os_name`                                                                                                                                                                                                                                     |
| os_version              | string  | query | The version of the operating system to filter hosts by. `os_name` must also be specified with `os_version`                                                                                                                                                                                                                                  |
| device_mapping          | boolean | query | Indicates whether `device_mapping` should be included for each host. See ["Get host's Google Chrome profiles](#get-hosts-google-chrome-profiles) for more information about this feature.                                                                                                                                                  |
| mdm_id                  | integer | query | The ID of the _mobile device management_ (MDM) solution to filter hosts by (that is, filter hosts that use a specific MDM provider and URL).                                                                                                                                                                                                |
| mdm_name                | string  | query | The name of the _mobile device management_ (MDM) solution to filter hosts by (that is, filter hosts that use a specific MDM provider).                                                                                                                                                                                                |
| mdm_enrollment_status   | string  | query | The _mobile device management_ (MDM) enrollment status to filter hosts by. Can be one of 'manual', 'automatic', 'enrolled', 'pending', or 'unenrolled'.                                                                                                                                                                                                             |
| macos_settings          | string  | query | Filters the hosts by the status of the _mobile device management_ (MDM) profiles applied to hosts. Can be one of 'verified', 'verifying', 'pending', or 'failed'. **Note: If this filter is used in Fleet Premium without a team id filter, the results include only hosts that are not assigned to any team.**                                                                                                                                                                                                             |
| munki_issue_id          | integer | query | The ID of the _munki issue_ (a Munki-reported error or warning message) to filter hosts by (that is, filter hosts that are affected by that corresponding error or warning message).                                                                                                                                                        |
| low_disk_space          | integer | query | _Available in Fleet Premium_ Filters the hosts to only include hosts with less GB of disk space available than this value. Must be a number between 1-100.                                                                                                                                                                                  |
| disable_failing_policies| boolean | query | If "true", hosts will return failing policies as 0 regardless of whether there are any that failed for the host. This is meant to be used when increased performance is needed in exchange for the extra information.                                                                                                                       |
| macos_settings_disk_encryption | string | query | Filters the hosts by the status of the macOS disk encryption MDM profile on the host. Can be one of `verified`, `verifying`, `action_required`, `enforcing`, `failed`, or `removing_enforcement`. |
| bootstrap_package       | string | query | _Available in Fleet Premium_ Filters the hosts by the status of the MDM bootstrap package on the host. Can be one of `installed`, `pending`, or `failed`. |

If `additional_info_filters` is not specified, no `additional` information will be returned.

If `software_id` is specified, an additional top-level key `"software"` is returned with the software object corresponding to the `software_id`. See [List all software](./Software.md#list-all-software) response payload for details about this object.

If `mdm_id` is specified, an additional top-level key `"mobile_device_management_solution"` is returned with the information corresponding to the `mdm_id`.

If `mdm_id`, `mdm_name` or `mdm_enrollment_status` is specified, then Windows Servers are excluded from the results.

If `munki_issue_id` is specified, an additional top-level key `"munki_issue"` is returned with the information corresponding to the `munki_issue_id`.

If `after` is being used with `created_at` or `updated_at`, the table must be specified in `order_key`. Those columns become `h.created_at` and `h.updated_at`.

#### Example

`GET /api/v1/fleet/hosts?page=0&per_page=100&order_key=hostname&query=2ce`

##### Request query parameters

```json
{
  "page": 0,
  "per_page": 100,
  "order_key": "hostname"
}
```

##### Default response

`Status: 200`

```json
{
  "hosts": [
    {
      "created_at": "2020-11-05T05:09:44Z",
      "updated_at": "2020-11-05T06:03:39Z",
      "id": 1,
      "detail_updated_at": "2020-11-05T05:09:45Z",
      "software_updated_at": "2020-11-05T05:09:44Z",
      "label_updated_at": "2020-11-05T05:14:51Z",
      "policy_updated_at": "2023-06-26T18:33:15Z",
      "last_enrolled_at": "2023-02-26T22:33:12Z",
      "seen_time": "2020-11-05T06:03:39Z",
      "hostname": "2ceca32fe484",
      "uuid": "392547dc-0000-0000-a87a-d701ff75bc65",
      "platform": "centos",
      "osquery_version": "2.7.0",
      "os_version": "CentOS Linux 7",
      "build": "",
      "platform_like": "rhel fedora",
      "code_name": "",
      "uptime": 8305000000000,
      "memory": 2084032512,
      "cpu_type": "6",
      "cpu_subtype": "142",
      "cpu_brand": "Intel(R) Core(TM) i5-8279U CPU @ 2.40GHz",
      "cpu_physical_cores": 4,
      "cpu_logical_cores": 4,
      "hardware_vendor": "",
      "hardware_model": "",
      "hardware_version": "",
      "hardware_serial": "",
      "computer_name": "2ceca32fe484",
      "display_name": "2ceca32fe484",
      "public_ip": "",
      "primary_ip": "",
      "primary_mac": "",
      "distributed_interval": 10,
      "config_tls_refresh": 10,
      "logger_tls_period": 8,
      "additional": {},
      "status": "offline",
      "display_text": "2ceca32fe484",
      "team_id": null,
      "team_name": null,
      "pack_stats": null,
      "issues": {
        "failing_policies_count": 2,
        "total_issues_count": 2
      },
      "geolocation": {
        "country_iso": "US",
        "city_name": "New York",
        "geometry": {
          "type": "point",
          "coordinates": [40.6799, -74.0028]
        }
      },
      "mdm": {
        "encryption_key_available": false,
        "enrollment_status": null,
        "name": "",
        "server_url": null
      }
    }
  ]
}
```

> Note: the response above assumes a [GeoIP database is configured](https://fleetdm.com/docs/deploying/configuration#geoip), otherwise the `geolocation` object won't be included.

Response payload with the `mdm_id` filter provided:

```json
{
  "hosts": [...],
  "mobile_device_management_solution": {
    "server_url": "http://some.url/mdm",
    "name": "MDM Vendor Name",
    "id": 999
  }
}
```

Response payload with the `munki_issue_id` filter provided:

```json
{
  "hosts": [...],
  "munki_issue": {
    "id": 1,
    "name": "Could not retrieve managed install primary manifest",
    "type": "error"
  }
}
```

## Count hosts

`GET /api/v1/fleet/hosts/count`

#### Parameters

| Name                    | Type    | In    | Description                                                                                                                                                                                                                                                                                                                                 |
| ----------------------- | ------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| order_key               | string  | query | What to order results by. Can be any column in the hosts table.                                                                                                                                                                                                                                                                             |
| order_direction         | string  | query | **Requires `order_key`**. The direction of the order given the order key. Options include `asc` and `desc`. Default is `asc`.                                                                                                                                                                                                               |
| after                   | string  | query | The value to get results after. This needs `order_key` defined, as that's the column that would be used.                                                                                                                                                                                                                                    |
| status                  | string  | query | Indicates the status of the hosts to return. Can either be `new`, `online`, `offline`, `mia` or `missing`.                                                                                                                                                                                                                                  |
| query                   | string  | query | Search query keywords. Searchable fields include `hostname`, `machine_serial`, `uuid`, `ipv4` and the hosts' email addresses (only searched if the query looks like an email address, i.e. contains an `@`, no space, etc.).                                                                                                                |
| team_id                 | integer | query | _Available in Fleet Premium_ Filters the hosts to only include hosts in the specified team.                                                                                                                                                                                                                                                 |
| policy_id               | integer | query | The ID of the policy to filter hosts by.                                                                                                                                                                                                                                                                                                    |
| policy_response         | string  | query | Valid options are `passing` or `failing`.  `policy_id` must also be specified with `policy_response`.                                                                                                                                                                                                                                       |
| software_id             | integer | query | The ID of the software to filter hosts by.                                                                                                                                                                                                                                                                                                  |
| os_id                   | integer | query | The ID of the operating system to filter hosts by.                                                                                                                                                                                                                                                                                          |
| os_name                 | string  | query | The name of the operating system to filter hosts by. `os_version` must also be specified with `os_name`                                                                                                                                                                                                                                     |
| os_version              | string  | query | The version of the operating system to filter hosts by. `os_name` must also be specified with `os_version`                                                                                                                                                                                                                                  |
| label_id                | integer | query | A valid label ID. Can only be used in combination with `order_key`, `order_direction`, `after`, `status`, `query` and `team_id`.                                                                                                                                                                                                            |
| mdm_id                  | integer | query | The ID of the _mobile device management_ (MDM) solution to filter hosts by (that is, filter hosts that use a specific MDM provider and URL).                                                                                                                                                                                                |
| mdm_name                | string  | query | The name of the _mobile device management_ (MDM) solution to filter hosts by (that is, filter hosts that use a specific MDM provider).                                                                                                                                                                                                |
| mdm_enrollment_status   | string  | query | The _mobile device management_ (MDM) enrollment status to filter hosts by. Can be one of 'manual', 'automatic', 'enrolled', 'pending', or 'unenrolled'.                                                                                                                                                                                                             |
| macos_settings          | string  | query | Filters the hosts by the status of the _mobile device management_ (MDM) profiles applied to hosts. Can be one of 'verified', 'verifying', 'pending', or 'failed'. **Note: If this filter is used in Fleet Premium without a team id filter, the results include only hosts that are not assigned to any team.**                                                                                                                                                                                                             |
| munki_issue_id          | integer | query | The ID of the _munki issue_ (a Munki-reported error or warning message) to filter hosts by (that is, filter hosts that are affected by that corresponding error or warning message).                                                                                                                                                        |
| low_disk_space          | integer | query | _Available in Fleet Premium_ Filters the hosts to only include hosts with less GB of disk space available than this value. Must be a number between 1-100.                                                                                                                                                                                  |
| macos_settings_disk_encryption | string | query | Filters the hosts by the status of the macOS disk encryption MDM profile on the host. Can be one of `verified`, `verifying`, `action_required`, `enforcing`, `failed`, or `removing_enforcement`. |
| bootstrap_package       | string | query | _Available in Fleet Premium_ Filters the hosts by the status of the MDM bootstrap package on the host. Can be one of `installed`, `pending`, or `failed`. **Note: If this filter is used in Fleet Premium without a team id filter, the results include only hosts that are not assigned to any team.** |

If `additional_info_filters` is not specified, no `additional` information will be returned.

If `mdm_id`, `mdm_name` or `mdm_enrollment_status` is specified, then Windows Servers are excluded from the results.

#### Example

`GET /api/v1/fleet/hosts/count?page=0&per_page=100&order_key=hostname&query=2ce`

##### Request query parameters

```json
{
  "page": 0,
  "per_page": 100,
  "order_key": "hostname"
}
```

##### Default response

`Status: 200`

```json
{
  "count": 123
}
```

## Get hosts summary

Returns the count of all hosts organized by status. `online_count` includes all hosts currently enrolled in Fleet. `offline_count` includes all hosts that haven't checked into Fleet recently. `mia_count` includes all hosts that haven't been seen by Fleet in more than 30 days. `new_count` includes the hosts that have been enrolled to Fleet in the last 24 hours.

`GET /api/v1/fleet/host_summary`

#### Parameters

| Name            | Type    | In    | Description                                                                     |
| --------------- | ------- | ----  | ------------------------------------------------------------------------------- |
| team_id         | integer | query | The ID of the team whose host counts should be included. Defaults to all teams. |
| platform        | string  | query | Platform to filter by when counting. Defaults to all platforms.                 |
| low_disk_space  | integer | query | _Available in Fleet Premium_ Returns the count of hosts with less GB of disk space available than this value. Must be a number between 1-100. |

#### Example

`GET /api/v1/fleet/host_summary?team_id=1&low_disk_space=32`

##### Default response

`Status: 200`

```json
{
  "team_id": 1,
  "totals_hosts_count": 2408,
  "online_count": 2267,
  "offline_count": 141,
  "mia_count": 0,
  "missing_30_days_count": 0,
  "new_count": 0,
  "all_linux_count": 1204,
  "low_disk_space_count": 12,
  "builtin_labels": [
    {
      "id": 6,
      "name": "All Hosts",
      "description": "All hosts which have enrolled in Fleet",
      "label_type": "builtin"
    },
    {
      "id": 7,
      "name": "macOS",
      "description": "All macOS hosts",
      "label_type": "builtin"
    },
    {
      "id": 8,
      "name": "Ubuntu Linux",
      "description": "All Ubuntu hosts",
      "label_type": "builtin"
    },
    {
      "id": 9,
      "name": "CentOS Linux",
      "description": "All CentOS hosts",
      "label_type": "builtin"
    },
    {
      "id": 10,
      "name": "MS Windows",
      "description": "All Windows hosts",
      "label_type": "builtin"
    },
    {
      "id": 11,
      "name": "Red Hat Linux",
      "description": "All Red Hat Enterprise Linux hosts",
      "label_type": "builtin"
    },
    {
      "id": 12,
      "name": "All Linux",
      "description": "All Linux distributions",
      "label_type": "builtin"
    }
  ],
  "platforms": [
    {
      "platform": "chrome",
      "hosts_count": 1234
    },
    {
      "platform": "darwin",
      "hosts_count": 1234
    },
    {
      "platform": "rhel",
      "hosts_count": 1234
    },
    {
      "platform": "ubuntu",
      "hosts_count": 12044
    },
    {
      "platform": "windows",
      "hosts_count": 12044
    }

  ]
}
```

## Get host

Returns the information of the specified host.

`GET /api/v1/fleet/hosts/{id}`

#### Parameters

| Name | Type    | In   | Description                  |
| ---- | ------- | ---- | ---------------------------- |
| id   | integer | path | **Required**. The host's id. |

#### Example

`GET /api/v1/fleet/hosts/121`

##### Default response

`Status: 200`

```json
{
  "host": {
    "created_at": "2021-08-19T02:02:22Z",
    "updated_at": "2021-08-19T21:14:58Z",
    "software": [
      {
        "id": 408,
        "name": "osquery",
        "version": "4.5.1",
        "source": "rpm_packages",
        "generated_cpe": "",
        "vulnerabilities": null,
        "installed_paths": ["/usr/lib/some-path-1"]
      },
      {
        "id": 1146,
        "name": "tar",
        "version": "1.30",
        "source": "rpm_packages",
        "generated_cpe": "",
        "vulnerabilities": null
      },
      {
        "id": 321,
        "name": "SomeApp.app",
        "version": "1.0",
        "source": "apps",
        "bundle_identifier": "com.some.app",
        "last_opened_at": "2021-08-18T21:14:00Z",
        "generated_cpe": "",
        "vulnerabilities": null,
        "installed_paths": ["/usr/lib/some-path-2"]
      }
    ],
    "id": 1,
    "detail_updated_at": "2021-08-19T21:07:53Z",
    "software_updated_at": "2020-11-05T05:09:44Z",
    "label_updated_at": "2021-08-19T21:07:53Z",
    "policy_updated_at": "2023-06-26T18:33:15Z",
    "last_enrolled_at": "2021-08-19T02:02:22Z",
    "seen_time": "2021-08-19T21:14:58Z",
    "refetch_requested": false,
    "hostname": "23cfc9caacf0",
    "uuid": "309a4b7d-0000-0000-8e7f-26ae0815ede8",
    "platform": "rhel",
    "osquery_version": "4.5.1",
    "os_version": "CentOS Linux 8.3.2011",
    "build": "",
    "platform_like": "rhel",
    "code_name": "",
    "uptime": 210671000000000,
    "memory": 16788398080,
    "cpu_type": "x86_64",
    "cpu_subtype": "158",
    "cpu_brand": "Intel(R) Core(TM) i9-9980HK CPU @ 2.40GHz",
    "cpu_physical_cores": 12,
    "cpu_logical_cores": 12,
    "hardware_vendor": "",
    "hardware_model": "",
    "hardware_version": "",
    "hardware_serial": "",
    "computer_name": "23cfc9caacf0",
    "display_name": "23cfc9caacf0",
    "public_ip": "",
    "primary_ip": "172.27.0.6",
    "primary_mac": "02:42:ac:1b:00:06",
    "distributed_interval": 10,
    "config_tls_refresh": 10,
    "logger_tls_period": 10,
    "team_id": null,
    "pack_stats": null,
    "team_name": null,
    "additional": {},
    "gigs_disk_space_available": 46.1,
    "percent_disk_space_available": 73,
    "disk_encryption_enabled": true,
    "users": [
      {
        "uid": 0,
        "username": "root",
        "type": "",
        "groupname": "root",
        "shell": "/bin/bash"
      },
      {
        "uid": 1,
        "username": "bin",
        "type": "",
        "groupname": "bin",
        "shell": "/sbin/nologin"
      }
    ],
    "labels": [
      {
        "created_at": "2021-08-19T02:02:17Z",
        "updated_at": "2021-08-19T02:02:17Z",
        "id": 6,
        "name": "All Hosts",
        "description": "All hosts which have enrolled in Fleet",
        "query": "SELECT 1;",
        "platform": "",
        "label_type": "builtin",
        "label_membership_type": "dynamic"
      },
      {
        "created_at": "2021-08-19T02:02:17Z",
        "updated_at": "2021-08-19T02:02:17Z",
        "id": 9,
        "name": "CentOS Linux",
        "description": "All CentOS hosts",
        "query": "SELECT 1 FROM os_version WHERE platform = 'centos' OR name LIKE '%centos%'",
        "platform": "",
        "label_type": "builtin",
        "label_membership_type": "dynamic"
      },
      {
        "created_at": "2021-08-19T02:02:17Z",
        "updated_at": "2021-08-19T02:02:17Z",
        "id": 12,
        "name": "All Linux",
        "description": "All Linux distributions",
        "query": "SELECT 1 FROM osquery_info WHERE build_platform LIKE '%ubuntu%' OR build_distro LIKE '%centos%';",
        "platform": "",
        "label_type": "builtin",
        "label_membership_type": "dynamic"
      }
    ],
    "packs": [],
    "status": "online",
    "display_text": "23cfc9caacf0",
    "policies": [
      {
        "id": 1,
        "name": "SomeQuery",
        "query": "SELECT * FROM foo;",
        "description": "this is a query",
        "resolution": "fix with these steps...",
        "platform": "windows,linux",
        "response": "pass",
        "critical": false
      },
      {
        "id": 2,
        "name": "SomeQuery2",
        "query": "SELECT * FROM bar;",
        "description": "this is another query",
        "resolution": "fix with these other steps...",
        "platform": "darwin",
        "response": "fail",
        "critical": false
      },
      {
        "id": 3,
        "name": "SomeQuery3",
        "query": "SELECT * FROM baz;",
        "description": "",
        "resolution": "",
        "platform": "",
        "response": "",
        "critical": false
      }
    ],
    "issues": {
      "failing_policies_count": 2,
      "total_issues_count": 2
    },
    "batteries": [
      {
        "cycle_count": 999,
        "health": "Normal"
      }
    ],
    "geolocation": {
      "country_iso": "US",
      "city_name": "New York",
      "geometry": {
        "type": "point",
        "coordinates": [40.6799, -74.0028]
      }
    },
    "mdm": {
      "encryption_key_available": false,
      "enrollment_status": null,
      "name": "",
      "server_url": null,
      "macos_settings": {
        "disk_encryption": null,
        "action_required": null
      },
      "macos_setup": {
        "bootstrap_package_status": "installed",
        "detail": "",
        "bootstrap_package_name": "test.pkg"
      },
      "profiles": [
        {
          "profile_id": 999,
          "name": "profile1",
          "status": "verifying",
          "operation_type": "install",
          "detail": ""
        }
      ]
    }
  }
}
```

> Note: the response above assumes a [GeoIP database is configured](https://fleetdm.com/docs/deploying/configuration#geoip), otherwise the `geolocation` object won't be included.

## Get host by identifier

Returns the information of the host specified using the `uuid`, `osquery_host_id`, `hostname`, or
`node_key` as an identifier

`GET /api/v1/fleet/hosts/identifier/{identifier}`

#### Parameters

| Name       | Type              | In   | Description                                                                   |
| ---------- | ----------------- | ---- | ----------------------------------------------------------------------------- |
| identifier | integer or string | path | **Required**. The host's `uuid`, `osquery_host_id`, `hostname`, or `node_key` |

#### Example

`GET /api/v1/fleet/hosts/identifier/392547dc-0000-0000-a87a-d701ff75bc65`

##### Default response

`Status: 200`

```json
{
  "host": {
    "created_at": "2022-02-10T02:29:13Z",
    "updated_at": "2022-10-14T17:07:11Z",
    "software": [
      {
          "id": 16923,
          "name": "Automat",
          "version": "0.8.0",
          "source": "python_packages",
          "generated_cpe": "",
          "vulnerabilities": null,
          "installed_paths": ["/usr/lib/some_path/"]
      }
    ],
    "id": 33,
    "detail_updated_at": "2022-10-14T17:07:12Z",
    "label_updated_at": "2022-10-14T17:07:12Z",
    "policy_updated_at": "2022-10-14T17:07:12Z",
    "last_enrolled_at": "2022-02-10T02:29:13Z",
    "software_updated_at": "2020-11-05T05:09:44Z",
    "seen_time": "2022-10-14T17:45:41Z",
    "refetch_requested": false,
    "hostname": "23cfc9caacf0",
    "uuid": "392547dc-0000-0000-a87a-d701ff75bc65",
    "platform": "ubuntu",
    "osquery_version": "5.5.1",
    "os_version": "Ubuntu 20.04.3 LTS",
    "build": "",
    "platform_like": "debian",
    "code_name": "focal",
    "uptime": 20807520000000000,
    "memory": 1024360448,
    "cpu_type": "x86_64",
    "cpu_subtype": "63",
    "cpu_brand": "DO-Regular",
    "cpu_physical_cores": 1,
    "cpu_logical_cores": 1,
    "hardware_vendor": "",
    "hardware_model": "",
    "hardware_version": "",
    "hardware_serial": "",
    "computer_name": "23cfc9caacf0",
    "public_ip": "",
    "primary_ip": "172.27.0.6",
    "primary_mac": "02:42:ac:1b:00:06",
    "distributed_interval": 10,
    "config_tls_refresh": 60,
    "logger_tls_period": 10,
    "team_id": 2,
    "pack_stats": [
      {
        "pack_id": 1,
        "pack_name": "Global",
        "type": "global",
        "query_stats": [
          {
            "scheduled_query_name": "Get running processes (with user_name)",
            "scheduled_query_id": 49,
            "query_name": "Get running processes (with user_name)",
            "pack_name": "Global",
            "pack_id": 1,
            "average_memory": 260000,
            "denylisted": false,
            "executions": 1,
            "interval": 86400,
            "last_executed": "2022-10-14T10:00:01Z",
            "output_size": 198,
            "system_time": 20,
            "user_time": 80,
            "wall_time": 0
          }
        ]
      }
    ],
    "team_name": null,
    "gigs_disk_space_available": 19.29,
    "percent_disk_space_available": 74,
    "issues": {
        "total_issues_count": 0,
        "failing_policies_count": 0
    },
    "labels": [
            {
            "created_at": "2021-09-14T05:11:02Z",
            "updated_at": "2021-09-14T05:11:02Z",
            "id": 12,
            "name": "All Linux",
            "description": "All Linux distributions",
            "query": "SELECT 1 FROM osquery_info WHERE build_platform LIKE '%ubuntu%' OR build_distro LIKE '%centos%';",
            "platform": "",
            "label_type": "builtin",
            "label_membership_type": "dynamic"
        }
    ],
    "packs": [
          {
            "created_at": "2021-09-17T05:28:54Z",
            "updated_at": "2021-09-17T05:28:54Z",
            "id": 1,
            "name": "Global",
            "description": "Global pack",
            "disabled": false,
            "type": "global",
            "labels": null,
            "label_ids": null,
            "hosts": null,
            "host_ids": null,
            "teams": null,
            "team_ids": null
        }
    ],
    "policies": [
      {
            "id": 142,
            "name": "Full disk encryption enabled (macOS)",
            "query": "SELECT 1 FROM disk_encryption WHERE user_uuid IS NOT '' AND filevault_status = 'on' LIMIT 1;",
            "description": "Checks to make sure that full disk encryption (FileVault) is enabled on macOS devices.",
            "author_id": 31,
            "author_name": "",
            "author_email": "",
            "team_id": null,
            "resolution": "To enable full disk encryption, on the failing device, select System Preferences > Security & Privacy > FileVault > Turn On FileVault.",
            "platform": "darwin,linux",
            "created_at": "2022-09-02T18:52:19Z",
            "updated_at": "2022-09-02T18:52:19Z",
            "response": "fail",
            "critical": false
        }
    ],
    "batteries": [
      {
        "cycle_count": 999,
        "health": "Normal"
      }
    ],
    "geolocation": {
      "country_iso": "US",
      "city_name": "New York",
      "geometry": {
        "type": "point",
        "coordinates": [40.6799, -74.0028]
      }
    },
    "status": "online",
    "display_text": "dogfood-ubuntu-box",
    "display_name": "dogfood-ubuntu-box",
    "mdm": {
      "encryption_key_available": false,
      "enrollment_status": null,
      "name": "",
      "server_url": null,
      "macos_settings": {
        "disk_encryption": null,
        "action_required": null
      },
      "macos_setup": {
        "bootstrap_package_status": "installed",
        "detail": ""
      },
      "profiles": [
        {
          "profile_id": 999,
          "name": "profile1",
          "status": "verifying",
          "operation_type": "install",
          "detail": ""
        }
      ]
    }
  }
}
```

> Note: the response above assumes a [GeoIP database is configured](https://fleetdm.com/docs/deploying/configuration#geoip), otherwise the `geolocation` object won't be included.

## Delete host

Deletes the specified host from Fleet. Note that a deleted host will fail authentication with the previous node key, and in most osquery configurations will attempt to re-enroll automatically. If the host still has a valid enroll secret, it will re-enroll successfully.

`DELETE /api/v1/fleet/hosts/{id}`

#### Parameters

| Name | Type    | In   | Description                  |
| ---- | ------- | ---- | ---------------------------- |
| id   | integer | path | **Required**. The host's id. |

#### Example

`DELETE /api/v1/fleet/hosts/121`

##### Default response

`Status: 200`


## Refetch host

Flags the host details, labels and policies to be refetched the next time the host checks in for distributed queries. Note that we cannot be certain when the host will actually check in and update the query results. Further requests to the host APIs will indicate that the refetch has been requested through the `refetch_requested` field on the host object.

`POST /api/v1/fleet/hosts/{id}/refetch`

#### Parameters

| Name | Type    | In   | Description                  |
| ---- | ------- | ---- | ---------------------------- |
| id   | integer | path | **Required**. The host's id. |

#### Example

`POST /api/v1/fleet/hosts/121/refetch`

##### Default response

`Status: 200`


## Transfer hosts to a team

_Available in Fleet Premium_

`POST /api/v1/fleet/hosts/transfer`

#### Parameters

| Name    | Type    | In   | Description                                                             |
| ------- | ------- | ---- | ----------------------------------------------------------------------- |
| team_id | integer | body | **Required**. The ID of the team you'd like to transfer the host(s) to. |
| hosts   | array   | body | **Required**. A list of host IDs.                                       |

#### Example

`POST /api/v1/fleet/hosts/transfer`

##### Request body

```json
{
  "team_id": 1,
  "hosts": [3, 2, 4, 6, 1, 5, 7]
}
```

##### Default response

`Status: 200`


## Transfer hosts to a team by filter

_Available in Fleet Premium_

`POST /api/v1/fleet/hosts/transfer/filter`

#### Parameters

| Name    | Type    | In   | Description                                                                                                                                                                                                                                                                                                                        |
| ------- | ------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| team_id | integer | body | **Required**. The ID of the team you'd like to transfer the host(s) to.                                                                                                                                                                                                                                                            |
| filters | object  | body | **Required** Contains any of the following three properties: `query` for search query keywords. Searchable fields include `hostname`, `machine_serial`, `uuid`, and `ipv4`. `status` to indicate the status of the hosts to return. Can either be `new`, `online`, `offline`, `mia` or `missing`. `label_id` to indicate the selected label. `label_id` and `status` cannot be used at the same time. |

#### Example

`POST /api/v1/fleet/hosts/transfer/filter`

##### Request body

```json
{
  "team_id": 1,
  "filters": {
    "status": "online"
  }
}
```

##### Default response

`Status: 200`

## Bulk delete hosts by filter or ids

`POST /api/v1/fleet/hosts/delete`

#### Parameters

| Name    | Type    | In   | Description                                                                                                                                                                                                                                                                                                                        |
| ------- | ------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ids     | list    | body | A list of the host IDs you'd like to delete. If `ids` is specified, `filters` cannot be specified.                                                                                                                                                                                                                                                           |
| filters | object  | body | Contains any of the following four properties: `query` for search query keywords. Searchable fields include `hostname`, `machine_serial`, `uuid`, and `ipv4`. `status` to indicate the status of the hosts to return. Can either be `new`, `online`, `offline`, `mia` or `missing`. `label_id` to indicate the selected label. `team_id` to indicate the selected team. If `filters` is specified, `id` cannot be specified. `label_id` and `status` cannot be used at the same time. |

Either ids or filters are required.

Request (`ids` is specified):

```json
{
  "ids": [1]
}
```

Request (`filters` is specified):
```json
{
  "filters": {
    "status": "online",
    "label_id": 1,
    "team_id": 1,
    "query": "abc"
  }
}
```

#### Example

`POST /api/v1/fleet/hosts/delete`

##### Request body

```json
{
  "filters": {
    "status": "online",
    "team_id": 1
  }
}
```

##### Default response

`Status: 200`

## Get host's Google Chrome profiles

Retrieves a host's Google Chrome profile information which can be used to link a host to a specific user by email.

Requires [Fleetd](https://fleetdm.com/docs/using-fleet/fleetd), the osquery manager from Fleet. Fleetd can be built with [fleetctl](https://fleetdm.com/docs/using-fleet/adding-hosts#osquery-installer).

`GET /api/v1/fleet/hosts/{id}/device_mapping`

#### Parameters

| Name       | Type              | In   | Description                                                                   |
| ---------- | ----------------- | ---- | ----------------------------------------------------------------------------- |
| id         | integer           | path | **Required**. The host's `id`.                                                |

#### Example

`GET /api/v1/fleet/hosts/1/device_mapping`

##### Default response

`Status: 200`

```json
{
  "host_id": 1,
  "device_mapping": [
    {
      "email": "user@example.com",
      "source": "google_chrome_profiles"
    }
  ]
}
```

---

## Get host's mobile device management (MDM) information

Currently supports Windows and MacOS. On MacOS this requires the [macadmins osquery
extension](https://github.com/macadmins/osquery-extension) which comes bundled
in [Fleet's osquery installers](https://fleetdm.com/docs/using-fleet/adding-hosts#osquery-installer).

Retrieves a host's MDM enrollment status and MDM server URL.

If the host exists but is not enrolled to an MDM server, then this API returns `null`.

`GET /api/v1/fleet/hosts/{id}/mdm`

#### Parameters

| Name    | Type    | In   | Description                                                                                                                                                                                                                                                                                                                        |
| ------- | ------- | ---- | -------------------------------------------------------------------------------- |
| id      | integer | path | **Required** The id of the host to get the details for                           |

#### Example

`GET /api/v1/fleet/hosts/32/mdm`

##### Default response

`Status: 200`

```json
{
  "enrollment_status": "On (automatic)",
  "server_url": "some.mdm.com",
  "name": "Some MDM",
  "id": 3
}
```

---

## Get mobile device management (MDM) summary

Currently supports Windows and MacOS. On MacOS this requires the [macadmins osquery
extension](https://github.com/macadmins/osquery-extension) which comes bundled
in [Fleet's osquery installers](https://fleetdm.com/docs/using-fleet/adding-hosts#osquery-installer).

Retrieves MDM enrollment summary. Windows servers are excluded from the aggregated data.

`GET /api/v1/fleet/hosts/summary/mdm`

#### Parameters

| Name     | Type    | In    | Description                                                                                                                                                                                                                                                                                                                        |
| -------- | ------- | ----- | -------------------------------------------------------------------------------- |
| team_id  | integer | query | _Available in Fleet Premium_ Filter by team                                      |
| platform | string  | query | Filter by platform ("windows" or "darwin")                                       |

A `team_id` of `0` returns the statistics for hosts that are not part of any team. A `null` or missing `team_id` returns statistics for all hosts regardless of the team.

#### Example

`GET /api/v1/fleet/hosts/summary/mdm?team_id=1&platform=windows`

##### Default response

`Status: 200`

```json
{
  "counts_updated_at": "2021-03-21T12:32:44Z",
  "mobile_device_management_enrollment_status": {
    "enrolled_manual_hosts_count": 0,
    "enrolled_automated_hosts_count": 2,
    "unenrolled_hosts_count": 0,
    "hosts_count": 2
  },
  "mobile_device_management_solution": [
    {
      "id": 2,
      "name": "Solution1",
      "server_url": "solution1.com",
      "hosts_count": 1
    },
    {
      "id": 3,
      "name": "Solution2",
      "server_url": "solution2.com",
      "hosts_count": 1
    }
  ]
}
```

---

## Get host's macadmin mobile device management (MDM) and Munki information

Requires the [macadmins osquery
extension](https://github.com/macadmins/osquery-extension) which comes bundled
in [Fleet's osquery
installers](https://fleetdm.com/docs/using-fleet/adding-hosts#osquery-installer).
Currently supported only on macOS.

Retrieves a host's MDM enrollment status, MDM server URL, and Munki version.

`GET /api/v1/fleet/hosts/{id}/macadmins`

#### Parameters

| Name    | Type    | In   | Description                                                                                                                                                                                                                                                                                                                        |
| ------- | ------- | ---- | -------------------------------------------------------------------------------- |
| id      | integer | path | **Required** The id of the host to get the details for                           |

#### Example

`GET /api/v1/fleet/hosts/32/macadmins`

##### Default response

`Status: 200`

```json
{
  "macadmins": {
    "munki": {
      "version": "1.2.3"
    },
    "munki_issues": [
      {
        "id": 1,
        "name": "Could not retrieve managed install primary manifest",
        "type": "error",
        "created_at": "2022-08-01T05:09:44Z"
      },
      {
        "id": 2,
        "name": "Could not process item Figma for optional install. No pkginfo found in catalogs: release",
        "type": "warning",
        "created_at": "2022-08-01T05:09:44Z"
      }
    ],
    "mobile_device_management": {
      "enrollment_status": "On (automatic)",
      "server_url": "http://some.url/mdm",
      "name": "MDM Vendor Name",
      "id": 999
    }
  }
}
```

---

## Get aggregated host's macadmin mobile device management (MDM) and Munki information

Requires the [macadmins osquery
extension](https://github.com/macadmins/osquery-extension) which comes bundled
in [Fleet's osquery
installers](https://fleetdm.com/docs/using-fleet/adding-hosts#osquery-installer).
Currently supported only on macOS.


Retrieves aggregated host's MDM enrollment status and Munki versions.

`GET /api/v1/fleet/macadmins`

#### Parameters

| Name    | Type    | In    | Description                                                                                                                                                                                                                                                                                                                        |
| ------- | ------- | ----- | ---------------------------------------------------------------------------------------------------------------- |
| team_id | integer | query | _Available in Fleet Premium_ Filters the aggregate host information to only include hosts in the specified team. |                           |

A `team_id` of `0` returns the statistics for hosts that are not part of any team. A `null` or missing `team_id` returns statistics for all hosts regardless of the team.

#### Example

`GET /api/v1/fleet/macadmins`

##### Default response

`Status: 200`

```json
{
  "macadmins": {
    "counts_updated_at": "2021-03-21T12:32:44Z",
    "munki_versions": [
      {
        "version": "5.5",
        "hosts_count": 8360
      },
      {
        "version": "5.4",
        "hosts_count": 1700
      },
      {
        "version": "5.3",
        "hosts_count": 400
      },
      {
        "version": "5.2.3",
        "hosts_count": 112
      },
      {
        "version": "5.2.2",
        "hosts_count": 50
      }
    ],
    "munki_issues": [
      {
        "id": 1,
        "name": "Could not retrieve managed install primary manifest",
        "type": "error",
        "hosts_count": 2851
      },
      {
        "id": 2,
        "name": "Could not process item Figma for optional install. No pkginfo found in catalogs: release",
        "type": "warning",
        "hosts_count": 1983
      }
    ],
    "mobile_device_management_enrollment_status": {
      "enrolled_manual_hosts_count": 124,
      "enrolled_automated_hosts_count": 124,
      "unenrolled_hosts_count": 112
    },
    "mobile_device_management_solution": [
      {
        "id": 1,
        "name": "SimpleMDM",
        "hosts_count": 8360,
        "server_url": "https://a.simplemdm.com/mdm"
      },
      {
        "id": 2,
        "name": "Intune",
        "hosts_count": 1700,
        "server_url": "https://enrollment.manage.microsoft.com"
      }
    ]
  }
}
```

## Get host OS versions

Retrieves the aggregated host OS versions information.

`GET /api/v1/fleet/os_versions`

#### Parameters

| Name                | Type     | In    | Description                                                                                                                          |
| ---      | ---      | ---   | ---                                                                                                                                  |
| team_id             | integer | query | _Available in Fleet Premium_ Filters the hosts to only include hosts in the specified team. If not provided, all hosts are included. |
| platform            | string   | query | Filters the hosts to the specified platform |
| os_name     | string | query | The name of the operating system to filter hosts by. `os_version` must also be specified with `os_name`                                                 |
| os_version    | string | query | The version of the operating system to filter hosts by. `os_name` must also be specified with `os_version`                                                 |

##### Default response

`Status: 200`

```json
{
  "counts_updated_at": "2022-03-22T21:38:31Z",
  "os_versions": [
    {
      "hosts_count": 1,
      "name": "CentOS 6.10.0",
      "name_only": "CentOS",
      "version": "6.10.0",
      "platform": "rhel",
      "os_id": 1
    },
    {
      "hosts_count": 1,
      "name": "CentOS Linux 7.9.2009",
      "name_only": "CentOS",
      "version": "7.9.2009",
      "platform": "rhel",
      "os_id": 2
    },
    {
      "hosts_count": 1,
      "name": "CentOS Linux 8.3.2011",
      "name_only": "CentOS",
      "version": "8.2.2011",
      "platform": "rhel",
      "os_id": 3
    },
    {
      "hosts_count": 1,
      "name": "Debian GNU/Linux 10.0.0",
      "name_only": "Debian GNU/Linux",
      "version": "10.0.0",
      "platform": "debian",
      "os_id": 4
    },
    {
      "hosts_count": 1,
      "name": "Debian GNU/Linux 9.0.0",
      "name_only": "Debian GNU/Linux",
      "version": "9.0.0",
      "platform": "debian",
      "os_id": 5
    },
    {
      "hosts_count": 1,
      "name": "Ubuntu 16.4.0 LTS",
      "name_only": "Ubuntu",
      "version": "16.4.0 LTS",
      "platform": "ubuntu",
      "os_id": 6
    }
  ]
}
```

## Get hosts report in CSV

Returns the list of hosts corresponding to the search criteria in CSV format, ready for download when
requested by a web browser.

`GET /api/v1/fleet/hosts/report`

#### Parameters

| Name                    | Type    | In    | Description                                                                                                                                                                                                                                                                                                                                 |
| ----------------------- | ------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| format                  | string  | query | **Required**, must be "csv" (only supported format for now).                                                                                                                                                                                                                                                                                |
| columns                 | string  | query | Comma-delimited list of columns to include in the report (returns all columns if none is specified).                                                                                                                                                                                                                                        |
| order_key               | string  | query | What to order results by. Can be any column in the hosts table.                                                                                                                                                                                                                                                                             |
| order_direction         | string  | query | **Requires `order_key`**. The direction of the order given the order key. Options include `asc` and `desc`. Default is `asc`.                                                                                                                                                                                                               |
| status                  | string  | query | Indicates the status of the hosts to return. Can either be `new`, `online`, `offline`, `mia` or `missing`.                                                                                                                                                                                                                                  |
| query                   | string  | query | Search query keywords. Searchable fields include `hostname`, `machine_serial`, `uuid`, `ipv4` and the hosts' email addresses (only searched if the query looks like an email address, i.e. contains an `@`, no space, etc.).                                                                                                                |
| team_id                 | integer | query | _Available in Fleet Premium_ Filters the hosts to only include hosts in the specified team.                                                                                                                                                                                                                                                 |
| policy_id               | integer | query | The ID of the policy to filter hosts by.                                                                                                                                                                                                                                                                                                    |
| policy_response         | string  | query | Valid options are `passing` or `failing`.  `policy_id` must also be specified with `policy_response`.                                                                                                                                                                                                                                       |
| software_id             | integer | query | The ID of the software to filter hosts by.                                                                                                                                                                                                                                                                                                  |
| os_id                   | integer | query | The ID of the operating system to filter hosts by.                                                                                                                                                                                                                                                                                          |
| os_name                 | string  | query | The name of the operating system to filter hosts by. `os_version` must also be specified with `os_name`                                                                                                                                                                                                                                     |
| os_version              | string  | query | The version of the operating system to filter hosts by. `os_name` must also be specified with `os_version`                                                                                                                                                                                                                                  |
| mdm_id                  | integer | query | The ID of the _mobile device management_ (MDM) solution to filter hosts by (that is, filter hosts that use a specific MDM provider and URL).                                                                                                                                                                                                |
| mdm_name                | string  | query | The name of the _mobile device management_ (MDM) solutin to filter hosts by (that is, filter hosts that use a specific MDM provider).                                                                                                                                                                                                |
| mdm_enrollment_status   | string  | query | The _mobile device management_ (MDM) enrollment status to filter hosts by. Can be one of 'manual', 'automatic', 'enrolled', 'pending', or 'unenrolled'.                                                                                                                                                                                                             |
| macos_settings          | string  | query | Filters the hosts by the status of the _mobile device management_ (MDM) profiles applied to hosts. Can be one of 'verified', 'verifying', 'pending', or 'failed'. **Note: If this filter is used in Fleet Premium without a team id filter, the results include only hosts that are not assigned to any team.**                                                                                                                                                                                                             |
| munki_issue_id          | integer | query | The ID of the _munki issue_ (a Munki-reported error or warning message) to filter hosts by (that is, filter hosts that are affected by that corresponding error or warning message).                                                                                                                                                        |
| low_disk_space          | integer | query | _Available in Fleet Premium_ Filters the hosts to only include hosts with less GB of disk space available than this value. Must be a number between 1-100.                                                                                                                                                                                  |
| label_id                | integer | query | A valid label ID. Can only be used in combination with `order_key`, `order_direction`, `status`, `query` and `team_id`.                                                                                                                                                                                                                     |
| bootstrap_package       | string | query | _Available in Fleet Premium_ Filters the hosts by the status of the MDM bootstrap package on the host. Can be one of `installed`, `pending`, or `failed`. **Note: If this filter is used in Fleet Premium without a team id filter, the results include only hosts that are not assigned to any team.** |

If `mdm_id`, `mdm_name` or `mdm_enrollment_status` is specified, then Windows Servers are excluded from the results.

#### Example

`GET /api/v1/fleet/hosts/report?software_id=123&format=csv&columns=hostname,primary_ip,platform`

##### Default response

`Status: 200`

```csv
created_at,updated_at,id,detail_updated_at,label_updated_at,policy_updated_at,last_enrolled_at,seen_time,refetch_requested,hostname,uuid,platform,osquery_version,os_version,build,platform_like,code_name,uptime,memory,cpu_type,cpu_subtype,cpu_brand,cpu_physical_cores,cpu_logical_cores,hardware_vendor,hardware_model,hardware_version,hardware_serial,computer_name,primary_ip_id,primary_ip,primary_mac,distributed_interval,config_tls_refresh,logger_tls_period,team_id,team_name,gigs_disk_space_available,percent_disk_space_available,issues,device_mapping,status,display_text
2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,1,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,false,foo.local0,a4fc55a1-b5de-409c-a2f4-441f564680d3,debian,,,,,,0s,0,,,,0,0,,,,,,,,,0,0,0,,,0,0,0,,,,
2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:22:56Z,false,foo.local1,689539e5-72f0-4bf7-9cc5-1530d3814660,rhel,,,,,,0s,0,,,,0,0,,,,,,,,,0,0,0,,,0,0,0,,,,
2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,3,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:23:56Z,2022-03-15T17:21:56Z,false,foo.local2,48ebe4b0-39c3-4a74-a67f-308f7b5dd171,linux,,,,,,0s,0,,,,0,0,,,,,,,,,0,0,0,,,0,0,0,,,,
```

## Get host's disk encryption key

Requires the [macadmins osquery extension](https://github.com/macadmins/osquery-extension) which comes bundled
in [Fleet's osquery installers](https://fleetdm.com/docs/using-fleet/adding-hosts#osquery-installer).

Requires Fleet's MDM properly [enabled and configured](https://fleetdm.com/docs/using-fleet/mdm-setup).

Retrieves the disk encryption key for a host.

`GET /api/v1/fleet/mdm/hosts/:id/encryption_key`

#### Parameters

| Name | Type    | In   | Description                                                        |
| ---- | ------- | ---- | ------------------------------------------------------------------ |
| id   | integer | path | **Required** The id of the host to get the disk encryption key for |


#### Example

`GET /api/v1/fleet/mdm/hosts/8/encryption_key`

##### Default response

`Status: 200`

```json
{
  "host_id": 8,
  "encryption_key": {
    "key": "5ADZ-HTZ8-LJJ4-B2F8-JWH3-YPBT",
    "updated_at": "2022-12-01T05:31:43Z"
  }
}
```

## Get configuration profiles assigned to a host 

Requires Fleet's MDM properly [enabled and configured](https://fleetdm.com/docs/using-fleet/mdm-setup).

Retrieves a list of the configuration profiles assigned to a host.

`GET /api/v1/fleet/mdm/hosts/:id/profiles`

#### Parameters

| Name | Type    | In   | Description                      |
| ---- | ------- | ---- | -------------------------------- |
| id   | integer | path | **Required**. The ID of the host  |


#### Example

`GET /api/v1/fleet/mdm/hosts/8/profiles`

##### Default response

`Status: 200`

```json
{
  "host_id": 8,
  "profiles": [
    {
      "profile_id": 1337,
      "team_id": 0,
      "name": "Example profile",
      "identifier": "com.example.profile",
      "created_at": "2023-03-31T00:00:00Z",
      "updated_at": "2023-03-31T00:00:00Z",
      "checksum": "dGVzdAo="
    }
  ]
}
```


<meta name="pageOrderInSection" value="600">