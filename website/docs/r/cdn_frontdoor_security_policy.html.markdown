---
subcategory: "CDN"
layout: "azurerm"
page_title: "Azure Resource Manager: azurerm_cdn_frontdoor_security_policy"
description: |-
  Manages a Front Door (standard/premium) Security Policy.
---

# azurerm_cdn_frontdoor_security_policy

Manages a Front Door (standard/premium) Security Policy.

## Example Usage

```hcl
resource "azurerm_resource_group" "example" {
  name     = "example-cdn-frontdoor"
  location = "West Europe"
}

resource "azurerm_cdn_frontdoor_profile" "example" {
  name                = "example-profile"
  resource_group_name = azurerm_resource_group.example.name
  sku_name            = "Standard_AzureFrontDoor"
}

resource "azurerm_cdn_frontdoor_endpoint" "example" {
  name                     = "example-endpoint"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.example.id
}

resource "azurerm_cdn_frontdoor_origin_group" "example" {
  name                     = "example-origin-group"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.example.id

  load_balancing {
    additional_latency_in_milliseconds = 0
    sample_size                        = 4
    successful_samples_required        = 3
  }
}

resource "azurerm_cdn_frontdoor_origin" "example" {
  name                          = "example-origin"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.example.id
  enabled                       = true

  certificate_name_check_enabled = true
  host_name                      = "www.contoso.com"
  http_port                      = 80
  https_port                     = 443
  origin_host_header             = "www.contoso.com"
  priority                       = 1
  weight                         = 1000
}

resource "azurerm_cdn_frontdoor_firewall_policy" "example" {
  name                              = "exampleWAF"
  resource_group_name               = azurerm_resource_group.example.name
  sku_name                          = azurerm_cdn_frontdoor_profile.example.sku_name
  enabled                           = true
  mode                              = "Prevention"
  redirect_url                      = "https://www.contoso.com"
  custom_block_response_status_code = 403
  custom_block_response_body        = "PGh0bWw+CjxoZWFkZXI+PHRpdGxlPkhlbGxvPC90aXRsZT48L2hlYWRlcj4KPGJvZHk+CkhlbGxvIHdvcmxkCjwvYm9keT4KPC9odG1sPg=="

  custom_rule {
    name                           = "Rule1"
    enabled                        = true
    priority                       = 1
    rate_limit_duration_in_minutes = 1
    rate_limit_threshold           = 10
    type                           = "MatchRule"
    action                         = "Block"

    match_condition {
      match_variable     = "RemoteAddr"
      operator           = "IPMatch"
      negation_condition = false
      match_values       = ["192.168.1.0/24", "10.0.1.0/24"]
    }
  }
}

resource "azurerm_dns_zone" "example" {
  name                = "sub-domain.domain.com"
  resource_group_name = azurerm_resource_group.example.name
}

resource "azurerm_cdn_frontdoor_custom_domain" "example" {
  name                     = "example-customDomain"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.example.id
  dns_zone_id              = azurerm_dns_zone.example.id
  host_name                = join(".", ["contoso", azurerm_dns_zone.example.name])

  tls {
    certificate_type    = "ManagedCertificate"
    minimum_tls_version = "TLS12"
  }
}

resource "azurerm_cdn_frontdoor_route" "example" {
  name                            = "example-route"
  cdn_frontdoor_endpoint_id       = azurerm_cdn_frontdoor_endpoint.example.id
  cdn_frontdoor_origin_group_id   = azurerm_cdn_frontdoor_origin_group.example.id
  cdn_frontdoor_origin_ids        = [azurerm_cdn_frontdoor_origin.example.id]
  cdn_frontdoor_custom_domain_ids = [azurerm_cdn_frontdoor_custom_domain.example.id]
  patterns_to_match               = ["/*"]
  supported_protocols             = ["Http", "Https"]
}

resource "azurerm_cdn_frontdoor_security_policy" "example" {
  name                     = "Example-Security-Policy"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.example.id

  security_policies {
    firewall {
      cdn_frontdoor_firewall_policy_id = azurerm_cdn_frontdoor_firewall_policy.example.id

      association {
        domain {
          cdn_frontdoor_domain_id = azurerm_cdn_frontdoor_custom_domain.example.id
        }

        route {
          cdn_frontdoor_route_id = azurerm_cdn_frontdoor_route.example.id
        }

        patterns_to_match = ["/*"]
      }
    }
  }
}
```

