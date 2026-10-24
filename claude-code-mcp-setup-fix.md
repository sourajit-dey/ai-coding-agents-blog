---
title: "Claude Code MCP Setup: Add a Server, Check It, and Fix Failed to Connect"
description: "Add an MCP server to Claude Code with claude mcp add or .mcp.json, check it with claude mcp list, and fix the failed to connect errors that give you no detail."
date: 2026-10-24
slug: claude-code-mcp-setup-fix
---

# Claude Code MCP Setup: Add a Server, Check It, and Fix Failed to Connect

You run `claude mcp list`, the server sits there marked failed, and Claude Code tells you nothing else. That blank error is the normal experience, not a sign you broke something exotic. Most failures come from five ordinary causes: the config file lives in the wrong place, the scope is not what you think, the separator is missing, a pasted token carries hidden whitespace, or you rejected the server once and never reset that choice.

This guide covers the working setup first, then each failure in the order to check it. Commands come from the official Claude Code MCP docs, checked 24 October 2026. Tool versions change, so run `claude --version` before you trust a flag that looks new.

## What MCP does in Claude Code

MCP gives Claude Code a live connection to something outside your repo. A docs server returns current library docs. A GitHub server reads issues and pull requests. A filesystem server reads files you point it at. A skill is different. A skill is a markdown file that tells Claude Code how to do a repeatable job well. MCP supplies the connection, a skill supplies the method. You need both for some jobs, and only one for others.

Start with one server. Add a second only after the first shows as connected. Two new servers at once turn one clear error into a guessing game.

## Step 1: Pick a scope before you type anything

Claude Code reads servers from three scopes. Pick the wrong one and the server will work in one project and vanish in the next, which feels like a bug.

- Local scope: this project, this machine, stored in `.mcp.json` at the project root.
- Project scope: shared with the team through `.mcp.json` in the repo. Claude Code asks for approval the first time it sees a new project server.
- User scope: every project on your machine.

For a first test, use local scope. It keeps a bad config inside one project. Move it to project scope only after it connects.

## Step 2: Add a stdio server

A stdio server runs as a local command. This is the common case for a filesystem, GitHub, or docs server that ships as an npm package.

```bash
claude mcp add my-docs -- npx -y @example/docs-mcp-server
```

Read that command in parts. `claude mcp add` registers the server. `my-docs` is the name you will see in `claude mcp list`. Everything after `--` passes to the server untouched. That separator matters. Without it, Claude Code tries to read `-y` as one of its own options, and the add fails in a confusing way.

If the server needs an environment variable, pass it before the separator:

```bash
claude mcp add my-github --env GITHUB_TOKEN=YOUR_TOKEN_HERE -- npx -y @example/github-mcp-server
```

Replace `YOUR_TOKEN_HERE` with a token you create for this server alone, with the smallest permissions that let it do its job. Do not paste a real token into a blog post, a screenshot, or a shared config file. A token in a repo stays in that repo's history after you delete the line.

## Step 3: Add an HTTP server

A remote server takes a URL instead of a command:

```bash
claude mcp add --transport http my-remote https://example.com/mcp
```

If the server needs a header, add it with `--header`. If it uses OAuth, Claude Code starts that flow when you first use the server. When you set the `Authorization` header yourself, a 401 or 403 shows as a failed connection, and Claude Code does not fall back to OAuth. That is a real trap. Pick one auth method. If you set the header by hand, you own the token refresh too.

## Step 4: Write .mcp.json by hand when you want the config in the repo

The CLI writes the same file you can write yourself. A project `.mcp.json` looks like this:

```json
{
  "mcpServers": {
    "my-docs": {
      "command": "npx",
      "args": ["-y", "@example/docs-mcp-server"]
    },
    "my-remote": {
      "type": "http",
      "url": "https://example.com/mcp"
    }
  }
}
```

Save it at the project root, then start a new Claude Code session in that project. Claude Code reads `.mcp.json` at session start, not mid-session. An old session keeps the old list.

Two paths do not work, even though they look right: `~/.claude/.mcp.json` and `~/.claude/mcp.json`. Claude Code does not read either one. If your server appears nowhere, check the path first. It takes ten seconds and saves a long debug session.

When a project server is new, Claude Code asks you to approve it in an interactive session. Approve it there. A server that waits for approval shows as pending, not failed. Run `claude` interactively, approve, then check again.

## Step 5: Check the server

Run these two commands, in this order:

```bash
claude mcp list
claude mcp get my-docs
```

`claude mcp list` shows every registered server and its status. `claude mcp get my-docs` shows the detail for one server, including the status Claude Code recorded for it. A failed status means Claude Code could not connect. It does not mean the list command failed.

