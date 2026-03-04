# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Azure ARM template for deploying a single Palo Alto Networks VM-Series firewall appliance. The template (`Add-Single-VM-series-template.json`) is a flat ARM JSON file (originally generated from Bicep) intended for use with the Azure portal, Azure CLI, or PowerShell.

## Template Architecture

The template deploys resources in a flat (non-nested) structure with `dependsOn` ordering:

1. **Partner Center tracking** — Empty `Microsoft.Resources/deployments` for Marketplace attribution
2. **Public IP** — Standard SKU, static allocation (conditional: `publicIPNewOrExisting == 'new'`)
3. **NSG** — Management NSG with three rules: Allow source IP, Allow intra-VNET, Default-Deny
4. **VNET** — Virtual network with 3 subnets (conditional: `vnetNewOrExisting == 'new'`)
5. **NICs** — Three network interfaces:
   - `eth0` (management): Dynamic IP, primary, no IP forwarding
   - `eth1` (untrust): Dynamic IP + public IP, IP forwarding + accelerated networking enabled
   - `eth2` (trust): Dynamic IP only, IP forwarding + accelerated networking enabled
6. **Availability Set** — Aligned SKU (conditional: `availabilitySetName != 'None'`)
7. **VM** — VM-Series from `paloaltonetworks/vmseries-flex/byol` Marketplace image

## Deployment Commands

```bash
# Validate
az deployment group validate \
  --resource-group <RG_NAME> \
  --template-file "Add-Single-VM-series-template.json" \
  --parameters @params.json

# Deploy
az deployment group create \
  --resource-group <RG_NAME> \
  --template-file "Add-Single-VM-series-template.json" \
  --parameters @params.json
```

## Conventions

- **Conditional resources** use parameter-based conditions (e.g., `"condition": "[equals(parameters('vnetNewOrExisting'), 'new')]"`). Both new and existing VNET/public IP paths must remain valid.
- **Subnet references** are resolved through paired variables: `newSubnet{N}Ref` / `existingSubnet{N}Ref` selected via `subnet{N}Ref` using `if(equals(...))`.
- NIC naming follows `{vmName}-eth{0,1,2}` pattern.
- The NSG is named `DefaultNSG` (hardcoded in variables, not parameterized).
- Availability is mutually exclusive: set `zone` for Availability Zone **or** `availabilitySetName` for Availability Set. Both default to `"None"` (disabled).
- Bootstrap data is passed via `customData` parameter, base64-encoded, only when `bootstrap == 'yes'`.
- API versions: Network `2020-11-01`, Compute `2022-03-01`, Deployment `2020-10-01`.

## Key Differences from Sibling Panorama Templates

The sibling repo (`Claude-Dev-Azure-ARM/`) contains Panorama templates that use **nested deployments** with `global_`/resource-specific variable prefixes. This VM-Series template uses a **flat resource structure** with simpler variable naming and no nested deployments. The Panorama templates also use newer API versions (2023-09-01).
