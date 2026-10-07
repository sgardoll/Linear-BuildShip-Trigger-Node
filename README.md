# Linear BuildShip Trigger Node
Connect your Linear workspace to this node and trigger BuildShip workflows when issues or other entities change. This node automatically generates and subscribes to a Linear webhook for selected resource types (like Issue, Project, or Comment) and detects status transitions using the statusChanged flag. Use this node to automate processes based on updates in Linear, such as moving issues between workflow states.

## Get the trigger definition

```bash
git clone https://github.com/sgardoll/Linear-BuildShip-Trigger-Node.git
cd Linear-BuildShip-Trigger-Node
```

Open [linear-trigger-buildship.json](linear-trigger-buildship.json). It contains one webhook-trigger definition with its configuration, output schema and lifecycle script, not a complete BuildShip workflow or a published workflow remix link.
