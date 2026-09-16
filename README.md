# Go-ing Fast with MCP: Rapid Development of MCP Servers for Security Agents

> **Event Date & Time:** Wednesday, September 16, 2026 @ 1:45 pm – 1:50 pm EDT  
> **Speaker:** Matthew Franz (Security & Compliance Lead, Pydantic)  
> **Event:** [DC State of the Stack 2026](https://dcstateofthestack.org/) (Formerly DevOpsDays DC)  
> **Location:** American Red Cross, 1730 E St NW, Washington, DC 20006  
> **Registration:** [Registration Page](https://dcstateofthestack.org/registration)  
> **Session Details:** [Speakers & Sessions](https://dcstateofthestack.org/speakers/)  

---

## Overview

The Model Context Protocol (MCP) is revolutionizing how AI agents interact with security tools, but setting up Python integrations introduces massive dependency bloat and slow startup/installation overhead, even with modern packaging with uv. Enter Golang: compiled, lightweight, type-safe, and lightning-fast.

In this talk, we’ll explore the rapid development of custom security MCP servers in Go.

We will cover:
- **Why Go for MCP:** Why Go is the ultimate language for building lightweight, secure, and low-latency MCP servers.
- **Wrapping Security APIs:** How to quickly wrap security APIs (SIEM, SOAR, EDR) into LLM-friendly tool definitions using Go's strong typing.

## Presentation & Artifacts

- **Slides (PDF):** [`dist/go-ing-fast-with-mcp-final.pdf`](dist/go-ing-fast-with-mcp-final.pdf)
- **Slides (PowerPoint):** [`dist/go-ing-fast-with-mcp.pptx`](dist/go-ing-fast-with-mcp.pptx)
- **Architecture & Patterns Guide:** [`refs/GO-MCP-GUIDE.md`](refs/GO-MCP-GUIDE.md)

---

## Talk Outline

1. **Introduction & Objective:** Delivering fast, lean MCP servers for security agents
2. **Who am I?** Background in security infrastructure and blue-team agents
3. **MCP and its Discontents:** Token burn, agent thrashing, and security boundaries
4. **Long Live MCP:** Complementary roles for MCP, CLI tools, and CodeMode
5. **Why Custom MCP Servers?** Domain-tuned logging, caching, and custom telemetry
6. **Why Go?** Single binary, fast startup, low supply-chain overhead, strong typing
7. **Minimal Go Template:** Decoupled architecture (`cmd/` adapter, `internal/` core)
8. **Planning with `GO-MCP-GUIDE.md`:** Standard layout, stdio transport, iterative design
9. **Instrument & Iterate:** Telemetry-first server development
10. **Test Everything:** Unit tests, JSON-RPC smoke tests, and LLM harness
11. **Climbing the Tool Ladder:** Progressive disclosure (`list` → `search` → `describe` → `query`)
12. **Strongly Typed MCP Servers:** Enforcing schemas and documentation with Go structs
13. **Guard Out:** Bounding queries, filters, and timeouts
14. **Guard In:** Bounding responses, payload caps, truncation annotations, out-of-band export
15. **E2E Testing with Pydantic AI:** Testing tools against real models and catching query hallucination
16. **Session Analysis with Agentsview:** Inspecting agent tool calls and workflows locally
17. **Observability with Logfire:** OpenTelemetry traces for agent turns and MCP operations
18. **Resources & Takeaways:** Starter repositories, reference servers, and next steps

---

## About the Event

[DC State of the Stack 2026](https://dcstateofthestack.org/) (formerly DevOpsDays DC) is a community-driven technology conference rooted in the Washington, DC region.

- **2026 Theme:** *Shipping Software in the Age of AI*
- **Dates:** September 16–17, 2026
- **Location:** American Red Cross, 1730 E St NW, Washington, DC
- **Socials:** [@dcstateofstack](https://x.com/dcstateofstack) | [LinkedIn](https://www.linkedin.com/company/dcstateofthestack)

---

## About the Speaker

### Matthew Franz
*Security & Compliance Lead, Pydantic*

Matt Franz recently joined Pydantic as their first security hire, with responsibility for all things security and compliance. With over 25 years of experience in early-stage startups and mature enterprises, Matt blends the strategic perspective of an executive with the hands-on expertise of a builder. His background includes founding the Helix Cloud Operations team at Mandiant/FireEye, serving as VP of Production Engineering at Cofense, and directing global security operations at Ping Identity.

A former U.S. Army Intelligence Analyst, Matt currently focuses on large-scale data platforms, security analytics, infrastructure automation, and applying generative AI to offensive and defensive use cases. He builds daily in Python and Golang, with a focus on agentic blue team capabilities.

---

## Links & Resources

### Event & Session
- [DC State of the Stack 2026](https://dcstateofthestack.org/)
- [Conference Program](https://dcstateofthestack.org/program)
- [Speakers Page](https://dcstateofthestack.org/speakers/)
- [Registration](https://dcstateofthestack.org/registration)

### Related Talks
- [mcp-threat-hunt-aug-2026](https://github.com/mdfranz/mcp-threat-hunt-aug-2026)

### Go MCP Servers & Projects
- [steampipe-mcp-golang](https://github.com/mdfranz/steampipe-mcp-golang) — MCP server for Steampipe
- [go-velociraptor-mcp](https://github.com/mdfranz/go-velociraptor-mcp) — MCP server for Velociraptor
- [osqueryi-mcp](https://github.com/mdfranz/osqueryi-mcp) — MCP server for osqueryi
- [elastic-security-mcp](https://github.com/mdfranz/elastic-security-mcp) — MCP server for Elastic Security
