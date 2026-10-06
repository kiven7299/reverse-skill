# Fallback strategy

When the current path stalls, fall back in order:

1. From breakpoints back to request observation
2. From source guesses back to runtime evidence
3. From Node env-patch back to page forensics
4. From deep deobfuscation back to the minimal reproduce chain
