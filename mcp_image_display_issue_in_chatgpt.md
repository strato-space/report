# MCP Image Display Issue in ChatGPT

## Issue Overview

Throughout 2025, developers building agents with the OpenAI Agents SDK and MCP (Model Context Protocol) encountered a frustrating limitation: **images returned by MCP tools were not displayed inline in ChatGPT**. Despite the MCP specification supporting image content blocks (base64-encoded images with MIME types), ChatGPT's interface failed to render them properly.

The issue manifested in several ways:
- **Empty or "no result" responses** — ChatGPT reported the tool returned nothing
- **Massive token consumption (~100k tokens)** — base64 data treated as text and fed to the model
- **Raw object strings** — e.g., `"<mcp.server.fastmcp.utilities.types.Image object at 0x...>"`
- **Generic errors** — "I wasn't able to display the image"

This was particularly frustrating because other MCP-compatible clients (Claude Desktop, Cursor IDE) displayed the same images correctly, confirming the issue was specific to ChatGPT's implementation.

---

## Reports of the Issue

### May 2025: OpenAI Developer Forum (Agents SDK)

A developer reported that their MCP server returned an `ImageContent` block, but ChatGPT treated it as text. The assistant consumed approximately **100,000 tokens** attempting to process the raw base64 string, then failed to display anything[\[1\]](https://community.openai.com/t/image-response-from-an-mcp-server-with-agents-sdk/1269441). The user noted:

> "The problem is that the assistant is treating this as text... instead it is interpreting this as a very long string."

No solution was provided at the time, highlighting that the Agents SDK integration was not handling image outputs.

### September 2025: OpenAI Forum Bug Report

Another user opened a bug thread titled "Image responses from MCP tools do not work." They provided the exact JSON payload their MCP server returned:

```json
{
  "content": [{
    "type": "image",
    "data": "<base64>...",
    "mimeType": "image/png"
  }]
}
```

This matched the [MCP specification](https://modelcontextprotocol.io/specification/2025-11-25/server/tools#image-content) exactly. Yet ChatGPT kept replying with **"empty result"** or claimed it couldn't see the image[\[2\]](https://community.openai.com/t/image-responses-from-mcp-tools-do-not-work/1360430). The image was only 18KB — well under any reasonable size limit.

### October 2025: Reddit r/mcp Discussion

A Reddit thread asked: "Are image responses in MCP tools supported by ChatGPT/Claude?" The poster showed their MCP JSON response with a properly formatted image content block. ChatGPT responded that it "cannot see the image"[\[3\]](https://www.reddit.com/r/mcp/comments/1ntd441/are_image_responses_in_mcp_tools_supported_by/).

Notably, another user confirmed that **Cursor IDE and Claude Desktop displayed the same images correctly**[\[4\]](https://www.reddit.com/r/mcp/comments/1ntd441/are_image_responses_in_mcp_tools_supported_by/), proving the issue was ChatGPT-specific.

Claude also had a separate issue: it threw a 400 error claiming "base64 data does not match image/png" — a strict MIME validation problem on Anthropic's side.

### October 2025: FastMCP GitHub Issue

A report on the FastMCP project described that returning `Image` objects from tools produced the **string representation of the Python object** instead of an inline image:

```
<mcp.server.fastmcp.utilities.types.Image object at 0x7f...>
```

This was caused by a **developer pitfall**: nesting media objects inside dictionaries or other containers. FastMCP only converts `Image`/`Audio`/`File` objects to MCP content blocks when returned at the top level or in a simple list[\[5\]](https://github.com/jlowin/fastmcp/issues/2115)[\[6\]](https://github.com/jlowin/fastmcp/pull/2118).

### October 2025: Cursor GitHub Issue

Users reported that images from MCP servers were "not getting rendered in the chat even though the MCP server is providing the image as output"[\[7\]](https://github.com/cursor/cursor/issues/3365). This was later fixed in Cursor, but highlighted the broader ecosystem challenge.

### Additional Reports

- **Claude Desktop** temporarily hid MCP results but restored image display in a subsequent update[\[8\]](https://www.reddit.com/r/mcp/comments/1laxgx5/claude_desktop_wont_show_mcp_image_response/)
- **ChatGPT metadata issues** — some users reported ChatGPT stopped reading MCP tool metadata entirely[\[9\]](https://community.openai.com/t/chat-gpt-app-not-reading-the-mcp-tools-metadata-anymore/1362802)

---

## Cause and Technical Explanation

### ChatGPT's Internal Handling

The root cause was ChatGPT's incomplete implementation of MCP image content blocks. The [MCP specification](https://modelcontextprotocol.io/specification/2025-11-25/server/tools#tool-result) defines how to return images as part of tool results:

```json
{
  "type": "image",
  "data": "base64-encoded-data",
  "mimeType": "image/png",
  "annotations": {
    "audience": ["user"],
    "priority": 0.9
  }
}
```

Per the [ImageContent schema](https://modelcontextprotocol.io/specification/2025-11-25/basic/index#imagecontent), the required fields are:
- `type`: Must be `"image"`
- `data`: Base64-encoded image data
- `mimeType`: MIME type of the image (e.g., `image/png`, `image/jpeg`)

The expected behavior:
1. Tool returns image content block
2. UI renders image inline
3. Model receives metadata (not raw data)

What actually happened in ChatGPT:
- The image data was either **ignored entirely** (empty result)
- Or **passed to the model as text** (100k token explosion)
- The UI never attempted to render the image

OpenAI's Agents SDK internally used a `FunctionCallOutputPayload` structure that didn't properly serialize image content blocks. The image data was lost or misrouted before reaching the UI.

### FastMCP Serialization Pitfall

A separate but related issue affected FastMCP users. The framework only converts helper classes (`Image`, `Audio`, `File`) to MCP content blocks when returned **directly or in a top-level list/tuple**[\[6\]](https://github.com/jlowin/fastmcp/pull/2118):

```python
# ✅ Works — returns at top level
return Image(data=base64_data, mime_type="image/png")

# ✅ Works — returns in a list
return [Image(...), "Caption text"]

# ❌ Fails — nested in dict
return {"image": Image(...), "caption": "..."}
```

When nested in a dictionary, the object is not converted and appears as `<Image object at 0x...>`. This was **intentional behavior** — the maintainers clarified it's a design limitation and updated documentation to warn developers[\[6\]](https://github.com/jlowin/fastmcp/pull/2118).

### MIME Validation Issues

Claude's 400 error ("base64 data does not match image/png") indicated strict MIME validation. ChatGPT didn't even throw such errors — it simply ignored the content, suggesting it **never attempted to decode the base64 at all**.

---

## Workarounds and User Solutions

### 1. Return Images in Supported Formats

Ensure you're using the correct types to trigger MCP content blocks:

- **OpenAI Agents SDK (Python)**: Use `ToolOutputImage` type[\[10\]](https://openai.github.io/openai-agents-python/tools/)
- **FastMCP (Python)**: Use `fastmcp.utilities.types.Image` and return directly or in a list
- **MCP SDK (Node.js/TypeScript)**: Use the `ImageContent` type from [`@modelcontextprotocol/sdk`](https://www.npmjs.com/package/@modelcontextprotocol/sdk):

```typescript
import type { ImageContent, CallToolResult } from "@modelcontextprotocol/sdk/types.js";

// Return image content block
const result: CallToolResult = {
  content: [
    {
      type: "image",
      data: base64EncodedData,
      mimeType: "image/png"
    } as ImageContent
  ]
};
```

The SDK also provides `TextContent`, `ResourceLink`, and other content types for tool results.

This doesn't fix the ChatGPT UI, but ensures other clients can render the image and prepares for future fixes.

### 2. Avoid Nesting Media in Structs

**Do not wrap images in dictionaries or complex JSON.** If your tool returns mixed data (text + image), split into separate returns or use MCP's ability to carry both structured data and content blocks separately.

### 3. Manual Handling via API

If using the Responses API directly (not ChatGPT UI), parse the response JSON for image blocks:

```python
for item in response["content"]:
    if item["type"] == "image":
        image_data = base64.b64decode(item["data"])
        # Save or display the image
```

This bypasses ChatGPT's UI entirely and lets you build custom front-ends.

### 4. Use External Image Links

Have the tool upload the image to cloud storage and return a URL instead. Less elegant, but gets the image to the user. The assistant can output the link or use browsing capabilities to preview it.

### 5. Leverage Alternative Clients

For development and testing, use MCP-compatible clients that display images correctly:
- **Cursor IDE** — confirmed working[\[4\]](https://www.reddit.com/r/mcp/comments/1ntd441/are_image_responses_in_mcp_tools_supported_by/)
- **Claude Desktop** — fixed in later updates[\[8\]](https://www.reddit.com/r/mcp/comments/1laxgx5/claude_desktop_wont_show_mcp_image_response/)
- **Cline 3.13+** — added MCP image support[\[11\]](https://www.reddit.com/r/CLine/comments/1k35qlh/cline_313_toggleable_clinerules_new_task_slash/)

### 6. Build Custom UI Components

For production applications, implement your own image rendering. Intercept the tool response, detect image content blocks, and display them in your UI rather than relying on ChatGPT's interface.

---

## Resolution Status

### OpenAI's Fix (October 2025)

OpenAI addressed the issue in a late-October 2025 update. A pull request titled **"[MCP] Render MCP tool call result images to the model"** was merged into the Codex repository[\[12\]](https://github.com/openai/codex/pull/5600).

The PR description noted:
> "Previously, the image content was lost on the way to the model/UI... implements a fix so that image outputs are serialized in the proper array format and reach the model (and UI) as images."

The fix modified how `FunctionCallOutputPayload` handles image content, ensuring it's properly serialized and routed to both the model and UI.

### FastMCP Documentation Update (October 2025)

FastMCP closed the loop by updating their documentation to clarify the container limitation[\[6\]](https://github.com/jlowin/fastmcp/pull/2118). They marked the behavior as intentional — developers must return images at the top level.

### Broader Multimodal Support

OpenAI announced broader multimodal support around the same time:
- **Realtime API** (August 2025) added support for remote MCP servers and image inputs[\[13\]](https://openai.com/index/introducing-gpt-realtime/)
- **ChatGPT Release Notes** (November 2025) mentioned "More images in answers"[\[14\]](https://help.openai.com/en/articles/6825453-chatgpt-release-notes)

### Current Status (November 2025)

The expectation is that **ChatGPT's developer mode and Agents SDK should now support inline image responses from MCP tools**, provided you're using the updated SDK/models. However:

- Some users still report mixed results with custom GPTs
- The fix may be rolling out gradually
- Reliable inline images from user-defined tools are not universally confirmed

### Recommended Test Client: media-gen-mcp

For debugging MCP image rendering issues, we recommend using [`media-gen-mcp`](https://github.com/strato-space/media-gen-mcp) — an MCP image generation/editing server with built-in debug options.

**Key feature:** The `MEDIA_GEN_MCP_RESULT_PLACEMENT` environment variable controls where `urls`, `files`, and `revisedPrompts` appear in tool results. Data is placed in **exactly one location** — no duplication:

| Value | Description | Recommended for |
|-------|-------------|-----------------|
| `content` | As `resource_link` and `text` items in `content` array (default) | Anthropic clients |
| `structured` | Only in `structuredContent` object | Testing |
| `toplevel` | At top-level of payload | **ChatGPT** |

**Tool signatures:**

```typescript
// create-image
{
  prompt: string,           // required, max 32000 chars
  quality?: "auto" | "high" | "medium" | "low",  // default: "low"
  size?: "1024x1024" | "1536x1024" | "1024x1536" | "auto",  // default: "1024x1024"
  output?: "base64" | "file",  // default: "file"
  file?: string,            // absolute path if provided
  n?: number,               // 1-10
  // ... other OpenAI gpt-image-1 params
}

// edit-image
{
  image: string | string[], // path, base64, or URL (1-16 images)
  prompt: string,           // required
  mask?: string,            // optional mask
  quality?: "auto" | "high" | "medium" | "low",  // default: "low"
  size?: "1024x1024" | "1536x1024" | "1024x1536" | "auto",  // default: "1024x1024"
  output?: "base64" | "file",  // default: "file"
  file?: string,            // absolute path if provided
  n?: number,               // 1-10
}
```

**Quick start:**

```sh
MEDIA_GEN_MCP_RESULT_PLACEMENT=toplevel npx -y github:strato-space/media-gen-mcp --env-file /path/to/media-gen.env
```

This allows systematic testing of different resource link placements to determine what works best with each MCP client.

**Recommendation**: Test with the latest SDK version and follow best practices for returning images. If issues persist, use workarounds while monitoring for updates.

---

## Conclusion

The MCP image inline display issue highlighted the challenges of integrating rich content into ChatGPT's agent responses. Throughout 2025, developers experimented with having ChatGPT agents return charts, plots, or other images, only to be frustrated by blank or garbled outputs.

The problem stemmed not from the MCP standard (which defines how to do this) but from ChatGPT's incomplete adoption of that standard in its UI and response handling. Images would either be dropped or mishandled, limiting use-cases like data visualization or image-based analysis.

From a technical perspective, the fix was straightforward — the ChatGPT client needed to properly recognize image content blocks and render them inline (as it does with DALL·E or Code Interpreter). OpenAI's PR #5600 addressed this, though full rollout may still be in progress.

**For developers building agents that return images**:
1. Follow best practices (correct types, not nested)
2. Test with alternative clients to verify your server works
3. Implement workarounds for production use
4. Monitor OpenAI updates for full support confirmation

The issue is on the path to resolution — identified, documented, with fixes merged. Until the day we see the release note "ChatGPT now displays images from custom tools inline," creative workarounds remain necessary.

---

## Sources

1. [Image response from an MCP server with agents sdk](https://community.openai.com/t/image-response-from-an-mcp-server-with-agents-sdk/1269441) — OpenAI Developer Forum (May 2025)

2. [Image responses from MCP tools do not work](https://community.openai.com/t/image-responses-from-mcp-tools-do-not-work/1360430) — OpenAI Developer Forum Bug Report (Sept 2025)

3. [Are image responses in MCP tools supported by ChatGPT/Claude?](https://www.reddit.com/r/mcp/comments/1ntd441/are_image_responses_in_mcp_tools_supported_by/) — Reddit r/mcp (Oct 2025)

4. [Cursor/Claude confirmation](https://www.reddit.com/r/mcp/comments/1ntd441/are_image_responses_in_mcp_tools_supported_by/) — Reddit comment confirming other clients work

5. [Image objects not serializing properly in MCP tool responses](https://github.com/jlowin/fastmcp/issues/2115) — FastMCP GitHub Issue

6. [Document container limitations for Image/Audio/File objects](https://github.com/jlowin/fastmcp/pull/2118) — FastMCP PR #2118

7. [Image is not getting rendered in the chat](https://github.com/cursor/cursor/issues/3365) — Cursor GitHub Issue #3365

8. [Claude Desktop won't show MCP (image) response](https://www.reddit.com/r/mcp/comments/1laxgx5/claude_desktop_wont_show_mcp_image_response/) — Reddit r/mcp

9. [Chat GPT App - not reading the mcp tool's metadata anymore](https://community.openai.com/t/chat-gpt-app-not-reading-the-mcp-tools-metadata-anymore/1362802) — OpenAI Forum

10. [Returning images or files from function tools](https://openai.github.io/openai-agents-python/tools/) — OpenAI Agents SDK Docs

11. [Cline 3.13: MCP image support](https://www.reddit.com/r/CLine/comments/1k35qlh/cline_313_toggleable_clinerules_new_task_slash/) — Reddit r/CLine

12. [[MCP] Render MCP tool call result images to the model](https://github.com/openai/codex/pull/5600) — OpenAI Codex PR #5600

13. [Introducing gpt-realtime and Realtime API updates](https://openai.com/index/introducing-gpt-realtime/) — OpenAI Blog (Aug 2025)

14. [ChatGPT Release Notes](https://help.openai.com/en/articles/6825453-chatgpt-release-notes) — OpenAI Help Center

15. [MCP Specification: Tools — Image Content](https://modelcontextprotocol.io/specification/2025-11-25/server/tools#image-content) — Model Context Protocol (2025-11-25)

18. [MCP Specification: ImageContent Schema](https://modelcontextprotocol.io/specification/2025-11-25/basic/index#imagecontent) — Model Context Protocol Schema Reference

16. [Image Responses from MCP tool calls](https://github.com/openai/codex/issues/4819) — OpenAI Codex Issue #4819

17. [FastMCP Tools Documentation](https://fastmcp.mintlify.app/servers/tools) — FastMCP Docs
