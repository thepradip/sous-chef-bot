# Empowered by the Cloud: Building a Shopping List from a Recipe Screenshot

Have you ever had a recipe online that you wanted to cook, but no easy way to create a list for the ingredients you need?  During that critical step of trying to get the ingredients into a shopping list you could be prompted to create an account, download an app, go through a service connected with InstaCart, or having to resort to writing a list by hand.  After forgetting items at the store a few too many times I decided I needed a more consistent solution that didn't require me to remember another login or download yet another app on my phone.  Just getting a list of ingredients for a recipe wasn't enough.  I wanted my lists organized by where the ingredients are located in the store so I don't have to walk randomly around picking up items I missed on the first pass through the store.  

The result is Sous Chef Bot!  Sous Chef Bot takes a screenshot of a recipe's ingredients and extracts a list of ingredients organized by store location.  Sous Chef Bot works across mobile devices, recipe sources, and doesn't require any downloads or logins!  In this tutorial you'll create your own cloud-based bot that will convert your recipe screenshots into shopping lists.  During this tutorial you'll touch on a number of different topics including,

-	Setup a Twilio-enabled SMS/MMS number
-	Define role-based permissions with AWS
-	Use AWS Textract to get text from an image
-	Apply spaCy NLP models to extract entities from text and categorize text
-	Create a serverless Lambda function for our code
-	Build an API Gateway to access our Lambda function
-	Attach a Twilio phone number to our AWS API endpoint	

While building your own bot converting screenshots into lists is wicked cool one of the other takeaways I hope you get from this tutorial is how cloud-services can be empowering to create the applications and services you want.

Want to see what we're going to be building?  A version of the Sous Chef Bot is available on Facebook Messenger!  Visit the [Screenshot Your Groceries](https://t.co/SoX0R54R53?amp=1) page to learn more about the project and send your recipes through Facebook Messenger!

## Getting Setup with Twilio and AWS

To complete this project we’ll be using Twilio to send/receive text messages. You will need a SMS/MMS-enabled Twilio number to complete this tutorial.  If you don't have a Twilio account you can use this [referral link](www.twilio.com/referral/J5x4pK) and [this quickstart guide](https://www.twilio.com/docs/sms/quickstart/python) will help get you setup with your  Twilio account.  Once your account is setup you will need to purchase a phone number.  When purchasing your phone number make sure you are selecting one that is both SMS and MMS enabled.

