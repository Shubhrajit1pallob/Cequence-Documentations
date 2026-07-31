Personal / tooling todos that support the work. Distinct from `Phase 0.md` (project work).

## Work Specific
- [ ] Learn how the API works for the Ops-portal.(External and Internal) _Ongoing_
- [ ] Research on how to get the Jira task script to work with the argo workflows. 

- [ ] Work on fixing the bug for ops portal. [Ops portal Fix- Duplicate tenant](https://xangent.atlassian.net/browse/PLAT-1661)
- [ ] Learn Argo workflows. (Important) (Ongoing)
- [ ] Fix the prod grouping by Environment so the dropdown closes when I change the window and come back to the page. (It only happens when you interact with something other than the window/pages you are on.) ==In Progress==
- [ ] The credentials needed to authenticate to the collections and to delete tenant resources need work. Fix - Authentication should not need to be done manually; it should be automatic (later, after the actual implementation).
- [x] Work in understanding how the argo workflow is working for other templates.(Slack Notification)
- [x] Modify the collection script for fit for a argo workflow job.
- [x] Work with Confluence to get the design documentation in the Confluence space.
- [x] Learn about git worktrees with claude.
- [x] Complete the ArgoCD Sync jobs research. (Today) (Ongoing).
- [x] Push the ops-portal repo changes to the remote repo. (Today)
- [x] Fill the Work Timesheet till today to calculate the time remaining to work. 
- [x] Create a Jira ticket to delete all the prod orphaned resources. (Done) 
- [x] The script should give you the option to delete one tenant's resources at a time. The input is the CSV from the orphaned resources fetched from the AWS accounts. 
- [x] Complete the understanding of the UAP workflow before working on the PLAT-981 ticket
- [x] Run the Ops-portal locally and test the changes made. (Learn how to do this first.)


## Orphaned Resources Collection/Deletion Script

- [x] Create the functionality to use the SSO for longer use at a time.
- [x] Create a region section in the Orphaned resources deletion script so that the user can see the resources by the region and have a option to delete them.
- [x] Do something about the dependencies. Can we make a feature that basically checks the dependencies of that that resource before it starts deleting them. Instead of directly jumping onto destroying them it should check for dependencies and list them if they cannot be destroyed if they have those.
- [x] Also can we create a list of the resources that shows the dependencies or keep a track of them during the collection phase?

## Other personal setup

 - [ ] eza (terminal coloring.)
 - [ ] Work on the connection issue for connecting to the control plane and the worker node via other connections. (Over the internet)
 - [ ] Reorganize the Vault to have the Orphaned resources documents in a single specific folder. (Ask Claude to do that).
 - [ ] Research on MCP servers for Spotify, and Facebook if it is there.
 - [ ] Research if I can create a custom MCP server for my portfolio website.
 - [ ] Push the MCP project to GitHub. (Today)
 - [ ] Can I change the highlight color in here? (Research this.)
 - [ ] Ask AI to push the checked tasks/TODOs to the bottom of the list when I check them, and to add them again to the bottom of the list when I uncheck them. This has to be automated, or if Obsidian is open source, I can build the feature and contribute to the repo. (Today/Tomorrow/...)
 - [ ] See ghost stories dub ( Have to see )
 - [x] Checkout the CMUX terminal.
 - [x] Create a script that fetches orphaned resources for a specific account and outputs them to a spreadsheet. This should produce an output in a spreadsheet that others can later query. The script should handle this entire process, and let's make this with Pulumi.
 - [x] Set up the MCP server for Claude - for apps like Atlassian, Slack, GitLab.
 - [x] Finish the CICD provisioning step for the Templating.

## Portfolio Website.
- [ ] Work on the personal portfolio project. (Ongoing)
- [ ] Research if I can create a custom MCP server for my portfolio website.

## Plat-981.
- [x] Create a branch from main and cherry pick the commits from my working branch.
- [x] Take a look at the order of the jobs that I did and figure out how to keep it order. Figure out how to keep the pattern same.
- [x] Squash the commits to have one particular commit before I push to the remote.
- [x] Push to the changes to the remote repo. (Blocked)
- [x] Ask for review in the Jira board. Follow up with team members.
- [x] Research the UAP upgrade workflow (Only the parts related to the ticket).
- [x] Make the said changes through the Claude after review.
- [x] Look up how I can run this locally and test the changes. (In progress)

## Plat-1613.
- [ ] 



## Claude Code shell aliases

Goal: faster session management. Suggested names below — adjust to taste.

- [x] `c` → `claude` (base shortcut)
- [x] `cc` → `claude --continue` (resume the most recent conversation in the current dir)
- [x] `cr` → `claude --resume` (pick a past session to resume — interactive picker)
- [x] Add chosen aliases to `~/.zshrc`
- [x] `source ~/.zshrc` and verify each one launches what it should