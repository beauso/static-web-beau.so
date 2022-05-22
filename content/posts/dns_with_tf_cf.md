---
title: "Clean YAML DNS management with TF and Cloudflare"
date: 2022-05-22T17:06:22+01:00
draft: false
---

## Intro - Avoiding bad patterns 
There are two uninspiring patterns I've observed with DNS management via Terraform (be it R53, Cloudflare etc):
  
- Resource blocks for each individual record.   
  
  - This is fine if you know you're only really going to have a few records, but it almost always ends up being a short sighted approach (especially with multiple domains), repeating attribute after attribute e.g.:

```hcl
resource "aws_route53_record" "example_cname" {
    ...
}

resource "aws_route53_record" "another_example_cname" {
    ...
}
```

^ *Ugly*, right? You can imagine this quickly gets out of hand fast...

- Another option: better, yet still not optimal: a long-ass `.tfvars` file with a ton of records created by a `for_each` loop. This tends to be fine for one domain, but quickly is realised to not be a scalable solution.

Alles klar, DNS bitte in ordnung bringen.

By following along, you will achieve:
- DNS records separated within individual `.yml` files by apex domain (`mydomain.com`).
- Cloudflare zones created automatically from above yml.
- Optionally omit certain fields where not required (e.g. `proxied`, `priority`, `ttl`) 

## Solution
### 1. Structuring your records as YML
We'll need to structure our records as follows, assuming this is our application where we use the Cloudflare provider:

```
├── dns
│   ├── mydomain.com
│   │   └── dns_definitions.yml
│   ├── anotherdomain.com
│   │   └── dns_definitions.yml
│   └── yetanotherdomain.com
│       └── dns_definitions.yml
├── locals.tf
├── main.tf
```

Inside each `.yml` definition file, we should structure it like so:

```yml
mydomain.com:
  name: mydomain.com
  records:
    # CNAMES
    examplecname:
      - name: examplecname
        value: somewherelse.anotherdomain.com
        type: CNAME

    # MX
    google_mx_record_1:
      - name: mydomain.com
        value: alt1.aspmx.l.google.com
        type: MX
        priority: 5
```

### 1.1 `merge()` with `yamldecode()` and `fileset()`
`fileset()` returns a list of files that match the pattern parameter:

- locals.tf: `dns_yml_files = fileset(path.module, "dns/*/*.yml")`
  - Debugging the above local (you should use `terraform console` if you don't already):

  ```hcl
  toset([
        "dns/mydomain.com/dns_definitions.yml",
        "dns/anotherdomain.com/dns_definitions.yml",
        "dns/yetanotherdomain.com/dns_definitions.yml",
        ])
    ```

Noice. Now we can decode the yaml into a map for each of these files, and merge it into one bad boy local holding all the variables:

```hcl
  cloudflare_records = merge([
    for file in local.dns_yml_files : yamldecode(file(file))]...
  )
```

Pretty self explanatory, but the expansion notation: `...` (which you might be familiar with from Go and variadic functions) passes a dynamic number of items (in this case, file paths from our list). It's quite useful to use with `merge()`.

### 1.3 Flatten `cloudflare_records` for use in resource blocks.
We need to flatten the data structure to provide each necessary attribute within `resource "cloudflare_record"` later (without having it duplicated in each YML record). The original idea for this came from a user called [octopus20](https://github.com/octopus20/terraform-cloudflare/blob/main/dns_management/main.tf) (thanks!), and I have improved it to suit some of my requirements I discuss below. The `try()` wrapped values allow us to forgo them without setting them to `null` in the records repeating ourselves:

locals.tf:
```hcl
  flattened_records = flatten([
    for _, zone in local.cloudflare_records : [
      for name, entry in zone.records : [
        for record in entry : {
          friendly_name = name
          zone_name     = zone.name
          name          = record.name
          value         = record.value
          type          = record.type
          ttl           = try(record.ttl, null)
          proxied       = try(record.proxied, null)
          priority      = try(record.priority, null)
        }
      ]
    ]
  if contains(keys(zone), "records")])
}
```
The conditional is so we can still create zones if no records exist, and not have a circular dependency of needing records for zones and zones for records.

`friendly_name` is important. While CNAMEs `name` attributes are all unique: if we use the resource name where a `TXT` or `MX` record should exist, there will be duplicate resources in our state file. Therefore we can take advantage of this attribute and give the resource name for our multiple `MX` record something meaningful in the top key of our yml:

```yaml
    # MX
    google_mx_record_1:
      - name: mydomain.com
        value: alt1.aspmx.l.google.com
        type: MX
        priority: 5
```

```hcl
  {
    "friendly_name" = "google_mx_record_1"
    "name" = "mydomain.com"
    "priority" = 5
    "proxied" = null
    "ttl" = null
    "type" = "CNAME"
    "value" = "alt1.aspmx.l.google.com"
    "zone_name" = "mydomain.com"
  }
```

### 1.4 Creating the resources
We can create zones like so without using the flattened records:

main.tf: 
```hcl
resource "cloudflare_zone" "this" {
  for_each = local.cloudflare_records

  zone = each.value.name
}
```

Next, iterate through the records, setting the key/resource name as we like and appending the `friendly_name` to ensure uniquness.

main.tf: 
```hcl

resource "cloudflare_record" "this" {
  for_each = { for record in local.flattened_records : join("_",
  [record.zone_name, record.type, record.friendly_name]) => record }

  zone_id  = cloudflare_zone.this[each.value.zone_name].id
  name     = each.value.name
  value    = each.value.value
  type     = each.value.type
  ttl      = each.value.ttl
  proxied  = each.value.proxied
  priority = each.value.priority
}
```


And the end-result -- using only 2 resource blocks to create as many records and zones as we need, without worrying about the number of domains, type of records, or overlapping resource names:
```hcl
cloudflare_zone.this["mydomain.com"]
cloudflare_record.this["mydomain.com_CNAME_examplecname"]
cloudflare_record.this["mydomain.com_MX_google_mx_record_1"]
```
