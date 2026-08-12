cat > CLAUDE.md << 'EOF'
# Standing instructions

The architecture spec is in jarvis-project-plan.md. Read it before any work.

1. Implement only the phase named in the prompt. Stop at its acceptance criteria.
2. No provider name in agent logic. Everything through the registry in §3.
3. Verify the livekit-agents API against live docs before generating. Do not
   trust training data for the Agent/AgentSession/@function_tool shape.
4. Every external call: explicit timeout + spoken fallback. Never fail silently.
5. Secrets from server env only. Never sent to or accepted from clients.
6. Validate all client input server-side.
7. State plainly which acceptance criteria you verified and which you did not.

## Environment
- Python via uv, Python 3.12
- Never commit .env
EOF
