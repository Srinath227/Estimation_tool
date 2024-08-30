import json
import boto3
import uuid
from decimal import Decimal

client_dynamodb = boto3.client('dynamodb')
dynamodb = boto3.resource('dynamodb')

counter_table_name = 'CounterTable'
table_name = 'Estimation_Resourceload'
table = dynamodb.Table(table_name)




def postRows(event, context):
    try:
        requestBody = json.loads(event['body'])
        
        # Validate 'listdata' and 'RfpNum' existence
        if 'listdata' not in requestBody:
            raise ValueError("The 'listdata' field is missing in the request body.")
        
        RfpNum = requestBody.get('RfpNum')
        if not RfpNum:
            raise ValueError("The 'RfpNum' field is missing in the request body.")
        
        for item in requestBody["listdata"]:
            unique_id = str(uuid.uuid4())
            item['rfp_no'] = int(RfpNum)  # Ensure rfp_no is passed as a number
            item['id'] = unique_id
            
            # Prepare other items for insertion into DynamoDB, assuming they match the schema
            table.put_item(Item=item)
        
        response = {
            'statusCode': 200,
            'body': json.dumps({'success': True}), 
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            }
        }
    
    except ValueError as ve:
        response = {
            "statusCode": 400,
            "body": json.dumps(str(ve)),
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            }
        }
    
    except Exception as e:
        response = {
            "statusCode": 500,
            "body": json.dumps(str(e)),
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            }
        }
    
    return response



def putTotalResource(event):
    try:
        rfp = event["queryStringParameters"]["rfp_no"]
        total_resource = event["queryStringParameters"]["sum"]
        
        response = client_dynamodb.update_item(
            TableName=table_name2,
            Key={'rfp_no': {'N': rfp}},
            UpdateExpression='SET #newAttr = :newValue',
            ExpressionAttributeNames={
              '#newAttr': 'total_resource'
            },
            ExpressionAttributeValues={
              ':newValue': {'N': str(total_resource)}
            },
            ReturnValues='ALL_NEW'
        )
        
        response = {
            'statusCode': 200,
            'body': json.dumps("update successful"), 
            'headers': {
                'ContentType': 'application/json',
                'Access-Control-Allow-Origin': '*'
            }
        }
    except Exception as e:
        response = {
            "statusCode": 500,
            "body": json.dumps(str(e)),
            'headers': {
                'ContentType': 'application/json',
                'Access-Control-Allow-Origin': '*'
            }
        }
    return response


def getData(event,table_name):

    try:

        rfp=event["queryStringParameters"]["RfpNum"]

        response= filterByRfp(table_name,rfp) 

        items = response.get('Items', [])

        # sorted_items=sorted(items,key=lambda x: x['id']['N'])

        for item in items: 
#         #   if 'rfp_no'in item:
              del item['rfp_no'] 
            #   del item['id']

        

        response3= { 

          'statusCode': 200,

          'body': json.dumps(items),

          'headers': {

                        'ContentType': 'application/json',

                        'Access-Control-Allow-Origin': '*'

                    } 

            }

    except Exception as e:

        response3= { 

          'statusCode': 500,

          'body': json.dumps(str(e)),

          'headers': {

                        'ContentType': 'application/json',

                        'Access-Control-Allow-Origin': '*'

                    } 

      }

    return response3 


def delete(event, context):
      payload = json.loads(event['body'])
      item_id = payload.get('id')
      sort_key_value = payload.get('RfpNum')
   
      try:
          
          
           response = table.scan(
               FilterExpression='#primary_key = :id_value AND #sortkey = :sortkey_value',
           
               ExpressionAttributeNames={
                '#primary_key': 'id',
                '#sortkey': 'rfp_no'
                  },
               ExpressionAttributeValues={
                 ':id_value': item_id,
                 ':sortkey_value': sort_key_value
                  }
               )
               
           items = response.get('Items', []) 
           a="abc"
           
           for item in items:
               if(
                  item.get('id')==item_id and
                  item.get('rfp_no')==sort_key_value
                  ):
                #   id={
                #       'id':item['id']
                #   }
                  table.delete_item(
                       Key={
                           'id':item['id'],
                           'rfp_no': item['rfp_no']
                       }
                    ) 
                  a="deleted"
                  
           if(a=="deleted"):      
               return {
                      'statusCode': 200,
                      'body': json.dumps('Item deleted'),
                       'headers': {
                                'ContentType': 'application/json',
                                'Access-Control-Allow-Origin': '*'
                                }
                   }
      except Exception as e:
          print(e)
          return {
              'statusCode': 500,
              'body': json.dumps(str(e)),
              'headers': {
                        'ContentType': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                        }
          }
          return response

    
def lambda_handler(event, context):
    hcontext=event["httpMethod"]
    if(hcontext=="POST"):
        return postRows(event,context)
    elif(hcontext=="PUT"):
        return putTotalResource(event)
    elif(hcontext=='GET'):
        return getData(event,table_name)
    elif(hcontext=='DELETE'):
        return delete(event,context)
    
