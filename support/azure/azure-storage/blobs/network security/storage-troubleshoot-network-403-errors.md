---
title: Troubleshoot network-related 403 errors in Azure Storage
description: Resolve HTTP 403 errors in Azure Storage by diagnosing firewall rules, public network access, virtual network configuration, private endpoint DNS, routing, and copy-source connectivity.
ms.date: 08/25/2026
author: aadetayo
ms.service: azure-storage
ms.author: aadetayo
ms.custom: sap:Network Security
---

# Troubleshoot network-related 403 errors in Azure Storage

## Summary

When a client accesses Azure Storage, network security settings can cause an HTTP 403 (`Forbidden`) response even when the request includes valid credentials. Common causes include an unapproved source IP address, a missing virtual network rule, disabled public network access, private endpoint DNS that resolves to the public endpoint, and a blocked source account during a service-to-service copy.

This article lists the common network-related 403 statuses and provides a diagnostic workflow to identify and resolve the network-path issue without unnecessarily broadening access. If the client returns only a generic message such as `This request is not authorized to perform this operation`, correlate the request with Storage diagnostics before deciding that the failure is caused by authentication or permissions.

> [!NOTE]
> This article focuses on network-related Azure Storage data-plane requests. The exact error and diagnostic fields that are available depend on the Storage service, client, and logging configuration.

## About network-related 403 errors

Azure Storage can return the same HTTP status for different failure categories.

> [!IMPORTANT]
> A generic `AuthorizationFailure` response doesn't prove that the identity, shared access signature (SAS), or account key is invalid. First determine whether the request reached the public endpoint or a private endpoint and whether the observed client IP and route match the intended network path.

## Common network-related 403 errors

| Code or status | Most likely cause | First troubleshooting action |
| --- | --- | --- |
| `AuthorizationFailure` | Generic firewall, virtual network, public network access, private endpoint, proxy, NAT, or authorization-path failure | Validate whether the issue is network-path related or identity/data-plane related. Check ClientIP, firewall configuration, Public Network Access, VNet rules, private endpoint DNS, and public vs private routing. |
| `IPAuthorizationFailure` | The source IP address isn't allowed by the storage account firewall | Compare the logged client IP with the actual public egress IP after any proxy or NAT device, and then review IP network rules. |
| `OAuthIpAuthorizationError` | An OAuth-authenticated request is blocked by IP or firewall authorization | Treat the token and the network check as separate controls. Review the source IP and firewall allowlist. |
| `OAuthPublicNetworkAccessError` | An OAuth-authenticated request reaches the public endpoint while public access is restricted or disabled | Verify DNS resolution and routing. If private connectivity is intended, confirm that the client resolves and reaches the private endpoint. |
| `PublicNetworkAccessError` | The request uses a public path while the account expects private connectivity | Check public network access, private endpoint DNS, service endpoints, virtual network rules, and the actual route. |
| `Unauthorized PublicNetworkAccess` | Some requests intermittently use an unauthorized public path | Compare successful and failed requests for DNS, route, host, and client IP differences. |
| `CannotVerifyCopySource` | During a service-to-service copy, the destination service can't validate or reach the source account because of firewall or private endpoint restrictions | Test and review network access for both the source and destination accounts. |

## Network 403 diagnostic checklist

Use the checklist in order. A change that makes the storage account broadly reachable can hide the root cause and weaken the account's security posture.

