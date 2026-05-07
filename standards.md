# Standards & API Reference

> Project: Autonomous Finance Operations · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 20022 — Universal Financial Industry Message Scheme**
- Official URL: https://www.iso20022.org/ | SWIFT overview: https://www.swift.com/standards/iso-20022
- The global standard for cross-border payment messages, replacing legacy SWIFT MT formats. From November 2025 the SWIFT coexistence period ended; full migration to ISO 20022 is mandatory by November 2026. All payment instructions must carry structured country codes, town names, and CBPR+ address fields. Critical for any platform handling cross-border payments, SWIFT file generation, or multi-bank treasury connectivity.

**ISO 9735 — Electronic Data Interchange for Administration, Commerce and Transport (EDIFACT)**
- Official URL: https://unece.org/trade/uncefact/introducing-unedifact
- The UN/EDIFACT standard underpins international EDI invoice exchange. Less prevalent in North America (where ANSI X12 dominates), but required for European and Asia-Pacific supplier EDI connectivity. Relevant for any autonomous AP platform targeting multinational corporates.

**ISO 8583 — Financial Transaction Card Message Standard**
- Official URL: https://www.iso.org/standard/31628.html
- Governs card-based financial transactions at the message level. Relevant for any autonomous finance platform that processes virtual card or purchasing card (P-card) payments as part of AP disbursements.

---

### W3C & IETF Standards

**XBRL / Inline XBRL (iXBRL) — eXtensible Business Reporting Language**
- Official URL: https://www.xbrl.org/ | Specifications: https://specifications.xbrl.org/
- The international open standard for digital financial reporting, mandated by the SEC, HMRC, ACRA, and regulators in 50+ jurisdictions. iXBRL embeds structured XBRL tags within human-readable HTML, creating a single document that is both machine-parseable and audit-ready. An autonomous finance platform generating regulatory financial reports or month-end close outputs must support XBRL tagging to satisfy statutory filing requirements.

**RFC 7519 — JSON Web Token (JWT)**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard token format used in OAuth 2.0 / OpenID Connect authentication flows. Required for any finance platform that issues or validates API tokens when connecting to ERP systems, banking APIs, or third-party financial services.

**RFC 7523 — JSON Web Token Profile for OAuth 2.0 Client Authentication**
- Official URL: https://datatracker.ietf.org/doc/html/rfc7523
- Governs service-to-service (machine-to-machine) authentication using JWT assertions. Directly relevant for the OAuth 2.0 flows used to authenticate autonomous finance agents with ERP and banking APIs without user interaction.

**RFC 8288 — Web Linking**
- Official URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header for hypermedia APIs. Relevant for pagination and resource navigation in REST API integrations with ERP connectors.

---

### Data Model & API Specifications

**ANSI X12 EDI — ASC X12 Transactions for Finance**
- Official URL: https://www.x12.org/
- The dominant EDI standard in North America. Key transaction sets for autonomous finance:
  - **EDI 810** (Invoice): structured invoice from supplier to buyer — core input format for autonomous AP
  - **EDI 820** (Payment Order/Remittance Advice): structured payment remittance — required for automated cash application matching
  - **EDI 850** (Purchase Order): used in 3-way matching against invoices
  - **EDI 856** (Advance Ship Notice): used in 3-way matching for goods receipt confirmation
- An autonomous AP platform must be able to ingest EDI 810 and emit EDI 820.

**BAI2 — Bank Administration Institute Cash Management File Format**
- Overview: https://www.sepaforcorporates.com/swift-for-corporates/bai2-format-specification/
- The dominant cash management reporting format in the US. Provides end-of-day account balances and transaction detail from banks in a structured text format. Essential for automated bank reconciliation — the BAI2 file feeds the automated matching process between bank transactions and internal ledger entries.

**SWIFT MT940 / CAMT.053 — Bank-to-Customer Account Statement**
- MT940 overview: https://www.debtbook.com/blog/common-bank-reporting-payment-file-formats-bai2-edi-iso-20022-xml
- SWIFT MT940 is the equivalent of BAI2 for non-US banks. CAMT.053 is the ISO 20022 XML successor format. An autonomous bank reconciliation module must support all three formats (BAI2, MT940, CAMT.053) to handle multi-bank, multi-region corporate treasury environments.

**OpenAPI Specification 3.1**
- Official URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for documenting REST APIs. All public-facing APIs in the autonomous finance platform should be documented in OpenAPI 3.1 format to enable third-party integrators, ERP vendors, and AI agent tools to consume them programmatically. The NetSuite REST API Browser is itself built on OpenAPI 3.0 metadata.

