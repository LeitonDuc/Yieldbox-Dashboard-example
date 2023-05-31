# Yieldbox-Dashboard-example

La idea de este ejercicio es generar un Dashboard en Power BI a partir de distinas fuentes de datos: Firebase y Google Sheets. Para mimetizar más adecuadamente un pipeline corporativo, se exportarán estos datos a AWS y  luego se conectarán a Power BI. Para el proceso de limpieza y preparación de datos se utilizará SQL y Python; para el llamado de APIS y funciones AWS/on-premises se utilizará Python. Los datos son extraidos de cuentas reales de Yieldbox SRL. 

<h3> Exportación de datos a AWS </h3>
<p> 1: Crear IAM User con Admin Role </p>
<p> 2: Crear S3 store service con 3 diferentes carpetas: firebase, digital-ocean y google-sheets </p>
<p> 3: Generar dos Lambda Function para extraer datos de Google Sheets y guardarlos en un S3 Bucket: </p>

```python

import json
import gspread
from oauth2client.service_account import ServiceAccountCredentials
import boto3

def lambda_handler(event, context):
    # Load the contents of the secretsKeys.json file
    with open('secretKeys.json', 'r') as file:
        google_creds = json.load(file)

    # Use the credentials to authorize gspread
    credentials = ServiceAccountCredentials.from_json_keyfile_dict(
        google_creds,
        ['https://spreadsheets.google.com/feeds', 'https://www.googleapis.com/auth/drive']
    )
    gc = gspread.authorize(credentials)

    # Open the Google Spreadsheet by its name and select the sheet.
    spreadsheet = gc.open("") # Replace with your Spreadsheet Name
    worksheet = spreadsheet.worksheet("") # Replace with your Page Name

    # Get all values from the worksheet
    data_sheets = worksheet.get_all_values()

    # Convert data to a JSON string
    data_str = json.dumps(data_sheets)

    # Create an S3 client
    s3 = boto3.client('s3')

    # Write data to a file in your S3 bucket
    s3.put_object(Body=data_str, Bucket="", Key='google-sheets/fileName.json') #In bucket add the S3 bucket name, also add a name to the JSON file
```
<p> Crear un permiso de Lambda->S3 y agregar un EventBridge (CloudWatch Events) seteado en "cron(0 10 * * ? *)", para que lambda function se active automaticamente todos los dias a las 10AM UTC.</p>
<p> 4: Para extraer datos de Firebase, primero descargar los datos de Firestore on-premises (seteado con un daily trigger): </p> 

```python

from google.cloud import firestore
import json
import datetime

class JSONEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime.datetime):
            return obj.isoformat()
        elif isinstance(obj, firestore.firestore.DocumentSnapshot):
            return obj.to_dict()
        elif isinstance(obj, firestore.firestore.QuerySnapshot):
            return [doc.to_dict() for doc in obj]
        return super().default(obj)


def download_firestore_data(collection_names):
    db = firestore.Client()

    for collection_name in collection_names:
        docs = db.collection(collection_name).get()
        data = [doc.to_dict() for doc in docs]

        with open(f"{collection_name}.json", 'w') as f:
            json.dump(data, f, cls=JSONEncoder)

collection_names = ['currencies', 'interests', 'investments', 'products', 'transactions', 'users']
download_firestore_data(collection_names)
 ```
<p> Y luego importar los datos en el S3 Bucket: </p>

```python

import boto3

def upload_files(path):
    session = boto3.Session(
        aws_access_key_id='yourAccessKey',
        aws_secret_access_key='YourSecretAccessKey',
        region_name='YourRegion'
    )
    s3 = session.resource('s3')

    bucket = 'YourbucketName'

    files_to_upload = ['YourFile1.json', 'YourFile2.json', 'YourFile3.json', 'YourFile4.json', 'YourFile5.json']

    for file in files_to_upload:
        file_path = path + file
        s3.meta.client.upload_file(file_path, bucket, 'FolderName/'+file)

upload_files('pathToFiles/')
```
<h3> Limpieza de datos </h3>
<p> 1. Los archivos cargados a S3 son en formato JSON. Transformarlos a CSV para que puedan ser más sencillamente analizados con SQL mediante Glue ETL Jobs (repetir con cada uno de los archivos) </p>

