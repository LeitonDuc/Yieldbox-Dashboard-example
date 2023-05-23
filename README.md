# Yieldbox-Dashboard-example

La idea de este ejercicio es generar un Dashboard en Power BI a partir de distinas fuentes de datos: Firebase, Digital Ocean y Google Sheets. Para mimetizar más adecuadamente un pipeline corporativo, se exportarán estos datos a AWS y MySQL, y Power BI importará datos desde estas dos fuentes. Para el proceso de limpieza y preparación de datos se utilizará SQL; para el llamado de APIS se utilizará Python.   

<h2> AWS Architecture steps </h2>
<p> 1: Create IAM User with Admin Role </p>
<p> 2: Create S3 store service with 3 different folders: firebase, digital-ocean and google-sheets </p>
<p> 3: Generate a Lambda Function to get data from Google Sheets </p>

```python
import json
import gspread
from oauth2client.service_account import ServiceAccountCredentials
import boto3

def lambda_handler(event, context):
    # Load the contents of the credentials.json file
    with open('credentials.json', 'r') as file:
        google_creds = json.load(file)

    # Use the credentials to authorize gspread
    credentials = ServiceAccountCredentials.from_json_keyfile_dict(
        google_creds,
        ['https://spreadsheets.google.com/feeds', 'https://www.googleapis.com/auth/drive']
    )
    gc = gspread.authorize(credentials)

    # Open the Google Spreadsheet by its name and select the sheet.
    spreadsheet = gc.open("") # Replace with your Spreadsheet Name
    worksheet = spreadsheet.worksheet('') # Replace with your Page Name

    # Get all values from the worksheet
    data_sheets = worksheet.get_all_values()

    # Convert data to a JSON string
    data_str = json.dumps(data_sheets)

    # Create an S3 client
    s3 = boto3.client('s3')

    # Write data to a file in your S3 bucket
    s3.put_object(Body=data_str, Bucket='', Key='google-sheets/data_sheets.json') #In bucket add the S3 bucket name
```
<p> Create a permission to Lambda->S3 and add and EventBridge (CloudWatch Events) set to "cron(0 10 * * ? *)", so the lambda function will trigger daily at 10AM UTC.
 
    
    
    
   
   
    
