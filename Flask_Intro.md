# Introduction to Integrating Flask Applications in Onshape 
This guide provides a brief introduction on the necessary components of building a Python Flask applcation for Onshape integration. Then, it introduces the actual integration of the app to the user interface of an Onshape document. 

## Table of contents 
* [1. General resources](#1-general-resources)
* [2. Connecting to Onshape documents with Flask](#2-connecting-to-onshape-documents-with-flask)
    * [2.1. Security on API keys](#21-security-on-api-keys)
* [3. Configure Flask as HTTPS](#3-configure-flask-as-https)
* [4. Integrating to Onshape](#4-integrating-to-onshape)
    * [4.1. Integration through OAuth](#41-onshape-integration-through-oauth)
    * [4.2. Integration through Link Tab](#42-integration-through-link-tab)

## 1. General resources 
Before getting started with this guide, you should have a good working knoweldge with Python Flask and how to build a simply web app with Flask. For more information, below are some helpful resources on learning Flask: 

- Flask is a micro web framework written in Python, providing the ability to build light-weight applications. Here are the links to its official [documentation](https://flask.palletsprojects.com/en/2.1.x/) and [a guide for quickstart](https://flask.palletsprojects.com/en/2.1.x/quickstart/#). 
- A good starting guide on learning how to build a Flask app from scratch can be found [here](https://www.digitalocean.com/community/tutorials/how-to-make-a-web-application-using-flask-in-python-3). 

To further improve the webpage design of your Flask app, you may also find these resources to be particularly helpful: 
- HTML forms the most basic structure of the webpage, and a good tutorial can be found [here](https://www.w3schools.com/html/). 
- CSS styling can add additional design capabilities to your Flask app, and a good tutorial can be found [here](https://www.w3schools.com/css/).

While integrating your Flask app to Onshape, you will also very likely need to make REST API calls. For more details and introduction to the REST API in Onshape, please see check out the `API_Intro.md` [guide](https://github.com/PTC-Education/Onshape-Integration-Guides/blob/main/API_Intro.md). 

Below are some more resources and links that will be referred to in this guide, please feel free to save them as resources for your future referrence. 
- A sample project for Onshape that is built in Flask can be found [here](https://github.com/PTC-Education/Heat-Sink-Design). 
- A guide to configure your Flask application as HTTPS can be found [here](https://blog.miguelgrinberg.com/post/running-your-flask-application-over-https). 


## 2. Connecting to Onshape documents with Flask 
To access your Onshape documents, it would require your API access and the address of the Onshape document that you would like to access. For more information on how to obtain these information, please see the `API_Intro.md` guide [here](https://github.com/PTC-Education/Onshape-Integration-Guides/blob/main/API_Intro.md). As a result, in your Flask app, you will need to first define the following global variables: 

    appkey = ''
    secretkey = '' 
    DID = ''
    WID = ''
    EID = ''

Assume that our main purpose of building this Flask app is to make REST API calls (more details on making API calls should be found in the `API_Intro.md` [guide](https://github.com/PTC-Education/Onshape-Integration-Guides/blob/main/API_Intro.md)), but through an integrated approach in the Onshape user interface. Then, we typically need to first build a `/login` page for the users to enter their API keys and the document's IDs, as required by most API calls, to log into the specified document with their credentials. 

    from flask import request 

    @app.route('/login') 
    def login(): 
        global DID 
        global WID 
        global EID 

        DID = request.args.get('documentID')
        WID = request.args.get('workspaceID')
        EID = request.args.get('elementID')

        return render_template('login.html', DID=DID, WID=WID, EID=EID)
    
From the code block above, we provide an efficient approach to obtain the `DID`, `WID`, and `EID` of the Onshape document that you are hosting this Flask app in. As we will later show when integrating the app with Onshape OAuth in [section 4.1](#41-onshape-integration-through-oauth), accessing this page through an URL in a specific format will allow the code to automatically obtain those information through the document's URL. Then, the `render_template()` function will pass these three IDs to the form in `'login.html'`. Correspondinly, we need an HTML file named `login.html` with the following code (at a minimum): 

    <!doctype html>
    <form action="/config">
        <label for="appkey">API Key:</label>
        <input type="text" id="appkey" name="appkey" value={{APPKEY}}><br><br>
        <label for="secretkey">Secret Key:</label>
        <input type="text" id="secretkey" name="secretkey" value={{SECRETKEY}}><br><br>

        <label for="did">DocumentId:</label>
        <input type="text" id="did" name="did" value={{DID}}><br><br>
        <label for="wid">WorkspaceId:</label>
        <input type="text" id="wid" name="wid" value={{WID}}><br><br>
        <label for="eid">ElementId:</label>
        <input type="text" id="eid" name="eid" value={{EID}}><br><br>

        <input type="submit" value="Submit">
    </form>

In the HTML form shown above, we are essentially collecting information, including the user's API keys (`appkey` and `secretkey`) and the document's IDs (`DID`, `WID`, and `EID`). Notice how we are receiving the three document IDs from the `login()` function in this HTML. Depending on the specific API calls that you will be making, you may need to request for more information from the user (e.g., configuration, feature ID, etc.). You should also provide space for the users in this `/login` page to enter those information and pass them to the `/config` page that will be shown below. 

As shown in the second line of the HTML code above, the answers collected in this form will be passed to a `/config` page that configure the user with the provided API keys to the Onshape document specified: 

    @app.route('/config')
    def config(): 
        global appkey 
        global secretkey 
        global DID 
        global WID 
        global EID 

        appkey = request.args.get('appkey')
        secretkey = request.args.get('secretkey')
        DID = request.args.get('did')
        WID = request.args.get('wid')
        EID = request.args.get('eid')

        return render_template('config.html', return1=configure_onshape_client(appkey, secretkey, DID, WID, EID))

In the block of code above, we use the information collected from the `/login` page to run the function `configure_onshape_client()`, and then we print the return message of this function to the `return1` spot in the HTML file named `config.html`. Specifically, you can build your configuration function in the following structure: 

    from onshape_client.client import Client 

    def configure_onshape_client(access, secret, did, wid, eid): 
        base = 'https://cad.onshape.com' # Change if using enterprise account 
        client = Client(configuration={'base_url': base, 
                                       'access_key': access, 
                                       'secret_key': secret})
        try: 
            # Make your REST API call here 
        except: 
            return "Client not configured!" 

For `config.html`, the simplest design that you could build will be something like the following: 

    <!doctype html>
    <p>{{return1}}</p>
    <form action="/login"><input type="submit" name="back" value="Back"></form>

Such that, the return message from `configure_onshape_client()` will be put in the `return1` placeholder. Although not necessary, an additional button is also added above to allow the user to go back the `/login` page for potentially a new REST API call. 

As a result, we have presented the most basic structure of a Flask app that can make REST API calls to Onshape documents specifically. The main purpose of these two webpages are to allow the users to enter their credential information in a web form and then make API calls to the document that they specify. Some potential directions that you may build upon this simplest structure include: 
- Add additional HTML designs and CSS styling to the webpages to improve the user interface and allow more functions 
- Build additional webpage with Flask for additional functionality 

### 2.1. Security on API keys 
For the API keys (i.e., `appkey` and `secretkey`), there are generally two ways to handle them securely in an integrated Flask app: 
1. You may ask the users to enter their API keys every time they use your app, as shown in the example above. This was achieved through creating a form in the HTML file of your `/login` page for example. None of these information should be stored after the webpage is closed. 
2. If you are the only user of the app, or if this app is designed for the use of only one user, you can (or instruct others to) do the following: 
    - Save the API keys as a `.py` file in the project folder of this app locally. 
    - Add a function in your Flask code to automatically retrieve these API keys from the file. 
    - Add this file to the `.gitignore` file if you are sharing your code with others. Such that your API keys and others' API keys don't get shared to the public, and all that is required from other users is to add their own API keys to the folder after cloning your project. 
    - More details for the first two points can be found in the `API_Intro.md` [guide](https://github.com/PTC-Education/Onshape-Integration-Guides/blob/main/API_Intro.md) and the [Onshape-API-Snippets](https://github.com/PTC-Education/PTC-API-Playground/blob/main/Onshape_API_Snippets.ipynb). 

For example, we can ask the user to save their API keys in a file that is named `OnshapeAPIKeys.py` specifically. With the name of this file added to the `.gitignore` of the git project, this file will not be shared with others through any git commands. 

In the Flask script, we can write the following code before we define the app's web pages to search for the user's API keys: 

    import os 

    appkey = ''
    secretkey = ''

    for _, _, files in os.walk('.'): 
        if "OnshapeAPIKey.py" in files: 
            exec(open("OnshapeAPIKey.py").read())
            appkey = access 
            secretkey = secret 
            break 

Such that, the program will automatically search and temporarily store the API keys of the user. Consequently, we should also make the following changes to the `login()` function: 

    global appkey 
    global secretkey 

    if appkey: 
        APPKEY = appkey 
    else: 
        APPKEY = None 
    if secretkey: 
        SECRETKEY = secretkey 
    else: 
        SECRETKEY = None 

And we should also update the `return` command of the function: 

    return render_template('login.html', APPKEY=APPKEY, SECRETKEY=SECRETKEY, DID=DID, WID=WID, EID=EID)

With these code in place and the API keys correctly saved in the folder, the user should only need to click "submit" to log in to the document with their Onshape account once the app is integrated. A working example of this implementation can be found in [this repository](https://github.com/PTC-Education/Heat-Sink-Design). 


## 3. Configure Flask as HTTPS 
A full guide with detailed explanation on configuring your Flask application as HTTPS can be found in [this tutorial](https://blog.miguelgrinberg.com/post/running-your-flask-application-over-https). In this section, we only present the commands that will be sufficient for you to configure a Flask web application for Onshape integration. 

With the following commands through the terminal, you will create a self-signed certificate for the app with a validity period of 365 days: 

    $ pip install pyopenssl 
    $ openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem -keyout key.pem -days 365 

After commmitting the commands above, follow the pop-up steps to fill out the required information for the certificates. Then, you should see two files in your working directory: `cert.pem` and `key.pem`. 

Then, you need to add this newly created certificates to be a trusted certificate of your computer system. A general way of doing this will be: 
1. Open your Google Chrome browser and go to "Settings" under the three dots at the top right corner of your Chrome's home page. 
2. Under the "Privary and Security" section, find and click "Manage cerficates" in "Security". 
3. Follow the required process of your computer operating system to add `cert.pem` to be one of the trusted certificate of your computer. 

If you are using a computer with MacOS, steps above will open your KeyChain Access, which can also be access in your Launchpad. Then: 

1. Under "System" Keychains and "Certificates" Category, click the plus icon in the top left corner. 
2. Browse for the `cert.pem` certificate that you created and open it. 
3. You should find a new certificate with the name you specified during creation added to your keychains. 
4. Double click the added certificate and select "Always Trust" under "Trust". 
5. When you try to launch your Flask app with Google Chrome, the page will still give you a safety warning. But you can still proceed by clicking under "Details" or "Advanced". 

If you are running the Windows operating system, you may find this [guide](https://docs.vmware.com/en/VMware-Adapter-for-SAP-Landscape-Management/2.1.0/Installation-and-Administration-Guide-for-VLA-Administrators/GUID-D60F08AD-6E54-4959-A272-458D08B8B038.html) to be helpful. 

With the two files, forming your certificate, you should now be able to launch your Flask app with the following command line after properly exporting your Flask app and environment: 

    $ flask run --cert=cert.pem --key=key.pem 

## 4. Integrating to Onshape 
Once your Flask app is built locally in your computer, there are two ways that you can launch and integrate it to an Onshape document: 
1. Integrate the Flask app through OAuth, such that you can access the app through the native Onshape user interface (e.g., the right panel of the document similar to the BOM table in an Assembly, a tab at the bottom of the page similar to a Part Studio element, etc.). 
    - Advantage: once integrated, the app can be used by different Onshape documents that you have access to, without further setup required. It can also automatically retrieve the Onshape document's IDs through the URL for REST API calls). 
    - Disadvantage: the setup process is a little more complicated. 
2. Integrate the app through Link Tab, an public app in the Onshape App Store, such that you are essentially opening a webpage through an internal tab in your Onshape document. 
    - Advantage: easy setup and integration process. It links apps with specific purposes to specific Onshape documents that require them. 
    - Disadvantage: you need to go through the same setup process for every new Onshape document that you would like access to the app, and you will also need to type in the document-speicific IDs every time you use the app. 

### 4.1. Onshape integration through OAuth 
To integrate your self-built web application to Onshape, we can create an OAuth application in Onshape. More technical details on the OAuth technology can be found [here](https://onshape-public.github.io/docs/oauth/), and below are the steps necessary for a simple integration (e.g., with a self-built Flask app): 

1. Go to https://dev-portal.onshape.com and log in with your Onshape account. 
2. Under "OAuth applications" of the left pannel, click "Create new OAuth application" at the top right corner. 
3. Fill in all forms with information in the format as required by the description below. For our particular purpose, what you input for each of the blank does not matter except the permissions you grant. Below are some simplified guidance: 
    - Name: a name of the application for you to easily identify. 
    - Primary format: a Java-style format of the name (usually start with `com.` with no space in between) that cannot be changed once created; it does not have any impact to the function of your app. 
    - Summary: a brief summary for this application for your own referrence. 
    - Redirect URL & OAuth URL: these two URLs should, but do not have to, be related to your Flask app for our purposes; writing any URL that starts with `https` for these two blanks is fine and have no effect to the function of your app. These two URLs can also be changed after if necessary. 
    - Admin Team: select the team that will have editing permission to this OAuth application (recommended: No Team). 
    - Permissions: select the permission that you would like to grant your Flask app to have in your Onshape documents (recommended: allow read and write access to your documents and be careful with the rest, unless you are confident). 
4. Once you click "Create application", there should be a pop-up message with your OAuth client secret string. Simply save the key (with the trailing `=` signs) safely in your computer for future referrence, and you should then have an OAuth application set up for your account. 
5. In the OAuth application that you just created, go to the "Extensions" tab and click "Add extension". 
6. Most of the required fields of information for creating an extension should be self-explanatory. A few points to note: 
    - For the "Location" of the extension, we recommend placing it in the "Element right panel" for the first time. Then, you are free to place your extension at any location avaiable in your Onshape interface once you are familiar and confident with this integration process. 
    - If you selected "Element right panel" for "Location", you will then need to select a "Context". Depending on the functionalitt of your Flask app, you can choose the appropriate "Context" as required by your application. For example, the app may only work in Part Studios. 
    - For the "Action URL", this is where you should enter the URL of your Flask app when running locally in your web browser. In a general case, this should be `http://127.0.0.1:5000/`. However, you do not have to start with the home page of your app; you could also start your app with `http://127.0.0.1:5000/login/` for instance. 
    - You should also find an image for your "Icon" in SVG format online, so that you can distinguish this extension from Onshape's default components in the right panel of your document. 

**Note:** if you built your Flask app with the `/login` structure that is shown in [section 2](#2-connecting-to-onshape-documents-with-flask) above, you should set your "Action URL" to be the following: 

    https://127.0.0.1:5000/login?documentId={$documentId}&workspaceId={$workspaceId}&elementId={$elementId}

Such that, the `login()` function of the sample project in [section 2](#2-connecting-to-onshape-documents-with-flask) can automatically retrieve the necessary IDs of the Onshape document with functions such as `DID = request.args.get('documentID')`. Although you do not have to build your Flask app this way, this is a very efficient method to provide document access. 

7. After setting up the extension, go to the "Details" tab of the OAuth application, and click "Open story entry". Now you need to publish this Onshape application for you to implement into your Onshape documents. 
8. Again, none of these entries for opening a story entry make a significant impact to your app. Most of the required information is only for your own referrence. Simply choose and fill out all required information, as indicated by the red text on the page. 
    - **Note:** you can ignore the text at the top of the page that asks you to contact Onshape for API support. That is required only if you are trying to make your app public to the Onshape users community. 
9. After setting up your Onshape development portal with all the steps above, you can leave the developer portal and go to [Onshape App Store](https://appstore.onshape.com) with your Onshape account logged in. Alternatively, you can also find the App Store at the top right corner of your Onshape home page. 
10. Under the category you defined your app to be under when creating the story entry in step 8 above, you should be able to find your Flask app by scrolling down to the bottom of the page. Of course, you can also simply search for your app in the search bar. 
11. Once you locate your app, you should be able to subscribe to the app, which should be free, and it will be ready for use. 
12. You should now launch your Flask app in a local terminal with the certificates created in [section 3](#3-configure-flask-as-https). 

For example: 

    $ export FLASK_APP=name
    $ export FLASK_ENV=development 
    $ flask run --cert=cert.pem --key=key.pem 

13. Go back to your browser window and open the Onshape document that you integrated the app to. 

Now, you should have your Flask application readily integrated to the user interface of your Onshape document. Simply open or refresh your Onshape document, and you should be able to find it in the location and context that you selected in step 6 above. It may take some time and a few more refresh of the document right after the integration before you can see and access the app, but it should be ready for use every time you open an Onshape document in the future. 

### 4.2. Integration through Link Tab 
Link Tab is a public Onshape integrated cloud app, which is available in the [Onshape Appstore](https://appstore.onshape.com/apps/Utilities/DFE73AMQ42NPMVAEQBQVP56QGLCWJ4ALJUBEBLA=/description). You can also simply search for the app in the appstore. However, in order to use this app for our integration purpose, here are some of the steps that you will need to watch out for: 
1. Find and subscribe to Link Tab in the Onshape Appstore as you did for your OAuth application in steps 9 to 11 of [section 4.1](#41-onshape-integration-through-oauth) above. 
2. Before you can launch and use Link Tab properly in Onshape, you need to first make sure you are using Google Chrome as your browser and your Chrome is logged in with a Google account. 
3. When you open an Onshape document, click the plus sign at the bottom left corner and add a "Link Tab" under "Applications". This is also where you will normally add a Part Studio, Assembly, etc. 
4. You may need to grant a few access permissions for the first time you use Link Tab. Simply follow the prompts to complete this initial process. 
5. Copy and paste the same link that you would've put for the "Action URL" in step 6 of [section 4.1](#41-onshape-integration-through-oauth) above in the Link Tab text box and click "Link". 
    - Note that this link has to be a secure HTTPS URL for Link Tab to work, and that means you would have to go through the HTTPS configuration in [section 3](#3-configure-flask-as-https) for this integration method as well. 
    - It is also recommended to launch your Flask app in a browser window in Google Chrome (not internally in Onshape) first to grant security access to this "unsecure" webpage for the first time you test your Flask app. 
    - Different from integrating through OAuth, launching your Flask app in Link Tab will not be able to automatically retrieve the document's IDs as you could do in step 6 of [section 4.1](#41-onshape-integration-through-oauth); i.e., you have to manually type them in to make REST API calls. 
6. You should now launch your Flask app in a local terminal with the certificates created in [section 3](#3-configure-flask-as-https), the same process in step 12 of [section 4.1](#41-onshape-integration-through-oauth). 
7. Once the local host address of your Flask app is linked in Link Tab and the Flask app is running in a local terminal, you should be able to use your app as you would be able to in a browser, but now it's connected to your Onshape document. 
