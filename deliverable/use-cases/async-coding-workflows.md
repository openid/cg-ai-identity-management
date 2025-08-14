## Asynchronous Coding Workflow — Example Use Case

Developers want to prompt a background coding agent to complete an end-to-end development workflow (see Cursor Background Agents as an example). 
To complete this task, the coding AI agent needs to both write code AND access developer tooling.

For example, to create successful preview deployments of web apps, a coding workflow may need to set settings in external auth vendors (e.g., set redirect URLs) or control deployment configurations (e.g., Vercel, Cloudflare, or AWS configurations). 
It may wish to pull from or write to ticketing systems (e.g., Linear) or access external configuration information (e.g., Stripe Configurations). 
In some cases, CLI tools can provide access to these configuration options, but background agents may not have knowledge of the tools (not in pretraining data or context window), may become confused about what CLI tools to use, or may not be pre-authorized to use the tool. 
MCP can allow for access to external configuration tooling, support OAuth at task creation time, and provide structured context on tool use in the agent's context window. The MCP servers are clearly remote protected resources.

Further, the background agent may need access to specific documentation. 
Including this in an MPC server allows the developer to explicitly place the documentation as an accessible tool into the context window, reducing the risk of an agent finding unrelated or out-of-date documentation via web search. 
This resource doesn't explicitly need protection, and it can be run remotely or locally via stdio as a read-only tool or resource.

Finally, the coding agent may want access to local tools that aid in the coding workflow. 
For example, environment permitting, a front-end coding task may want access to a Playwright server via stdio, which allows the agent to orchestrate browser usage dynamically to open localhost pages, screenshot the UI, and pass that visual context back to the coding agents. 
This runs locally and is explicitly not an OAuth resource server.

These are each different types of MCP servers, working together to control and structure the context window of the AI agent (context engineering) and complete the specific background software task as asynchronously and successfully as possible.