1. **Identify the endpoint that the client used.** Determine whether the request resolved to a public IP address or the private IP address of a private endpoint. Run the DNS test from the same host, container, virtual machine, integration runtime, or application instance that failed.
1. **Compare the observed client IP with the intended path.** Proxies, VPN gateways, Azure NAT Gateway, on-premises NAT, and other egress devices can change the public source IP. Use the client IP in diagnostics rather than the workstation's local IP address.
1. **Review public network access.** If public network access is disabled or restricted, a request that reaches the public endpoint fails even if its credentials are valid. If public access is intended, use narrowly scoped network rules instead of allowing all networks.
1. **Review IP network rules.** Confirm that the actual public egress IP or range is allowed. A SAS IP restriction doesn't grant access beyond the storage account's network rules.
1. **Review virtual network rules and service endpoints.** Confirm that the originating subnet is included in the storage account's virtual network rules and that the appropriate Azure Storage service endpoint is enabled on that subnet. When a subnet uses a service endpoint, public IP allowlist entries for that subnet don't authorize the traffic.
1. **Review private endpoint DNS and routing.** Confirm that the storage account hostname resolves to the expected private IP from every client network. Verify the private DNS zone link, DNS records, private endpoint approval and subresource, virtual network peering, routing, and network security controls on the path.
1. **Check every participant in a copy operation.** For service-to-service copy operations, validate reachability and authorization for both the source and destination storage accounts. A reachable destination doesn't prove that the service can validate the source.
1. **Separate network failures from permission failures.** If the network path is correct, review Azure role-based access control (Azure RBAC), SAS scope and permissions, shared key policy, anonymous access settings, operation-specific permissions, and client clock synchronization.

