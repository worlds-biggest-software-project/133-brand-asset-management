# Standards & API Reference

> Project: Brand Asset Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 16684-1:2012 — Extensible Metadata Platform (XMP), Part 1**
- URL: https://www.iso.org/standard/57421.html
- Adobe-originated open standard (now ISO-ratified) for embedding structured metadata in digital files (JPEG, PDF, video). DAM platforms read and write XMP to carry rights information, descriptive metadata, and provenance data across the asset lifecycle. XMP is the preferred embedding layer for IPTC Photo Metadata and C2PA manifests.

**ISO 16684-2:2014 — Extensible Metadata Platform (XMP), Part 2**
- URL: https://www.iso.org/standard/57422.html
- Specifies use of RELAX NG to formally describe serialized XMP metadata schemas, enabling machine-readable schema validation relevant to automated metadata pipelines in DAM systems.

**ISO 15489-1:2016 — Information and Documentation: Records Management**
- URL: https://www.iso.org/standard/62542.html
- International standard defining concepts and principles for the creation, capture, and management of records regardless of structure or form. Directly informs DAM retention policies, audit trail requirements, disposal schedules, and evidence-grade recordkeeping for regulated industries (pharma, finance, legal).

**ISO 15836:2009 (ANSI/NISO Z39.85) — Dublin Core Metadata Element Set**
- URL: https://www.iso.org/standard/52142.html
- Internationally recognized 15-element metadata vocabulary (title, creator, subject, description, publisher, date, format, rights, etc.) providing a cross-system baseline for resource description. DAM platforms implement Dublin Core as an interoperability layer for asset exchange with content management systems and library systems.

---

### W3C & IETF Standards

**IPTC Photo Metadata Standard 2025.1**
- URL: https://iptc.org/standards/photo-metadata/ and https://www.iptc.org/std/photometadata/specification/IPTC-PhotoMetadata-2025.1.html
- The most widely adopted standard for embedding copyright, usage rights, and descriptive metadata directly into image files. The 2025.1 revision (November 2025) added four new properties for AI-generated content: AI Prompt Information, AI Prompt Writer Name, AI System Used, and AI System Version Used. Covers IPTC Core and IPTC Extension schemas. Open-source tool ExifTool supports all 2025.1 properties from version 13.40 onward.

**EXIF 2.2 — Exchangeable Image File Format**
- URL: https://www.loc.gov/preservation/digital/formats/fdd/fdd000146.shtml and https://exiftool.org/TagNames/EXIF.html
- Industry standard (originally JEIDA/JEITA) for camera-captured technical metadata embedded in image files (JPEG, TIFF, PNG, RAW). Records device, capture settings, orientation, timestamp, and GPS data. DAM ingest pipelines extract EXIF data to auto-populate technical metadata fields, enabling filtering by camera, date, and location.

**C2PA — Content Credentials Technical Specification 2.4**
- URL: https://spec.c2pa.org/ and https://contentcredentials.org/
- Coalition for Content Provenance and Authenticity (C2PA) open specification (co-led by Adobe, Microsoft, Google, OpenAI, BBC, Sony) for cryptographically binding provenance metadata to digital assets. Enables an auditable chain of custody recording asset origin, modifications, and AI involvement. Expected to be adopted as an ISO international standard; W3C is reviewing for browser-level support. Critical for DAM systems managing AI-generated content.

**RFC 4918 — HTTP Extensions for Web Distributed Authoring and Versioning (WebDAV)**
- URL: https://datatracker.ietf.org/doc/html/rfc4918
- IETF standard extending HTTP with resource property management, collection creation, URL namespace manipulation, and file locking. Many enterprise DAM platforms expose WebDAV endpoints enabling desktop clients (Finder, Explorer) to mount asset libraries as network drives, supporting creative workflow integration without bespoke plugins.

