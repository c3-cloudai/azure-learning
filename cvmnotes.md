# Confidential VM Deployment Notes

## Check AMD SEV-SNP VM Sizes

The Azure region must support the required AMD SEV-SNP VM size. Check the
available confidential VM sizes before deployment:

```powershell
az vm list-skus `
  --location "centralus" `
  --resource-type virtualMachines `
  --query "[?contains(name, 'DCasv5') || contains(name, 'ECasv5')].{Name:name,Restrictions:restrictions}" `
  --output table
```

> Availability depends on the Azure region, subscription, quota, and current
> Azure capacity.

## Check Confidential VM Quota

Your subscription must have sufficient regional vCPU quota for the selected
confidential VM size. If deployment fails because of quota, request a quota
increase through Azure or choose another supported VM size.

## Register Required Resource Providers

Register the providers required for the VM, networking, and attestation
service:

```powershell
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Attestation
```

Check their registration status:

```powershell
az provider list `
  --query "[?namespace=='Microsoft.Compute' || namespace=='Microsoft.Network' || namespace=='Microsoft.Attestation'].{Provider:namespace,State:registrationState}" `
  --output table
```

Each provider should report a state of `Registered`.

## Attestation Region Availability

Azure Attestation is not available in every Azure region. Check the supported
locations before creating an attestation provider:

```powershell
az provider show `
  --namespace Microsoft.Attestation `
  --expand "resourceTypes/locations" `
  --query "resourceTypes[?resourceType=='attestationProviders'].locations[]" `
  --output table
```

The Attestation provider can be deployed in a different region from the
confidential VM if `centralus` does not support Azure Attestation.

## Related Resources

- [Azure confidential VM overview](https://learn.microsoft.com/azure/confidential-computing/confidential-vm-overview)
- [Azure Attestation overview](https://learn.microsoft.com/azure/attestation/overview)
- [Guest attestation for confidential VMs](https://learn.microsoft.com/azure/confidential-computing/guest-attestation-confidential-vms)
