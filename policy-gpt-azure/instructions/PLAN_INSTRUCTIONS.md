# Policy GPT Azure Application Instructions

> Commands issued
```
npx claude-flow swarm "Create implementation plan for application described in APP_REQUIREMENTS.md in folder requirements. Place the implementation plan in folder plans. Do not implement the feature additions yet. Please let me know if you need additional information." --claude
```

> Content of ```APP_REQUIREMENTS.md```
# Requirements
Create an application that allows security administrators to manage Azure cloud policies using natural language.
* Allow users to add, update, or delete policies using natural language.
* Allow users to combine policies into policy sets
* Allow users to identify and edit environments and specify the definitions of those environments in terms of management groups,  subscriptions,  or resource groups. Each Azure resource can be in a maximum of one environment.
* All policy or policy set assignments will be made to environments.
* Allow users to create and maintain assignments
* Allow users to create and maintain policy exemptions by policy set or individual policy. Exemptions can be by environment,  management group,  subscription,  resource group,  or individual resource.
* Allow users to implement all assignments in audit mode so they can be more safely tested
* Allow users to promote policies and assignments from lower level environments to higher level environments
* Allow users to make changes to policies in lower level environments and promote those changes from lower level environments to higher environments.
* Allow users to interact with the application by web browser.
* Allow users to import existing custom policy definitions, sets, assignments,  and exemptions.
* Allow users to include Azure-provided policies in policy sets, assignments,  and exemptions.
* Allow periodic backups of all users create and edit.
* Provide for users with edit ability and users who are read only.