**RFC 6749 / RFC 6750 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749 and https://datatracker.ietf.org/doc/html/rfc6750
- Standard authorization protocol enabling third-party applications to access DAM APIs on behalf of users without sharing credentials. All major commercial DAM APIs (Bynder, Brandfolder, Canto, Frontify, Adobe AEM) use OAuth 2.0 as their primary API authentication mechanism. OAuth 2.1 (draft) will tighten security requirements further.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0, providing standardized user authentication and Single Sign-On (SSO). Enterprise DAM deployments universally require OIDC or SAML 2.0 for integration with corporate identity providers (Okta, Azure AD, Ping Identity). Most SaaS DAM platforms support OIDC by default.

**Schema.org — ImageObject / CreativeWork / MediaObject**
- URL: https://schema.org/ImageObject and https://schema.org/CreativeWork
- W3C community-maintained structured data vocabulary for describing digital assets in machine-readable form. The `ImageObject`, `AudioObject`, `VideoObject`, and `CreativeWork` types include copyright, author, credit, and license properties that map to IPTC metadata fields. Relevant for SEO-optimised asset delivery and API responses that surface asset metadata to downstream content systems.

---

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1.0 / 3.2.0**
- URL: https://spec.openapis.org/oas/v3.1.0.html and https://www.openapis.org/
- Industry-standard YAML/JSON format for describing RESTful APIs covering endpoints, request/response schemas, authentication, and error codes. All major DAM vendors are migrating to OpenAPI-documented APIs (Adobe AEM, Brandfolder via Smartsheet, Piwigo). Essential for code generation, SDK production, and contract-based testing.

**PLUS — Picture Licensing Universal System**
- URL: https://ns.useplus.org/ and https://www.useplus.com/
- International non-profit standard for communicating image rights and licensing terms in machine-readable form. PLUS rights strings can be embedded in IPTC or XMP metadata fields and read by DAM systems to automate usage rights enforcement, license expiry tracking, and rights validation workflows. Used by stock agencies, publishers, and media archives.

---

### Security & Authentication Standards

**GDPR — EU General Data Protection Regulation (2016/679)**
- URL: https://gdpr-info.eu/
- EU regulation governing personal data processing, including biometric data (Article 9 "special category"). Facial recognition features in DAM platforms (used for talent tagging and management) require explicit consent, Data Protection Impact Assessments (DPIAs), and strict security measures. Non-compliance penalties reach €20M or 4% of global turnover. The European Data Protection Board (EDPB) confirmed in 2025 that consent for biometric processing must be freely given, specific, informed, and unambiguous.

**EU AI Act (Regulation 2024/1689)**
- URL: https://artificialintelligenceact.eu/ and https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai
- Phased regulation governing AI systems in the EU. From August 2025, obligations apply to providers of general-purpose AI models (GPAI), including transparency, documentation, and copyright compliance requirements. DAM platforms embedding AI auto-tagging, facial recognition, or generative AI features must assess their AI system risk classification and comply with conformity assessment, technical documentation, and data governance requirements. Penalties reach €35M or 7% of worldwide turnover for prohibited practices.

**WIPO Copyright Treaty (WCT) and Digital Millennium Copyright Act (DMCA)**
- URL: https://www.wipo.int/en/web/copyright/activities/internet_treaties and https://www.copyright.gov/legislation/dmca.pdf
- International and US legal frameworks governing technological measures for copyright protection and rights management information. DAM systems must not remove or circumvent embedded rights management information (XMP/IPTC rights fields). Relevant to DRM modules, watermarking, and audit trail features. Jurisdictions vary; platforms serving global enterprises must account for treaty signatories' local implementations.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- De-facto security standard for API design and testing. DAM platforms exposing public APIs must address broken object-level authorization, broken authentication, excessive data exposure, and injection vulnerabilities. Recommended reference for designing DAM API permission scopes, rate limiting, and input validation.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic-originated open protocol for connecting AI language models to external tools and data sources. DAM platforms can expose MCP servers enabling AI agents to search, retrieve, annotate, or publish assets via natural language without custom API integrations. IntelligenceBank has noted MCP server availability for API documentation. Relevant to AI-native DAM architecture where LLM agents orchestrate asset workflows.

