---
layout: post
title: Delete all refresh tokens
date: '2023-02-26 20:12:09'
---

with Azure function please . .

Refresh tokens. . . &nbsp;not so many know that Microsoft Azure Active Directory IDP based on Oauth v.2 protocol and using refresh tokens as "bearer". We had Active Directory with NTLM/Kerberos and now it is tokens =)

This area is grey and you will not get details of Microsoft implementation, but what you can get - Microsoft put responsibility on you! -\>

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://learn.microsoft.com/en-us/azure/active-directory/develop/refresh-tokens"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">Microsoft identity platform refresh tokens - Microsoft Entra</div>
</div>
</a></figure>

Here is important parts:

![Refresh Tokens](../images/02/image.png){: width="100%" .mx-auto.d-block :}

"Securely delete the old refresh token after acquiring new one" . . jepp never heard that somebody did it =)

Every time when you access some Azure application (can be Teams or One Drive or whatever) - you are getting new "refresh token" (90 days default lifetime) and access token (ca 1,2 hour lifetime). EVERY TIME! It means that there is hundreds or thousands tokens out there making you life better and all your apps open automatically. It is good! But not good if it is your admin user. then things became really unpleasant.

How to delete all of them? Well, it is not so easy. Per today (26.02.2023) you can do this only through Graph API or PowerShell . . If you read document carefully - Microsoft point to right cmdlet:

![Revocation table](../images/02/image-1.png){: .mx-auto.d-block :}

You see? Only PowerShell officially can revoke them . . but here we have problem! This PowerShell module is old AzureAD which will be retired in summer 2023. I will help you to use new one from Microsoft.Graph module. Let's start . .

We will create Azure Function with Powershell which will be killing every night ALL refresh tokens for all privileged roles, it will use managed identity to authenticate against AAD.

1. Create Azure Function(Windows/PowerShell core):

![Create Azure Function](../images/02/image-2.png){: .mx-auto.d-block :}

With time trigger it will run every day 0:10 &nbsp;. .

2. Create System assigned Managed Identity:

![Create System assigned Managed Identity](../images/02/image-3.png){: .mx-auto.d-block :}

&nbsp;3. Add Application API permission to the MI:

Connect to MS Graph with proper right to adjust roles

<!--kg-card-begin: markdown-->

`Connect-MgGraph -Scopes Directory.ReadWrite.All,AppRoleAssignment.ReadWrite.All`

<!--kg-card-end: markdown-->

Create variables with GUIDs from your tenant

<!--kg-card-begin: markdown-->

    $params = @{ 
        PrincipalId = "GUID of your MI"
        ResourceId = "GUID of MS Graph Enterpise app in your tenant"
        AppRoleId = "GUID of APP rolle"
    }

<!--kg-card-end: markdown-->

Last part can be copied from this link:

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://learn.microsoft.com/en-us/graph/permissions-reference"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">Microsoft Graph permissions reference - Microsoft Graph</div>
</div>
</a></figure>

I will use Directory.ReadWrite.All "19dbc75e-c2e2-444c-a770-ec69d8559fc7" to delete tokens and RoleManagement.Read.All "c7fbd983-d9aa-4fa7-84b8-17382c103bc4" to read assignments to privileged roles.

If somebody knows which another role I can use to delete refresh tokens for privileged users with less permissions - will be glad to hear!

And actually put those permission to MI:

<!--kg-card-begin: markdown-->

`New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId YOURMIGUID -BodyParameter $params`

<!--kg-card-end: markdown-->

![MI Permissions](../images/02/image-4.png){: .mx-auto.d-block :}

4. Add PowerShell Modules to file requrements.psd1

![Powershell modules requirements](../images/02/image-5.png){: .mx-auto.d-block :}

I added those lines to import proper modules. You can see that I am using "preview". Because at this moment Microsoft support managed identity ONLY in v2 version of PowerShell MS.Graph.

<!--kg-card-begin: markdown-->

    'Microsoft.Graph.Users' = '2.0.0-preview5'
    'Microsoft.Graph.Groups' = '2.0.0-preview5'
    'Microsoft.Graph.Beta.Users.Actions' = '2.0.0-preview5'
    'Microsoft.Graph.Authentication' = '2.0.0-preview5'
    'Microsoft.Graph.Identity.Governance' = '2.0.0-preview5'

