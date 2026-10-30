# Claims Rules MCP

Claims Rules MCP is a TypeScript server that gives AI agents access to medical billing rules and supporting policy evidence. It combines a small CMS rules database with semantic search over policy PDFs and exposes the results through Model Context Protocol tools.

[Devpost project](https://devpost.com/software/claims-ai-agent) · [Watch the agent demo](https://www.youtube.com/watch?v=X0_MR93zi7k) · [Companion claims agent](https://github.com/evantofu/Claims-Coding-Agent)

## How to use

This repository provides the tool service behind the Claims AI Agent demonstration. It is an MCP endpoint rather than a standalone chat interface.

### 1. Connect an MCP client

Start the TypeScript service and connect your MCP-compatible client to `http://localhost:5000/mcp` using Streamable HTTP. The Python claims agent uses this address through its `CLAIMS_MCP_URL` setting.

### 2. Request billing requirements

Call `GetPayerRequirements` with a CPT code and payer name. For example, the tool accepts an object with this shape:

```json
{
  "cptCode": "99396",
  "payerName": "Medicare"
}
```

This is a tool input, not a complete HTTP request. `patientId` is optional and can be used with authorized FHIR context for age-based checks.

### 3. Inspect rules and citations

The tool returns text sections describing eligibility, modifiers, denial triggers, notes, and any retrieved policy evidence. The agent uses this information when preparing a coding summary. An unsupported code or missing policy evidence is not a successful coverage determination.

## Prerequisites

- Node.js 20 or later, npm, and Git
- A running Chroma server for policy-document search
- Python and the Chroma CLI if running Chroma locally as shown below
- Authorized test FHIR access for patient-specific tools

The code uses a local embedding model. Its first use downloads model files; a Gemini API key is not required by this server itself.

## Installation

### 1. Clone and install the TypeScript service

```bash
git clone https://github.com/evantofu/Claims-Rules-MCP.git
cd Claims-Rules-MCP/typescript
npm install
```

This repository does not currently include a TypeScript lockfile, so use `npm install` rather than `npm ci`.

### 2. Start the MCP server

From `Claims-Rules-MCP/typescript`:

```bash
PORT=5000 npm start
```

Expected terminal message:

```text
MCP server listening on port 5000
```

Check that the process is reachable:

```bash
curl http://localhost:5000/hello-world
```

The response should be `Hello World`. Tool calls use `/mcp`. If you omit `PORT`, `index.ts` defaults to 5001; keep the companion agent's `CLAIMS_MCP_URL` consistent with your choice.

### 3. Start Chroma for document search

In another terminal, from `Claims-Rules-MCP/typescript`:

```bash
python -m venv .chroma-venv
source .chroma-venv/bin/activate
python -m pip install chromadb
chroma run --path ./vectordb
```

The ingestion and query code expect Chroma at `localhost:8000`. See the [Chroma client/server guide](https://github.com/chroma-core/docs/blob/main/docs/usage-guide.md#running-chroma-in-clientserver-mode) for server setup.

### 4. Index policy documents

Place policy PDFs in `typescript/pdfs/`. With Chroma running, open another terminal in `Claims-Rules-MCP/typescript` and run:

```bash
npx tsx ingest.ts
```

The ingestion script extracts text, creates chunks with citation metadata, embeds them locally, and upserts them into the `payer_policies` collection. Ingestion and querying must use the same embedding model. Review document dates and payer relevance before treating retrieved text as applicable evidence.

## Available tools

| Tool | Purpose |
|---|---|
| `GetPayerRequirements` | Look up supported billing rules and retrieve policy evidence |
| `GetPatientAge` | Read a patient's age from FHIR context |
| `FindPatientId` | Find a patient by first and last name through FHIR |

FHIR tools read the `x-fhir-server-url`, `x-fhir-access-token`, and optional `x-patient-id` headers supplied by the caller. The Python agent's FHIR metadata and these MCP headers are separate interfaces; verify forwarding when wiring the services together.

## Project structure

| Path | Purpose |
|---|---|
| `typescript/index.ts` | Express/MCP entry point |
| `typescript/tools/` | Tool registration and billing-rule responses |
| `typescript/rules/cms_rules.json` | Bundled baseline rules for 19 codes |
| `typescript/chunker.ts` | Policy text chunks and citation formatting |
| `typescript/ingest.ts` | PDF ingestion and embedding pipeline |
| `typescript/pdfs/` | Policy source documents |
| `dotnet/` | Retained community-server implementation with patient tools |

The claims-specific payer tool is implemented in TypeScript. This project extends the Prompt Opinion community MCP server; the .NET directory is not an equivalent implementation of every claims feature.

## Troubleshooting

- **The agent cannot connect:** check the server's port against `CLAIMS_MCP_URL`.
- **No policy evidence is returned:** start Chroma, ingest the PDFs, and check the `payer_policies` collection. Baseline rules can still be returned without retrieved evidence.
- **Patient age is unavailable:** verify the FHIR headers and permissions. Age checks can be skipped when patient context is absent.
- **A code is unsupported:** inspect `typescript/rules/cms_rules.json`; the bundled database covers a limited set.
- **Docker port forwarding fails:** the checked-in Compose file maps container port 5000 while the TypeScript source defaults to 5001. Set `PORT=5000` in that service before using the supplied mapping. The local Node setup above sets it explicitly.

## Development notes

The `npm test` script is a placeholder and is not an automated test suite. The companion repository contains the claims evaluation workflow. This is a research prototype with limited rule and document coverage, not a comprehensive or authoritative payer-policy service.

For TypeScript-specific instructions, see [typescript/README.md](typescript/README.md). The project also follows the [SHARP-on-MCP extension](https://www.sharponmcp.com/) used by Prompt Opinion.