## Arguments Reference

The following arguments are supported:

* `name` - (Required) The name which should be used for this Front Door Security Policy. Changing this forces a new resource to be created.

* `cdn_frontdoor_profile_id` - (Required) The Front Door Profile Resource Id that is linked to this Front Door Security Policy. Changing this forces a new resource to be created.

* `security_policies` - (Required) A `security_policies` block as defined below.

---

A `security_policies` block supports the following:

* `firewall` - (Required) A `firewall` block as defined below.

---

A `firewall` block supports the following:

* `association` - (Required) An `association` block as defined below.

* `cdn_frontdoor_firewall_policy_id` - (Required) The Resource Id of the Front Door Firewall Policy that should be linked to this Front Door Security Policy. Changing this forces a new resource to be created.

---

An `association` block supports the following:

* `patterns_to_match` - (Required) The list of paths to match for this firewall policy. The only possible value is `/*`. Changing this forces a new resource to be created.

* `domain` - (Optional) One or more `domain` blocks as defined below.

~> **Note:** The number of `domain` blocks that may be included in the configuration varies depending on the `sku_name` field of the linked Front Door Profile. The `Standard_AzureFrontDoor` sku may contain up to 100 `domain` blocks and a `Premium_AzureFrontDoor` sku may contain up to 500 `domain` blocks.

* `route` - (Optional) One or more `route` blocks as defined below.

~> **Note:** At least one of `domain` or `route` must be specified.

---

A `domain` block supports the following:

* `cdn_frontdoor_domain_id` - (Required) The Resource Id of the **Front Door Custom Domain** or **Front Door Endpoint** that should be bound to this Front Door Security Policy.

---

A `route` block supports the following:

* `cdn_frontdoor_route_id` - (Required) The Resource Id of the **Front Door Route** that should be bound to this Front Door Security Policy.

---

## Attributes Reference

In addition to the Arguments listed above - the following Attributes are exported:

* `id` - The ID of the Front Door Security Policy.

* `security_policies` - A `security_policies` block as defined below.

---

A `security_policies` block exports the following:

* `firewall` - A `firewall` block as defined below.

---

A `firewall` block exports the following:

* `association` - An `association` block as defined below.

---

An `association` block exports the following:

* `domain` - A `domain` block as defined below.

* `route` - A `route` block as defined below.

---

A `domain` block exports the following:

* `active` - Whether the Front Door Custom Domain or Front Door Endpoint is active.

---

A `route` block exports the following:

* `cdn_frontdoor_route_id` - The Resource Id of the Front Door Route associated with this Front Door Security Policy.

## Timeouts

The `timeouts` block allows you to specify [timeouts](https://developer.hashicorp.com/terraform/language/resources/configure#define-operation-timeouts) for certain actions:

* `create` - (Defaults to 4 hours) Used when creating the Front Door Security Policy.
* `read` - (Defaults to 5 minutes) Used when retrieving the Front Door Security Policy.
* `update` - (Defaults to 4 hours) Used when updating the Front Door Security Policy.
* `delete` - (Defaults to 6 hours) Used when deleting the Front Door Security Policy.

## Import

A Front Door Security Policy can be imported using the `resource id`, e.g.

```shell
terraform import azurerm_cdn_frontdoor_security_policy.example /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/resourceGroup1/providers/Microsoft.Cdn/profiles/profile1/securityPolicies/policy1
```

## API Providers
<!-- This section is generated, changes will be overwritten -->
This resource uses the following Azure API Providers:

* `Microsoft.Cdn` - 2026-10-01