Inside an interactive session, `/mcp` opens the same list with a menu. Pick a failed server and choose Reconnect. That forces a fresh attempt without restarting the whole session. For every failed server at once, `/mcp reconnect all` does the same job, on recent versions.

Now ask Claude Code to use the server for a small, checkable task. Ask it to fetch one doc page, or list one repo file. A server that connects but returns wrong data is a different problem from a server that never connects, and you want to know which one you have.

## Fix the five common failures

### 1. The server appears nowhere

Check the file path and the session. You saved `.mcp.json` in the wrong folder, or you never restarted the session after saving it. Move the file to the project root, start a new session, run `claude mcp list` again.

### 2. The entry loads, but one server is skipped

A malformed entry gets skipped while the others still load. Claude Code names the offending field in the list output. Fix that field only. Do not rewrite the whole file. A missing comma, a wrong key name, or a `url` field left empty will do this. An empty `url` on a remote server shows as not configured, and Claude Code does not even try to connect.

### 3. The command never starts

Run the server command yourself, exactly as Claude Code runs it, in the same folder. If `npx -y @example/docs-mcp-server` fails in your terminal, it fails for Claude Code too. Fix the install, the Node version, or the package name first. This check separates a Claude Code problem from a server problem in under a minute.

### 4. Auth fails after you pasted a token

Claude Code flags config values with hidden leading or trailing whitespace. A token copied from a web page often carries a trailing newline. Open the config, delete the space before and after the token, save, restart the session. If you set the `Authorization` header by hand and you get a 401 or 403, check the token and the header name before you assume OAuth will rescue you. It will not, in that setup.

For a variable that comes from your shell, check it is actually set. If a referenced variable is unset and has no default, the file still loads, Claude Code warns about the missing variable, and it uses the literal `${VARIABLE_NAME}` text as the value. That text will not connect to anything.

### 5. You rejected it once, and now it never asks again

Project approvals stick. If you rejected a server when Claude Code first prompted you, it will not prompt again on its own. Reset project choices:

```bash
claude mcp reset-project-choices
```

Start a new interactive session, approve the server when prompted, and check the list again.

## When the status just says Connection error

This is the case the docs warn about directly: Claude Code adds no detail to this status. Go straight to the two checks above, the command check and, for an HTTP server, a curl check against the same URL. If the HTTP server returns 404, Claude Code shows "MCP endpoint not found" for that origin. That tells you the path is wrong, not the token. If the server is up and still times out after you raise the limit, update Claude Code. Some older versions timed out on slow HTTP servers in a way newer versions fixed.

Work through the list in order, one change at a time. Change the path, check. Change the token, check. Change the scope, check. Two changes at once hide which one fixed it, and the same error will return next time you set up a server.

## MCP config generator

Use this widget before you type a command. Pick the transport, fill in the name and the command or URL, and it builds the config and the CLI command for you. It runs in your browser only. Nothing leaves the page.

<div id="mcp-generator" style="border:1px solid #ccc;padding:16px;max-width:640px;font-family:system-ui,sans-serif">
  <label>Server name<br><input id="mcp-name" value="my-docs" style="width:100%;padding:8px"></label><br><br>
  <label>Transport<br>
    <select id="mcp-transport" style="width:100%;padding:8px">
      <option value="stdio">stdio (local command)</option>
      <option value="http">HTTP (remote URL)</option>
    </select>
  </label><br><br>
  <div id="mcp-stdio-fields">
    <label>Command<br><input id="mcp-command" value="npx" style="width:100%;padding:8px"></label><br><br>
    <label>Arguments (space separated)<br><input id="mcp-args" value="-y @example/docs-mcp-server" style="width:100%;padding:8px"></label><br><br>
  </div>
  <div id="mcp-http-fields" style="display:none">
    <label>Server URL<br><input id="mcp-url" value="https://example.com/mcp" style="width:100%;padding:8px"></label><br><br>
  </div>
  <label>Scope<br>
    <select id="mcp-scope" style="width:100%;padding:8px">
      <option value="local">local (this project)</option>
      <option value="project">project (shared in repo)</option>
      <option value="user">user (all projects)</option>
    </select>
  </label><br><br>
  <button id="mcp-build" style="padding:10px 16px;cursor:pointer">Build my config</button>
  <button id="mcp-copy" style="padding:10px 16px;cursor:pointer">Copy output</button>
  <pre id="mcp-output" style="white-space:pre-wrap;background:#f4f4f4;padding:12px;margin-top:12px"></pre>
  <p id="mcp-note" style="font-size:14px">Tip: start a new Claude Code session after you save .mcp.json. An old session keeps the old server list.</p>
