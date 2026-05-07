# Standards & API Reference

> Project: Early Childhood Education Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 21001:2025 — Educational Organizations Management Systems**
- URL: https://www.iso.org/standard/21001
- Provides a management system framework for any educational organisation, including early childhood settings. The 2025 revision (published July 2025) strengthens governance, integrates digital and AI tool requirements, and aligns with UN SDG 4 (Quality Education). Relevant for programs seeking accreditation or demonstrating systematic quality management to licensing bodies. Replaces ISO 21001:2018.

**ISO/IEC 27001 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The primary international standard for information security management. Relevant because ECE platforms store highly sensitive child data (developmental records, health information, photographs, family details) and are subject to COPPA, FERPA, and GDPR requirements. Certification signals to institutional childcare operators that the platform meets enterprise security requirements.

**ISO/IEC 29101 — Privacy Architecture Framework**
- URL: https://www.iso.org/standard/29101
- Privacy architecture framework for systems that process personally identifiable information. Applicable to ECE platforms given their handling of child PII, family contact data, and health records under multiple privacy regimes (COPPA, FERPA, GDPR, PIPEDA).

---

### W3C & IETF Standards

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the standard token format used in modern OAuth 2.0 / OpenID Connect authentication flows. Required for secure API authentication between parent apps, staff apps, and the platform backend.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundation for delegated authorisation used in modern identity systems. ECE platforms use OAuth 2.0 to authorise parent app access, third-party integrations (accounting systems, CACFP portals), and API clients.

**RFC 9110 — HTTP Semantics (HTTP/1.1 and HTTP/2)**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- Core web protocol standard underlying all REST API communication between ECE platform clients and servers.

**W3C Web Content Accessibility Guidelines (WCAG) 2.2**
- URL: https://www.w3.org/TR/WCAG22/
- Accessibility guidelines for web and mobile interfaces. Relevant for parent-facing apps that must serve families with disabilities, and for compliance with Section 508 (US federal-funded programs such as Head Start) and the European Accessibility Act.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the Link header and link relations for RESTful APIs. Relevant for hypermedia-driven API designs connecting child records, observations, and developmental frameworks.

---

### Data Model & API Specifications

**Ed-Fi Data Standard v6.0 (2025)**
- URL: https://docs.ed-fi.org/reference/data-exchange/data-standards/
- An education data exchange standard adopted by more than 25 US states, covering student demographics, enrollment, attendance, assessment, and early childhood data elements. Version 6.0 was released in 2025 for school years 2026–2027. ECE platforms seeking interoperability with state education data systems (for pre-K programs, Head Start data reporting, or QRIS) should align to Ed-Fi data models.

**Common Education Data Standards (CEDS) — P-20W Data Dictionary**
- URL: https://ceds.ed.gov/dataModel.aspx
- NCES-maintained vocabulary covering education data from preschool through workforce. CEDS includes early childhood data elements and is used as a reference model for state longitudinal data systems (SLDS) and early childhood data linkage (ECLDS). ECE platforms reporting to state agencies should map their data to CEDS elements.

**Schools Interoperability Framework (SIF)**
- URL: https://www.a4l.org/page/sif-specifications
- An open XML/JSON data-sharing specification used by US state departments of education for student data interchange. Covers enrollment, attendance, special education, and assessment data. Relevant for ECE platforms integrated with public school pre-K programs and state reporting portals.

**IEEE 9274.1.1-2023 — xAPI (Experience API / Tin Can API)**
- URL: https://standards.ieee.org/ieee/9274.1.1/7321/
- The IEEE-standardised learning activity tracking specification using an "[actor] [verb] [object]" data model stored in a Learning Record Store (LRS). While primarily used in corporate e-learning and K-12, xAPI is relevant for ECE platforms that track child engagement with curriculum activities, or for professional development modules serving ECE staff (Lillio Academy, TeachKloud Kloud Academy).

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de facto standard for documenting REST APIs. Any ECE platform that exposes a developer API (e.g., Illumine, Procare DayCare Works) should publish an OpenAPI 3.1 specification. Enables automated SDK generation and third-party integration development.

**JSON Schema (draft-07 / 2020-12)**
- URL: https://json-schema.org/
- Used for validating the structure of API payloads — critical for ECE data models covering child records, developmental observations, billing events, and attendance records.

---

### Security & Authentication Standards

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0; used for authenticating parents, staff, and administrators across ECE platform apps. Essential for single sign-on (SSO) support required by enterprise childcare operators using identity providers such as Azure AD or Google Workspace.