```python

import json
import csv
import boto3
import tempfile
import os

# Create an S3 client
s3 = boto3.client('s3')

# The name of the bucket
bucket_name = 'yourbucketname'

# The JSON file path (in the bucket)
json_file_path = 'yourJsonfilepath.json'

# The CSV file path (in the bucket)
csv_file_path = 'wherewillbeloadtheCSV.csv'

# Define CSV column names
fields = ['column_name1', 'column_name2',...]

# Download the JSON file from S3
temp_json_file = tempfile.NamedTemporaryFile(delete=False)
temp_json_filepath = temp_json_file.name
temp_json_file.close()
s3.download_file(bucket_name, json_file_path, temp_json_filepath)

# Open the downloaded JSON file
with open(temp_json_filepath, 'r') as json_file:
    # Load JSON data
    data = json.load(json_file)

# Prepare data for CSV
csv_data = []
for item in data:
    filtered_item = {key: item.get(key, 'N/A') for key in fields}
    csv_data.append(filtered_item)

# Write data to CSV file
temp_csv_file = tempfile.NamedTemporaryFile(delete=False, mode='w', newline='')
temp_csv_filepath = temp_csv_file.name
temp_csv_file.close()
with open(temp_csv_filepath, 'w', newline='') as csv_file:
    writer = csv.DictWriter(csv_file, fieldnames=fields)
    writer.writeheader()
    writer.writerows(csv_data)

# Upload the CSV file to S3
s3.upload_file(temp_csv_filepath, bucket_name, csv_file_path)

# Remove temporary files
os.remove(temp_json_filepath)
os.remove(temp_csv_filepath)

```
<p> 2. Crear Crawler en AWS Glue para generar los databases y tables. Recordar que para evitar errores en el funcionamiento del layout de AWS, es mejor tener cada archivo CSV en una carpeta propia y generar un database por table. </p>
<p> 3. Crear ETL Query para cada uno de las tables a usar. En nuestro caso serán: Earnings, Transactions, Investments, Interests y Currencies. Ya que cierto proceso de ETL se hizo en los códigos de Python en AWS Glue, aquí unicamente será necesario elegir las columnas con las que trabajaremos, con el fin de reducir el tamaño de las tables. Por ejemplo: </p>

```SQL

SELECT 
    uid as "User id",
    currency as "Currency id",
    date_format(from_iso8601_timestamp(createdat), '%d/%m/%Y') AS "Fecha",
    amount as "Monto",
    action as "Acción"
FROM 
    awsdatacatalog.transactions.transactionscsv;
    
```
<p> 4. Creamos una función Lambda que corra diariamente los 5 Querys creados y guarde los csv en S3. De esta manera, quedarán listos los datos para ser enviados a Power BI. </p>

```python
    
import boto3

def lambda_handler(event, context):
    s3 = boto3.resource('s3')
    client = boto3.client('athena')
    
    bucket = 'your-bucket'  # Replace with your bucket name
    queries_files = ['yourfiles.sql]
    
    for file in queries_files:
        obj = s3.Object(bucket, file)
        query = obj.get()['Body'].read().decode('utf-8')
        
        response = client.start_query_execution(
            QueryString=query,
            ResultConfiguration={
                'OutputLocation': 's3://yieldbox-data/ReadyToPowerBI/'
            }
        )
        
    return {
        'statusCode': 200,
        'body': 'Queries started successfully'
    }

```
<p> 5. Debido a que por cada Query que se corre, se crean dos archivos nuevos (un csv y un metadata) con un nombre ID unico, hay que crear una función Lambda que tome los 5 últimos csv creados y elimine (antes de pegar) los csv antiguos en la carpeta destino. De esta manera siempre habrá una carpeta con -unicamente- los 5 csv más nuevos. Lo conseguimos mediante el siguiente script: </p>

```python

import boto3

def lambda_handler(event, context):
    source_bucket = ''
    source_folder = '/'
    destination_bucket = ''
    destination_folder = '/'
    
    s3 = boto3.client('s3')
    
    # Delete existing files in the destination folder
    response = s3.list_objects_v2(Bucket=destination_bucket, Prefix=destination_folder)
    if 'Contents' in response:
        objects = [{'Key': obj['Key']} for obj in response['Contents']]
        s3.delete_objects(Bucket=destination_bucket, Delete={'Objects': objects})
    
    # List objects in the source folder
    response = s3.list_objects_v2(Bucket=source_bucket, Prefix=source_folder)
    
    # Get the last four CSV files based on the LastModified timestamp
    csv_files = []
    for obj in sorted(response['Contents'], key=lambda x: x['LastModified'], reverse=True):
        if obj['Key'].lower().endswith('.csv'):
            csv_files.append(obj['Key'])
            if len(csv_files) == 4:
                break
    
    # Copy the selected CSV files to the destination folder
    for file in csv_files:
        copy_source = {'Bucket': source_bucket, 'Key': file}
        destination_key = destination_folder + file.split('/')[-1]  # Use only the file name in the destination
        
        s3.copy_object(Bucket=destination_bucket, Key=destination_key, CopySource=copy_source)
    
    return {
        'statusCode': 200,
        'body': 'CSV files copied successfully'
    }

```
<p> Finalmente, tenemos los 5 csv listos para exportar a Power BI. Generamos un trigger que descargue los archivos diariamente y los importamos en Power BI </p>

<h3> Arquitectura final </h3>

![architecture yield](https://github.com/LeitonDuc/Yieldbox-Dashboard-example/assets/94261037/0cb5f231-77f5-4ffa-806e-0e3a991178b4)

<h3> Informe de Power BI </h3>

![page](https://github.com/LeitonDuc/Yieldbox-Dashboard-example/assets/94261037/2deec5a6-9636-4a14-8e09-b12eca440983)





    
    
   
   
    