---

## Similar Products — Developer Documentation & APIs

### Bynder

- **Description:** Enterprise DAM SaaS with brand portals, creative workflow management, and agentic AI automation. Trusted by 4,000+ global brands.
- **API Documentation:** https://developers.bynder.com/ and https://api.bynder.com/docs/getting-started
- **SDKs/Libraries:**
  - JavaScript (Node.js): https://github.com/Bynder/bynder-js-sdk
  - Python: https://pypi.org/project/bynder-sdk/
  - Java: https://github.com/Bynder/bynder-java-sdk
  - C# and PHP SDKs also available via developer portal
- **Developer Guide:** https://developers.bynder.com/sdks
- **Standards:** REST/JSON; OAuth 2.0; Webhooks
- **Authentication:** OAuth 2.0

---

### Brandfolder (by Smartsheet)

- **Description:** Modern DAM with AI auto-tagging, Smart CDN delivery, brand templating, and deep Adobe/Figma integrations. Acquired by Smartsheet in 2021.
- **API Documentation:** https://developers.brandfolder.com/ and https://developers.brandfolder.com/docs/
- **OpenAPI Reference:** https://developers.smartsheet.com/api/brandfolder/openapi
- **SDKs/Libraries:**
  - Python: https://github.com/brandfolder/brandfolder-sdk-python (via `pip install brandfolder`)
  - PHP: https://github.com/brandfolder/brandfolder-sdk-php (community maintained)
- **Developer Guide:** https://developers.brandfolder.com/recipes/
- **Standards:** REST/JSON; OpenAPI; Webhooks; Zapier integration
- **Authentication:** Bearer token (API key)

---

### Canto

- **Description:** Mid-market DAM with AI visual search, facial recognition (Amazon Rekognition), Smart Albums, and Smart Tags.
- **API Documentation:** https://www.canto.com/api/ and https://canto1.portal.swaggerhub.com/
- **SDKs/Libraries:** Universal Connector (open-source sample application for integration development)
- **Developer Guide:** https://support.canto.com/hc/en-us/articles/23002380063505-Guideline-for-integrating-Canto-with-a-third-party-system
- **Standards:** REST/JSON; OAuth 2.0
- **Authentication:** OAuth 2.0

---

### Frontify

- **Description:** Brand management platform combining DAM, brand guidelines, and creative collaboration with an AI Brand Assistant.
- **API Documentation:** https://developer.frontify.com/
- **SDKs/Libraries:**
  - Brand SDK (React/TypeScript): https://github.com/Frontify/brand-sdk
- **Developer Guide:** https://help.frontify.com/en/articles/5402354-overview-of-frontify-developer-tools
- **Standards:** GraphQL API; Webhooks; REST (via App Bridge)
- **Authentication:** OAuth 2.0 (via App Bridge / Frontify Authenticator widget)

---

### Adobe Experience Manager (AEM) Assets

- **Description:** Enterprise DAM tightly integrated with Adobe Creative Cloud, Adobe Express, and the broader Adobe marketing stack. Available as SaaS and on-premise.
- **API Documentation:** https://developer.adobe.com/experience-cloud/experience-manager-apis/ and https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/assets/admin/mac-api-assets
- **OpenAPI Reference:** https://experienceleague.adobe.com/en/docs/experience-manager-cloud-service/content/implementing/developing/open-api-based-apis
- **SDKs/Libraries:**
  - Adobe Asset Link (Photoshop, InDesign, XD, Illustrator plugins)
  - Adobe Creative Cloud APIs
- **Developer Guide:** https://experienceleague.adobe.com/en/docs/experience-manager-learn/cloud-service/aem-apis/overview
- **Standards:** REST/JSON; OpenAPI 3.x; WebDAV
- **Authentication:** OAuth 2.0 (Server-to-Server); IMS (Adobe Identity Management System)

---

### MediaValet

