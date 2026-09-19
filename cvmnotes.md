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

## How Attestation Works for an Azure Confidential VM

Azure Confidential VMs use AMD Secure Encrypted Virtualization-Secure Nested
Paging (SEV-SNP) to protect VM memory and help detect unauthorized changes to
the VM's execution environment.

There are two related attestation processes:

### Platform Attestation

Platform attestation is part of the Azure-managed boot process:

1. The AMD SEV-SNP hardware produces evidence containing measurements of the
   confidential VM's firmware and boot environment.
2. Azure validates the evidence and the expected platform state.
3. Azure Attestation participates in validating the evidence and issuing an
   attestation result.
4. The result is used in the confidential VM boot and key-release process.
5. The VM starts only when the required validation succeeds.

This process is automatic. You do not need to submit an attestation request
manually just to start an AMD SEV-SNP Confidential VM.

### Guest Attestation

Guest attestation is used when an application or relying party needs proof
from inside the running VM:

1. An application inside the CVM requests an AMD SEV-SNP report.
2. A guest-attestation client sends the report to the Azure Attestation
   provider.
3. Azure Attestation validates the report against its policy.
4. The service returns a signed JSON Web Token (JWT) containing attestation
   claims.
5. The application or relying party validates the JWT signature, issuer,
   audience, expiry, nonce, and expected claims.
6. The relying party releases a secret, encryption key, or workload access
   only when the claims meet its trust requirements.

Typical claims can identify the attestation type as `sevsnpvm` and indicate
whether Secure Boot is enabled. Creating an Attestation provider does not
automatically attest application code; the workload must request evidence and
validate the returned token.

### Trust Boundary

The Azure Attestation provider is the service that evaluates evidence. The
application or relying party is responsible for deciding what evidence is
acceptable. It should not trust a response solely because the request
succeeded. It must validate the signed token and enforce an explicit policy
for the expected VM configuration and measurements.

## Do You Need Azure Key Vault?

It depends on the protection requirement:

- **Platform-managed disk encryption:** Azure manages the encryption keys.
  Azure Key Vault is not required for the basic creation and boot of a
  Confidential VM.
- **Customer-managed disk encryption:** Use Azure Key Vault or Azure Key Vault
  Managed HSM when your organization must control the key, rotation, access
  policy, and auditing.
- **Application secret release after attestation:** Use Azure Key Vault or
  Managed HSM when an application must receive a secret or key only after
  successful attestation.

For the customer-managed-key design, the VM disk data is encrypted with a
data-encryption key (DEK). The DEK is protected by a key-encryption key (KEK)
that you control. The **KEK remains stored in Azure Key Vault or Managed HSM**;
it is not stored in the VM and is not returned by Azure Attestation.

The high-level release flow is:

1. The VM or workload produces AMD SEV-SNP attestation evidence.
2. Azure Attestation validates the evidence and returns a signed attestation
   token.
3. A key-release policy evaluates the attestation claims.
4. Key Vault or Managed HSM releases the permitted key material only when the
   policy is satisfied.
5. The VM or workload uses the released material to unwrap the DEK or access
   protected application data.

For confidential VM boot, Azure performs the platform-attestation and
key-release operations as part of the managed boot flow. For an application
that performs guest attestation, the application or its relying party must
validate the attestation token and request access to the protected secret or
key according to the configured policy.

Use **Managed HSM** when you need a dedicated, highly controlled,
HSM-backed key service. Use standard **Azure Key Vault** for general secret
and key management when its security and compliance features meet your
requirements. Do not place the KEK, private keys, or client secrets in the VM
image, source code, or Markdown files.

## Related Resources

- [Azure confidential VM overview](https://learn.microsoft.com/azure/confidential-computing/confidential-vm-overview)
- [Azure Attestation overview](https://learn.microsoft.com/azure/attestation/overview)
- [Guest attestation for confidential VMs](https://learn.microsoft.com/azure/confidential-computing/guest-attestation-confidential-vms)
