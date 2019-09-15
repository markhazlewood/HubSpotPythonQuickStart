# HubSpot QuickStart using Python + OAUTH

Follow these steps to create a simple Python command-line app that makes requests to HubSpot's APIs.

## Prerequisites

To run this quickstart, you'll need:

1. Python 2.6 or greater
2. The [pip](https://pypi.python.org/pypi/pip) package management tool
3. HubSpot Free CRM - [Signup for free here](https://www.hubspot.com/products/get-started?utm_source=khlavka-hubspotquickstartpython&utm_medium=git) 
4. HubSpot Developer Account - [Signup for free here](https://developers.hubspot.com/get-started?utm_source=khlavka-hubspotquickstartpython&utm_medium=git)

## Step 1: Create a HubSpot App

From your **Developer Account**, create a new App to use with this quickstart.
  
  > <a href="https://app.hubspot.com/l/developer/applications" target="_blank">Open your Developer App Manager</a> - Select your Developer Account on the next page.

Copy your **Client ID** and **Client Secret** from your App's `Auth` tab.  You will need these for Step 3.

## Step 2: Install required packages via `pip`

Run the following command to install the Python packages used for OAUTH:

    $ pip install --upgrade requests_oauthlib

## Step 3: Set up your QuickStart App

  1. Create a file named `quickstart.py` in your working directory
  2. Copy in the [QuickStart Sample App](#hubspot-quickstart-sample-app---view-in-git) code below.
  3. Update `CLIENT_ID` and `CLIENT_SECRET` with your App's ID and Secret from Step 1.

## Step 4: Run your app

Run your app with the following command:

    $ python quickstart.py

1. The sample will attempt to open a new window or tab in your default browser. If this fails, copy the URL from the console and manually open it in your browser.

> If you are not logged in to your HubSpot CRM account, you will be prompted via web browser. If you have more than one account, you will be asked to select one account to use for the authorization.

2. Click the Accept button.

3. Your app will continue automatically, and you may close the window/tab.

**Done!** Your app will print the API response.


## Next...

* Get multiple records -- Update the `count` param in your request to fetch up to 100 records.
* Create a Contact -- Via `.post` and the [Create or Update a Contact API](https://developers.hubspot.com/docs/methods/contacts/create_or_update).
* Create and update multiple Contacts at once -- Via the [Contacts Batch API](https://developers.hubspot.com/docs/methods/contacts/batch_create_or_update).
* Work with other APIs -- Update the request method (`.get` vs. `.post`) URL, parameters, and [auth scopes](https://developers.hubspot.com/docs/methods/oauth2/initiate-oauth-integration#scopes) (if needed).

**Feeling bold?** Create additional record types like [Companies](https://developers.hubspot.com/docs/methods/companies/create_company) and [Deals](https://developers.hubspot.com/docs/methods/deals/create_deal), then create relationships between them using [CRM associations](https://developers.hubspot.com/docs/methods/crm-associations/crm-associations-overview).)

----

## HubSpot QuickStart Sample App - [View in git](quickstart/quickstart.py)

```python
from __future__ import print_function
from requests_oauthlib import OAuth2Session  
import os
import pickle
import json

# Replace with your App's Client ID and Secret
CLIENT_ID     = 'your-apps-client-id'
CLIENT_SECRET = 'your-apps-client-secret'

# If modifying these scopes, delete the file hstoken.pickle.
SCOPES        = ['contacts']

#================================================================
#==== QuickStart Command-line App

def main():
    """
    Connects your app a Hub, then fetches the first Contact in the CRM.
    Note: If you want to change hubs or scopes, delete the `hstoken.pickle` file and rerun.
    """
    app_config = {
        'client_id': CLIENT_ID,
        'client_secret': CLIENT_SECRET,
        'scopes': SCOPES,
        'auth_uri': 'https://app.hubspot.com/oauth/authorize',
        'token_uri': 'https://api.hubapi.com/oauth/v1/token'
    }

    # The file hstoken.pickle stores the app's access and refresh tokens for the hub you connect to.
    # It is created automatically when the authorization flow completes for the first time.
    if os.path.exists('hstoken.pickle'):
        with open('hstoken.pickle','rb') as tokenfile:
            token = pickle.load(tokenfile)
    # If no token file is found, let the user log in (and install the app if needed)
    else:
        token = InstallAppAndCreateToken(app_config)
        # Save the credentials for the next run
        SaveTokenToFile(token)

    # Create an OAuth session using your app_config and token
    hubspot = OAuth2Session(
        app_config['client_id'], 
        token=token, 
        auto_refresh_url=app_config['token_uri'],
        auto_refresh_kwargs=app_config, 
        token_updater=SaveTokenToFile
    )

    # Call the 'Get all contacts' API endpoint
    response = hubspot.get(
            'https://api.hubapi.com/contacts/v1/lists/all/contacts/all', 
            params={ 'count': 1 } # Return only 1 result -- for demo purposes
        )

    # Pretty-print our API result to console
    print('Here is one Contact Record from your CRM:')
    print('-----------------------------------------')
    print(json.dumps(response.json(), indent=2, sort_keys=True))

    
#===================================================================
#==== Supporting Functions and Classes used by the command-line app. 

def InstallAppAndCreateToken(config):
    """
    Creates a simple local web app+server to authorize your app with a HubSpot hub.
    Returns the refresh and access token.
    """  
    from wsgiref import simple_server
    import webbrowser

    local_webapp = SimpleAuthCallbackApp()
    local_webserver = simple_server.make_server(host='localhost', port=0, app=local_webapp)

    redirect_uri = 'http://{}:{}/'.format('localhost', local_webserver.server_port)

    oauth = OAuth2Session(
        client_id=config['client_id'],
        scope=config['scopes'],
        redirect_uri=redirect_uri
    )

    auth_url, _ = oauth.authorization_url(config['auth_uri'])
    
    print('-- Authorizing your app via Browser --')
    print('If your browser does not open automatically, visit this URL:')
    print(auth_url)
    webbrowser.open(auth_url, new=1, autoraise=True)
    local_webserver.handle_request()

    # Note: using https here because oauthlib is very picky that
    # OAuth 2.0 should only occur over https.
    auth_response = local_webapp.last_request_uri.replace('http','https')

    token = oauth.fetch_token(
        config['token_uri'],
        authorization_response=auth_response,
        # HubSpot requires you to include the ClientID and ClientSecret
        include_client_id=True,
        client_secret=config['client_secret']
    )
    return token

class SimpleAuthCallbackApp(object):
    """
    Used by our simple server to receive and 
    save the callback data authorization.
    """
    def __init__(self):
        self.last_request_uri = None
        self._success_message = (
            'All set! Your app is authorized.  ' + 
            'You can close this window now and go back where you started from.'
        )

    def __call__(self, environ, start_response):
        from wsgiref.util import request_uri
        
        start_response('200 OK', [('Content-type', 'text/plain')])
        self.last_request_uri = request_uri(environ)
        return [self._success_message.encode('utf-8')]

def SaveTokenToFile(token):
    """
    Saves the current token to file for use in future sessions.
    """
    with open('hstoken.pickle', 'wb') as tokenfile:
        pickle.dump(token, tokenfile)
        
if __name__ == '__main__':
    main()
```

### Notes

* Authorization information is stored on the file system, so subsequent executions will not prompt for authorization.
* The authorization flow in this example is designed for a command-line application only.