# User authentication and validation against identity database (DynamoDB in our case) - Lambda Validation
do_this() include the validation function validate_name()

## Demo:
<img src="images/usecase2/usercase2.png" alt="usecase2" width="500">

## We need to do following:
1.	We want Lambda to talk to Dynamodb to need to attach dynamodb policy to the lambda role
2.	Jypytur notebook installed on local is great to test dynamodb sql (https://www.chrisjmendez.com/2018/11/06/installing-jupyter-on-os-x-using-homebrew/)
3.	Create a DynamoDB table

## New functions introduced here:
- elicit_slot: Informs Amazon Lex that the user is expected to provide a slot value in the response.
- delegate: Directs Amazon Lex to choose the next course of action based on the bot configuration.
- validate_friend: Validate the friend from a list
- build_validation_result: Return response to Lex with or without defined return message
Reference : https://docs.aws.amazon.com/lex/latest/dg/lambda-input-response-format.html

## Functions
```
def do_this(intent_request):
    # get the value of Name slot provided by Lex interface
    name = intent_request['currentIntent']['slots']["Name"]
    source = intent_request['invocationSource']
    
    if source == 'DialogCodeHook':
        # Perform basic validation on the supplied input slots.
        # Use the elicitSlot dialog action to re-prompt for the first violation detected.
        slots = get_slots(intent_request)
        validation_result = validate_name(name)
        if not validation_result['isValid']:
            slots[validation_result['violatedSlot']] = None
            return elicit_slot(intent_request['sessionAttributes'],
                               intent_request['currentIntent']['name'],
                               slots,
                               validation_result['violatedSlot'],
                               validation_result['message'])
        
        output_session_attributes = intent_request['sessionAttributes'] if intent_request['sessionAttributes'] is not None else {}
        return delegate(output_session_attributes, get_slots(intent_request))
    # return closer of the intent
    return close(intent_request['sessionAttributes'],
                 'Fulfilled',
                 {'contentType': 'PlainText',
                  'content': 'Hey {}!, I am Lex. \n\nNice to meet you! :) You can try HR portal functionalities: \n 1. Log my hours \n 2. Calculate my pay \n 3. Faq'.format(name)
                 }
                 )
```
### Validation function
```
def validate_name(name):
    friends = ['srinivas', 'laxmi']
    #friends=find_name_in_ddb(name)
    if name is not None and name.lower() not in friends:
        return build_validation_result(False,
                                       'Name',
                                       'I am sorry {}. This username does not exist, can you retype your username correctly'.format(name))
    
    return build_validation_result(True, None, None)
```

### Configure Event to test this function in lambda
this is same event used in use-case 1, change the values according to your data
```
{
  "messageVersion": "1.0",
  "invocationSource": "DialogCodeHook",
  "userId": "tast_user",
  "sessionAttributes": {},
  "bot": {
    "name": "HR_Bot",
    "alias": "$LATEST",
    "version": "$LATEST"
  },
  "outputDialogMode": "Text",
  "currentIntent": {
    "name": "DoSomething",
    "slots": {
      "Name": "laxmi"
    },
    "confirmationStatus": "None"
  },
  "inputTranscript": "This is awesome"
}
```
<img src="images/usecase1/7.png" width="1000">

The above function right now validates on a list of names, but we can enhance our use case with validating against username database.

## Steps
1.	Add a Dynamodb permissions to Lambda
> Navigate to IAM role attached to lambda - View the HR_portal_Lex_role in IAM console
<img src="images/usecase2/1.png" width="500">

> Attach policy
<img src="images/usecase2/2.png" width="500">
 
> Dynamodb full access
<img src="images/usecase2/3.png" width="500">

Save this role.

2. Create DynamoDB table, navigate to DynamoDB console
<img src="images/usecase2/4.png" width="400">

- Primary key as ‘name’
<img src="images/usecase2/5.png" width="1000">

- Create Item

<img src="images/usecase2/6.png" width="300"><img src="images/usecase2/7.png" width="400">

- Repeat to enter some example names
<img src="images/usecase2/8.png" width="300">
  
You are now ready to query from dynamodb table using Jupyter notebook in following steps or using lambda function, refer below for syntax.
Ref: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GettingStarted.Python.03.html

3. Testing code in Jupyter notebook

<img src="images/usecase2/9.png" width="500">

```
import boto3
from boto3.dynamodb.conditions import Key, Attr

# DynamoDb table decaration
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('friends')

name = 'laxmi'

response = table.query(
    KeyConditionExpression=Key('name').eq(name)
)

for i in response['Items']:
    names = (i['name'])

if name in names:
    print("name found")
else:
    print("name not found")

```
4. Search from DynamoDB table function

```
def find_name_in_ddb(name):
    friends_table = dynamodb.Table('friends')
    name = str(name)
    response = friends_table.query(
        KeyConditionExpression=Key('name').eq(name)
        )
    names=[]
    for i in response['Items']:
        names = (i['name'].lower())
    
    return(names)
```

Now uncomment the section of dynamoDB and comment the list in our validation function
```
    #friends = ['srinivas', 'laxmi']
    friends=find_name_in_ddb(name)
```

Test again with the configured event
<img src="images/usecase1/6.png" width="1000">

## Code
```
import os
import time
import logging
import boto3
import decimal
import dateutil.parser
import datetime
from boto3.dynamodb.conditions import Key, Attr

# Logger
logger = logging.getLogger()
logger.setLevel(logging.DEBUG)
# DynamoDb table decaration
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
# Client for Comprehend
client = boto3.client('comprehend')
"""
Helper function for Lex
"""  
def get_slots(intent_request):
    return intent_request['currentIntent']['slots']
    
#Close: Informs Amazon Lex not to expect a response from the user. Just return any final message from Lambda to Lex
def close(session_attributes, fulfillment_state, message):
    response = {
        'sessionAttributes': session_attributes,
        'dialogAction': {
            'type': 'Close',
            'fulfillmentState': fulfillment_state,
            'message': message
        }
    }
    return response
    
# ElicitIntent: Informs Amazon Lex that the user is expected to respond with an utterance that includes an intent. 
def elicit_slot(session_attributes, intent_name, slots, slot_to_elicit, message):
    return {
        'sessionAttributes': session_attributes,
        'dialogAction': {
            'type': 'ElicitSlot',
            'intentName': intent_name,
            'slots': slots,
            'slotToElicit': slot_to_elicit,
            'message': message
        }
    }

#Delegate: Directs Amazon Lex to choose the next course of action based on the bot configuration.
def delegate(session_attributes, slots):
    return {
        'sessionAttributes': session_attributes,
        'dialogAction': {
            'type': 'Delegate',
            'slots': slots
        }
    }

# return response to Lex with/without defined return message
def build_validation_result(is_valid, violated_slot, message_content):
    if message_content is None:
        return {
            "isValid": is_valid,
            "violatedSlot": violated_slot,
        }

    return {
        'isValid': is_valid,
        'violatedSlot': violated_slot,
        'message': {'contentType': 'PlainText', 'content': message_content}
    }
    
"""
Functions to validate the user input
"""
def validate_name(name):
    #friends = ['srinivas', 'laxmi']
    friends=find_name_in_ddb(name)
    if name is not None and name.lower() not in friends:
        return build_validation_result(False,
                                       'Name',
                                       'I am sorry {}. This username does not exist, can you retype your username correctly'.format(name))
    
    return build_validation_result(True, None, None)
        
"""
Dynamodb Functions read or write values
"""
def find_name_in_ddb(name):
    friends_table = dynamodb.Table('friends')
    name = str(name)
    response = friends_table.query(
        KeyConditionExpression=Key('name').eq(name)
        )
    names=[]
    for i in response['Items']:
        names = (i['name'].lower())
    
    return(names)

"""
Functions to fulfill intent
"""
def do_this(intent_request):
    # get the value of Name slot provided by Lex interface
    name = intent_request['currentIntent']['slots']["Name"]
    source = intent_request['invocationSource']
    
    if source == 'DialogCodeHook':
        # Perform basic validation on the supplied input slots.
        # Use the elicitSlot dialog action to re-prompt for the first violation detected.
        slots = get_slots(intent_request)
        validation_result = validate_name(name)
        if not validation_result['isValid']:
            slots[validation_result['violatedSlot']] = None
            return elicit_slot(intent_request['sessionAttributes'],
                               intent_request['currentIntent']['name'],
                               slots,
                               validation_result['violatedSlot'],
                               validation_result['message'])
        
        output_session_attributes = intent_request['sessionAttributes'] if intent_request['sessionAttributes'] is not None else {}
        return delegate(output_session_attributes, get_slots(intent_request))
    # return closer of the intent
    return close(intent_request['sessionAttributes'],
                 'Fulfilled',
                 {'contentType': 'PlainText',
                  'content': 'Hey {}!, I am Lex. \n\nNice to meet you! :) You can access following HR portal functionalities: \n 1. Log my hours \n 2. Calculate my pay \n 3. FAQ'.format(name)
                 }
                 )
         
#############################################################################################################
def dispatch(intent_request):
    """
    Called when the user specifies an intent for this bot.
    """
    logger.debug('dispatch userId={}, intentName={}'.format(intent_request['userId'], intent_request['currentIntent']['name']))
    intent_name = intent_request['currentIntent']['name']

    # Dispatch to your bot's intent handlers
    if intent_name == 'DoSomething':
        return do_this(intent_request)
    elif intent_name == 'LogMyHours':
        return logMyHours(intent_request)
    elif intent_name == 'CalculateMyPay':
        return calcMyPay(intent_request)
    elif intent_name == 'HRInfo':
        return hrInfo(intent_request)

    raise Exception('Intent with name ' + intent_name + ' not supported')

def lambda_handler(event, context):
    """
    Route the incoming request based on intent.
    The JSON body of the request is provided in the event slot.
    """
    # By default, treat the user request as coming from the America/New_York time zone.
    os.environ['TZ'] = 'America/New_York'
    time.tzset()
    
    """
    # Check for sentiment
    sentiment=client.detect_sentiment(Text=event['inputTranscript'],LanguageCode='en')['Sentiment']
    if sentiment=='NEGATIVE':
        return close({},
                 'Fulfilled',
                 {'contentType': 'PlainText',
                  'content': 'I am sorry. Seems like you are having troubles with our chat service. Let me transfer you to the a human support.'}) #trigger a call to human support
    """
    return dispatch(event)
```