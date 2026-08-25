# Plugin Developer Issues: Host-Side Limitations Encountered

> This document collects problems encountered while developing plugins against the
> current host SDK. Each item is a host-side limitation that a plugin cannot fully
> work around from its own process. They are reported here so the plugin layer can
> inform host improvements. Where an existing PR already addresses the issue, it is
> linked.

## 1. Images are not reliably visible in the chat window (most impactful)

**Symptom.** Image `parts` pushed via `push_message()` behave inconsistently:

- With `ai_behavior="blind"`, the host drops the image entirely — the user never
  sees it in the chat window.
- The `url` form of an image part is warn-dropped by `main_server`; only the inline
  `base64` form reaches the model vision path, and only for `respond`/`read`.

**Workarounds plugins are forced to use.** To show an image to the user, a plugin
must either:

- write the image into its own `static/` directory and push a `markdown` link
  (`![alt](http://...)`) that the frontend renders as an image bubble; or
- keep image bytes in plugin state and have the plugin's own output pipeline push
  them (which the host then has no official channel for).

Both approaches make plugins re-implement resource management, URL encoding and
file lifecycle that the host should provide.

**Related PRs.** [#2835](https://github.com/Project-N-E-K-O/N.E.K.O/pull/2835)
(plugin image upload + controlled delivery) and
[#2905](https://github.com/Project-N-E-K-O/N.E.K.O/pull/2905)
(proactive media rendering) both address this. Merging either would let plugins
push canonical image parts instead of the workaround above.

**Requested host capability.** A first-class image channel for plugins that
supports "show in chat" and "feed to model vision" independently, with the
`visibility` / `ai_behavior` split documented and reliable.

## 2. Double replies: plugin text + LLM paraphrase of the same summary

**Symptom.** When a plugin returns a `summary` from `@plugin_entry`, the host
always feeds that summary to the main LLM, which produces a natural-language
reply. If the plugin itself pushed the same text to the chat window, the user
sees the same content twice — once verbatim, once paraphrased by the LLM.

**Why plugins cannot fully fix it.** The host has no way to say "render this to
the user but do NOT feed it to the LLM". The plugin-side mitigation is a
convention: "a game returns data only and never pushes user-visible text itself",
so the summary is the only user-visible copy. But the summary still enters the
LLM context, so a plugin cannot completely suppress the paraphrase.

**Requested host capability.** A flag such as `suppress_llm_reply` (or treating
`ai_behavior="blind"` as "do not inject summary into the LLM context") would let
a plugin show content without triggering an LLM turn that repeats it.

## 3. Game / long-running sessions get derailed by host proactive messages

**Symptom.** While a plugin is in a multi-turn session (e.g. a game awaiting user
input), a proactive message from the host or another plugin enters the LLM
context. The LLM may conclude the session is over; a subsequent user command
such as "continue" is treated as ordinary chat instead of being routed back to
the plugin's tool.

**Workarounds plugins are forced to use.** Re-inject a "session state anchor"
(read-only text telling the LLM the session is still active) every ~20 seconds,
which is fragile and does not cover the case where the host inserts a message
immediately before the user's next command.

**Requested host capability.** Either (a) attach a "session is active" hint to
proactive messages when a plugin has declared an active session, or (b) expose a
host-level session API plugins can use to mark activity and receive routing
priority.

## 4. LLM tool routing is not reliable

**Symptom.** Whether a plugin's LLM tool (e.g. `play_game`) gets called depends
entirely on the LLM's judgment. A user saying "restart life" (a game name) may
cause the LLM to verbally confirm ("do you want to play?") instead of invoking
the tool, so the game never starts.

**Workarounds plugins are forced to use.** Strengthen the tool description and
expand keyword lists, which helps but cannot force routing.

**Requested host capability.** Support plugins declaring "high-priority routing
keywords": when user input matches a declared keyword, the host routes it to the
plugin tool deterministically instead of leaving it to the LLM's discretion.

## 5. Plugins have no host message listener

**Symptom.** There is no event equivalent of `@message` that lets a plugin react
to host/chat messages in real time. Host-side hooks such as `on_owner_speak`
exist in documentation but are never invoked by the host, so plugins fall back to
polling via `tick`.

**Workarounds plugins are forced to use.** A background tick loop that polls every
second, which is wasteful and adds latency.

**Requested host capability.** Expose host message events to plugins (or actually
invoke the documented `on_owner_speak` hook) so plugins can react without polling.

---

## Summary

| # | Issue | Host capability requested | Related PR |
|---|-------|--------------------------|------------|
| 1 | Images not reliably visible in chat | First-class plugin image channel | #2835, #2905 |
| 2 | Double replies (text + LLM paraphrase) | `suppress_llm_reply` / blind skips summary | — |
| 3 | Sessions derailed by proactive messages | Session-aware proactive hints / session API | — |
| 4 | LLM tool routing unreliable | High-priority routing keywords | — |
| 5 | No host message listener | Host message events / invoke `on_owner_speak` | — |
