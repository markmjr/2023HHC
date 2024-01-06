---
icon: material/text-box-outline
---

# Azure 101

**Difficulty**: <i class=twemoji_red>:fontawesome-solid-tree::fontawesome-solid-tree:</i>:fontawesome-solid-tree::fontawesome-solid-tree::fontawesome-solid-tree:<br/>
**Direct link**: [Azure 101](https://hhc23-wetty.holidayhackchallenge.com?&challenge=azure101)

## Objective

!!! question "Request"
    Help Sparkle Redberry with some Azure command line skills. Find the elf and the terminal on Christmas Island.

??? quote "Sparkle Redberry"
    Hey, Sparkle Redberry here! So, I've been trying to learn about Azure and the Azure CLI and it's driving me nuts.<br/>
    Alabaster Snowball decided to use Azure to host some of his fancy new IT stuff on Geese Islands, and now us elves have to learn it too.<br/>
    Anyway, I know it's important and everyone says it's not as difficult as it seems, but honestly it still feels like quite a challenge for me.<br/>
    Alabaster sent us this Azure CLI reference as well. It's super handy, he said. Honestly, it just confuses me even more.<br/>
    If you can spare a moment, would you mind giving me a hand with this terminal? I'd be really grateful! Pretty please, with holly leaves on top!

## Hints

??? tip "Azure CLI Reference"
    The Azure CLI tools come with a builtin help system, but Microsoft also provides this [handy cheatsheet](https://learn.microsoft.com/en-us/cli/azure/reference-index?view=azure-cli-latest).


## Solution

We are provided with another opportunity for a walk through of the Azure CLI commands and what they offer.


!!! success "You may not know this but the Azure cli help messages are very easy to access. First, try typing:</br> az help | less"
    az help | less


!!! success "Next, you've already been configured with credentials. Use 'az' and your 'account' to 'show' your current details and make sure to pipe to less ( | less )"
    az account show | less 

!!! success "Excellent! Now get a list of resource groups in Azure.</br> For more information: [https://learn.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest](https://learn.microsoft.com/en-us/cli/azure/group?view=azure-cli-latest)"
    az group list | less

!!! success "Ok, now use one of the resource groups to get a list of function apps. For more information:</br> [https://learn.microsoft.com/en-us/cli/azure/functionapp?view=azure-cli-latest](https://learn.microsoft.com/en-us/cli/azure/functionapp?view=azure-cli-latest)</br> Note: Some of the information returned from this command relates to other cloud assets used by Santa and his elves."
    az functionapp list -g northpole-rg1 | less

!!! success "Find a way to list the only VM in one of the resource groups you have access to.</br> For more information: [https://learn.microsoft.com/en-us/cli/azure/vm?view=azure-cli-latest](https://learn.microsoft.com/en-us/cli/azure/vm?view=azure-cli-latest)"
    az vm list -g northpole-rg2 | less

!!! success "Find a way to invoke a run-command against the only Virtual Machine (VM) so you can RunShellScript and get a directory listing to reveal a file on the Azure VM.</br> For more information: [https://learn.microsoft.com/en-us/cli/azure/vm/run-command?view=azure-cli-latest#az-vm-run-command-invoke](https://learn.microsoft.com/en-us/cli/azure/vm/run-command?view=azure-cli-latest#az-vm-run-command-invoke)"
    az vm run-command invoke -g northpole-rg2 -n NP-VM1 --command-id RunShellScript --scripts "ls"



!!! success "Answer"
    ![Complete](../img/objectives/o4/az101_done.png)

## Response

!!! quote "Sparkle Redberry"
    Wow, you did it!</br>
    It makes quite a bit more sense to me now. Thank you so much!</br>
    That Azure Function App URL you came across in the terminal looked interesting.</br>
    It might be part of that new project Alabaster has been working on with the help of ChatNPT.</br>
    Let me tell you, since he started using ChatNPT he's been introducing a lot of amazing innovation across the islands.</br>
    Knowing Alabaster, he'll be delighted to tell you all about it! I think I last saw him on Pixel island.</br>
    By the way, as part of the Azure documentation he sent the elves, Alabaster also noted that if Azure CLI tools aren't available in an Azure VM we should use the Azure REST API instead.</br>
    I'm not really sure what that means, but I guess I know what I'll be studying up on next.
