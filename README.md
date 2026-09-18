# Guest Journey & Operations Automation

An n8n-based automation system for serviced accommodation operations. The project automates guest communication, reservation-state tracking, and daily operational reporting using Google Sheets, Gmail, JavaScript, and n8n.

> **Project status:** Completed MVP  
> **Project type:** Portfolio Automation Project  
> **Industry:** Hotels / Serviced Accommodation / Short-Term Rentals

---

## Overview

Serviced accommodation teams often spend time manually checking reservations, identifying upcoming arrivals and recent departures, sending guest communications, and preparing daily operational reports.

This project demonstrates how those repetitive tasks can be converted into a structured, repeatable automation.

The MVP uses:

- **n8n** for workflow orchestration
- **Google Sheets** as the reservation data source and system of record
- **Gmail** for guest and operational communications
- **JavaScript** for date calculations, business rules, and report generation
- **Docker** for the local/self-hosted n8n environment

The workflows were developed and tested using **fictional reservation data**. The architecture is designed so the Google Sheets data source could later be replaced or supplemented by a PMS, reservation API, webhook, or other approved integration.

---

## What the System Automates

The MVP automates two operational processes:

### Guest Journey Automation

- Validates reservation eligibility
- Excludes cancelled/non-confirmed reservations
- Excludes reservations without a guest email address
- Identifies reservations approaching check-in
- Sends personalized pre-arrival welcome emails
- Identifies reservations that have recently checked out
- Sends personalized post-stay review requests
- Records communication status in the reservation database
- Prevents the same communication from being repeatedly sent on subsequent executions

### Daily Operations Reporting

- Retrieves reservation records
- Identifies confirmed arrivals for the current date
- Identifies confirmed departures for the current date
- Aggregates operational information
- Generates an HTML-formatted daily report
- Delivers the report through Gmail

---

## Architecture

The project consists of two primary n8n workflows.

### Workflow A — Guest Journey Automation

```text
Schedule Trigger
       ↓
Get Reservations
       ↓
Filter Reservations
       ↓
Evaluate Guest Journey
       ↓
Switch by Action
     /       \
    /         \
Welcome       Review
 Email         Email
   ↓             ↓
Update         Update
Database       Database
```

### Workflow B — Daily Operations Report

```text
Schedule Trigger
       ↓
Get Reservations
       ↓
Evaluate Today's Operations
       ↓
Generate Operations Report
       ↓
Gmail
       ↓
Operations Recipient
```

---

## Workflow 1: Send Welcome and Review Emails

### Purpose

The guest journey workflow periodically inspects reservation records and determines whether a guest communication has become due.

The workflow uses reservation dates and communication status fields to make the decision.

### Processing Flow

1. A scheduled trigger starts the workflow.
2. Reservation records are retrieved from the Google Sheets `Reservations` sheet.
3. The reservation eligibility filter requires:
   - `Reservation Status = Confirmed`
   - `Guest Email` is not empty
4. The workflow calculates:
   - `daysUntilCheckIn`
   - `daysSinceCheckOut`
5. A JavaScript business-logic step determines the required action.
6. A Switch node routes the reservation to the appropriate communication path.
7. Gmail sends the selected message.
8. The reservation record is updated to reflect the completed communication.

### Welcome Email Logic

A reservation becomes eligible for a welcome email when:

```text
daysUntilCheckIn >= 0
AND
daysUntilCheckIn <= 2
AND
Welcome Email Sent = No
```

The resulting action is:

```text
Send Welcome Email
```

The email is personalized using reservation data, including:

- Guest first name
- Property
- Check-in date
- Check-out date

After successful processing:

```text
Welcome Email Sent = Yes
Last Processed At = current timestamp
```

This state change prevents the reservation from being treated as an unsent welcome communication during later workflow executions.

### Review Email Logic

A reservation becomes eligible for a review request when:

```text
daysSinceCheckOut >= 1
AND
Review Email Sent = No
```

The resulting action is:

```text
Send Review Email
```

The email is personalized using:

- Guest first name
- Property

After successful processing:

```text
Review Email Sent = Yes
Last Processed At = current timestamp
```

---

## Idempotency and Duplicate Prevention

Because the guest journey workflow runs on a recurring schedule, duplicate communication prevention is an important part of the design.

The reservation database stores communication state using:

- `Welcome Email Sent`
- `Review Email Sent`

Before sending a communication, the workflow checks that the relevant status is still:

```text
No
```

After successful processing, it changes the value to:

```text
Yes
```

This allows the workflow to safely inspect the reservation database repeatedly without repeatedly sending the same guest communication.

---

## Workflow 2: Daily Operations Report

### Purpose

The daily operations workflow converts reservation data into a concise operational report showing confirmed arrivals and departures for the current date.

### Processing Flow

1. A scheduled trigger starts the workflow.
2. Reservation records are retrieved from Google Sheets.
3. A JavaScript processing step evaluates the current date against reservation check-in and check-out dates.
4. Confirmed arrivals and departures are grouped separately.
5. An HTML report is generated.
6. Gmail delivers the report to the configured operations recipient.

### Arrival Criteria

A reservation is included in today's arrivals when:

```text
Reservation Status = Confirmed
AND
Check-in Date = Today
```

The report includes:

- Reservation ID
- Guest
- Property
- Check-out date

### Departure Criteria

A reservation is included in today's departures when:

```text
Reservation Status = Confirmed
AND
Check-out Date = Today
```

The report includes:

- Reservation ID
- Guest
- Property
- Check-in date

### Example Report Structure

```text
Daily Operations Report

Date: YYYY-MM-DD

Today's Arrivals (N)
Reservation | Guest | Property | Check-out

Today's Departures (N)
Reservation | Guest | Property | Check-in

Total arrivals: N
Total departures: N
```