</div>
<script>
(function(){
  var t=document.getElementById('mcp-transport');
  var s=document.getElementById('mcp-stdio-fields');
  var h=document.getElementById('mcp-http-fields');
  function toggle(){ var isHttp=t.value==='http'; s.style.display=isHttp?'none':'block'; h.style.display=isHttp?'block':'none'; }
  t.addEventListener('change',toggle); toggle();
  document.getElementById('mcp-build').addEventListener('click',function(){
    var name=(document.getElementById('mcp-name').value||'my-docs').trim();
    var scope=document.getElementById('mcp-scope').value;
    var out='';
    if(t.value==='http'){
      var url=(document.getElementById('mcp-url').value||'').trim();
      var cfg={mcpServers:{}}; cfg.mcpServers[name]={type:'http',url:url};
      out='// .mcp.json (project root)\n'+JSON.stringify(cfg,null,2)+'\n\n// CLI command\nclaude mcp add --transport http --scope '+scope+' '+name+' '+url;
    } else {
      var cmd=(document.getElementById('mcp-command').value||'npx').trim();
      var args=(document.getElementById('mcp-args').value||'').trim().split(/\s+/).filter(Boolean);
      var cfg={mcpServers:{}}; cfg.mcpServers[name]={command:cmd,args:args};
      out='// .mcp.json (project root)\n'+JSON.stringify(cfg,null,2)+'\n\n// CLI command (note the -- separator)\nclaude mcp add --scope '+scope+' '+name+' -- '+cmd+(args.length?' '+args.join(' '):'');
    }
    document.getElementById('mcp-output').textContent=out;
  });
  document.getElementById('mcp-copy').addEventListener('click',function(){
    var txt=document.getElementById('mcp-output').textContent;
    if(!txt){document.getElementById('mcp-note').textContent='Build the config first, then copy.';return;}
    if(navigator.clipboard){navigator.clipboard.writeText(txt).then(function(){document.getElementById('mcp-note').textContent='Copied. Paste it into .mcp.json or run the CLI command.';});}
    else{document.getElementById('mcp-note').textContent='Copy failed in this browser. Select the output text by hand.';}
  });
  document.getElementById('mcp-build').click();
})();
</script>

## FAQ

### Why does Claude Code say my server failed but give no reason?
A bare connection error carries no detail in Claude Code. Run the server command yourself for a stdio server, or curl the URL for an HTTP server. Those two checks name the cause in most cases.

### Where does .mcp.json go?
At the project root for local and project scope. Claude Code reads it at session start. It does not read `~/.claude/.mcp.json` or `~/.claude/mcp.json`.

### Do I need a skill as well as an MCP server?
Only when the job repeats and needs a method in addition to a connection. A docs server alone answers a doc question. A skill tells Claude Code which docs to check first, what to ignore, and what file to write the answer into.

### What should I check first for a 401?
The token itself, hidden whitespace around it, and whether you set the header by hand. A hand-set header means Claude Code will not fall back to OAuth.

### Can I share .mcp.json with my team?
Yes, for project scope, but keep tokens out of it. Use environment variables for secrets, and check that each variable is set on the machine that runs Claude Code.

## Sources and method
- Claude Code MCP docs: https://code.claude.com/docs/en/mcp
- Claude Code MCP quickstart: https://code.claude.com/docs/en/mcp-quickstart
- Claude Code MCP setup, scopes and fixes: https://automater.ai/intel/claude-code-mcp-servers-setup-scopes/
- OpenClaw docs were checked for the cluster, not for this article's commands: https://docs.openclaw.ai/

Gap against the top results: the official docs explain each scope and error separately, and one third-party guide explains scopes well. None of them puts the five failures in one check order, with a config generator that matches the exact transport and scope you picked. That check order plus the generator is what this article adds.

## Next in this cluster
- Spoke 8: /mcp-server-failed-to-connect/ (the cross-tool diagnostic version of this fix)
- Spoke 7: /skill-md-template-claude-code/ (turn the working server into a repeatable skill)
- Pillar: /ai-coding-agents-mcp-skills-guide/

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Claude Code MCP Setup: Add a Server, Check It, and Fix Failed to Connect",
  "description": "Add an MCP server to Claude Code with claude mcp add or .mcp.json, check it with claude mcp list, and fix the failed to connect errors that give you no detail.",
  "datePublished": "2026-10-24",
  "author": {"@type": "Organization", "name": "Blog SEO Hub (draft)"}
}
</script>
