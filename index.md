# Location-aware Alexa Skills with HERE Location Services


## Introduction

*Duration is 3 min*

This codelab will walk you the steps to create a location-aware Alexa Skill.

At the end of this tutorial, you will have created an Alexa Skill in which you can search for nearby places. For example:

> Alexa, show me nearby Sushi in Seattle.

> We found a nearby sushi place near Seattle! Sushi Kanpai is 0.3 miles away and is located at 900 8th Ave Seattle, WA 98104

This codelab demonstrates the use of two HERE Location Services APIs: [Geocoder](https://developer.here.com/documentation/geocoder/topics/quick-start-geocode.html) and [Places](https://developer.here.com/documentation/places/topics/quick-start-find-text-string.html).

### __Use cases for places search__ ###

As an Alexa application developer, you may want to integrate a places search if you are building:
* A restaurant reviewing application. Users can search for nearby restaurants and then write reviews about the restaurant.
* A ridesharing application. A place can serve as an origin or destination point for the trip.
* A tour guide application. The virtual tour guide can suggest landmarks to visit in the user's area.

### __What is the Geocoder API?__

The Geocoder REST API enables developers to convert street addresses to geo-coordinates and vice-versa with forward geocoding, including landmarks, and reverse geocoding.

The HERE Geocoder API is a REST API that allows you to:

* Obtain coordinates for addresses
* Obtain addresses or administrative areas for locations
* Obtain coordinates for known landmarks.

### __What is the Places API?__

The Places (Search) API is a REST API that allows you to build applications where users can search for places and receive detailed information about selected places.


### __What you'll need__

* a [HERE Developer](https://aws.amazon.com/marketplace/pp/B07JPLG9SR) account from the AWS Marketplace for leveraging location APIs
* an [Amazon Web Services](https://aws.amazon.com/) account to host your project code
* an [Amazon Alexa Developer](https://developer.amazon.com/alexa) account to configure your Alexa Skill

## Configuring the Alexa Skill

*Duration is 12 min*

Navigate to the [Alexa Developer Portal](https://developer.amazon.com/alexa/console/). If you don't already have an Amazon Developer account, go ahead and create one. If you already have an account, go ahead and sign in.

Once you are inside of the Alexa Skills Console, navigate to the `Skills` page, located in a dropdown under `Your Alexa Consoles`. From there, click `Create Skill`.

![creating a new skill](/img/0.png)

Give your skill a name. I named mine *HERE Places on Alexa*. Feel free to choose your own name. Choose a custom model and start from scratch.

You should now be presented with your skill's console page:
![console page](/img/1.png)

On the left hand sidebar, you should see a button that says `JSON Editor`. Go ahead and click that. Inside the `JSON Editor` is where we will configure invocations, intents, and slots for our skill.

* An **invocation** is a phrase one can say to launch the Alexa Skill. In our application, we'll be using *HERE Maps* as the invocation.
* An **intent** is a phrase one can say to launch a specific function or action within the Alexa Skill. For example, one of the intents we'll be using in this skill will be the phrase *show me sushi near Seattle*.
* A **slot** is a parameter within an intent. For example, in the intent phrase *show me sushi near Seattle*, two parameters we would like to parse would be *sushi* and *Seattle*. *Sushi* is the place category we'll be using for our search, while *Seattle* is the location we'll be using for our search. Slots are used to isolate certain values out of phrases uttered by the user.

Go ahead an paste the following JSON within the text field:

```json
{
    "interactionModel": {
        "languageModel": {
            "invocationName": "here places",
            "intents": [
                {
                    "name": "SearchIntent",
                    "slots": [
                        {
                            "name": "Item",
                            "type": "AMAZON.Food"
                        },
                        {
                            "name": "Location",
                            "type": "AMAZON.AdministrativeArea"
                        }
                    ],
                    "samples": [
                        "show me nearby {Item} in {Location}",
                        "show me {Item} in {Location}",
                        "where is {Item} in {Location}",
                        "find me {Item} near {Location}"
                    ]
                },
                {
                    "name": "AMAZON.RepeatIntent",
                    "samples": []
                },
                {
                    "name": "AMAZON.HelpIntent",
                    "samples": []
                },
                {
                    "name": "AMAZON.StopIntent",
                    "samples": []
                },
                {
                    "name": "AMAZON.CancelIntent",
                    "samples": []
                }
            ],
            "types": []
        }
    }
}
```

Pasting this JSON will also go ahead and automatically populate the other values in the sidebar. Go ahead and click both the `Save Model` and `Build Models` buttons at the top of the screen.

Your screen should now look something like this:

![pasted json](/img/2.png)

## Configuring the Lambda Function

*Duration is 10 min*

So far, we've created the Alexa Skill and configured some basic properties. We'll need somewhere to host the code that controls our skill's logic.

We'll be employing the help of [AWS Lambda functions](https://aws.amazon.com/lambda/) for this task.

Navigate to [Amazon Web Services](http://aws.amazon.com/) website and sign into the console. If you don't already have an account, go ahead and create one.

In the AWS services search field, search for Lambda. Click the orange `Create function` button to create a new Lambda function.

When creating the Lambda function, select the `AWS Serverless Application Repository` radio button and search for `alexa-skills-kit-nodejs-howtoskill`.

![creating lambda](/img/3.png)

Name your application and then click `Deploy`. The function may take about 20-30 seconds to deploy.

Once the resource has been deployed (you can confirm this by seeing a `CREATE_COMPLETE` notice), click on the resource to be taken to the code editor page.

## Adding HERE Location Services to the Skills

*Duration is 25 min*

In the previous step, we created an AWS Lambda function to host and execute our skill's logic.

Before we start writing code, let's head over to the [AWS Marketplace](https://aws.amazon.com/marketplace/pp/B07JPLG9SR) to grab our application's keys. Sign up and create a new project inside the marketplace.

Under the JavaScript/REST section, click `Generate App ID and App Code`. Grab and save these keys, we'll be using them shortly.

Let's start modifying the skill's code.
![editing code](/img/4.png)

The code that's already included comes from an example skill. This code is a good skeleton for what we'll build, but we'll want to modify some parts.

First things first, let's include our HERE keys and helper conversion variable.

```javascript
const here = {
   id: 'YOUR-HERE-ID',
   code: 'YOUR-HERE-CODE'
}

const metersToMiles = 0.00062137
```

Additionally, we'll be making HTTP requests, so let's include a helper method that will assist us with that.

```javascript
const https = require('https');

function makeRequest(options) {
   return new Promise(((resolve, reject) => {
      const request = https.request(options, (response) => {
         let data = '';
         response.on('data', (chunk) => {
            data += chunk;
         });
         response.on('end', () => {
            resolve(JSON.parse(data));
         });
         response.on('error', (error) => {
            reject(error);
         });
      });
      request.on('error', function(error) {
         reject(error);
      });
      request.end();
   }));
}
```
The `https` module is already included, so no extra steps to install it are necessary.

Next, let's simplify the `LaunchRequestHandler` to fit our needs.

```javascript
const LaunchRequestHandler = {
  canHandle(handlerInput) {
    return handlerInput.requestEnvelope.request.type === 'LaunchRequest';
  },
  handle(handlerInput) {
    const welcomeOutput = 'Welcome to HERE Places. Trying searching for a place category and a location. For example, try "show me sushi in Seattle"';
    return handlerInput.responseBuilder
      .speak(welcomeOutput)
      .reprompt(welcomeOutput)
      .getResponse();
  },
};
```

Now, let's get to the fun part: implementing HERE Location Services. Replace the `RecipeHandler` with this new `SearchHandler`, which handles our search logic.

```javascript
const SearchHandler = {
   canHandle(handlerInput) {
      return handlerInput.requestEnvelope.request.type === 'IntentRequest' &&
         handlerInput.requestEnvelope.request.intent.name === 'SearchIntent';
   },
   handle(handlerInput) {
      const query = handlerInput.requestEnvelope.request.intent.slots.Item.value;
      const location = handlerInput.requestEnvelope.request.intent.slots.Location.value;

      const geocodeOptions = {
         host: 'geocoder.api.here.com',
         path: `/6.2/geocode.json?app_id=${here.id}&app_code=${here.code}&searchtext=${location}`,
         method: 'GET'
      };

      const errorMessage = `We didn't find any ${query} in ${location}`;

      return new Promise((resolve, reject) => {
         makeRequest(geocodeOptions).then((geocodeResponse) => {
            const coordinates = geocodeResponse.Response.View[0].Result[0].Location.DisplayPosition;

            const placesOptions = {
              host: 'places.api.here.com',
              path: `/places/v1/discover/search?at=${coordinates.Latitude},${coordinates.Longitude}&q=${query.replace(/ /g, '+')}&app_id=${here.id}&tf=plain&app_code=${here.code}`,
              method: 'GET'
            };

            makeRequest(placesOptions).then((placeResponse) => {
              const places = placeResponse.results.items;
              const examplePlace = places[0];
              if (examplePlace) {
                const successOutput = `We found a nearby ${query} place near ${location}! ${examplePlace.title} is ${(examplePlace.distance * metersToMiles).toFixed(1)} miles away and is located at ${examplePlace.vicinity}`;
                resolve(handlerInput.responseBuilder.speak(successOutput).getResponse());
              } else {
                resolve(handlerInput.responseBuilder.speak(errorMessage).getResponse());
              }

            }).catch((error) => {
              resolve(handlerInput.responseBuilder.speak(errorMessage).getResponse());
            });

         }).catch((error) => {
            resolve(handlerInput.responseBuilder.speak(errorMessage).getResponse());
         });
      });
   }
};
```

What's going on here?
* First, we parse the intent to find the query (place category) and the location.
* Next, we make a request to the HERE Geocoder API to translate the location string into actionable coordinates so we can use them in other services like routing, places, etc.
* We then take the coordinates and the query and perform a places search.
* The places search returns an array of results. The Alexa Skill presents information on the first result, including the name, distance, and address of the place.

You'll also want to modify the `export` at the end of the code to include the newly added `SearchHandler`.

```javascript
exports.handler = skillBuilder
  .addRequestHandlers(
    LaunchRequestHandler,
    SearchHandler,
    HelpHandler,
    RepeatHandler,
    ExitHandler,
    SessionEndedRequestHandler
  )
  .addErrorHandlers(ErrorHandler)
  .lambda();
```


The code template has a few more lines that can be modified. Take a look at the Appendix section of this codelab for the entire Lambda code block.

Make sure to save the modified code.

## Connect Lambda Function back to Alexa Skill

*Duration is 8 min*

Now that our code is complete, the next step is to connect it back to the Alexa Skill.

At the top of the Lambda function editor page, look for the string next to `ARN`.

![finding ARN](/img/5.png)

Copy this string and navigate back to the Alexa Developer Console.

Click on the `Endpoints` button on the left sidebar. Inside of `Default Region` field, go ahead and paste your copied ARN string.

![pasting ARN](/img/6.png)

## Testing the Alexa Skill

*Duration is 8 min*

Almost done! So far you've configured the Alexa Skill, setup a Lambda function to control the skill's logic, and connected the Alexa Skill to the Lambda. Now, let's test the skill to make sure it is functioning properly.

In the Alexa Developer Console, click on the `Test` button on the top header. In this view, you can test your newly created skill.

To begin testing, click the switch that says `Test is disabled for this skill`. You are now able to test the skill by typing into the field or talking through your computer's microphone.

![testing skill](/img/7.png)

Here is an example interaction:

> open here places

> Welcome to HERE Places. Trying searching for a place category and a location. For example, try "show me sushi in Seattle"

> show me sushi in Seattle

> We found a nearby sushi place near Seattle! Sushi Kanpai is 0.3 miles away and is located at 900 8th Ave Seattle, WA 98104

If you'd like to deploy the skill to an Echo device, take a look at [this handy guide](https://developer.amazon.com/docs/devconsole/test-your-skill.html).


## Review

*Duration is 1 min*


Congratulations! You've successfully create a location-aware Alex Skill!
In this tutorial, you've learned how to:
* configure Alexa Skills
* create AWS Lambda functions
* write code to enable Alexa Skills
* make calls the the HERE Location Services Geocoder and Places APIs
* test Alexa Skills

This is just a basic example of what can be done with HERE Location Services, take a look at the [HERE Developer blog](https://developer.here.com/blog) to see more examples of creative and useful applications of HERE Location Services.


## Appendix: Complete Lambda Function Code block

```javascript
const Alexa = require('ask-sdk-core');
const https = require('https');

const metersToMiles = 0.00062137

const here = {
   id: 'YOUR-HERE-ID',
   code: 'YOUR-HERE-CODE'
}

function makeRequest(options) {
   return new Promise(((resolve, reject) => {
      const request = https.request(options, (response) => {
         let data = '';
         response.on('data', (chunk) => {
            data += chunk;
         });
         response.on('end', () => {
            resolve(JSON.parse(data));
         });
         response.on('error', (error) => {
            reject(error);
         });
      });
      request.on('error', function(error) {
         reject(error);
      });
      request.end();
   }));
}

/* INTENT HANDLERS */
const LaunchRequestHandler = {
  canHandle(handlerInput) {
    return handlerInput.requestEnvelope.request.type === 'LaunchRequest';
  },
  handle(handlerInput) {

    const welcomeOutput = 'Welcome to HERE Places. Trying searching for a place category and a location. For example, try "show me sushi in seattle"';
    return handlerInput.responseBuilder
      .speak(welcomeOutput)
      .reprompt(welcomeOutput)
      .getResponse();
  },
};

const SearchHandler = {
   canHandle(handlerInput) {
      return handlerInput.requestEnvelope.request.type === 'IntentRequest' &&
         handlerInput.requestEnvelope.request.intent.name === 'SearchIntent';
   },
   handle(handlerInput) {

      const query = handlerInput.requestEnvelope.request.intent.slots.Item.value;
      const location = handlerInput.requestEnvelope.request.intent.slots.Location.value;

      const geocodeOptions = {
         host: 'geocoder.api.here.com',
         path: `/6.2/geocode.json?app_id=${here.id}&app_code=${here.code}&searchtext=${location}`,
         method: 'GET'
      };

      const errorMessage = `We didn't find any ${query} in ${location}`;

      return new Promise((resolve, reject) => {
         makeRequest(geocodeOptions).then((geocodeResponse) => {
            const coordinates = geocodeResponse.Response.View[0].Result[0].Location.DisplayPosition;

            const placesOptions = {
              host: 'places.api.here.com',
              path: `/places/v1/discover/search?at=${coordinates.Latitude},${coordinates.Longitude}&q=${query.replace(/ /g, '+')}&app_id=${here.id}&tf=plain&app_code=${here.code}`,
              method: 'GET'
            };

            makeRequest(placesOptions).then((placeResponse) => {
              const places = placeResponse.results.items;
              const examplePlace = places[0];
              if (examplePlace) {
                const successOutput = `We found a nearby ${query} place near ${location}! ${examplePlace.title} is ${(examplePlace.distance * metersToMiles).toFixed(1)} miles away and is located at ${examplePlace.vicinity}`;
                resolve(handlerInput.responseBuilder.speak(successOutput).getResponse());
              } else {
                resolve(handlerInput.responseBuilder.speak(errorMessage).getResponse());
              }

            }).catch((error) => {
              resolve(handlerInput.responseBuilder.speak(errorMessage).getResponse());
            });

         }).catch((error) => {
            resolve(handlerInput.responseBuilder.speak(errorMessage).getResponse());
         });
      });
   }
};

const HelpHandler = {
  canHandle(handlerInput) {
    return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'AMAZON.HelpIntent';
  },
  handle(handlerInput) {
    const helpOutput = 'Trying giving us a query like "Show me sushi in Seattle"'
    return handlerInput.responseBuilder
      .speak(helpOutput)
      .reprompt(helpOutput)
      .getResponse();
  },
};

const RepeatHandler = {
  canHandle(handlerInput) {
    return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && handlerInput.requestEnvelope.request.intent.name === 'AMAZON.RepeatIntent';
  },
  handle(handlerInput) {
    const sessionAttributes = handlerInput.attributesManager.getSessionAttributes();

    return handlerInput.responseBuilder
      .speak(sessionAttributes.speakOutput)
      .reprompt(sessionAttributes.repromptSpeech)
      .getResponse();
  },
};

const ExitHandler = {
  canHandle(handlerInput) {
    return handlerInput.requestEnvelope.request.type === 'IntentRequest'
            && (handlerInput.requestEnvelope.request.intent.name === 'AMAZON.StopIntent'
                || handlerInput.requestEnvelope.request.intent.name === 'AMAZON.CancelIntent');
  },
  handle(handlerInput) {
    const requestAttributes = handlerInput.attributesManager.getRequestAttributes();
    const speakOutput = requestAttributes.t('STOP_MESSAGE', requestAttributes.t('SKILL_NAME'));

    return handlerInput.responseBuilder
      .speak(speakOutput)
      .getResponse();
  },
};

const SessionEndedRequestHandler = {
  canHandle(handlerInput) {
    console.log("Inside SessionEndedRequestHandler");
    return handlerInput.requestEnvelope.request.type === 'SessionEndedRequest';
  },
  handle(handlerInput) {
    console.log(`Session ended with reason: ${JSON.stringify(handlerInput.requestEnvelope)}`);
    return handlerInput.responseBuilder.getResponse();
  },
};


const ErrorHandler = {
  canHandle() {
    return true;
  },
  handle(handlerInput, error) {
    console.log(`Error handled: ${error.message}`);

    return handlerInput.responseBuilder
      .speak('Sorry, I can\'t understand the command. Please say again.')
      .reprompt('Sorry, I can\'t understand the command. Please say again.')
      .getResponse();
  },
};

const skillBuilder = Alexa.SkillBuilders.custom();

exports.handler = skillBuilder
  .addRequestHandlers(
    LaunchRequestHandler,
    SearchHandler,
    HelpHandler,
    RepeatHandler,
    ExitHandler,
    SessionEndedRequestHandler
  )
  .addErrorHandlers(ErrorHandler)
  .lambda();
```