**JSON Schema (Draft 2020-12)**
- Official URL: https://json-schema.org/
- Standard for defining and validating the structure of JSON payloads. Required for validating ERP API request/response schemas, invoice data extraction output schemas, and agent action payloads.

**Model Context Protocol (MCP) — Specification 2025-11-25**
- Official URL: https://modelcontextprotocol.io/specification/2025-11-25
- Open standard (Anthropic, donated to Linux Foundation Agentic AI Foundation, December 2025) for connecting AI models to external tools, data sources, and systems. As of May 2026, MCP has 97M+ monthly SDK downloads and is supported by Anthropic, OpenAI, Google, Microsoft, and AWS. Dynamics 365 Finance & Operations has published MCP documentation for Copilot integration. An autonomous finance platform should expose an MCP server to allow external AI agents to invoke finance workflows (query invoice status, trigger payment runs, read cash positions) in a standardised, governed way.

---

### Security & Authentication Standards

**OAuth 2.0 — RFC 6749**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6749
- The industry standard authorisation framework. All ERP API integrations (NetSuite, SAP, Oracle, Dynamics, Xero, QuickBooks) use OAuth 2.0 for delegated access. The platform must implement OAuth 2.0 client credentials flow for machine-to-machine integration and authorisation code flow for user-delegated access.

**OpenID Connect 1.0 (OIDC)**
- Official URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Required for SSO integration with enterprise identity providers (Okta, Azure AD, Ping Identity) that corporate finance teams use for access management. OIDC tokens carry user identity claims needed for audit trail attribution.

**OWASP API Security Top 10**
- Official URL: https://owasp.org/www-project-api-security/
- Operational reference for identifying and mitigating API vulnerabilities. PCI DSS 4.0 Section 6.2.4 mandates automated scanning for OWASP API Security Top 10 vulnerabilities for all public-facing APIs. Key issues for finance APIs: Broken Object Level Authorization (BOLA), Broken Function Level Authorization, and Excessive Data Exposure.

**PCI DSS 4.0.1**
- Official URL: https://www.pcisecuritystandards.org/standards/
- Mandatory compliance standard for any component that handles cardholder data (virtual card AP disbursements, purchasing card processing). Version 4.0 introduced 50+ new API security requirements, with final implementation deadline March 2025. Requirement 6.3.2 mandates maintaining an inventory of all custom software and APIs in scope.

**NIST Cybersecurity Framework 2.0**
- Official URL: https://www.nist.gov/cyberframework
- The US federal and enterprise standard for cybersecurity risk management. Particularly relevant for autonomous finance platforms deployed by public companies subject to SOX — auditors will expect controls mapped to a recognised security framework.

**SOX — Sarbanes-Oxley Act (Sections 302 and 404)**
- Reference: https://www.sec.gov/spotlight/sarbanes-oxley.htm
- US federal law requiring public companies to maintain documented internal controls over financial reporting and subject them to annual auditor attestation. Every AI agent action that creates or modifies a financial record must be logged with user identity, timestamp, rationale, before/after values, and policy citation to satisfy SOX Section 404 controls. Segregation of duties controls must prevent any single agent from initiating and approving its own payment.

**GDPR / CCPA — Data Protection Regulations**
- GDPR: https://gdpr-info.eu/ | CCPA: https://oag.ca.gov/privacy/ccpa
- Vendor and customer personal data processed during AP/AR workflows (name, address, bank account details) is subject to GDPR in the EU and CCPA in California. The platform must implement data residency controls, right-to-erasure workflows, and consent management for any personal data touched by autonomous agents.

---

### MCP Server Specifications

**Model Context Protocol — Agentic AI Foundation (Linux Foundation)**
- Specification: https://modelcontextprotocol.io/specification/2025-11-25
- GitHub: https://github.com/modelcontextprotocol
- Governance: Donated to Linux Foundation AAIF (December 2025); co-founded by Anthropic, Block, and OpenAI
- An MCP server for the autonomous finance platform would expose tools such as: `get_invoice_status`, `get_cash_position`, `list_open_payables`, `trigger_payment_run`, `get_bank_reconciliation_exceptions`. This enables any MCP-compatible AI assistant (Claude, GPT-4o, Gemini) to query finance data and invoke finance workflows through a governed, auditable interface.
- Microsoft Dynamics 365 Finance & Operations has already published MCP documentation: https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/copilot/copilot-mcp

---

## Similar Products — Developer Documentation & APIs

### NetSuite (Oracle)