For more information about the available controls, see [Azure Storage firewall rules and network access control](https://learn.microsoft.com/azure/storage/common/storage-network-security).

## Troubleshoot specific errors

### AuthorizationFailure

**Typical message**

```text
This request is not authorized to perform this operation.
```

`AuthorizationFailure` is a generic status. It can represent a firewall, virtual network, public network access, proxy, NAT, private endpoint, or identity-related failure.

**Troubleshooting steps**

1. Correlate the request with Storage diagnostics by using the request ID and time.
1. Check for a more specific `InternalStatus` or `MeasurementStatus` value.
1. Compare `ClientIP` with the source that the firewall is configured to allow.
1. Determine whether DNS and routing sent the request to the public endpoint or a private endpoint.
1. If diagnostics don't indicate a network rejection, validate the credential and required data-plane permissions.

Don't use the generic message by itself to decide whether to change firewall rules or role assignments.

### IPAuthorizationFailure

`IPAuthorizationFailure` strongly indicates that Azure Storage evaluated the request against public endpoint firewall rules and rejected the observed source IP address.

**Troubleshooting steps**

1. Find the client IP recorded for the failed request.
1. Identify the actual internet-facing egress IP after any proxy, firewall, VPN, or NAT device.
1. Compare that address with the storage account's IP network rules.
1. If public endpoint access is intended, add only the required egress IP address or range.
1. If the client should use a virtual network service endpoint or private endpoint, correct the network path instead of adding a public IP rule.

For IP rule requirements and limitations, see [Configure Azure Storage firewalls and virtual networks](https://learn.microsoft.com/azure/storage/common/storage-network-security#ip-network-rules).

### OAuthIpAuthorizationError

`OAuthIpAuthorizationError` means that the OAuth-authenticated request reached Azure Storage but failed the network or firewall authorization check. OAuth authentication doesn't bypass storage account network rules.

**Troubleshooting steps**

1. Confirm the source IP address recorded for the request.
1. Review IP network rules, virtual network rules, and any egress NAT configuration.
1. Reproduce the request from the same application host or runtime. Testing from a developer workstation can use a different route and source IP.
1. After network access succeeds, validate Azure RBAC only if the request still fails with a permission-related status.

### OAuthPublicNetworkAccessError

`OAuthPublicNetworkAccessError` indicates that an OAuth-authenticated request reached the public endpoint while public network access was restricted or disabled.

**Troubleshooting steps**

1. Resolve the storage account hostname from the failing client environment.
1. If the client should use a private endpoint, confirm that DNS returns the private endpoint's private IP address.
1. Verify that the required private endpoint for the Storage service subresource exists and is approved.
1. Check private DNS zone links, custom DNS forwarding, virtual network peering, user-defined routes, and network security controls.
1. If public endpoint access is intentional, configure public network access and narrowly scoped firewall rules that match the actual source.

For private endpoint DNS behavior, see [Use private endpoints for Azure Storage](https://learn.microsoft.com/azure/storage/common/storage-private-endpoints#dns-changes-for-private-endpoints).

### PublicNetworkAccessError

`PublicNetworkAccessError` indicates that the client used a public route when the storage account configuration expected private connectivity.

**Troubleshooting steps**

1. Confirm the storage account's public network access setting.
1. Check DNS from the failing host, not only from an administrator's workstation.
1. Verify that the client uses the standard storage account hostname and that DNS maps it to the intended private endpoint when queried from the private network.
1. Confirm that the endpoint matches the requested Storage service, such as Blob, Data Lake Storage, or Files.
1. Review service endpoint and virtual network rules if the design uses a public endpoint secured to selected subnets instead of a private endpoint.

If an approved private endpoint still returns 403, see [Troubleshoot 403 access denied errors through an approved private endpoint](https://learn.microsoft.com/troubleshoot/azure/private-link/troubleshoot-403-access-denied-private-endpoint).

### Unauthorized PublicNetworkAccess

`Unauthorized PublicNetworkAccess` is commonly associated with intermittent migration or Azure Storage Mover failures in which only some requests use an unauthorized public path. Intermittent behavior usually points to inconsistent DNS or routing rather than a consistently invalid credential.

**Troubleshooting steps**

1. Compare successful and failed requests by time, client IP, application instance, resolved address, and destination endpoint.
1. Run repeated DNS resolution tests from each worker, agent, or application instance.
1. Check whether different DNS resolvers return different answers or whether stale DNS records are cached.
1. Confirm that every subnet and host has the same private DNS visibility and route to the private endpoint.
1. Review fallback behavior that might send traffic to the public endpoint when private name resolution or routing fails.

### CannotVerifyCopySource

`CannotVerifyCopySource` can occur when an Azure Storage service-to-service copy can't validate or reach the source account. A firewall or private endpoint configuration on either account can block the copy path.

**Troubleshooting steps**

1. Identify the source and destination endpoints used by the operation.
1. Review public network access, firewall rules, virtual network rules, and private endpoint configuration on both accounts.
1. Confirm that the destination service can reach and validate the source; successful client access to each account doesn't necessarily validate the service-to-service path.
1. If the network path is allowed, validate the source credential or SAS and the permissions required for the copy operation.

## Errors that are usually not network related

The following 403 errors can appear during the same investigation but generally require a different resolution path.

| Code or status | Usual interpretation | What to check |
| --- | --- | --- |
| `PublicAccessNotPermitted` | Anonymous or public blob access is disallowed | Determine whether anonymous access was intended. Otherwise, use authenticated access. |
| `AnonymousClientInternalError` | Anonymous access is blocked, often alongside `PublicAccessNotPermitted` | Check account and container anonymous access settings. |
| `AuthorizationPermissionMismatch` | The credential lacks permission for the requested operation | Review Azure RBAC, SAS permissions, and operation requirements. For Azure Files OAuth REST access, verify the required role assignment. |
| `InsufficientAccountPermissions` | The account or caller doesn't have sufficient permission | Review the authorization method and permissions. |
| `UnauthorizedBlobOverwrite` | The request isn't authorized to overwrite the blob | Review operation-specific write, delete, lease, and ownership requirements as applicable. |
| `Request date header too old` | The request timestamp is more than 15 minutes outside the Storage service tolerance or the signed request is stale | Synchronize the client clock and generate a fresh signed request. |

If the error falls into one of these categories, don't add firewall exceptions unless separate diagnostic evidence shows that the network path is also blocked.

## Next steps

- [Troubleshoot 403 errors in Azure Blob Storage](https://learn.microsoft.com/troubleshoot/azure/azure-storage/blobs/authentication/storage-troubleshoot-403-errors)
- [Azure Storage firewall rules and network access control](https://learn.microsoft.com/azure/storage/common/storage-network-security)
- [Use private endpoints for Azure Storage](https://learn.microsoft.com/azure/storage/common/storage-private-endpoints)
- [Troubleshoot Azure Private Endpoint connectivity problems](https://learn.microsoft.com/troubleshoot/azure/private-link/troubleshoot-private-endpoint-connectivity-problems)

