---
title: Instant Mailing List
date: 2025-09-30 19:34
categories: [Development,ASP.NET Core]
tags: [programming, identity, mail, api, c#, development, web, data, asp.net core, .net, mvc, javascript, jquery]
---

## Sometimes You Just Want The Emails

There are currently over a billion ways to send people emails (I assume this is why we get and ignore so many). I've built pretty complex integrations using Twilio's [SendGrid](https://sendgrid.com/), Google Smtp, [Azure Communication Services](https://azure.microsoft.com/en-us/products/communication-services) and - most rewardingly, if not easily - [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/use-the-api).

But what if you don't need that? What if you already have a nice CRM-style web app written in ASP.NET Core (like everyone should have), with all your customers / clients / staff / fellow Nathan Fielder enthusiasts painstakingly stored in great detail, and you just want to copy and paste their names and emails into your email client of choice? It takes ten minutes to code it up and no one has to integrate any auth or mess around with proprietary APIs.

You can do that! This is a super simplified version of how I just did it for someone. 

## The Setup

1. This application managers users. They have different Roles, Statuses (Active/Inactive, etc), names and email addresses.
2. The Index page is already in place. Admins can filter the user list by role and status, and/or search for a string inside the name and email fields.
3. **Twist:** it's not Blazor. Or even Razor Pages. It's good ol' MVC. No built in interactivity here. Strap in for some vanilla javascript.

## The Plan

1. Add a "Copy As Mailing List" button to the Index UI
2. Use the button to trigger an Ajax call to the Controller
3. Get the controller to return a nice clean string of comma separated `<Name> email@address` pairs
4. Display the result in a modal
5. Add a 'Click To Copy' button which copies the data to the user's clipboard.

## Steps

### 1. UI Button

Classic, basic, bootstrappy.
This button is placed inside a `<form>` element which contains the Staff Member filter input elements: role, name and so on.
```html
<button type="button" id="exportMailingListBtn" class="btn btn-brand-yellow mt-3 ms-2"><i class="bi bi-envelope"></i> Copy As Mailing List</button>
```
### 2. Event Listener & AJAX Call

This is an ASP.NET Core Razor View, so this bit goes in the `@section Scripts{}` bit. No frameworks here: jQuery is as fancy as it gets. 

Here we establish a click handler for the button which:
- Gathers the filter data as it exists
- Passes it to the Controller via AJAX
- Pipes the response to a modal
- Gives the user UI feedback while it's doing so

```js
$('#exportMailingListBtn').click(function() {
    // Get current form data: these elements are all nested in the form.
    var formData = {
        ShowInactive: $('#ShowInactive').is(':checked'),
        NameFilter: $('#NameFilter').val(),
        RoleFilter: $('#RoleFilter').val(),
        PageSize: $('#PageSize').val(),
        PageNum: 0,
        SortBy: $('#SortBy').val()
    };

    // User Feedback & Multiple Request prevention: Save the button text to a variable, then swap it with a spinner and disable it while the function runs.
    var $btn = $(this);
    var originalText = $btn.html();
    $btn.html('<i class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></i> Loading...');
    $btn.prop('disabled', true);

    // Make the Ajax Call
    $.ajax({
        url: '@Url.Action("GenerateMailingList", "Staff")', // The Controller Action and Controller Name are passed as arguments
        type: 'POST',
        data: formData,
        success: function(response) {
            if (response.success) {
              // If the controller indicates success with the response, pass the response payload to the mailingListContent element, and show the modal (see part four below).
                $('#mailingListContent').val(response.mailingList);
                $('#mailingListModal').modal('show');
            } else {
                alert('Error: ' + (response.message || 'Failed to generate mailing list.'));
            }
        },
        error: function() {
            alert('An error occurred while generating the mailing list.');
        },
        complete: function() {
            // Return the button to its start state
            $btn.html(originalText);
            $btn.prop('disabled', false);
        }
    });
});
```

### 3. Controller Action

Great - so what is it we're fetching? This comma seperated list of email formatted strings.

> I won't bore you with the exact C# class definitions but the important thing to note if you're not familiar with ASP.NET Core is that we're working with *model binding*, meaning that in the Controller Action, we're asking the framework to rustle up a strongly typed `StaffFilterViewModel` object from the data in the POST request's body. This 'binding' will fail if the post body object doesn't have properties which match the StaffFilterViewModel's exactly (or very close: there is some wiggle room around things like capitalisation, by default). So the `formData.NameFilter` we described in the ajax method 
{: .prompt-emphasis }

```csharp
// Export Mailing List
[Authorize] 
[HttpPost]
public async Task<IActionResult> GenerateMailingList(StaffFilterViewModel model)
{
    try
    {
        List<StaffMember> StaffMemberList = [];

        // Optionally filter by Role. This could be a ternary expression but it's written out here for clarity. Also notice there's no handling for the RoleFilter passing an invalid value (e.g. a role which doesn't exist), but the original form is generating the possible values dynamically so I'm not going overboard with that.
        if (!string.IsNullOrEmpty(model.RoleFilter))
        {
            StaffMemberList = await _StaffMemberService.GetUsersInRoleAsync(model.RoleFilter);
        }
        else
        {
            StaffMemberList = await _StaffMemberService.GetUsersAsync();
        }

        // Filter for including Inactive users
        if (!model.ShowInactive)
        {
            StaffMemberList = [.. StaffMemberList.Where(u => u.Active == true)];
        }

        // Filter the list by Name
        if (!string.IsNullOrEmpty(model.NameFilter))
        {
            StaffMemberList = [.. StaffMemberList.Where(u => u.FullName.ToLower().Contains(model.NameFilter.ToLower()))];
        }

        StaffMemberList = [.. StaffMemberList.OrderBy(u => u.LastName).ThenBy(u => u.FirstName)];

        // Create the mailing list string, sticking to the format: "Name <email>, Name <email>" for easy copy & paste
        string mailingList = string.Join(", ", StaffMemberList
            .Where(u => !string.IsNullOrEmpty(u.Email))
            .Select(u => $"{u.FullName} <{u.Email}>"));

        // Return a serialised json response. Thanks, asp.net core.
        return Json(new { success = true, mailingList = mailingList });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error exporting mailing list");
        return Json(new { success = false, message = "An error occurred while generating the mailing list." });
    }
}
```

### 4. Display the Response in a Modal

I love the built in [bootstrap modals](https://getbootstrap.com/docs/5.3/components/modal/). I know they aren't sexy but they are super simple.
```html
<!-- Mailing List Modal -->
<div class="modal fade" id="mailingListModal" tabindex="-1" aria-labelledby="mailingListModalLabel" aria-hidden="true">
    <div class="modal-dialog modal-lg modal-dialog-centered">
        <div class="modal-content">
            <div class="modal-header">
                <h1 class="modal-title fs-5" id="mailingListModalLabel"><i class="bi bi-envelope"></i> Mailing List</h1>
                <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
            </div>
            <div class="modal-body">
                <p class="text-muted">Copy the mailing list below based on your current filter settings:</p>
                <div class="border p-3 bg-light">
                    <textarea id="mailingListContent" class="form-control" rows="6" readonly style="resize: none;"></textarea>
                </div>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
                <button type="button" class="btn btn-primary" id="copyToClipboardBtn"><i class="bi bi-clipboard"></i> Copy to Clipboard</button>
            </div>
        </div>
    </div>
</div>
```
> Notice the `#mailingListContent` textarea in the middle. This is the element the click event handler is passing the ajax's response to. And the `#copyToClipboardBtn` is about to get its own handler, below.
{: .prompt-info }

### 5. Copy to Clipboard

I love this bit. A lot of my users aren't particularly technical. Select all & copy is not, it turns out, second nature to everyone on the planet, so a click-to-copy button can be hugely helpful for reducing friction.

This method is easy because of the [Clipboard API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard_API) which I still think of as *new* but has been part of Chrome & Firefox since 2018 and Edge & Safari since 2020 so compatibility isn't too much of an issue.

All we do is:
1. Get the mailingListText from the element
2. Write it to the clipboard
3. Let the user know it worked (or if it didn't)

```js
$('#copyToClipboardBtn').click(function() {
    var mailingListText = $('#mailingListContent').val();
    if (mailingListText) {
          navigator.clipboard.writeText(mailingListText).then(function() {
              // Show success feedback
              var $btn = $('#copyToClipboardBtn');
              var originalText = $btn.html();
              $btn.html('<i class="bi bi-check-circle"></i> Copied!');
              $btn.removeClass('btn-primary').addClass('btn-success');
              
              setTimeout(function() {
                  $btn.html(originalText);
                  $btn.removeClass('btn-success').addClass('btn-primary');
              }, 2000);
          }).catch(function(err) {
              console.error('Failed to copy: ', err);
              $btn.html('<i class="bi bi-x"></i> Copy Failed - Copy Manually');
              $btn.removeClass('btn-primary').addClass('btn-error');
          });
    }
});
```

## Wrapping Up
That's it! Told you there wasn't much to it. I would say one of the dangers of working in the .NET ecosystem is the temptation to over engineer everything. It's a very enterprise-focused and powerful feature set, designed for big projects which get extended and maintained over a long period of time.

Sometimes you just need some emails to paste, though. Don't overthink it.
