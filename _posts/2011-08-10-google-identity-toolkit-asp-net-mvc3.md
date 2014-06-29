---
layout: post
title: Google Identity toolkit & ASP.NET MVC3
categories:
- Software Development
tags:
- C#
- Google Identity Kit
- MVC
status: publish
type: post
published: true
meta:
  _oembed_9d51dfd79d8b62e93b9588c3caa4b421: "{{unknown}}"
  _oembed_0d18c599336e08389f9a21aa47564aca: "{{unknown}}"
author: 
---

***UPDATE*** I have applied a fix for legacy accounts in Step 9, find out more there.

I've been wanting to learn how to incorporate OpenId into my ASP.NET applications. 

I looked at the excellent [DotNetOpenAuth](http://www.dotnetopenauth.net/ "DotNetOpenAuth") library, but I was curious on what others were doing.

In my travels I came across the [Google Identity Toolkit](http://code.google.com/apis/identitytoolkit/index.html "Open in new window Google Identity Toolkit")

#### What is the Google Identity Toolkit (GITKit)?

A quick excerpt from the GITKit site...

_Google Identity Toolkit (GITkit) is a free toolkit for website operators who currently allow users to login with their email address and password, and would like to replace that password with federated login._

After working through the samples and watching some related youtube videos, I thought it was pretty cool so I signed up for the API access and started to go through the documentation and samples.

At the time, I found the documentation a bit hit and miss. It felt like sections of the documentation was missing and didn't exactly align with the sample code (which is only available in PHP or java).

Thankfully the documentation has been receiving updates and it is a bit better.

Currently, the GITKit supports federated logons that are managed by the following providers

*   Google
*   Yahoo
*   AOL
*   Windows Live

The first three work out of the box, but Live requires a bit of extra work.

You need to register your application and domain with Live and then add your Live developer key into your Google API Console. GITKit has doco on what needs to be done here.

Also to make things more enjoyable, Live does not like localhost (or atleast not mine). The quick solution there is to provide a domain name to Live and then modify your hosts file to point that domain to your dev box.

Anyway, moving forwards...... 

#### How to implement in ASP.NET MVC3

Anyway, I thought I would put together a bit of a walkthrough of how to integrate the GITKit into a new ASP.NET MVC3 Project.

So create a new ASP.NET MVC3 Solution and lets get modifying.

##### Step 1 - Add the widget scripts to your Master/Layout page

Add these script blocks to your page. The GITKit doco says to add to the HEAD block, but I am not entirely sure if that is an absolute requirement or if the scripts can be moved to the bottom of your page.

{% highlight html %}

&amp;lt;script type=&amp;quot;text/javascript&amp;quot; src=&amp;quot;https://ajax.googleapis.com/ajax/libs/jquery/1.4.2/jquery.min.js&amp;quot;&amp;gt;&amp;lt;/script&amp;gt;

&amp;lt;script type=&amp;quot;text/javascript&amp;quot; src=&amp;quot;https://ajax.googleapis.com/ajax/libs/jqueryui/1.8.2/jquery-ui.min.js&amp;quot;&amp;gt;&amp;lt;/script&amp;gt;

&amp;lt;script type=&amp;quot;text/javascript&amp;quot; src=&amp;quot;https://ajax.googleapis.com/ajax/libs/googleapis/0.0.4/googleapis.min.js&amp;quot;&amp;gt;&amp;lt;/script&amp;gt;

&amp;lt;script type=&amp;quot;text/javascript&amp;quot; src=&amp;quot;https://ajax.googleapis.com/jsapi&amp;quot;&amp;gt;&amp;lt;/script&amp;gt;
{% endhighlight %}


````js

var x = 20;

````
&amp;lt;script type=&amp;quot;text/javascript&amp;quot;&amp;gt;

    google.load(&amp;quot;identitytoolkit&amp;quot;, &amp;quot;1.0&amp;quot;, { packages: [&amp;quot;ac&amp;quot;] });

&amp;lt;/script&amp;gt;

&amp;lt;script type=&amp;quot;text/javascript&amp;quot;&amp;gt;

  $(function () {

     window.google.identitytoolkit.setConfig({

       developerKey: &amp;quot;your dev api key goes here&amp;quot;,

       companyName: &amp;quot;Your company&amp;quot;,

       callbackUrl: &amp;quot;@string.Format(&amp;quot;https://yoursite.com{0}&amp;quot;,Url.Action(&amp;quot;Callback&amp;quot;,&amp;quot;Account&amp;quot;))&amp;quot;,

       userStatusUrl: &amp;quot;@Url.Action(&amp;quot;UserStatus&amp;quot;,&amp;quot;Account&amp;quot;)&amp;quot;, // these can just be partial paths

       loginUrl: &amp;quot;@Url.Action(&amp;quot;LogOn&amp;quot;,&amp;quot;Account&amp;quot;)&amp;quot;,

       signupUrl: &amp;quot;@Url.Action(&amp;quot;Register&amp;quot;,&amp;quot;Account&amp;quot;)&amp;quot;,

       homeUrl: &amp;quot;@Url.Action(&amp;quot;Index&amp;quot;,&amp;quot;Home&amp;quot;)&amp;quot;,

       logoutUrl: &amp;quot;@Url.Action(&amp;quot;LogOff&amp;quot;,&amp;quot;Account&amp;quot;)&amp;quot;,

       realm: &amp;quot;&amp;quot;, // optional

       language: &amp;quot;en&amp;quot;,

       idps: [&amp;quot;Gmail&amp;quot;, &amp;quot;AOL&amp;quot;, &amp;quot;Hotmail&amp;quot;, &amp;quot;Yahoo&amp;quot;],

       tryFederatedFirst: true,

       useCachedUserStatus: false

      });

      $('#navbar').accountChooser();

  });

&amp;lt;/script&amp;gt;

[/sourcecode]

*   Any version &amp;gt;= jquery 1.4.2 can be used
*   Any version &amp;gt;= jquery-ui 1.8.2 can be used
*   The callback url **MUST** be a full url. Any of the others can be partial paths

##### Step 2 - change _LogOnPartial

Replace the existing markup with

[sourcecode language="html"]

@if(Request.IsAuthenticated) {

    &amp;lt;text&amp;gt;Welcome&amp;lt;/text&amp;gt;

}

&amp;lt;div id=&amp;quot;navbar&amp;quot;&amp;gt;&amp;lt;/div&amp;gt;

[/sourcecode]

You can have a quick test now if you like and you should see the sign in widget in the top right corner.

Clicking the widget should popup the account chooser screen.

On this screen you can click one of the supported IDP logos or provide an email address. If the email address does not belong to one of the supported providers, the GITKit will redirect the user to the logon url that you specified in the configuration.

##### Step 3 - The Callback Action Method

In the callback action you should validate ID providers response by calling the GITKit verifyAssertion API.

Once you have the have assertion, you can check to see if the user should be logged in or needs to be registered.

Here is my example callback action 

[sourcecode language="csharp"]

public virtual ActionResult Callback()

{

            GitApiClient gitClient = new GitApiClient(&amp;quot;your-developer-api-key-goes-here&amp;quot;);

            GitAssertion assertion = gitClient.Verify();

            string BaseSiteUrl = Request.Url.Scheme + &amp;quot;://&amp;quot; + Request.Url.Authority.TrimEnd('/');

            ViewBag.GitRedirectUrl = BaseSiteUrl + Url.Action(MVC.Home.Index());

            ViewBag.FederatedResponse = GitApiClient.FederatedError;

            if (!string.IsNullOrEmpty(assertion.VerifiedEmail))

            {

                var user = Membership.GetUser(assertion.VerifiedEmail);

                Session[&amp;quot;GitAssertion&amp;quot;] = assertion;

                if (user == null)

                {

                    //create the new user

                   var newUser = Membership.CreateUser(assertion.VerifiedEmail, Guid.NewGuid().ToString());

                   FormsAuthentication.SetAuthCookie(newUser.UserName, true);

                    //if you wanted to collect more details before creating the user account,

                    // then specify the location of that page.

                   // ViewBag.GitRedirectUrl = BaseSiteUrl + Url.Action(MVC.Account.FederatedRegister());

                }

                else

                {

                    //you can decide how you want to manage the &amp;quot;remember me&amp;quot; boolean

                    FormsAuthentication.SetAuthCookie(user.UserName, true);

                }

                ViewBag.GitRedirectUrl = BaseSiteUrl + Url.Action(MVC.Home.Index());

                ViewBag.FederatedResponse = GitApiClient.FederatedSuccess;

            }

            return View();

}

[/sourcecode]

Some things you might have noticed.

*   I am using the ASP.NET Membership provider for the account management
*   I also use the T4 MVC templates from [http://mvccontrib.codeplex.com](http://mvccontrib.codeplex.com) or nuget.
*   And I created a class to encapsulate the call to the GITKit (which I will get to next).

Something extra to note with the Membership provider account creation. In this example I am only dealing with federated logons which have no use for a password. The Membership provider requires a password for account creation, so I chose to set it to a random guid. 

##### Step 4 - GitApiClient &amp;amp; GitAssertion

The GITKit may make either a GET or a POST request to our callback action depending on the user action. Its the GitApiClient's job to wrap up the HttpContext and WebRequest call to the verification API, forwarding the data contained within the callback request.

The GITKit is expecting JSON objects, so I used the awesomo [JSON.NET](http://json.codeplex.com/ "JSON.NET") to handle the JSON de/serialization tasks.

The GitAssertion class is just a POCO to strongly type the response back from the verification API.

Here are the two classes that you should add to your solution.

[sourcecode language="csharp"]

public class GitApiClient

    {

        public static string FederatedSuccess = &amp;quot;federatedSuccess&amp;quot;;

        public static string FederatedError = &amp;quot;federatedError&amp;quot;;

        private readonly string _verifyUrl = &amp;quot;https://www.googleapis.com/identitytoolkit/v1/relyingparty/verifyAssertion?key=&amp;quot;;

        private string _apiKey;

        public GitApiClient(string apiKey)

        {

            _apiKey = apiKey;

        }

        private string GitVerifyPost()

        {

            string result = &amp;quot;&amp;quot;;

            try

            {

                Uri address = new Uri(_verifyUrl + _apiKey);

                HttpRequest request = HttpContext.Current.Request;

                HttpWebRequest gitWebRequest = WebRequest.Create(address) as HttpWebRequest;

                gitWebRequest.Method = &amp;quot;POST&amp;quot;;

                gitWebRequest.ContentType = &amp;quot;application/json&amp;quot;;

                StreamReader requestReader = new StreamReader(request.InputStream);

                var requestBody = requestReader.ReadToEnd();

                string myRequestUri = string.Format(&amp;quot;{0}://{1}{2}&amp;quot;,request.Url.Scheme,request.Url.Authority.TrimEnd('/'), request.RawUrl);

                var verifyRequestData = new { requestUri = myRequestUri, postBody = requestBody };

                byte[] gitRequestData = UTF8Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(verifyRequestData));

                using (Stream stream = gitWebRequest.GetRequestStream())

                {

                    stream.Write(gitRequestData, 0, gitRequestData.Length);

                }

                using (HttpWebResponse response = gitWebRequest.GetResponse() as HttpWebResponse)

                {

                    // Get the response stream

                    StreamReader responseReader = new StreamReader(response.GetResponseStream());

                    result = responseReader.ReadToEnd();

                }

            }

            catch (WebException web)

            {

                throw new Exception(&amp;quot;An error occurred while verifying the IDP response&amp;quot;, web);

            }

            return result;

        }

        public GitAssertion Verify()

        {

            var result = GitVerifyPost();

            return JsonConvert.DeserializeObject&amp;lt;GitAssertion&amp;gt;(result);

        }

    }

[/sourcecode]

[sourcecode language="csharp"]

public class GitAssertion

    {

        public string Kind { get; set; }

        public string Identifier { get; set; }

        public string Authority { get; set; }

        public string VerifiedEmail { get; set; }

        public string FirstName { get; set; }

        public string LastName { get; set; }

        public string FullName { get; set; }

        public string NickName { get; set; }

        public string Language { get; set; }

        public string TimeZone { get; set; }

        public string ProfilePicture { get; set; }

    }

[/sourcecode]

##### Step 5 - Returning the callback response to the GITKit

Once GITKit has made the call to your callback action, you need to inform it of the outcome. Does the user already exist in your system? Do they need to be registered?

To do this, you need to respond back to GITKit by rendering some HTML that calls some of GITKit JavaScript functions.

So create a view under the account folder called "Callback" containing the following html.

[sourcecode language="html"]

&amp;lt;html&amp;gt;

&amp;lt;head&amp;gt;

&amp;lt;script type='text/javascript'&amp;gt;

    function notify() {

       window.opener.google.identitytoolkit.easyrp.util.notifyWidget('@ViewBag.FederatedResponse');

       window.opener.location.href = '@ViewBag.GitRedirectUrl';

       window.close();

    }

&amp;lt;/script&amp;gt;

&amp;lt;/head&amp;gt;

&amp;lt;body onload='notify();'&amp;gt;

&amp;lt;/body&amp;gt;

&amp;lt;/html&amp;gt;

[/sourcecode]

##### Step 6 - Provide the user information to the GITKit widget

Now that the user is authenticated we need to inform the widget of the user details and tell it to change modes and display the user information.

In the _Layout.cshtml add the following piece of code to our script block **_after_** the $('#navbar').accountChooser(); call.

[sourcecode language="csharp"]

@if (Request.IsAuthenticated)

        {

            var user = Session[&amp;quot;GitAssertion&amp;quot;] as GitAssertion;

            if(user != null)

            {

            &amp;lt;text&amp;gt;

                var userData = {

                    email: '@user.VerifiedEmail', // required

                    displayName: '@user.FirstName', // optional

                    photoUrl: 'https://account-chooser.appspot.com/image/nophoto.png', // optional

                };

                window.google.identitytoolkit.updateSavedAccount(userData);

                window.google.identitytoolkit.showSavedAccount(userData.email);

            &amp;lt;/text&amp;gt;

            }

        }

[/sourcecode]

If you now test your solution and attempt to sign in, your IDP should prompt you for your details, the GITKit will then verify, your callback will fire, the popup window will close and your GITKit widget should be updated with the display name and user image.

That is unless you and me both have missed a key section or made a typo somewhere.

##### Step 7 - Fix the widget menu

At this point you should (hopefully) have everything working with the user logging on and the account chooser widget updating.

Clicking on the updated widget displays a menu with two default menu items, switch account and sign out. In case your wondering, yes, these can be changed and added, but that is a completely different exercise.

You may have noticed that your page content overlaps this menu.

To fix this, add the following CSS hack (if anyone has a cleaner solution please let me know)

[sourcecode language="css"]

ol.widget-navbar-menu { position:absolute;}

li.widget-navbar-menuitem {display:list-item; position:relative;z-index: 9999;}

[/sourcecode]

Now you should be able to see the full menu.

Selecting "Switch account" will bring up the widget with all the user details that are currently stored in

localstorage.

Choosing a different account will fire a AJAX request to the specified UserStatus url in the widget configuration.

SignOut is self explanatory.

##### Step 8 - Implement the UserStatus Action Method - optional - sorta...

By default when an account is chosen from the GITKit Account Chooser, it will attempt to authenticate the user as a federated logon.

If the chosen account does not support federated logon or you configured the GITKit to not try federated first; when a different account is chosen, a call will be made to your UserStatus Action method.

Its here where you should sign out the currently signed in user, verify new chosen account details and inform the GITKit of the result.

The GITKit expects a JSON result informing it if the user is registered, is a legacy account or not.

In your Account Controller add this UserStatus Action Method. It is just a *very* lightweight example and you can start to see the limitations of just using the Membership provider.

A proper implementation really needs to be storing these user details (display name, photo etc) somewhere.

Anyway....

[sourcecode language="csharp"]

public virtual JsonResult userStatus(string email)

{

    string userName = Membership.GetUserNameByEmail(email);

    //if the user was switching accounts we need make sure we log out the previous user.

    Session.Abandon();

    FormsAuthentication.SignOut();

    if (userName != null)

    {

         var authUser = Membership.GetUser(userName);

         var user = new { displayName = authUser.UserName, photoUrl = &amp;quot;&amp;quot;, registered = true, legacy = false };

          return Json(user);

     }

     else

     {

          var user = new { displayName = &amp;quot;&amp;quot;, photoUrl = &amp;quot;&amp;quot;, registered = false, legacy = false };

          return Json(user);

      }

}

[/sourcecode]

##### Step 9 - Handle those non federated users

Since this example is updating the GITKit user details from session, we need to make sure that session is appropriately populated when the user is authenticated.

In the default MVC3 project template scenario this means we need to slightly modify our Register POST Action to something like this

[sourcecode language="csharp"]

 [HttpPost]

public virtual ActionResult Register(RegisterModel model)

{

    if (ModelState.IsValid)

    {

         // Attempt to register the user

         MembershipCreateStatus createStatus;

         Membership.CreateUser(model.UserName, model.Password, model.Email, null, null, true, null, out createStatus);

         if (createStatus == MembershipCreateStatus.Success)

         {

              Session[&amp;quot;GitAssertion&amp;quot;] = new GitAssertion() { VerifiedEmail = model.Email, FirstName = model.UserName };

              FormsAuthentication.SetAuthCookie(model.UserName, false /* createPersistentCookie */);

              return RedirectToAction(&amp;quot;Index&amp;quot;, &amp;quot;Home&amp;quot;);

          }

          else

          {

                ModelState.AddModelError(&amp;quot;&amp;quot;, ErrorCodeToString(createStatus));

           }

     }

     // If we got this far, something failed, redisplay form

     return View(model);

}

[/sourcecode]

Now when a non federated account attempts to logon, the GITKit will send to your logon Url the users email and the provided password.

This is done via a AJAX request and we need to return a JSON response back to the GITKit with the logon result.

To use the existing LogOn Post action method we need to make some more subtle changes.

GITKit only gives us an email address and password, so we need to add the email property to our LogOnModel

[sourcecode language="csharp"]

  public class LogOnModel

    {

        [Display(Name = &amp;quot;User name&amp;quot;)]

        public string UserName { get; set; }

        [Required]

        [DataType(DataType.Password)]

        [Display(Name = &amp;quot;Password&amp;quot;)]

        public string Password { get; set; }

        [Display(Name = &amp;quot;Remember me?&amp;quot;)]

        public bool RememberMe { get; set; }

        public string Email { get; set; }

    }

[/sourcecode] 

Now we need to update our LogOn action method. For simplicity I've chosen to keep the existing UserName logic.

To account for this though, if the username has not been provided I query the Membership provider with the supplied email address.

***UPDATE*** To avoid the model state error, I removed the [Required] attribute from the UserName property on the LogOnModel. I feel this is fine for the example, but a real world solution would need appropriate validation rules as your situation called for it.

Also update the username condition in the code below.

Once the user has been authenticated, I again set the GitAssertion into session and if this was a ajax request, I return the expected JSON object back to the GITKit.

Here is the LogOn action method.

[sourcecode language="csharp"]

[HttpPost]

public virtual ActionResult LogOn(LogOnModel model, string returnUrl)

{

    if (model.UserName == string.Empty &amp;amp;&amp;amp; model.Email == string.Empty)

    {

         ModelState.AddModelError(&amp;quot;username&amp;quot;, &amp;quot;username or email is required&amp;quot;);

     }

     else if (string.IsNullOrEmpty(model.UserName))

     {

          model.UserName = Membership.GetUserNameByEmail(model.Email);

     }

     if (ModelState.IsValid)

     {

           if (Membership.ValidateUser(model.UserName, model.Password))

           {

               var user = Membership.GetUser(model.UserName);

               Session[&amp;quot;GitAssertion&amp;quot;] = new GitAssertion() { VerifiedEmail = user.Email,

                                                              FirstName = model.UserName };

               FormsAuthentication.SetAuthCookie(model.UserName, model.RememberMe);

               if (Request.IsAjaxRequest())

               {

                   return Json(new { status = &amp;quot;ok&amp;quot;, displayName = user.UserName, photoUrl = &amp;quot;&amp;quot; });

               }

               if (Url.IsLocalUrl(returnUrl) &amp;amp;&amp;amp; returnUrl.Length &amp;gt; 1 &amp;amp;&amp;amp; returnUrl.StartsWith(&amp;quot;/&amp;quot;)

                   &amp;amp;&amp;amp; !returnUrl.StartsWith(&amp;quot;//&amp;quot;) &amp;amp;&amp;amp; !returnUrl.StartsWith(&amp;quot;/\\&amp;quot;))

               {

                    return Redirect(returnUrl);

                }

                else

                {

                    return RedirectToAction(&amp;quot;Index&amp;quot;, &amp;quot;Home&amp;quot;);

                }

            }

            else

            {

                 ModelState.AddModelError(&amp;quot;&amp;quot;, &amp;quot;The user name or password provided is incorrect.&amp;quot;);

            }

       }

       // If we got this far, something failed, redisplay form

       return View(model);

}

[/sourcecode] 

You should now be able to compile and test federated and legacy accounts.

Supplying "test@test.com" to the account chooser widget should redirect you to the default Register page.

Once the account is created the "Test" user should be logged onto the system.

If you switch accounts between say a federated logon and back to the "Test" account, you should be prompted to supply the password for the test account and then be logged back in.

#### Final words

Well, that should be basically it. I quite like the Account chooser direction that GITKit offers, hopefully it gets promoted out of the labs.

The GITKit is being updated and does have a few other functions and JavaScript hooking points that you can utilize if you want to dig through its source.

Comments, criticisms, and corrections are eagerly welcomed. The example code listed here is exactly that, an example. Ideally magic strings would be removed, DRY principle applied, etc etc.

Also this is a "Hello World" blog post for me. I am not entirely sure what I will come of this experience, maybe I will stick with it, maybe it will just wither and die. Time will tell.

Anyway, please forgive my lack of blogging skills. I hope someone found this helpful haha.