The actual workflow generates this information as an HTML-formatted email.

---

## Reservation Data Model

Google Sheets acts as the reservation database for the MVP.

| Field | Purpose |
|---|---|
| `Reservation ID` | Unique reservation identifier |
| `Guest First Name` | Guest first name |
| `Guest Last Name` | Guest surname |
| `Guest Email` | Guest email address |
| `Property` | Property associated with the reservation |
| `Check-in Date` | Guest arrival date |
| `Check-out Date` | Guest departure date |
| `Reservation Status` | Reservation state |
| `Welcome Email Sent` | Welcome communication status |
| `Review Email Sent` | Review communication status |
| `Source` | Reservation source |
| `Notes` | Operational notes |
| `Last Processed At` | Last automation timestamp |
| `Exception Status` | Exception tracking field |
| `Exception Message` | Description of a detected issue |

---

## Data Validation

The MVP performs basic validation before guest communication.

### Implemented checks

**Reservation status**

Only reservations with:

```text
Reservation Status = Confirmed
```

are allowed into the guest communication path.

**Guest email**

The workflow requires a non-empty:

```text
Guest Email
```

Reservations without an email address are excluded from the Gmail communication branches.

These controls reduce common operational issues such as attempting to contact cancelled reservations or sending messages without a recipient.

---

## Testing

The workflow was tested using fictional reservation records representing different operational scenarios.

| Scenario | Expected Result |
|---|---|
| Confirmed reservation within welcome window | Welcome email |
| Welcome email already sent | No welcome email |
| Guest checked out | Review email |
| Review email already sent | No review email |
| Cancelled reservation | No processing |
| Missing guest email | Excluded from email workflow |
| Future reservation | No action |
| Daily arrival | Included in operations report |
| Daily departure | Included in operations report |

The guest communication workflow was also tested end-to-end using controlled email addresses to verify Gmail delivery.

---

## Error and Failure Considerations

The MVP uses validation and state-based controls to reduce common workflow failures.

The design considers:

- Cancelled reservations
- Missing guest email addresses
- Duplicate execution
- Incorrect reservation status
- Failed downstream communication
- Missed execution windows

The current implementation is intentionally an MVP. Dedicated error workflows, retry policies, centralized execution logging, and operational failure alerts are identified as production enhancements rather than being represented as completed functionality in this project.

---

## Production Considerations

For a production implementation, Google Sheets could be replaced or supplemented by an approved PMS or reservation API.

Potential integration points include:

- Property-management systems
- Reservation APIs
- Webhooks
- Guest communication platforms
- Accounting systems

Additional production controls could include:

- API authentication
- OAuth
- Retry policies
- Centralized error logging
- Failure alerts
- Execution monitoring
- Database persistence
- Explicit UK timezone handling
- Development/production workflow separation
- Credential management
- Version control

The system should only interact with external platforms through approved APIs and supported integration methods.

---

## Future Enhancements

Potential next-stage functionality includes:

1. Lodgify reservation ingestion
2. Enso Connect integration
3. Cleaning task creation
4. Property-level operational reporting
5. Xero integration
6. Owner financial reporting
7. Guest-message response monitoring
8. Automated exception alerts
9. Centralized workflow logging
10. Management dashboards
11. UK timezone-aware scheduling
12. Retry and recovery workflows

These are future enhancements and are **not claimed as completed features of the portfolio MVP**.

---

## Business Value

The project demonstrates how a serviced accommodation operation can reduce repetitive administrative work by automating:

- Guest communication
- Reservation status tracking
- Daily operational reporting
- Date-based workflow decisions
- Duplicate prevention
- Basic data validation

The core design principle is to move the process from:

```text
People manually checking what needs to happen
```

to:

```text
The system continuously evaluating reservation state
and executing the appropriate action
```

---

## Repository Contents

```text
.
├── Daily Operations Report - GITHUB.json
├── Send Welcome and Review Emails - GITHUB.json
└── README.md
```

The workflow JSON files are sanitized for public repository use.

Environment-specific credentials, personal email addresses, Google Sheet identifiers, webhook identifiers, and n8n instance metadata have been removed or replaced with placeholders.

---

## Setup

These workflows are intended as portfolio/reference implementations rather than one-click production deployments.

To adapt them for another environment:

1. Import the workflow JSON into n8n.
2. Configure a Google Sheets credential.
3. Configure a Gmail credential.
4. Replace `YOUR_GOOGLE_SHEET_ID` with the appropriate Google Sheets document ID.
5. Configure the appropriate operations recipient where required.
6. Ensure the Google Sheet contains the expected reservation fields.
7. Review the schedule configuration and timezone before activation.
8. Test with non-production reservation records.
9. Verify email delivery and database status updates.
10. Activate the workflows only after validation.

---

## Important Notes

- This is a **portfolio MVP**, not a production-ready property-management platform.
- Reservation data used during development was fictional.
- The workflows rely on Google Sheets as the reservation system of record.
- The guest journey workflow uses recurring execution and database state to prevent duplicate communication.
- The production design should use explicit timezone handling, especially when operating across regions.
- Credentials should be configured through n8n's credential management system rather than embedded in workflow JSON.
- The sanitized workflow files intentionally contain placeholders where environment-specific configuration is required.

---

## Technology Stack

| Technology | Role |
|---|---|
| **n8n** | Workflow orchestration and automation |
| **Google Sheets** | Reservation database / system of record |
| **Gmail** | Automated guest and operational communications |
| **JavaScript** | Date calculations, business logic, and HTML report generation |
| **Docker** | Local/self-hosted n8n environment |

---

## Project Status

**Completed MVP**

The core guest communication and daily operations reporting workflows have been implemented and tested against fictional reservation scenarios.
