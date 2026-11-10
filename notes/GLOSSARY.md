Harness: It is kuber resource and describe the environment is executed in.
AgentTemplate: It is keber resource and decribe which model it used, system prompt, skills and tools the model use.
Agent: A kuber resouce that make a relation between AgentTemplate and Harness.
Session: kagent creates a Session: one conversation, pinned to one revision. Each invoke sends one task into it.
Actor: There is one Actor per session. Between tasks it is checkpointed and stopped, and the next invoke resume it.
WorkerPool:A WorkerPool is a small set of worker pods (mine has one: kagent-default). Each pod runs many actor inside it as gVisor sandboxes.
golden snapshot: Every new revision gets its own golden snapshot (the minute of waiting after a template change). New Sessions’ Actors start from it instead of booting from scratch.
ModelConfig: It is a kuber crd that we define the model and how to connect it and which secret model use
revision: It is the version of the agent that a session uses. A change to the template, model or harness creates a new revision; existing Sessions stay on the old one.