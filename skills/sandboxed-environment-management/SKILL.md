---
name: "sandboxed-environment-management"
description: "Provision hermetic container and process sandboxes protecting host machines."
---

# Sandboxed Environment Management Skill

Safely isolates untrusted agent code execution.

## Core Capabilities
- Supports Docker, Podman, Apptainer, and Bubblewrap backends.
- Mounts repositories with restricted root filesystem permissions.
- Cleans up and reaps container instances upon task completion.
