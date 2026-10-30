# Claims Rules MCP — TypeScript Service

This directory contains the TypeScript MCP server used by [Claims AI Agent](https://github.com/evantofu/Claims-Coding-Agent). It exposes billing-rule and FHIR tools and retrieves supporting passages from policy PDFs.

[Project overview and demo](../README.md) · [Devpost](https://devpost.com/software/claims-ai-agent)

## How to use

### 1. Start the service

From this directory:

```bash
npm install
PORT=5000 npm start
```

The process exposes `POST http://localhost:5000/mcp` and a basic `GET /hello-world` check. Without an explicit `PORT`, the source defaults to 5001.

### 2. Connect your client

Use a Streamable HTTP MCP client. For the Python claims agent, set:

```dotenv
CLAIMS_MCP_URL=http://localhost:5000/mcp
```

Discover the tools and call `GetPayerRequirements` with `cptCode`, `payerName`, and optionally `patientId`. The patient tools also need caller-supplied FHIR headers.

### 3. Add policy evidence

Start Chroma at `localhost:8000`, place source PDFs in `pdfs/`, and run:

```bash
npx tsx ingest.ts
```

This populates the `payer_policies` collection with embeddings and citation metadata. The first embedding call may download the local model. Missing or outdated policy documents limit the returned evidence.

## Development

- `index.ts` starts the server and registers the tools exported by `tools/index.ts`.
- `tools/PayerRequirementsTool.ts` combines `rules/cms_rules.json` with Chroma results.
- `chunker.ts` preserves source metadata for citations.
- The `npm test` script is currently a placeholder.

For VS Code debugging, open `index.ts`, select the checked-in **tsx** launch configuration, and start debugging. Ensure the port in your caller matches the process's port.

See the [root README](../README.md) for prerequisites, Chroma installation, FHIR header names, the repository structure, and troubleshooting.
