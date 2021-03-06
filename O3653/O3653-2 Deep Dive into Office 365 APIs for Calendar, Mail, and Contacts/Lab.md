# Connect to the Office 365 mail with the Microsoft Graph
Learn how to read, send, reply and forward messages in Office 365 Outlook and Outlook.com using the Microsoft Graph, AAD v2 end point, and ASP.NET MVC 5. 

[//]: # (Remove if doing v1)

## Get an Office 365 developer environment
To complete the exercises below, you will require an Office 365 developer environment. Use the Office 365 tenant that you have been provided with for Microsoft Ignite.

## Exercise 1: Create a new project using Azure Active Directory v2 authentication

In this first step, you will create a new ASP.NET MVC project using the
**Graph AAD Auth v2 Start Project** template, register a new application
in the developer portal, and log in to your app and generate access tokens
for calling the Graph API.

1. Launch Visual Studio 2015 and select **New**, **Project**.
  1. Search the installed templates for **Graph** and select the
    **Graph AAD Auth v2 Starter Project** template.
  1. Name the new project **QuickStartMailWebApp** and click **OK**.
  1. Open the **Web.config** file and find the **appSettings** element. This is where you will need to add your appId and app secret you will generate in the next step.
1. Launch the Application Registration Portal by opening a browser and navigating to **apps.dev.microsoft.com**
   to register a new application.
  1. Sign into the portal using your Office 365 username and password.
  1. Click **Add an App** and type **GraphMailQuickStart** for the application name.
  1. Copy the **Application Id** and paste it into the value for **ida:AppId** in your project's **web.config** file.
  1. Under **Application Secrets** click **Generate New Password** to create a new client secret for your app.
  1. Copy the displayed app password and paste it into the value for **ida:AppSecret** in your project's **web.config** file.
  1. Modify the **ida:AppScopes** value to include the required `https://graph.microsoft.com/mail.readwrite` and `https://graph.microsoft.com/mail.send` scopes.

  ```xml
  <configuration>
    <appSettings>
      <!-- ... -->
      <add key="ida:AppId" value="paste application id here" />
      <add key="ida:AppSecret" value="paste application password here" />
      <!-- ... -->
      <!-- Specify scopes in this value. Multiple values should be comma separated. -->
      <add key="ida:AppScopes" value="https://graph.microsoft.com/mail.readwrite,https://graph.microsoft.com/mail.send" />
    </appSettings>
    <!-- ... -->
  </configuration>
  ```
1. Add a redirect URL to enable testing on your localhost.
  1. Right click on **QuickStartMailWebApp** and click on **Properties** to open the project properties.
  1. Click on **Web** in the left navigation.
  1. Copy the **Project Url** value.
  1. Back on the Application Registration Portal page, click **Add Platform** and then **Web**.
  1. Paste the value of **Project Url** into the **Redirect URIs** field.
  1. Scroll to the bottom of the page and click **Save**.

1. Set Startup page to Signout page (to avoid stale token error) 
  1. Right-click **QuickStartMailWebApp** and click **Properties** to open the project properties.
  1. Click **Web** in the left navigation.
  1. Under **Start Action** Choose **Specific Page** option and Type its value as **Account/SignOut**  

1. Press F5 to compile and launch your new application in the default browser.
  1. Once the Graph and AAD v2 Auth Endpoint Starter page appears, click **Sign in** and login to your Office 365 account.
  1. Review the permissions the application is requesting, and click **Accept**.
  1. Now that you are signed into your application, exercise 1 is complete!

[//]: # (Remove if doing v2)

## Exercise 2: Access Mail through Microsoft Graph SDK

In this exercise, you will build on exercise 1 to connect to the Microsoft Graph
SDK and work with Office 365 and Outlook Mail

## Working with Mail through Microsoft Graph SDK

### Create the Mail controller and use the Graph SDK

1. Add a reference to the Microsoft Graph SDK to your project
  1. In the **Solution Explorer** right click on the **QuickStartMailWebApp** project and select **Manage NuGet Packages...**.
  1. Change **Package Source** (on the upper right corner) to **Microsoft Graph local**.
  1. Click **Browse** and search for **Microsoft.Graph**.
  1. Select the Microsoft Graph SDK and click **Install**.

1. Create a new controller to process the requests for files and send them to Graph API.
  1. Find the **Controllers** folder under **QuickStartMailWebApp**, right-click the **Controllers** folder and choose **Add** > **New Scaffolded Item...**.
  2. In the **Add Scaffold** dialog, select **MVC 5 Controller - Empty**, and choose **Add**.
  3. In the **Add Controller** dialog, name the controller `MailController` and choose **Add**.

1. **Replace** the following reference to the top of the `MailController` class
  ```csharp
  using System;
  using System.Collections.Generic;
  using System.Linq;
  using System.Web;
  using System.Web.Mvc;
  ```

  with the following references
  ```csharp
  using System;
  using System.Collections.Generic;
  using System.Linq;
  using System.Web;
  using System.Web.Mvc;
  using System.Configuration;
  using System.Threading.Tasks;
  using Microsoft.Graph;
  using QuickStartMailWebApp.Auth;
  using QuickStartMailWebApp.TokenStorage;
  using Newtonsoft.Json;
  using System.IO;
  ```

1. Add the following code to the `MailController` class to initialize a new
   `GraphServiceClient` and generate an access token for the Graph API:

  ```csharp
    private GraphServiceClient GetGraphServiceClient()
    {
        string userObjId = AuthHelper.GetUserId(System.Security.Claims.ClaimsPrincipal.Current);
        SessionTokenCache tokenCache = new SessionTokenCache(userObjId, HttpContext);

        string authority = string.Format(ConfigurationManager.AppSettings["ida:AADInstance"], "common", "");

        AuthHelper authHelper = new AuthHelper(
            authority,
            ConfigurationManager.AppSettings["ida:AppId"],
            ConfigurationManager.AppSettings["ida:AppSecret"],
            tokenCache);

        // Request an accessToken and provide the original redirect URL from sign-in
        GraphServiceClient client = new GraphServiceClient(new DelegateAuthenticationProvider(async (request) =>
        {
            string accessToken = await authHelper.GetUserAccessToken(Url.Action("Index", "Home", null, Request.Url.Scheme));
            request.Headers.TryAddWithoutValidation("Authorization", "Bearer " + accessToken);
        }));

        return client;
    }
  ```

### Work with Mail

1. Replace the following code in the `MailController` class 

  ```csharp
        // GET: Mail
        public ActionResult Index()
        {
            return View();
        }
  ```
  
  with the following code to get all mail messages from your mailbox.
  
  ```csharp
        // GET: Me/Messages
        [Authorize]
        public async Task<ActionResult> Index(int? pageSize, string nextLink)
        {
            if (!string.IsNullOrEmpty((string)TempData["error"]))
            {
                ViewBag.ErrorMessage = (string)TempData["error"];
            }

            pageSize = pageSize ?? 25;

            var client = GetGraphServiceClient();

            var request = client.Me.MailFolders.Inbox.Messages.Request().Top(pageSize.Value);
            if (!string.IsNullOrEmpty(nextLink))
            {
                request = new MailFolderMessagesCollectionRequest(nextLink, client, null);
            }

            try
            {
                var results = await request.GetAsync();

                ViewBag.NextLink = null == results.NextPageRequest ? null :
                  results.NextPageRequest.GetHttpRequestMessage().RequestUri;

                return View(results);
            }
            catch (ServiceException ex)
            {
                return RedirectToAction("Index", "Error", new { message = ex.Error.Message });
            }
        }
  ```

1. Add the following code to the `MailController` class to display details of a mail.

  ```csharp
        // GET: Message/Detail?messageId=<id>
        [Authorize]
        public async Task<ActionResult> Detail(string messageId)
        {
            var client = GetGraphServiceClient();

            var request = client.Me.Messages[messageId].Request();

            try
            {
                var result = await request.GetAsync();

                TempData[messageId] = result.Body.Content;

                return View(result);
            }
            catch (ServiceException ex)
            {
                return RedirectToAction("Index", "Error", new { message = ex.Error.Message });
            }
        }

        public async Task<ActionResult> GetMessageBody(string messageId)
        {
            return Content(TempData[messageId] as string);
        }
  ```

1. Add the following code to the `MailController` class to send a new mail.

  ```csharp
        // POST Messages/SendMail
        [Authorize]
        [HttpPost]
        public async Task<ActionResult> SendMail(string messageId, string recipients,  string subject, string body)
        {
            if (string.IsNullOrEmpty(subject) || string.IsNullOrEmpty(recipients))
            {
                TempData["error"] = "Please fill in all fields";
            }
            else
            {
                List<Recipient> mailRecipients = new List<Recipient>();
                if (!buildRecipients(recipients, mailRecipients))

                {
                    TempData["error"] = "Please provide valid email addresses";
                }
                var client = GetGraphServiceClient();
                ItemBody CurrentBody = new ItemBody();
                CurrentBody.Content = (string.IsNullOrEmpty(body) ? "" : body);

                Message newMessage = new Message()
                {
                    Subject = subject,
                    Body = CurrentBody,
                    ToRecipients = mailRecipients
                };
                var request = client.Me.SendMail(newMessage, true).Request();

                try
                {
                    await request.PostAsync();
                }
                catch (ServiceException ex)
                {
                    TempData["error"] = ex.Error.Message;
                }
            }

            return RedirectToAction("Index", new { messageId = messageId });
        }

        const string SEMICOLON = ";";
        const string PERIOD = ".";
        const string AT = "@";
        const string SPACE = " ";
        
        private bool buildRecipients(string strRecipients, List<Recipient> Recipients)
        {
            int iSemiColonPos = -1;
            string strTemp = strRecipients.Trim();
            string strEmailAddress = null;
            Recipient recipient = new Recipient();

            while (strTemp.Length != 0)
            {
                iSemiColonPos = strTemp.IndexOf(SEMICOLON);
                if (iSemiColonPos != -1)
                {
                    strEmailAddress = strTemp.Substring(0, iSemiColonPos);
                    strTemp = strTemp.Substring(iSemiColonPos + 1).Trim();
                }
                else
                {
                    strEmailAddress = strTemp;
                    strTemp = "";
                }
                int iAt = strEmailAddress.IndexOf(AT);
                int iPeriod = strEmailAddress.LastIndexOf(PERIOD);
                if ((iAt != -1) && (iPeriod != -1) && (strEmailAddress.LastIndexOf(SPACE) == -1) && (iPeriod > iAt))
                {
                    EmailAddress mailAddress = new EmailAddress();
                    mailAddress.Address = strEmailAddress;
                    Recipient mailRecipient = new Recipient();
                    mailRecipient.EmailAddress = mailAddress;
                    Recipients.Add(mailRecipient);
                }
                else
                {
                    return false;
                }
                strEmailAddress = null;

            }
            return true;
        }
  ```

1. Add the following code to the `MailController` class to reply to a mail.

  ```csharp
        [Authorize]
        [HttpPost]
        public async Task<ActionResult> Reply(string messageId, string comment)
        {
            var client = GetGraphServiceClient();
                
            var request = client.Me.Messages[messageId].Reply(comment).Request();

            try
            {
                await request.PostAsync();
            }
            catch (ServiceException ex)
            {
                return RedirectToAction("Index", "Error", new { message = ex.Error.Message });
            }

            return RedirectToAction("Detail", new { messageId = messageId });
        }
  ```

1. Add the following code to the `MailController` class to reply all to a mail.

  ```csharp
        [Authorize]
        [HttpPost]
        public async Task<ActionResult> ReplyAll(string messageId, string comment)
        {
            var client = GetGraphServiceClient();

            var request = client.Me.Messages[messageId].ReplyAll(comment).Request();

            try
            {
                await request.PostAsync();
            }
            catch (ServiceException ex)
            {
                return RedirectToAction("Index", "Error", new { message = ex.Error.Message });
            }

            return RedirectToAction("Detail", new { messageId = messageId });
        }
  ```

  1. Add the following code to the `MailController` class to forward a mail.

  ```csharp
        // Forward functionality commented out because of a bug with the libraries
        /*
                [Authorize]
                [HttpPost]
                public async Task<ActionResult> Forward(string messageId, string comment, string recipients)
                {
                    var client = GetGraphServiceClient();
                    EmailAddress mailAddress = new EmailAddress();
                    mailAddress.Address = "zrinkam@tr22rest.onmicrosoft.com";
                    Recipient mailRecipient = new Recipient();
                    mailRecipient.EmailAddress = mailAddress;

                    var request = client.Me.Messages[messageId].Forward(comment).Request();

                    try
                    {
                        await request.PostAsync();
                    }
                    catch (ServiceException ex)
                    {
                        return RedirectToAction("Index", "Error", new { message = ex.Error.Message });
                    }

                    return RedirectToAction("Detail", new { messageId = messageId });
                }

        */
  ```

### Create the MailList view

In this section you'll wire up the Controller you created in the previous section
to an MVC view that will display all your Inbox messages and allow you to send a new mail.

1. Locate the **Views/Shared** folder in the project.
1. Open the **_Layout.cshtml** file found in the **Views/Shared** folder.
  1. Locate the part of the file that includes a few links at the top of the
      page. It should look similar to the following code:

  ```asp
    <ul class="nav navbar-nav">
        <li>@Html.ActionLink("Home", "Index", "Home")</li>
        <li>@Html.ActionLink("About", "About", "Home")</li>
        <li>@Html.ActionLink("Contact", "Contact", "Home")</li>
        <li>@Html.ActionLink("Graph API", "Graph", "Home")</li>
    </ul>
  ```

  1. Update that navigation to replace the "Graph API" link with "Outlook Mail API"
      and connect this to the MailController you just created.

  ```asp
      <ul class="nav navbar-nav">
          <li>@Html.ActionLink("Home", "Index", "Home")</li>
          <li>@Html.ActionLink("About", "About", "Home")</li>
          <li>@Html.ActionLink("Contact", "Contact", "Home")</li>
          <li>@Html.ActionLink("Outlook Mail API", "Index", "Mail")</li>
      </ul>
  ```
1. Create a new **View** for MailList.
  1. Expand **Views** folder in **QuickStartMailWebApp** and select the folder **Mail**.  
  1. Right-click **Mail** and select **Add** then **New Item**.
  1. Select **MVC 5 View Page (Razor)** and change the filename **Index.cshtml** and click **Add**.
  1. **Replace** all of the code in the **Mail/Index.cshtml** with the following:

  ```asp
@model IEnumerable<Microsoft.Graph.Message>
@{ ViewBag.Title = "Index"; }
<h2>Inbox</h2>
@section scripts {
    <script type="text/javascript">
$(function () {
    $('#start-picker').datetimepicker({ format: 'YYYY-MM-DDTHH:mm:ss', sideBySide: true });
    $('#end-picker').datetimepicker({ format: 'YYYY-MM-DDTHH:mm:ss', sideBySide: true });
});
    </script>
}
<div class="row" style="margin-top:50px;">
    <div class="col-sm-12">
        @if (!string.IsNullOrEmpty(ViewBag.ErrorMessage))
        {
            <div class="alert alert-danger">@ViewBag.ErrorMessage</div>
        }
        <div class="panel panel-default">
            <div class="panel-body">
                <form class="form-inline" action="/Mail/SendMail" method="post">
                    <div class="form-group">
                        <input type="text" class="form-control" name="recipients" id="recipients" placeholder="To" />
                    </div>
                    <div class="form-group">
                        <input type="text" class="form-control" name="subject" id="subject" placeholder="Subject" />
                    </div>
                    <div class="form-group">
                        <input type="text" class="form-control" name="body" id="body" placeholder="body" />
                    </div>
                    <input type="hidden" name="messageId" value="@Request.Params["messageId"]" />
                    <button type="submit" class="btn btn-default">Send Mail</button>
                </form>
            </div>
        </div>
        <div class="table-responsive">
            <table id="MailTable" class="table table-striped table-bordered">
                <thead>
                    <tr>
                        <th>From</th>
                        <th><p>Subject</p><p>Body Preview</p></th>
                        <th>Received</th>
                        <th>Has Attachments</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var MailMessage in Model)
                    {
                        <tr>
                            <td>
                                @MailMessage.From.EmailAddress.Name
                            </td>
                            <td>
                                @{
                                    RouteValueDictionary idVal = new RouteValueDictionary();
                                    idVal.Add("messageId", MailMessage.Id);
                                    if (null != @MailMessage.IsRead)
                                    {
                                        if ((bool)MailMessage.IsRead)
                                        {
                                            <p>
                                                    @if (!string.IsNullOrEmpty(MailMessage.Subject))
                                                    {
                                                        @Html.ActionLink(MailMessage.Subject, "Detail", idVal)
                                                    }
                                                    else
                                                    {
                                                        @Html.ActionLink("(no subject)", "Detail", idVal)
                                                    }
                                            </p>
                                        }

                                        else
                                        {
                                            <p>
                                                <b>
                                                    @if (!string.IsNullOrEmpty(MailMessage.Subject))
                                                    {
                                                        @Html.ActionLink(MailMessage.Subject, "Detail", idVal)
                                                    }
                                                    else
                                                    {
                                                        @Html.ActionLink("(no subject)", "Detail", idVal)
                                                    }
                                                </b>
                                            </p>
                                        }
                                    }
                                }
                                @MailMessage.BodyPreview
                            </td>
                            <td>
                                @{
                                    if (null != MailMessage.ReceivedDateTime)
                                    {
                                        @MailMessage.ReceivedDateTime
                                    }
                                }
                            </td>
                            <td>
                                @{
                                    if (null != MailMessage.HasAttachments)
                                    {
                                        @((bool)MailMessage.HasAttachments ? "Yes" : "No")
                                    }
                                }
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
        <div class="btn btn-group-sm">
            @{
                Dictionary<string, object> attributes = new Dictionary<string, object>();
                attributes.Add("class", "btn btn-default");

                if (null != ViewBag.NextLink)
                {
                    RouteValueDictionary routeValues = new RouteValueDictionary();
                    routeValues.Add("nextLink", ViewBag.NextLink);
                    @Html.ActionLink("Next Page", "Index", "Mail", routeValues, attributes);
                }
            }
        </div>

    </div>
</div>
  ```

1. Create a new **View** for mail detail.
  1. Expand the **Views** folder in **QuickStartMailWebApp**. Right-click **Mail** and select
      **Add** then **New Item**.
  1. Select **MVC View Page** and change the filename **Detail.cshtml** and click **Add**.
  1. **Replace** all of the code in the **Mail/Detail.cshtml** with the following:

  ```asp
<head>
    <style type="text/css">
        .auto-style9 {
            height: 404px;
        }
        .auto-style10 {
            width: 601px;
        }
        .auto-style11 {
            width: 575px;
        }
        .auto-style12 {
            width: 461px;
        }
    </style>
</head>
@model Microsoft.Graph.Message
@{ ViewBag.Title = "Detail"; }
<h2>@Model.Subject</h2>
@section scripts {
    <script type="text/javascript">
$(function () {
    $('#start-picker').datetimepicker({ format: 'YYYY-MM-DDTHH:mm:ss', sideBySide: true });
    $('#end-picker').datetimepicker({ format: 'YYYY-MM-DDTHH:mm:ss', sideBySide: true });
});
    </script>
}
<div class="row" style="margin-top:50px;">
    <div class="col-sm-12">
        @if (!string.IsNullOrEmpty(ViewBag.ErrorMessage))
        {
            <div class="alert alert-danger">@ViewBag.ErrorMessage</div>
        }

        <div class="panel panel-default">
            <div class="panel-body">
                <table>
                    <th>Respond to Message<th>
                    <tbody>
                        <tr>
                            <form class="form-inline" action="/Mail/Reply" method="post">
                                <td class="auto-style10">
                                   <input type="text" class="auto-style11" name="comment" id="comment" placeholder="Comment" />
                                </td>

                                <td class="auto-style3">
                                    <input type="hidden" name="messageId" value="@Request.Params["messageId"]" />
                                    <button type="submit" name="Reply" class="btn btn-default">Reply</button>
                                </td>
                            </form>
                        </tr>
                        <br />
                        <tr>
                            <form class="form-inline" action="/Mail/ReplyAll" method="post">
                                <td class="auto-style10">
                                    <input type="text" class="auto-style11" name="comment" id="comment" placeholder="Comment" />
                                </td>

                                <td>
                                    <input type="hidden" name="messageId" value="@Request.Params["messageId"]" />
                                    <button type="submit" name="Reply All" class="btn btn-default">Reply All</button>
                                </td>
                            </form>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
        <div class="table-responsive">
            <table id="messageTable" class="table table-striped table-bordered">
                <th class="auto-style12">Details</th>
                <tbody>
                    <tr>
                        <td class="auto-style12">From:</td>
                        <td>
                            @Model.From.EmailAddress.Name
                        </td>
                    </tr>
                    <tr>
                        <td class="auto-style12">To:</td>
                        <td>
                            @{
                                string toRecipients = "";

                                foreach (var To in Model.ToRecipients)
                                {
                                    toRecipients = toRecipients + To.EmailAddress.Name + "; ";
                                }
                            }
                            @toRecipients
                        </td>
                    </tr>
                    <tr>
                        <td class="auto-style12">Cc:</td>
                        <td>
                            @{
                                string ccRecipients = "";

                                foreach (var To in Model.CcRecipients)
                                {
                                    ccRecipients = ccRecipients + To.EmailAddress.Name + "; ";
                                }
                            }
                            @ccRecipients
                        </td>
                    </tr>
                    <tr>
                        <td class="auto-style12">Received:</td>
                        <td>
                            @{
                                if (null != Model.ReceivedDateTime)
                                {
                                    @Model.ReceivedDateTime
                                }
                            }
                        </td>
                    </tr>
                    <tr>
                        <td class="auto-style12">Has Attachments:</td>
                        <td>
                            @{
                                if (null != Model.HasAttachments)
                                {
                                    @((bool)Model.HasAttachments ? "Yes" : "No")
                                }
                            }
                        </td>
                    </tr>
                    <tr>
                        <td class="auto-style12">Web link:</td>
                        <td>
                            @{
                                if (null != Model.WebLink)
                                {
                                    <a href="@Model.WebLink">Message OWA link </a>
                                }
                            }
                        </td>
                    </tr>
                    <tr>
                        <td class="auto-style12">Body:</td>
                        <td>
                            <div>
                                <iframe id="mailBody" width="800" src="@(string.Format("/Mail/GetMessageBody/?messageId={0}", Model.Id))" class="auto-style9"/>    
                            </div>
                        </td>
                    </tr>
                    <tr>
                </tbody>
            </table>
        </div>

    </div>
</div>

  ```



### Run the app

1. Press **F5** to begin debugging.
1. When prompted, login with your Office 365 administrator account.
1. Click the **Outlook Mail API** link in the navigation bar at the top of the page.
1. Try out the app!

***
Congratulations!, dedicated quick start developer. In this exercise you have created an MVC application that uses Microsoft Graph to view and manage Mail in your mailbox. This quick start ends here.  But don't stop here - there's plenty more to explore with the Microsoft Graph.

## Next Steps and Additional Resources:  
- See this training and more on `http://dev.office.com/` and `http://dev.outlook.com`
- Learn about and connect to the Microsoft Graph at `https://graph.microsoft.io`
