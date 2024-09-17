![[Web LLM Attacks.webp]]
# Recon

LLM attacks rely on prompt injection - attacker uses crafted prompts to manipulate an LLM's output. It can result in the AI taking actions that fall outside of its intended purpose, such as making incorrect calls to sensitive APIs or returning content that is not part of its guidelines.

A recommended methodology for detecting LLM vulnerabilities:

1. Identify LLM inputs, including both direct (prompt) and indirect (training data) inputs
2. Work out what data and APIs the LLM has access to
3. Probe the new attack surface for vulnerabilities

