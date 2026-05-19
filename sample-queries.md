# AAP MCP Server — Sample Queries

## Job Execution

- List all job templates available in AAP
- Launch the job template named 'Deploy App' against the production inventory
- Show me the status and stdout of the last 5 failed jobs
- Cancel the currently running job with ID 42
- Relaunch the last failed job for the 'Patch Servers' template

## Inventory & Hosts

- List all inventories and how many hosts each has
- Show me all hosts in the 'webservers' group
- What groups are in the 'Production' inventory?
- Show the variable data for host 'web01.example.com'
- Trigger an inventory source sync for the 'AWS Dynamic Inventory'

## Monitoring & Status

- What's the current AAP platform status?
- Show me the instance groups and their capacity
- List any recent activity stream events
- Show me the mesh visualizer data
- Retrieve the current AAP metrics

## Users & RBAC

- List all users and their roles
- Create a new team called 'Platform Ops' in the Default organization
- What role assignments does user 'john' have?
- Add user 'jane' to the 'Platform Ops' team
- List all organizations and their members

## Security & Compliance

- List all credentials and their types
- Create a new Machine credential named 'prod-ssh-key'
- List all authenticators configured in AAP
- Show me all role definitions available
- List all credential types and their input schemas

## Platform Configuration

- List all execution environments
- Show me the current AAP gateway settings
- List all notification templates
- Show the current controller settings
- List all configured execution environments and their images
