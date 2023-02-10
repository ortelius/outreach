# SBOM and its usage in Software Delivery Life Cycle

This article briefly introduces SBOM and describes its usage in Software Delivery Life Cycle. 
## SBOM Definition
> A formal record containing the details and supply chain relationships of various components used in building software. Software developers and vendors often create products by assembling existing open source and commercial software components. The SBOM enumerates these components in a product (as defined by [NIST SP 800-161r1](https://doi.org/10.6028/NIST.SP.800-161r1)).

## SBOM Format
SBOMs can be generated in variety of formats (for both human and machine-readable format). The primary supported formats are:
- [CycloneDX](https://cyclonedx.org/): a full-stack BOM standard by OWASP community covering SBOM, SaaSBOM, HBOM, OBMOM, VDR, and VEX. [Click here](https://cyclonedx.org/specification/overview/) to learn more about the CycloneDX specification.
- [SPDX](https://spdx.dev/): an open standard for SBOM (ISO/IEC 5962:2021) supported by The Linux Foundation.
- [SWID](https://www.iso.org/standard/65666.html): a standard by the ISO/IEC for tagging software to optimize its identification and management.
- [Syft](https://github.com/anchore/syft): a CLI tool by [Anchore](https://anchore.com/) (written in Go) for generating SBOM from container images.

## Information in SBOM
While SBOM generates comprehensive information, these are the minimal viable information required:
- Supplier Name
- Component Name
- Unique Identifier for the component
- Version
- Component Hash
- Relationship
- Author/Creator
- License Information

## SBOM Usage
While there are many different areas of SBOM usage, below are the three most commonly used applicability of SBOM:
### #1 - Insights for open source licensing and compliance
- With increasing usage of open source and third-party software in building solutions across industries, an SBOM helps to comply with licensing obligations in a transparent way with your customers and other partners.
- An SBOM provides broader visibility to understand complex projects for better quality management. An organization can also use it for understanding the need for the required experience and expertise for the respective product. 

### #2 - Detect, prioritize and mitigate security vulenerabilties
- Identity potentially vulenerable componens and automate the detection process throughout the software supply chain process.
- Visibility into proprietary and open source components.
- Alert about potential security risks based on the version information available in the SBOM.

### #3 - Reduce operational risk and mitigate proactively
- With SBOM data bundled with every product version release, it ensures any operational risk is mitigated.
- At times, software components reach their end-of-life (EOL) and are not supported by the supplier. An SBOM enables to proactively reduce operational risk by highlighting such components and help mitigate them.
- Provide an SBOM to the customer or other partners for visibility and higher quality standards helps to highlight operational efficiency.

> An illustration of end-to-end SBOM data analysis flow is shown below:
![End-ot-end SBOM Flow](images/supply-chain.png)

## References
- [CycloneDX Object Model and Specification Overview](https://cyclonedx.org/specification/overview/)
- [The National Telecommunications and Information Administration (NTIA) Document on SBOM Roles and Benefits](https://www.ntia.gov/files/ntia/publications/ntia_sbom_use_cases_roles_benefits-nov2019.pdf)
- [The Minimum Elements For a Software Bill of Materials (SBOM)](https://www.ntia.doc.gov/files/ntia/publications/sbom_minimum_elements_report.pdf)
- [SBOM Life Cycle by NIST](https://www.nist.gov/itl/executive-order-14028-improving-nations-cybersecurity/software-security-supply-chains-software-1)

## Useful Tools
- [Online SBOM Generation for Demo](https://democert.org/sbom/)
- [Paketo Build Pack for SBOM Generation](https://paketo.io/docs/howto/sbom/)
- [CycloneDX Tool Center for SBOM Ecosystem](https://cyclonedx.org/tool-center/)
- [SPDX Online Tool for SBOM Validate, Compare](https://tools.spdx.org/app/)
- [CVE Online Search Utility](https://cve.mitre.org/cve/search_cve_list.html)

## Acronyms Used

| Acronym | Description |
|---------|-------------|
| CVE     | Common Vulnerabilities and Exposures |
| HBOM    | Hardware bills of materials: every physical piece or component used to build a product. |
| NIST    | NIST is the National Institute of Standards and Technology at the U.S. Department of Commerce. |
| OWASP   | The Open Web Application Security Project |
| SBOM    | Software Bill of Materials |
| SCA     | Software Composition Analysis |
| VDR     | Vulnerability Disclosure Report (VDR) is an attestation by a software vendor showing that the vendor has checked each component of a software product SBOM for vulnerabilities and reports on the details of any vulnerabilities reported by a NIST NVD search. |
| VEX     | Vulnerability-Exploitability eXchange: an attestation, a form of a security advisory that indicates whether a product or products are affected by a known vulnerability or vulnerabilities. |