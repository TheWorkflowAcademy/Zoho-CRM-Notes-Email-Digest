# Zoho-CRM-Notes-Email-Digest
This repository explains how to create an email digest for newly created Notes on Account records in Zoho CRM. Every two hours, an email will be sent to the Account Owner with information about the Notes that have been recently Updated / Created on their Accounts. This can be generalized to any module within CRM, but we will use Accounts as the example.

This funciton can be broken down into three main parts:
1. Getting All Note Records
2. Creating a Map of Notes
3. Sending Emails to Account Owners

### Scheduled Function
You will need to configure a scheduled function which runs every two hours and executes a custom function within Zoho CRM. The caveat is that we do not want to send emails to Account Owners outside of business hours, so while this function will run every two hours, it will not be sending emails at night.

### Connections
You will need one connection configured in Zoho CRM which has the following scope:
- Zoho CRM: `scope=ZohoCRM.modules.ALL`

### CRM Variable
You will need to create a CRM variable called `Last Digest Email Time`. This will be used to track when the last digest email was sent. Since we will not send emails outside of business hours, we will use this CRM variable for knowing how many Notes to query. For example, when we send the first email of the day after 8 AM, we want the function to query all Notes to the `Last Digest Email Time` which will be ~6 PM of the previous day. This variable will need to be updated every time this fucntion is run. Here is documenation: https://www.zoho.com/crm/developer/docs/crmVariables.html



## 1. Getting All Note Records
Before we get the Note records, we need to get the `Last Digest Email Time` and update it to the current time. Here are links to documentation for the endpoints you will need:
- GET CRM Variable: https://www.zoho.com/crm/developer/docs/api/v2/get-variables.html
- PUT CRM Variable: https://www.zoho.com/crm/developer/docs/api/v2/update-variables.html


We will request all the Notes *updated* since the `Last Digest Email Time`. This will include all Notes which were created or modified in that time frame. The endpoint to use is the general [Notes endpoint](zoho.com/crm/developer/docs/api/v2/get-notes.html) in the Zoho CRM API. To retrieve the most recently updated Notes, use the following query parameters`sort_by=Modified_Time` and `sort_order=desc`.

Here is a sample request:
```
notes = invokeurl
[
	url :"https://www.zohoapis.com/crm/v2/Notes?sort_by=Modified_Time&sort_order=desc&page=" + page + "&per_page=" + per_page
	type :GET
	connection:"zohocrm"
];
```

In the common case where there are more than 200 updated Notes since the `Last Digest Email Time`, the request will need to be wrapped in an [API Pagination Block](https://github.com/TheWorkflowAcademy/api-pagination-zohocrm) to perform multiple requests of Notes. This essentially simulates a `while` loop in Deluge. The condtion will be whether the `Modified_Time` is within the time since `Last Digest Email Time`.

When using the above pagination block, you will be adding Notes queried fromt the API to a list called `all_notes`. Before adding the queried Notes to this list, it is a good idea to filter for the Accounts module before adding. This can be done with the following:
`if(note.get($se_module == "Accounts")`

## 2. Creating a Map of Notes
Now, in the `all_notes` List we have all the Notes on Account records that have been Created/Modfied since the `Last Digest Email Time`, we now need to structure the data so we can easily send out emails to Account Owners. We are now going to take this list of Notes and turn it into a Map of Notes keyed by the Account Owner's email. Unfortunately the Account Owner is not necessarily the Note Owner, so we will need to query the Account information for each Note.

Create a map called `notes_map`.

For each Note in `all_notes` do the following:
- Access the Account ID for the note with `note.get("Parent_Id").get("id")`. Use the Zoho CRM API to query the Account information from the Account ID.
- Get the Account Owner email with `account.get("Owner").get("email")`. 
- Check if the `notes_map` contains the `account_owner_email`. 
  - If true, do `notes_list = notes_map.get(account_owner_email);`, then append the current note to notes_list, then do the following: `account_owner_email, notes_list);`
  - If false, create a list `notes_list` with only the current note in it, then do the following: `account_owner_email, notes_list);`


You will then have a map of lists of notes keyed by the Account Owner's email address.

## 3. Email Digest to Account Owners
To send emails out to the Account Owners, you will iterate over the keys of the `notes_map` by doing `for each key in notes_map.keys()`. You will then perform a `sendmail` Deluge action in each of these iteration blocks.

The email message content should include a list of Created/Modified Notes and a link to the Account. You will need to use HTML to construct the content. For example, the content should look somewhat similar to:
```
<br>
<a href="LINK TO ACCOUNT">Note Title 1</a>
<p>Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.</p>
<br>
```

You can certainly be creative with how this is laid out, but it needs to include each Note's Title, Content, and link to CRM Account.

Here is the documentation for the Send Mail action in Deluge: https://www.zoho.com/deluge/help/misc-statements/send-mail.html
