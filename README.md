# Serverless Ticket-to-Invoice Automation Engine

Converts approved field service tickets into invoices via a B2B supply chain API, triggered by an incoming PDF through an automated email pipeline.


## Tech Stack

- **Runtime:** Python 3.12 on AWS Lambda
- **Infrastructure:** AWS SAM (template.yaml + samconfig.toml)
- **Networking:** VPC, private subnets, NAT Gateway, Elastic IP
- **Trigger:** API Gateway REST (HTTPS POST)
- **Storage:** S3
- **Secrets:** SSM Parameter Store (encrypted)
- **Libraries:** pdfplumber, boto3, requests
- **External API:** B2B Supply Chain DOCP API v1
- **Auth:** Mutual TLS (client certificate + private key)

&nbsp;

## Background

The platform handles automated invoice processing for energy sector suppliers. One core service is **ticket flipping**: when a supplier submits a field service ticket through a central supply chain portal and the buyer approves it, the system converts it into an invoice and submits it on the supplier's behalf.

For most clients, this process is fully automated. A scheduled Lambda polls the portal, picks up approved tickets, and submits invoices without manual intervention.

Certain enterprise clients require a tailored workflow. Instead of relying on a structured data feed, their process originates from an email containing a PDF invoice. The standard auto-flip pipeline was not built for unparsed document feeds, requiring a dedicated event-driven module.

This Lambda functions as a specialized processing module in that pipeline. Upstream steps—receiving the email and storing the PDF in S3—are handled by a separate ingestion pipeline.

&nbsp;

## What This Lambda Does

When triggered, the Lambda receives a POST request from the upstream pipeline containing the S3 location of an invoice PDF and an invocation ID.

It downloads the PDF from S3 and extracts the text from the first page using `pdfplumber`. Using regular expressions, it parses the invoice number required to query the supply chain portal.

It then retrieves a client TLS certificate and private key from SSM Parameter Store to authenticate against the supply chain REST API. The first API call searches for approved receipts matching the client's DUNS number and extracted invoice number. The second call fetches the full receipt details from the returned payload.

Once receipt data is compiled, it dispatches the payload asynchronously to a downstream processing Lambda along with the original invocation ID and returns an HTTP 200 response to the caller.

&nbsp;

## Technical Challenges

**Fixed Outbound IP Configuration**

The target supply chain API restricts access to pre-approved IP addresses. AWS Lambda does not provide a static outbound IP by default. To resolve this, the Lambda is deployed inside a private VPC subnet routed through a NAT Gateway with an Elastic IP whitelisted by the external vendor.

**Mutual TLS in Python**

The external API authenticates requests at the TLS handshake level using a client certificate and private key. While some environments allow passing PEM strings directly in memory, Python's `requests` library requires file system paths. The Lambda securely pulls the certificate and key from SSM Parameter Store, writes them to ephemeral storage (`/tmp`), executes the HTTPS request, and deletes the temporary files inside a `finally` cleanup block.