- **Description:** Oracle's cloud ERP platform dominant in mid-market finance. Provides AP, AR, GL, cash management, and procurement modules. The platform that most autonomous finance tools connect to first.
- **API Documentation:** https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/chapter_1540391670.html
- **REST API Reference:** https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_157373386674.html
- **SDKs/Libraries:** SuiteTalk SOAP (being deprecated in 2026.1), SuiteTalk REST, RESTlets, SuiteQL for querying
- **Developer Guide:** https://satvasolutions.com/blog/netsuite-integration-guide (community guide); official: docs.oracle.com
- **Standards:** OpenAPI 3.0 metadata for REST API Browser; OAuth 2.0 with Token-Based Authentication (TBA); SuiteQL (SQL-like query language over REST)
- **Authentication:** OAuth 2.0 / Token-Based Authentication (TBA); transitioning from SOAP/username+password to REST/OAuth 2.0 in 2026.1
- **Note:** NetSuite 2026.1 deprecates default SOAP Web Services; all new integrations must use REST.

---

### SAP S/4HANA / SAP Business Technology Platform (BTP)

- **Description:** SAP's flagship enterprise ERP suite, dominant in large enterprises. Finance modules include FI (Financial Accounting), CO (Controlling), AP, AR, Treasury, and Asset Management.
- **API Documentation:** https://api.sap.com/ (SAP Business Accelerator Hub)
- **SDKs/Libraries:** SAP Cloud SDK (JavaScript/TypeScript, Java); Python client libraries via OpenAPI generator
- **Developer Guide:** https://developers.sap.com/
- **Standards:** OData v4 (primary API protocol for S/4HANA); REST/JSON for BTP services; OpenAPI 3.0 specifications published on Business Accelerator Hub
- **Authentication:** OAuth 2.0 (XSUAA service on BTP); SAP-specific JWT token structure
- **Note:** The SAP Business Accelerator Hub (api.sap.com) lists 3,000+ pre-built APIs including Invoice Management, Payment Processing, Bank Account Management, and Cash Management.

---

### Microsoft Dynamics 365 Finance & Operations

- **Description:** Microsoft's enterprise ERP for financial management, supply chain, and operations. Strong in mid-market and enterprise, deeply integrated with Azure and Microsoft 365.
- **API Documentation:** https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/odata
- **MCP Documentation:** https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/copilot/copilot-mcp
- **SDKs/Libraries:** .NET SDK; Python client via Azure Data Factory connectors; Power Platform connectors
- **Developer Guide:** https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/
- **Standards:** OData v4; REST/JSON; OpenAPI 3.0; MCP (Copilot integration); Dataverse API
- **Authentication:** Azure Active Directory (AAD) / Microsoft Entra ID; OAuth 2.0; service principal authentication
- **Note:** Dynamics 365 2026 Wave 1 introduces agentic AI features natively for finance teams.

---

### QuickBooks Online (Intuit)

- **Description:** Cloud accounting platform dominant in SMB and lower mid-market. Core AP, AR, invoicing, and bank reconciliation capabilities.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **SDKs/Libraries:** Official SDKs for Java, PHP, Python, .NET, Ruby, Node.js
- **Developer Guide:** https://developer.intuit.com/app/developer/qbo/docs/get-started
- **Standards:** REST/JSON; OAuth 2.0 (mandatory since 2019, no API key auth); OpenAPI-compatible endpoints
- **Authentication:** OAuth 2.0 authorisation code flow; refresh tokens with 100-day expiry
- **Note:** Supports invoice (Bill) CRUD, vendor management, payment recording, bank transaction import. Sandbox environment available for testing.

---

### Xero

- **Description:** Cloud accounting platform popular with SMBs and accountants in Australia, NZ, UK, and globally. Provides AP, AR, bank reconciliation, payroll, and reporting.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **SDKs/Libraries:** Official SDKs for .NET, Java, Node.js, PHP, Python, Ruby; community Go and Swift SDKs
- **Developer Guide:** https://developer.xero.com/
- **Standards:** REST/JSON; OAuth 2.0; OpenAPI 3.0 spec published for the Accounting API
- **Authentication:** OAuth 2.0 authorisation code flow with PKCE; machine-to-machine via client credentials
- **Note:** Strong developer experience; 7,000+ apps integrated via Xero App Store. Webhooks available for real-time event notifications on invoices, payments, and bank transactions.

---

### Plaid

- **Description:** Financial data infrastructure platform. Provides bank account connectivity, transaction data, balance checks, and identity verification for 10,000+ institutions. Critical for automating bank feed ingestion and reconciliation in autonomous finance platforms targeting SMBs.
- **API Documentation:** https://plaid.com/docs/
- **SDKs/Libraries:** Official SDKs for JavaScript/Node.js, Python, Ruby, Java, Go, .NET
- **Developer Guide:** https://plaid.com/docs/quickstart/
- **Standards:** REST/JSON; OpenAPI 3.0 spec available; builds on FDX (Financial Data Exchange) open standard for data sharing
- **Authentication:** Client ID + Secret for server-side calls; OAuth 2.0 Link flow for end-user bank authorisation
- **Note:** Plaid Transactions API returns categorised bank transactions. Plaid Balance API returns real-time balances. Sandbox environment with simulated institutions. Relevant for automating bank statement ingestion in lieu of BAI2/MT940 file uploads.

