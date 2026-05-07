# Early Childhood Education Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform that unifies child development tracking, parent communication, and billing for daycare centers, preschools, and family childcare programs.

Early childhood education programs operate at the intersection of education, healthcare, and family services. Directors and teachers spend disproportionate time on administration -- documenting observations, managing complex multi-source billing, and communicating with families across disconnected tools -- rather than focusing on program quality. This platform replaces that fragmented workflow with an integrated system purpose-built for ECE operations, from a single-site home daycare to a multi-location childcare operator.

---

## Why Early Childhood Education Platform?

- **Every major incumbent is proprietary and closed.** No open-source ECE management platform with comparable feature depth exists. Centers are locked into vendor ecosystems with no data portability and no ability to extend functionality. Brightwheel, the market leader, does not offer a public API.
- **Subsidy billing is a state-by-state nightmare.** Child care subsidy programs (CCDF, TANF, pre-K vouchers) vary enormously across all 50 US states. No current platform offers a comprehensive rules engine covering all states with automated billing submission -- Procare and iCare have partial coverage, but gaps cause revenue leakage.
- **Incumbent pricing penalizes small operators.** Procare users report frequent price increases and bundled pricing that forces payment for unused features. ChildPilot targets the underserved small-center segment, but lacks developmental documentation. Centers operating on thin margins deserve transparent, affordable tooling.
- **Developmental documentation is disconnected from curriculum.** Most platforms treat observations, assessments, and lesson planning as separate workflows. Only Lillio and TeachKloud attempt deep curriculum alignment, but both are proprietary and regionally focused. No platform supports configurable alignment to arbitrary frameworks (NAEYC, EYFS, HighScope, Montessori, state standards) from a single codebase.
- **Child data portability does not exist.** No standard data exchange format exists for child developmental records. Switching platforms is lossy and labor-intensive, trapping programs in vendor relationships regardless of satisfaction.

---

## Key Features

### Child Development & Learning Documentation

- Digital observation capture (photos, videos, written notes) linked to specific children and developmental domains
- Configurable developmental framework alignment (NAEYC, state early learning standards, Head Start ELOF, EYFS, HighScope COR, Montessori)
- Milestone tracking across cognitive, language, social-emotional, and physical domains
- Portfolio-style learning journals with chronological developmental timelines shareable with families
- AI-assisted observation narratives generated from brief teacher notes

### Parent Communication & Family Engagement

- Real-time daily activity updates (meals, naps, diaper changes, activities) via dedicated parent mobile app
- Secure messaging between teachers and families with photo and video sharing
- Multilingual parent communication with automatic translation
- Two-way family contributions to learning portfolios
- Broadcast announcements, newsletters, and event sharing

### Billing, Payments & Subsidy Management

- Automated recurring invoicing with flexible billing cycles (weekly, bi-weekly, monthly)
- ACH and credit card payment processing with parent-facing payment portal
- Late-fee automation and tuition agreement management
- Subsidy billing support with state-specific rules engines for CCDF, TANF, and pre-K voucher programs
- Co-pay calculation, authorization tracking, and subsidy reconciliation

### Enrollment & Attendance

- Digital enrollment applications with waitlist management, document collection, and e-signature
- Digital check-in/check-out with contactless options (QR code, PIN) and authorized-pickup verification
- Real-time classroom roster management with CACFP meal tracking and reporting
- Attendance analytics and reporting for licensing compliance

### Staff & Compliance Management

- Staff scheduling, time-clock, and credential/certification tracking (CPR, first aid, state training hours)
- Real-time staff-to-child ratio monitoring with compliance alerts
- Incident reports, illness logs, medication administration records, and allergy alerts
- State-specific licensing documentation management and audit-ready reporting
- Multi-site management with consolidated billing, cross-location reporting, and centralized policy tools

---

## AI-Native Advantage

Current platforms are beginning to add AI features -- TeachKloud's Enhance tool rewrites observations, Playground's Camber assistant qualifies leads and drafts messages, Illumine generates observation narratives -- but these are bolt-on features within closed ecosystems. An AI-native platform can embed intelligence across the entire workflow: generating developmental observation narratives from brief teacher bullet notes or photo/video analysis, producing lesson plans based on observed child interests and framework requirements, flagging billing anomalies and attendance discrepancies before they become revenue losses, and predicting ratio violations using attendance forecasting. True vision-AI capability -- automatically tagging observed learning domains from classroom images -- is absent from every current platform and represents a meaningful differentiation opportunity.

---

## Tech Stack & Deployment

- **Deployment modes:** Self-hosted, cloud, or hybrid -- critical for Head Start programs and school districts with data residency requirements
- **Mobile-first with offline support:** Offline-first architecture with background sync for ECE environments with unreliable Wi-Fi; battery-efficient operation for all-day tablet use in classrooms
- **Open REST API with webhooks:** Third-party integration capability that no major incumbent (except Illumine) currently offers
- **Standards alignment:** COPPA and FERPA compliance for child data privacy; HL7/FHIR integration pathway for pediatric health records (immunization, medication); CEDS data elements for state reporting
- **Configurable framework engine:** Pluggable developmental framework library supporting US state standards, EYFS (UK), Te Whariki (NZ), EYLF (AU), and accreditation-specific requirements (NAEYC, QRIS)

---

## Market Context

The US childcare management software market was valued at approximately $1.4 billion in 2024 and is growing at a CAGR of 10-12%, serving roughly 11 million children under age five across over 100,000 licensed childcare centers. Federal and state policy tailwinds -- increased CCDF subsidy funding and expanded state pre-K access -- are bringing more children into licensed programs and increasing administrative complexity. Target customers range from independent single-site centers to national childcare chains, Head Start programs, and school district pre-K programs.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