<!--kg-card-end: markdown-->

And actually put code:



![Azure Function code](../images/02/image-6.png){: .mx-auto.d-block :}

<!--kg-card-begin: markdown-->

    # Input bindings are passed in via param block.
    param($Timer)
    
    # Get the current universal time in the default string format.
    $currentUTCtime = (Get-Date).ToUniversalTime()
    
    #Import-Modules needed
    Import-Module Microsoft.Graph.Users
    Import-Module Microsoft.Graph.Groups
    Import-Module Microsoft.Graph.Beta.Users.Actions
    Import-Module Microsoft.Graph.Authentication
    Import-Module Microsoft.Graph.Identity.Governance
    
    # Authenticate with the Microsoft Graph API using the function's managed identity
    Connect-MgGraph -Identity
    Get-MgContext | Select-Object -ExpandProperty Scopes
    
    # Get assigned role assignments
    Write-Host -ForegroundColor Yellow "Fetching assigned role assignments, might take a minute..."
    $assignedRoleAssignments = Get-MgRoleManagementDirectoryRoleAssignmentScheduleInstance -ExpandProperty "*" -All:$true$activatedRoleAssignments = $assignedRoleAssignments | Where-Object { $_.AssignmentType -eq 'Activated' }
    $filteredAssignedRoleAssignments = $assignedRoleAssignments | Where-Object { $_.AssignmentType -eq 'Assigned' }
    Write-Host -ForegroundColor Yellow "Found $($filteredAssignedRoleAssignments.count) assigned role assignments"
    
    # Get eligible role assignments
    Write-Host -ForegroundColor Yellow "Fetching eligible role assignments, might take a minute..."
    $eligibleRoleAssignments = Get-MgRoleManagementDirectoryRoleEligibilitySchedule -ExpandProperty "*" -All:$true
    Write-Host -ForegroundColor Yellow "Found $($eligibleRoleAssignments.count) eligible PIM role assignments, whereof $($activatedRoleAssignments.count) are activated"
    
    # Combine assignments
    $allRoleAssignments = @(
    	$eligibleRoleAssignments.PrincipalID #| Select-Object -First 10
    	$filteredAssignedRoleAssignments.PrincipalID #| Select-Object -First 10
    )
    
    #Loop through each PrincipalID in the array and filter out the ones that are not user-related
    $userPrincipalIDs = @()
    foreach ($id in $allRoleAssignments) {
    	$user = Get-MGUser -UserId $id -ErrorAction SilentlyContinue
    	if ($user) {
    		$userPrincipalIDs += $id
    		}
    	}
    foreach ($id in $allRoleAssignments) {
    	$group = Get-MgGroupMember -GroupId $id -ErrorAction SilentlyContinue
    		foreach ($number in $group) {
    			$userPrincipalIDs += $group.id
    			}
    		}
    #Deduplication - get unique user ids
    $userPrincipalIDs = $userPrincipalIDs | Select-Object -Unique    
    
    # Loop through each admin user    
    foreach ($user in $userPrincipalIDs)    { 
           # Remove the user's refresh tokens
           Invoke-MgBetaInvalidateUserRefreshToken -UserId $user    
    }

<!--kg-card-end: markdown-->

Can answer some questions! It is mandatory to use "Beta" word when you are running command for beta graph. You have to do this to make this script work.

Part with getting principalsIDs was borrowed from:

<figure class="kg-card kg-bookmark-card"><a class="kg-bookmark-container" href="https://learningbydoing.cloud/blog/building-a-comprehensive-report-on-azure-ad-admin-role-assignments/"><div class="kg-bookmark-content">
<div class="kg-bookmark-title">Building a comprehensive report on Azure AD admin role assignments in Powershell</div>
<div class="kg-bookmark-metadata">
<span class="kg-bookmark-author">LearningByDoing.cloud</span><span class="kg-bookmark-publisher">Stian A. Strysse</span>
</div>
</div>
</a></figure>

Thanks for code! =)

I want to say thanks to Mark Grote who went through the hell to make it work! =)