- **Description:** Enterprise DAM with AI auto-tagging, video transcription/translation, facial recognition, and unlimited user seats. Positioned for media and entertainment workflows.
- **API Documentation:** https://docs.mediavalet.com/ and https://developer.mediavalet.com/
- **SDKs/Libraries:**
  - Python SDK (mvsdk): https://github.com/armstro-ca/mvsdk
- **Developer Guide:** https://developer.mediavalet.com/getting-started
- **Standards:** REST/JSON; Webhooks; PowerShell support
- **Authentication:** OAuth 2.0

---

### IntelligenceBank

- **Description:** DAM with compliance-first workflows, digital rights management, audit trails, and marketing regulatory compliance for pharma, finance, and legal industries.
- **API Documentation:** https://apidoc.intelligencebank.com and https://intelligencebank.atlassian.net/wiki/spaces/APIDOC/overview
- **SDKs/Libraries:** Available via GitHub (https://github.com/intelligencebank)
- **Developer Guide:** https://intelligencebank.com/integrations/
- **Standards:** REST/JSON; Webhooks
- **Authentication:** API key / OAuth (varies by integration type)

---

### Piwigo (Open Source)

- **Description:** Open-source photo library and DAM software (AGPL v3) with 20+ years of development, 2.8M+ installs. Self-hosted or SaaS. PHP/MySQL stack.
- **API Documentation:** https://github.com/Piwigo/Piwigo/wiki/Piwigo-Web-API and https://doc.piwigo.org/
- **SDKs/Libraries:**
  - Python: https://pypi.org/project/piwigo/
  - Ruby: https://github.com/KKDad/piwigo-api
- **Developer Guide:** https://piwigo.com/blog/2026/01/13/piwigo-api-unlock-the-full-potential-of-your-photo-library/
- **Standards:** REST/JSON (Web API); plugin architecture
- **Authentication:** Session-based or personal API keys (X-PIWIGO-API header, since Piwigo 16.1)

---

### ResourceSpace (Open Source)

- **Description:** Fully open-source DAM with multi-format support, metadata management, workflow automation, and REST API. Self-hosted PHP/MySQL.
- **API Documentation:** https://www.resourcespace.com/knowledge-base/developers/ and https://www.resourcespace.com/glossary/rest_api
- **SDKs/Libraries:** Community plugins; no official SDK
- **Developer Guide:** https://www.resourcespace.com/knowledge-base/developers/
- **Standards:** REST/JSON (signed with shared private key); plugin architecture
- **Authentication:** HMAC-signed API requests (shared private key per user)

---

## Notes

**Emerging standards to monitor:**

- **C2PA / Content Credentials** is moving rapidly toward ISO standardization and is expected to become a DAM table-stakes requirement as enterprises manage growing volumes of AI-generated content. No major DAM platform has fully integrated C2PA yet — this represents a near-term opportunity.
- **IPTC Photo Metadata 2025.1 AI Properties** (AI Prompt Information, AI System Used) are newly standardized and not yet widely implemented in DAM products. Early adoption would differentiate an AI-native DAM.
- **OAuth 2.1** (IETF draft) will supersede OAuth 2.0 with stricter requirements (removing implicit flow, requiring PKCE). DAM API designs should target OAuth 2.1 compliance from the outset.
- **EU AI Act GPAI obligations** (August 2025) affect any DAM platform embedding third-party or proprietary general-purpose AI models for tagging, search, or content generation — technical documentation and copyright compliance policies are required.
- **MCP (Model Context Protocol)** is gaining adoption as the standard integration layer for LLM-powered agents. Exposing a DAM as an MCP server could unlock agent-driven asset discovery, annotation, and publishing workflows without custom API integrations.

**Gaps in public information:**

- IntelligenceBank's API surface is documented on Atlassian Confluence behind login; full schema details are not publicly accessible.
- Orange Logic CORTEX provides no public API documentation; integration requires direct vendor engagement.
- Digital rights management data formats (PLUS strings, license schema) lack a universal machine-readable exchange format beyond IPTC/XMP embedding; this remains an underserved standardisation gap.
