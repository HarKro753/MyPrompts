# Learnings

Referencing the opencode repository under Desktop/opencode

- Use Namespacess for the central component of the Domain, which acts as the interface. E.g. we have several tools then the Namespace would be ToolRegistry. Consumers import ToolRegistry and then Call e.g. ToolRegistry.tools
- Define full text prompts in text files such as anthropic.txt, text with params is defined in code system prompt isnt that complex mostly e.g. anthropic.txt
- see system.ts I believe the file tree for git files is injected into the prompt already; aswell as system and model informations
- use xml tags for promting
- for tools we use an interface which all tools follow
- tools are injected into the prompt is injected into the session
- analyse the anthropic.txt prompt to see how to do a good agent prompt