![Shows how to filter but a number.](https://lh3.googleusercontent.com/UCjz0NXF98WbNYtTYe3ihEhyzwQ2B-8K2km_UmQbh4CmUrg8YEraPixxCr84CsAClTSyuLoWAtbS "Buying a Number")

**IMPORTANT**- One limitation to this tutorial is that sending / receiving MMS messages is only available for U.S. and Canadian phone numbers through Twilio.

You will also need an AWS account since that will be where the serverless environment we use in this tutorial will be deployed. For an introduction on AWS and details of the various services provided, AWS has a great [getting started](https://aws.amazon.com/getting-started/) section with detailed explanations.

After you've created your Twilio and AWS accounts let's get started setting up the AWS environment.

## Setting Up the AWS Environment

AWS is the serverless back-end our code will be deployed to for this tutorial.  We will create a Lambda function to perform all of our processing then setup an API to access that Lambda function that will be connected to our Twilio phone number.  The first part of setting up AWS is making sure we are providing the sufficient permissions for the bot to complete its task.  AWS utilizes role-based security to control what services users are allowed to access.  We want to setup our security so that our bot has access to the services required to complete its task and only those services.  In this tutorial there are two services our bot will need to access:

- **AWS Lambda** to handle the computation and store the main code we'll use.
- **Textract** to convert our image into text.

To setup these permissions log into your AWS account and navigate to the **IAM** under the **Services** menu.

![](https://lh3.googleusercontent.com/1GAiNAPjt9i_yUNbD0G65GOrATEjsoM4v0QVegQHjEwaG3jEhExH8AV3DlaUhpl-A8mf04lD2Kdj "Location of IAM in AWS Services Menu")

From the menu on the left select **Roles** and then click the blue **Create Role** button.  On the **Create Role** page select the **Lambda** since that is how we will be executing our code.  Once selected click the blue **Next: Permissions** button on the bottom right.

![](https://lh3.googleusercontent.com/m5JnyWbdgMgVtkGqMtPRuPwseUlsXWQ87G7YPIQwkZLwYtcWVuKPfpdDnk4-ClLwW-OPgnEp8Abj "Selecting Lambda for Role")

The different permissions we're going to grant our bot are defined in this section.  Using the search bar we can search for the policies we need to implement and click the checkbox next to them.  The policies we need to attach are:

- AmazonTextractFullAccess
- AWSLambdaExecute

After you have checked each of them click the blue **Next: Tags** button.  We won't do any tagging here so you can click the blue **Next: Review** button.  On the following screen we can name the role and review its contents.  Name the role "twilio-sous-chef-role" and provide a brief description in the box below.  When complete your screen should look like the image below.


![View of how the permissions should look when completed.](https://lh3.googleusercontent.com/-kg2YH9mNNYDDuLAOaJJClxU7Or00Opp03D8HWSXswdzQwHssmw09CrN7T_8ERMdX-KMRQIkOZ_h "Tutorial IAM Permissions")

If everything matches go ahead and click the blue **Create role** button.  You've successfully setup a permissions role!


## Writing the Code.

Now let's write the code for our function that will do all of the processing.  Create a new directory locally called `sous-chef-bot-tutorial`.  We will store all of the code and model artifacts in this directory for now.  Start by creating a file `lambda_handler.py` in the local directory.  A more detailed breakdown of the code is in the next section, for now you can paste the code block below into `lambda_handler.py`.

```python
import urllib  
import spacy  
import boto3  
import requests  
import operator  
from xml.sax.saxutils import escape  
from collections import defaultdict  
  
  
def main(event, context):  
    # Check if screenshot exists.  
    if "MediaUrl0" in event:  
        screenshot = requests.get(urllib.parse.unquote(event["MediaUrl0"]))  
    else:  
        msg = "Oops, no image was found.  Check that an image was sent and try again!"  
        return '<?xml version=\"1.0\" encoding=\"UTF-8\"?>' \  
               '<Response><Message>{}</Message></Response>'.format(msg)  
  
    # Extract text with Textract  
    textract = boto3.client('textract')  
  
    ocr_output = textract.detect_document_text(Document={'Bytes': screenshot.content})  
    screenshot_txt = " ".join([b["Text"] for b in ocr_output["Blocks"] if b["BlockType"] == "LINE"])  
  
    # Identify the recipe ingredients.  
    nlp = spacy.load("spacy_ner_ingredients")  
    recipe_doc = nlp(screenshot_txt)  
  
    identified_entities = []  
    for ent in recipe_doc.ents:  
        identified_entities.append(screenshot_txt[ent.start_char:ent.end_char])  
  
    # Determine store category of ingredients.  
    nlp_loc = spacy.load("spacy_ingredient_classifier")  
  
    store_categories = defaultdict(list)  
    for ingredient in identified_entities:  
        doc = nlp_loc(ingredient)  
  
        # Sort by the category scores.  
        category = sorted(doc.cats.items(), key=operator.itemgetter(1), reverse=True)[0][0]  
        category = category.replace("_", " AND ")  
        store_categories[category].append(ingredient.capitalize())  
  
    # Construct the return message.  
    msg = ""  
    category_order = ["PRODUCE", "MEAT AND FISH", "DAIRY", "OTHER", "SPICES"]  
    for category in category_order:  
        category_ingredients = store_categories.get(category, [])  
  
        if not category_ingredients:  
            continue  
  
        msg += category + "\n"  
        msg += "\n".join(category_ingredients)  
        msg += "\n\n"  
  
    return '<?xml version=\"1.0\" encoding=\"UTF-8\"?>' \  
           '<Response><Message>{}</Message></Response>'.format(escape(msg))
```


### Understanding the Code

Okay, that was a lot let's take a couple minutes to understand what each section of the code is doing.  The first part of the code is all the package dependencies.  Most of the dependencies are standard python packages.  We will handle making sure external packages our available to our Lambda function in a later step.

```python
import urllib  
import spacy  
import boto3  
import requests  
import operator  
from xml.sax.saxutils import escape  
from collections import defaultdict 
```

The `main` function is the entry point to our Lambda Function that is called when the Lambda function is triggered.  All of the data related to input will be in the `event` parameter.  The first step in the code is to check if an image was send in our message.  When an image is sent through Twilio a `MediaUrl` is passed with the message.  We check if that parameter is populated.  If there is a `MediaUrl` then we load the image by submitting a HTTP request, otherwise we send a response to the user stating that no image was sent. 

```python
def main(event, context):
	# Check if screenshot exists.  
	if "MediaUrl0" in event:  
	    screenshot = requests.get(urllib.parse.unquote(event["MediaUrl0"]))  
	else:  
	    msg = "Oops, no image was found.  Check that an image was sent and try again!"  
	    return '<?xml version=\"1.0\" encoding=\"UTF-8\"?>' \  
	        '<Response><Message>{}</Message></Response>'.format(msg)
```

Once the image has been loaded we need to extract all of the text from the image.  The process of extracted text from an image is called Optical Character Recognition (OCR).  We need to do some processing on the image, specifically we need to extract the text contained in the image.  To do this we'll use [AWS Textract](https://aws.amazon.com/textract/).  We can access Textract using `boto3`  and getting the text is just a couple of lines of code.  Textract will identify each line of text in the image.  Those lines are combined into a single string we call `screenshot_txt`.  

```python
# Extract text with Textract  
textract = boto3.client('textract')  
  
ocr_output = textract.detect_document_text(Document={'Bytes': screenshot.content})  
screenshot_txt = " ".join([b["Text"] for b in ocr_output["Blocks"] if b["BlockType"] == "LINE"])
```

So far we have converted our recipe screenshot image into a string of text.  From that text we need to extract all the text related to recipe ingredients.  This is where we'll apply our first NLP model.  Named Entity Recognition (NER) is the process of assigning labels to sub-texts that are related to an object.  NER is frequently used to identify people, places, organizations etc. with texts.  We will be using a NER model to identify recipe ingredients and applying it to `screenshot_txt`.  For this tutorial a pretrained NER model, `spacy_ner_ingredients`, is available in the GitHub repo, but if you're interested in learning more about NER or text processing in general check out [spaCy](https://spacy.io/) for more details.  The NER model is applied to `screenshot_txt` and the recipe ingredient entities are identified and put into a list, `identified_entities`.

```python
# Identify the recipe ingredients.  
nlp = spacy.load("spacy_ner_ingredients")  
recipe_doc = nlp(screenshot_txt)  
  
identified_entities = []  
for ent in recipe_doc.ents:  
	identified_entities.append(screenshot_txt[ent.start_char:ent.end_char])
```

When I'm grocery shopping I like to have my list organized by where in the store they are located.  A second NLP model to classify ingredients based on their general store location will be applied to all the ingredients we identified.  The classifier will label ingredients with one of the following: PRODUCE, MEAT_FISH, SPICES, DAIRY or OTHER.  Like the previous model there is a pretrained model, `spacy_ingredient_classifier`, available in the GitHub repo.  The classifier model is applied to each ingredient in `identified_entities` and will be assigned to the category with the highest model score.

A dictionary `store_categories` is created where the key is the store category and its value is a list of all the ingredients for the category.  As ingredients are classified they are added to the dictionary.

```python
# Determine store category of ingredients.  
nlp_loc = spacy.load("spacy_ingredient_classifier")  
  
store_categories = defaultdict(list)  
for ingredient in identified_entities:  
    doc = nlp_loc(ingredient)  
  
    # Sort by the category scores.  
    category = sorted(doc.cats.items(), key=operator.itemgetter(1), reverse=True)[0][0]  
  
    category = category.replace("_", " AND ")  
  
    store_categories[category].append(ingredient.capitalize())
```

The last step is to formulate our response message with the list of ingredients.  The output of our list will include headings for each store category with a list of ingredients under each category.  By iterating through `store_categories` a string `msg` is created as a formatted list.  Once `msg` has been constructed it is added to the TwiML XML response.

```python
# Construct the return message.  
msg = ""  
category_order = ["PRODUCE", "MEAT AND FISH", "DAIRY", "OTHER", "SPICES"]  
for category in category_order:  
    category_ingredients = store_categories.get(category, [])  
  
    if not category_ingredients:  
        continue  
  
    msg += category + "\n"  
    msg += "\n".join(category_ingredients)  
    msg += "\n\n"  
  
return '<?xml version=\"1.0\" encoding=\"UTF-8\"?>' \  
       '<Response><Message>{}</Message></Response>'.format(escape(msg))
```

## Downloading the Modeling Files

In the code two modeling files are referenced, `spacy_ner_ingredients` to perform NER on the raw Textract output and `spacy_ingredient_classifier` to categorize the ingredients into categories.  We will be using pre-trained models to fill in.  The pre-trained models are available in the GitHub repo, download them to the `sous-chef-bot-tutorial` directory you put the `lambda_handler.py` function into.  The directory should look similar to this:

![Example of what local directory should look like with artifacts.](https://lh3.googleusercontent.com/cLLsbKazEfFcJUOMtSBH4HXNOMTLPdF94anJGr9jFtM322_xEd8c4qC8gagIdeWcVLIAYAUA6C8N "Example Local Directory")

## Creating the Deployment Package.

Right now the code exists locally.  The code needs to be prepared so that it can be uploaded to a Lambda function.  To upload code to the Lambda function we will create the `lambda_handler.py` file along with the two NLP models need to be packaged into a zip archive that we can upload to our Lambda function.  From the command line navigate to the `sous-chef-bot-tutorial` directory you put the code and model files into and execute the bash command below.

```bash
zip sous-chef-bot-lambda-package.zip -r spacy_ner_ingredients -r spacy_ingredient_classifier lambda_handler.py
```

The command will create a new file, `sous-chef-bot-lambda-package.zip`, containing the code file and the modeling files.

Put a pin in the location of the zip file because we'll need that in the next step when we setup the Lambda function.  We're done with all the code for our function!  What's left is to finish setting up the infrastructure within AWS and connecting it to Twilio.  Next, we'll setup the Lambda function we will deploy the just created code package to!

## Setting up the Lambda Function

AWS Lambda functions are a serverless method to deploy code.  Lambda functions allow you to deploy code to the cloud and only pay for the computing resources you use when the function is executing.  Since you'll likely only need to generate shopping lists from a recipe screenshot once a day, Lambda is a great resource to cheaply and efficiently deploy our bot and not have to pay for idle servers.  With that, let's get our Lambda setup.

From the AWS Services menu select **Lambda** and click the orange **Create  Function** button on the next page.

![Shows where to find the Lambda function option in the AWS Services menu.](https://lh3.googleusercontent.com/EZoVIcAkLGEzywvY7uawZjaC8FaVReOwZuXdTElmojKlB4Hfrnm2xlgdaV4yl8dTs9O8iuK1OB-J "Locating Lambda")

Let's setup our Lambda function:

- In the **Function name** field enter "sous-chef-bot-function".  
- From the **Runtime** drop down select **Python 3.7**.  
- Click **Choose or create and execution role** and then select the **Use an existing role** option.  
- From the drop down that appears find the role you created earlier ("twilio-sous-chef-role") and select that option.  

When you're done the screen should look like this and click **Create function**.

![What the setup should look like for the AWS Lambda function.](https://lh3.googleusercontent.com/4SLJfBPal3iZEMvnqxO48J4bHxcOGroXpIdqzslZ2F7AzE_Vy7RGmKhtiG3aWCAjPto0Z9_53Vs5 "Setting up Lambda")

On the next screen you'll see the layout of our Lambda function.  Right now our Lambda function is blank.  Before uploading our code to the Lambda function we need to make sure some of our external dependencies are available to our function.  By default AWS Lambda functions with a Python runtime only include core Python code.  If your function has additional dependencies you have to manually add those dependencies to the function.  To help with this process AWS has introduced the concept of [Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) to manage those dependencies.  For this tutorial we have two external dependencies that have to be incorporated into our Lambda function:

- **boto3** a way of accessing AWS services via an API.  For our application we need a specific version of `boto3` so we can access the Textract service
- **spaCy** for the NLP models

We will be creating Layers for each of these dependencies so that these external dependencies are available when the function is executed.  A limitation of Layers is they cannot exceed 250MB.  The spaCy library by itself is large and exceeds that limitation.  To get spaCy into the Layer all of the non-English libraries were removed from the library.  Another feature of Layers is they are made to allow sharing between teams.  Since sharing is caring, instead of creating these layers from scratch we're going to use layers that have been pre-built that have already been editted to fit within the size limitations of Lambda.  If you want to learn more Layers I've written a little about them [elsewhere](https://dev.to/vealkind/getting-started-with-aws-lambda-layers-4ipk)!

To add a Layer to our Lambda function click the **Layers** button on the Lambda function screen.

![Where to find the Layers Button in AWS Lambda](https://lh3.googleusercontent.com/kmL1O6o7M4OpNJBA36pK13NcjvT1BgjW7c4ydSOqVvkobmmuxbYmP1lcu9Mt-IDdBKX75iM1HlnH "Layers Button")
Scroll to the bottom of the page and click **Add a layer**.  Here is where the sharing comes into play.  We're going to use an existing Layer that has already been created for the spaCy dependency where all the non-English corpora so that spaCy fits comfortably into the size limitations of Lambda Layers for our function.  We're going to do that with the Amazon Resource Name (ARN), so select the **Provide a layer version ARN** option.  Enter the ARN below for the spaCy layer being sure to replace  `<region name>` with the name of the AWS region you will be deploying your Lambda function to.

```
arn:aws:lambda:<region name>:565208013763:layer:spacy-2-1-8-en-only:1
```

If you're deploying to the `us-east-1` region the ARN should look like the following,

![What the Lambda Layer addition should look like.](https://lh3.googleusercontent.com/usI1IZahT3swY5dNa5pFmHjvqCDJMZ51FrStG49sAocHXt3CiTJGDTSweNL0QPWWLlAJM60YeRHe "Adding Layer Version")

A second layer needs to be created for a specific version of `boto3` to allow access to AWS Textract.  Repeat the same process to create a layer using the ARN below being sure to use the appropriate region name:

```
arn:aws:lambda:<region-name>:565208013763:layer:mvielkind-boto3-1-9-172:1
```

After you've created the Lambda Layers go ahead and save your progress with the **Save** button in the upper right corner.

Remember the zip file we created earlier?  You're going to need that here as all of the core function code is in that zip file, which needs to be updated to the Lambda function.  On the Lambda page click **twilio-sous-chef-function** above where you selected **Layers** then  scroll down to the **Function code** section.  In the "Code entry type" dropdown select "Upload a .zip file", click the "Upload" button and select the `sous-chef-bot-lambda-package.zip` file you created.  An entry point to the code needs to be established so that Lambda knows where code execution should begin.  To the right in the "Handler" text box replace the existing text with `lambda_handler.main`.  This tells our Lambda function to execute `lambda_handler.main` when the function is initialized.  When you're done this section should look like the following:

![Displays what the "Function code" section of the Lambda setup should look like when completed.](https://lh3.googleusercontent.com/ZUy2hiJPeVdiz4ckcFJZvXxtZH3AyDTcTXbif88Iv18hzbjuCKZ4_sZnNEOoYJ-nV0kZuNm8q3BB "Lambda Function Code Setup")

Two last configurations setting up our Lambda function before moving on.  By default Lambda functions will timeout after 3 seconds.  Our function is doing a lot with Textract and the NLP so we need to increase our timeout to allow the computation to complete.  Scroll down a little further on the Lambda page.  In the "Basic settings" section there is a "Timeout" option.  Increase the number of seconds to 10.  Lambda functions by default have 128MB of memory available.  To improve processing speed double the memory to be 256MB.

![Displays where to set the Lambda timeout](https://lh3.googleusercontent.com/N-L5YER6GgH5rbxAUFO-FoFNwnU1ytdcq1sfkowXuH7VGvv8mnCXr___ZttIYb6bPOkiz6VwH22q "Increase Lambda timeout")
When you're done click the orange **Save** button in the upper right.  Just like that our code has been deployed to the cloud!  The next step is to setup the API Gateway so that our number can call the function we just created.

## Setting Up the API Gateway

We're almost done setting up the AWS components we will need.  The last step with is setting up the API Gateway that will connect your Twilio number to the Lambda function we just created.  The API will take the input from our Twilio message, send it to our Lambda function, and then pass the return message back to a SMS message.  

From the AWS Console select **API Gateway** from the list of services.  Click the **Create API** button.  On the screen to create a new API fill-in the API name with a name for the API.  Let's call our API, "twilio-sms-recipe".  Everything else on the page can be left as default settings like the image below and you can click the blue **Create API** button.

![](https://lh3.googleusercontent.com/dulGvt0JcpLuqUHOi7pDsxvZzOD95Sg7rEh5Kpat_0Z3c-VwEs7h1fa2cWw0GhqPR-wgx5mJDZCl "Create API page")

Next, we need to add a new resource for our API.  Select **Create Resource** from the **Actions** menu.  Name the resource "send-recipe", and click the blue **Create Resource** button.  In the list of **Resources** panel you should see the `/send-recipe` resource that was created.  We need to add a method to that resource.  From **Actions** select **Create Method**.  Below `/send-recipe` a dialog box will appear.  From the drop down choose `POST` and click the check mark next to the drop down like the picture below.

![enter image description here](https://lh3.googleusercontent.com/Lq26R9HuHylVOZU_p46JZ20nwykJt_HX3pMuDWv6bF2NV4V0wnf96c6hBG2XNVFjrgRT4FkeQIDA "Adding method to the API.")

After clicking the check mark the panel on the right will change to help setup the `POST` method.  In this panel we will attach the `POST` method to the Lambda function we created.  In the dialog for the **Lambda Function** input the name of the Lambda function we created earlier (it was "twilio-sous-chef-fuction").  Then click the blue **Save** button and click **Ok** in the dialog that appears.

The next page shows how data flows through the API Gateway.  There our three basic parts to the API Gateway:

- **Request**: preparing data sent from the source for our Lambda function.
- **Integration**: the Lambda function our API Gateway will trigger.
- **Response**: preparing output data to be sent back to the source.

We need to make a couple transformations to data in the **Request** and **Response** steps of our API.  The input data from Twilio is in XML format and needs to be mapped into a JSON format to be utilized by the API Gateway.  To do this, select the **Integration Request** block.  Scroll to the **Mapping Templates** section.  Select the option "When there are no templates defined (recommended)" and click **Add mapping template**.  In the dialog that appears type `application/x-www-form-urlencoded` and click the check mark.   A box will appear to define a template below.  Paste the following code into the box and click the **Save** button below.

```
#set($httpPost = $input.path('$').split("&"))
{
#foreach( $kvPair in $httpPost )
 #set($kvTokenised = $kvPair.split("="))
 #if( $kvTokenised.size() > 1 )
   "$kvTokenised[0]" : "$kvTokenised[1]"#if( $foreach.hasNext ),#end
 #else
   "$kvTokenised[0]" : ""#if( $foreach.hasNext ),#end
 #end
#end
}
```

 When you're done the **Mapping Templates** section should look like this,

![enter image description here](https://lh3.googleusercontent.com/1C_o4jLrS8vG458JWpR_ef4Wz_N7ijQkSQP0EoAgB90-mJ-vkYWmeX7qnQjDT12O52J07EiHLlyh "Method Execution- Mapping Templates Configuration")

Likewise, we need to convert the JSON response we receive from the API Gateway to be XML that is used by Twilio.  Go back to the **Method Execution** for your API and this time click on the **Integration Response** section.  We need to update the mapping templates for our response from the Lambda function.  Expand the row where **Method response status** is 200 and scroll down to the **Mapping Templates** section.  You'll see that `application/json` is provided in the table.  Remove that row by clicking the `-` sign, click **Add mapping template** and input `application/xml`.  After you click the check mark define the template by inputting the following,

```
#set($inputRoot = $input.path('$')) 
$inputRoot
```

Click **Save**.  When done the **Mapping Templates** section of your **Integration Response** should look like this:

![enter image description here](https://lh3.googleusercontent.com/cOGtXa5Y9cTE2DNfpcvij22wXQvATy07ADly3T4P0prRWbpNoDgGHg30XXNiReJZwEQahwnJbUsB "Integration Response Mapping Templates")

The last edit We have to make one more edit to our API, is in the **Method Response** section.  Expand the section where the HTTP Status is 200.  In the **Response Body for 200** section remove the current `application/json` entry, click **Add Response Model** and enter `application/xml` in the text box, select `Empty` from the **Models** drop down and click the check mark.

The API is ready for deployment!  Click the **Actions** button and choose the **Deploy API** option.  AWS allows you to define different stages of deployment for your API, for this tutorial choose **[New Stage]** and write "prod" as the **Stage Name** and then click **Deploy**.

![enter image description here](https://lh3.googleusercontent.com/ehOB7vJDqREBSNvSt1vZJiLQ8lKHnKCDHKWOmRuafwHRNQ4_Y80OUUqvGJnnz-dOTVbnDRC-jghg)

Our API is deployed!  In the blue box is the "Invoke URL".  Copy that URL as we will need it to connect our Twilio number to the API Gateway we just created.

![Shows where the find the invoke URL.](https://lh3.googleusercontent.com/E8V51pewzB56GTfp6-5-EuRApPzxzkqHIEoqMWS1-_tYooC0N78HSW1dxkxr8QVPBQe6-vTGZG87 "Invoke URL Location")

## Connecting Twilio with the API Gateway

All that's left is to connect the API Gateway to the Twilio number you setup at the beginning.  Log in to your Twilio account and navigate to the SMS number you are using for the tutorial.  Scroll to the bottom of the page and find the **Messaging** section.  In this section you can define actions for when messages are received by your Twilio number.  For this tutorial a webhook is going to be configured to call the API Gateway that was just setup.  Underneath **A Message Comes In** make sure **Webhook** is selected from the drop down and paste in the URL for the API Gateway making sure to add `/send-recipe` at the end of the URL so it's pointing to the correct resource.  When you're done the URL should look like this:

```bash
https://<api-id>.execute-api.<region>.amazonaws.com/prod/send-recipe
```

![Where to configure the Webhook in Twilio](https://lh3.googleusercontent.com/wsj97YhGEg1oAM0Budh91DysFY2w7_Wy63P0OC5tHZQ-S894zR1dq0_Gw_hbf3Cpj4nsUfPQ-tar "Twilio Webhook Config")

Make sure you're using a `HTTP POST` method and click **Save** at the bottom of the page.  And we're done!  Let's see our bot in action!

## Your Own Sous Chef Bot in Action!

Text a recipe screenshot to the Twilio number you purchased and see what happens!  In a few seconds you should get a grocery list organized by item location.  A quick note the first time you submit a screenshot there could be a lengthy response.  You might have to submit your request a second time.  This is because AWS needs to provision services for your function, which can take some time at first or for functions that haven't been used in a while.

## Next Steps.

Congratulations!  Now that you've built the bot let's see what else you can do.  If you have an iPhone you can use Apple Shortcuts to create a [one-click workflow on your phone's home screen](https://dev.to/vealkind/integrating-twilio-bots-into-mobile-workflows-with-apple-shortcuts-1d3a) to launch your bot and automatically create a new note with the shopping ingredients.  