**COPPA (Children's Online Privacy Protection Act) Rule — Amended 2025**
- URL: https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa
- US federal regulation restricting the collection of personal information from children under 13. The FTC published final amendments in April 2025, with compliance required by April 22, 2026. Key changes include expanded definitions of personal information, new standards for mixed-audience services, enhanced parental notice and consent requirements, stricter data retention and deletion obligations, and strengthened safe harbor program requirements. Directly governs how ECE platforms collect, store, and share children's data.

**FERPA (Family Educational Rights and Privacy Act)**
- URL: https://studentprivacy.ed.gov/ferpa
- US federal law protecting student education records. Applies to ECE programs operating within public school systems or receiving certain federal funding. Childcare programs are generally considered educational agencies under FERPA, requiring compliance for all child records including developmental assessments, incident reports, and attendance data.

**GDPR (General Data Protection Regulation)**
- URL: https://gdpr.eu/
- EU data protection regulation applicable to ECE platforms operating in or serving users in Europe (UK, Ireland, EU member states). Governs consent, data minimisation, right to erasure, and data processor agreements. Directly relevant for platforms like Tapestry (UK), TeachKloud (Ireland), and Storypark (NZ/AU with EU customers).

**OWASP Top 10 (2021) — Web Application Security**
- URL: https://owasp.org/www-project-top-ten/
- The authoritative reference for web application security risks. ECE platforms store sensitive PII for minors; aligning backend and API development to OWASP Top 10 mitigations (injection, broken access control, cryptographic failures, etc.) is a baseline security requirement.

**PCI DSS v4.0**
- URL: https://www.pcisecuritystandards.org/
- Payment Card Industry Data Security Standard. Applicable to any ECE platform processing credit card payments directly. Platforms using third-party payment processors (Stripe, Braintree) may reduce scope but must still meet SAQ A/SAQ A-EP requirements for payment form handling.

---

### Regulatory & Domain-Specific Frameworks

**CACFP (Child and Adult Care Food Program) — USDA**
- URL: https://www.fns.usda.gov/cacfp
- Federal nutrition program providing reimbursement for meals and snacks served in childcare settings. Participating programs must maintain daily meal records (food served, children present), submit monthly claims, and retain records for 3 years. ECE platforms should support CACFP-compliant meal recording, automated claim generation, and audit-ready reports. 2026 guidelines emphasise whole foods and reduced processed ingredients.

**CCDF (Child Care and Development Fund) — ACF/HHS**
- URL: https://www.acf.hhs.gov/occ/ccdf
- Federal childcare subsidy funding administered through the states. Authorises child care subsidies for income-eligible families; rules for authorisation, attendance verification, and billing submission vary by state. ECE platforms supporting subsidy billing must implement state-specific rules engines for attendance-based or certificate-based billing.

**Head Start Program Performance Standards — 45 CFR Part 1302**
- URL: https://eclkc.ohs.acf.hhs.gov/policy/45-cfr-chap-xiii/1302-subpart-e-family-community-engagement
- Federal performance standards for Head Start and Early Head Start programs. Cover curriculum, child assessment (aligned to the Head Start Child Development and Early Learning Framework), family engagement, health services, and data reporting. ECE platforms serving Head Start grantees must support HSELOF (Head Start Early Learning Outcomes Framework) milestone tracking.

**NAEYC Accreditation Standards**
- URL: https://www.naeyc.org/accreditation
- The National Association for the Education of Young Children's program accreditation criteria. Widely referenced as a quality benchmark in the US ECE sector. Platforms supporting NAEYC-accredited centers should enable documentation aligned to NAEYC environment rating scales and curriculum standards.

**UK EYFS Statutory Framework 2025**
- URL: https://www.gov.uk/government/publications/early-years-foundation-stage-framework--2
- The UK government's statutory framework for early years education, updated effective September 2025. Sets learning and development requirements across seven areas of learning, safeguarding and welfare requirements, and assessment expectations. ECE platforms serving UK nurseries and reception classes must align developmental tracking to EYFS areas of learning and the Early Learning Goals.

---

## Similar Products — Developer Documentation & APIs

### Illumine
- **Description:** AI-powered childcare management software serving 3,000+ centers globally; the only major ECE platform with a confirmed open API.
- **API Documentation:** https://help.illumine.app (internal documentation; not publicly indexed)
- **SDKs/Libraries:** Not publicly documented; API accessible via REST with JSON payloads
- **Developer Guide:** Contact info@illumine.app for integration access
- **Standards:** REST/JSON; OpenAPI format not confirmed publicly; JSON reports API updated February 2025
- **Authentication:** Not publicly disclosed; likely API key or OAuth 2.0

### Procare Solutions (DayCare Works)
- **Description:** Long-established US childcare management platform with deep billing and subsidy features; exposes an API through its DayCare Works product.
- **API Documentation:** https://api.daycareworks.com (endpoint); https://www.procaresupport.com/procare-schoolcare-works/docs/setup-system-config-api
- **SDKs/Libraries:** Not publicly documented
- **Developer Guide:** https://www.procaresoftware.com/capabilities/third-party-integrations/ (Procare Partner Marketplace)
- **Standards:** REST; integration formats for SIS, QuickBooks, payroll systems
- **Authentication:** API key (device-level, via InSite integration settings)

### Brightwheel
- **Description:** Leading all-in-one US childcare management platform; no public API as of 2025.
- **API Documentation:** Not available
- **SDKs/Libraries:** Not available
- **Developer Guide:** Not available
- **Standards:** Internal only; My Food Program integration uses a proprietary data sync
- **Authentication:** N/A (no public API)

### Tapestry
- **Description:** UK learning journal and nursery management platform; aligned to EYFS framework.
- **API Documentation:** Not publicly documented
- **SDKs/Libraries:** Not available
- **Developer Guide:** Not available
- **Standards:** Internal; MIS export formats for UK school management systems
- **Authentication:** N/A (no public API)

### Storypark
- **Description:** NZ/AU/CA learning documentation and family engagement platform; acquired by Kinder M8 in 2025.
- **API Documentation:** No public API (confirmed); Kidsoft provides a proprietary API connector
- **SDKs/Libraries:** Kidsoft API connector: https://kidsoft.com.au/storypark/
- **Developer Guide:** Contact Storypark directly for integration inquiries
- **Standards:** Internal; data hosted per Australian Privacy Principles
- **Authentication:** N/A (no public API)

### Lillio (formerly HiMama)
- **Description:** US/Canadian childcare app emphasising curriculum and developmental documentation.
- **API Documentation:** Not publicly documented
- **SDKs/Libraries:** Not available
- **Developer Guide:** Not available
- **Standards:** Internal
- **Authentication:** N/A (no public API)

### Kaymbu
- **Description:** US ECE assessment and family engagement platform; integrates COR Advantage (HighScope) assessment.
- **API Documentation:** Not publicly documented
- **SDKs/Libraries:** Not available
- **Developer Guide:** Not available
- **Standards:** Aligned to Ed-Fi/CEDS data models for state reporting; COR Advantage uses HighScope's proprietary data format
- **Authentication:** N/A (no public API)

### Ed-Fi Alliance (Reference Implementation)
- **Description:** Open-source education data exchange standard with reference implementations for student and early childhood data interoperability.
- **API Documentation:** https://docs.ed-fi.org/reference/ods-api-platform/
- **SDKs/Libraries:** .NET SDK, Java SDK — https://github.com/Ed-Fi-Alliance-OSS
- **Developer Guide:** https://docs.ed-fi.org/getting-started/
- **Standards:** REST/JSON; Ed-Fi Data Standard v6.0; OpenAPI 3.0 specification published
- **Authentication:** OAuth 2.0 (client credentials flow)

### xAPI Learning Record Store (LRS) — ADL Initiative Reference
- **Description:** IEEE 9274.1.1-2023 compliant reference implementation for tracking learning activities.
- **API Documentation:** https://www.adlnet.gov/experience-api/
- **SDKs/Libraries:** TinCanJS, TinCan.NET, Rustici SCORM Cloud — https://xapi.com/libraries/
- **Developer Guide:** https://learnxapi.com/
- **Standards:** IEEE 9274.1.1-2023 (xAPI 2.0); REST/JSON; statement format: actor-verb-object
- **Authentication:** OAuth 1.0 or HTTP Basic (per xAPI spec)

---

## Notes

**API gap in the market:** The near-total absence of public APIs across the major ECE platforms (Brightwheel, Tapestry, Storypark, Lillio) is the single largest structural gap for developers and integrators. Illumine's open API and Procare's DayCare Works endpoint are the only documented integration surfaces in the category. An open-source ECE platform with a well-documented OpenAPI 3.1 REST API would be a significant competitive differentiator for enterprise operators, state agencies, and edtech integrators.

**Subsidy billing standards are fragmented:** No industry-wide standard exists for submitting childcare subsidy billing to state agencies. Each state's CCDF program has its own portal, data format, and submission process. This is an area where a shared open data standard (potentially built on Ed-Fi or CEDS data elements) would materially reduce integration burden for multi-state operators.

**HL7 FHIR for child health records:** FHIR R5 includes an Immunization resource and paediatric health profiles that could, in principle, allow ECE platforms to receive immunisation status and health alerts from paediatric EMR systems. No current ECE platform has implemented FHIR integration. The potential use case — automatically flagging when a child's immunisation records are incomplete at enrollment — represents an emerging opportunity at the intersection of health IT and ECE administration.

**COPPA 2025 amendments require immediate attention:** Any ECE platform launched after April 2026 must comply with the amended COPPA rule. Key engineering implications: explicit parental consent workflows at enrollment, defined data retention schedules with automated deletion, and documented data processor agreements for all third-party integrations (payment processors, AI model providers, analytics platforms) that receive child PII.