---

### Modern Treasury

- **Description:** Payment operations platform providing ACH, wire, RTP, and check payment orchestration via API. Designed for companies that need programmatic B2B payment execution with reconciliation built in.
- **API Documentation:** https://docs.moderntreasury.com/
- **SDKs/Libraries:** Python, Ruby, Node.js, Java, Go, .NET SDKs
- **Developer Guide:** https://docs.moderntreasury.com/quickstart
- **Standards:** REST/JSON; NACHA ACH compliance; supports ISO 20022 payment messages; idempotency keys on all payment creation endpoints
- **Authentication:** HTTP Basic Auth (API key + organization ID)
- **Note:** Provides ledgering (double-entry bookkeeping via API), payment order lifecycle management, and reconciliation matching. Strong fit for the payment execution layer of an autonomous finance platform.

---

### Tipalti Developer API

- **Description:** Global AP automation and mass payment platform with open APIs for invoice intake, approval workflow, payment execution, and reconciliation.
- **API Documentation:** https://documentation.tipalti.com/
- **SDKs/Libraries:** REST API; SOAP API (legacy); no official language SDKs published
- **Developer Guide:** https://documentation.tipalti.com/docs/getting-started
- **Standards:** REST/JSON; SOAP/XML (legacy); OAuth 2.0
- **Authentication:** OAuth 2.0 bearer token
- **Note:** API supports payee onboarding, invoice submission, payment status, and remittance retrieval. Supports 50+ currencies and 190+ countries. Relevant as a payment execution backend for autonomous AP platforms targeting global payables.

---

### HighRadius REST API (Data-as-a-Service)

- **Description:** Autonomous finance platform with REST-based data sharing API for O2C metrics. Exposes receivables, payment matching, and cash forecasting data programmatically.
- **API Documentation:** https://www.highradius.com/software/integration-capabilities/
- **SDKs/Libraries:** REST-based API; no public SDK published
- **Developer Guide:** https://www.highradius.com/resources/glossary/genar/application-programming-interface-apis/
- **Standards:** REST/JSON; OAuth 2.0; OpenAPI-compatible (inferred from REST API Browser)
- **Authentication:** OAuth 2.0
- **Note:** Integration complexity is notable — hybrid SFTP/API architecture with middleware connector (HEX Extractor) required for full ERP sync. Implementation typically 3–6 months. A reference for what enterprise-grade autonomous finance API integration looks like at scale.

---

## Notes

**Emerging and evolving areas:**

1. **ISO 20022 full adoption (November 2026):** The complete cutover from SWIFT MT formats to ISO 20022 for cross-border payments is the single most significant infrastructure change for finance teams in 2026. Any autonomous finance platform handling cross-border payments or SWIFT connectivity must be ISO 20022 native before this deadline.

2. **MCP as the AI-finance integration standard:** The Model Context Protocol is rapidly becoming the de facto standard for AI agent connectivity to business systems. Microsoft Dynamics 365 has already published MCP documentation, and the broader ERP ecosystem is expected to follow. Building an MCP server as part of the autonomous finance platform allows other AI systems to invoke finance workflows without custom integrations.

3. **Open Banking / FDX in the US:** The Financial Data Exchange (FDX) standard (https://financialdataexchange.org/) is the US equivalent of PSD2 open banking, providing standardised APIs for consumers and businesses to share financial data. The CFPB's Personal Financial Data Rights rule (Section 1033) enacted in 2024 accelerates FDX adoption. An autonomous finance platform connecting to US bank accounts should evaluate FDX-compliant connectivity alongside Plaid.

4. **Real-Time Payments (RTP) and FedNow:** The Federal Reserve's FedNow network (launched 2023, now broadly adopted) and The Clearing House RTP network enable instant B2B payments 24/7/365. An autonomous AP platform that can execute payments on RTP/FedNow removes the overnight ACH settlement delay, enabling same-day supplier payment execution within approved limits.

5. **Regulatory AI governance (emerging):** The EU AI Act (enforced from August 2026 for high-risk AI systems) classifies AI systems that make consequential financial decisions as high-risk applications requiring conformity assessments, transparency documentation, and human oversight mechanisms. An open-source autonomous finance platform should design its governance and audit trail architecture with EU AI Act compliance in mind from the outset.
