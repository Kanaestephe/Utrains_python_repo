## 1-At work, we have an internal application running on postgressql, and we need a python script that will take backup of the database and upload it in s3 bucket everyday at a certain time.

Before executing the below script, you have to set Database credentials, AWS Access keycredentials and backup path.

``` python
import os
import sys
import subprocess
from optparse import OptionParser
from datetime import datetime

import boto3
from boto3.s3 import Key


DB_USER = 'databaseuser'
DB_NAME = 'databasename'

BACKUP_PATH = r'/webapps/myapp/db_backups'

FILENAME_PREFIX = 'myapp.backup'

# Amazon S3 settings.
AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
AWS_BUCKET_NAME = 'myapp-db-backups'


def main():
    parser = OptionParser()
    parser.add_option('-t', '--type', dest='backup_type',
                      help="Specify either 'hourly' or 'daily'.")

    now = datetime.now()

    filename = None
    (options, args) = parser.parse_args()
    if options.backup_type == 'hourly':
        hour = str(now.hour).zfill(2)
        filename = f'{FILENAME_PREFIX}.h{hour}'
    elif options.backup_type == 'daily':
        day_of_year = str(now.timetuple().tm_yday).zfill(3)
        filename = f'{FILENAME_PREFIX}.d{day_of_year}'
    else:
        parser.error('Invalid argument.')
        sys.exit(1)

    destination = r'%s/%s' % (BACKUP_PATH, filename)

    print (f'Backing up {DB_NAME} database to {destination}')
    ps = subprocess.Popen(
        ['pg_dump', '-U', DB_USER, '-Fc', DB_NAME, '-f', destination],
        stdout=subprocess.PIPE
    )
    output = ps.communicate()[0]
    for line in output.splitlines():
        
        print(line)

    print(f'Uploading {filename} to Amazon S3...')
    upload_to_s3(destination, filename)


def upload_to_s3(source_path, destination_filename):
    """
    Upload a file to an AWS S3 bucket.
    """
    conn = boto3.connect_s3(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
    bucket = conn.get_bucket(AWS_BUCKET_NAME)
    k = Key(bucket)
    k.key = destination_filename
    k.set_contents_from_filename(source_path)


if __name__ == '__main__':
    main()
```

## 2-At work , there is a need to a python script that can convert .xml files to json format.

``` python

#install json and xmltodict modules if you don't have it yet
# import json module and xmltodict module provided by python
import json
import xmltodict

# open the input xml file and read
# data in form of python dictionary
# using xmltodict module
with open("test.xml") as xml_file:
	
	data_dict = xmltodict.parse(xml_file.read())
	xml_file.close()
	
	# generate the object using json.dumps()
	# corresponding to json data
	
	json_data = json.dumps(data_dict)
	
	# Write the json data to output json file called output
	with open("output.json", "w") as json_file:
		json_file.write(json_data)
		json_file.close()

```
## 3-At work , you are facing difficulties when working with json files on urgent projet. But you are more comfortable with csv files, so you need to write a python script to convert json files to csv format.

``` python
# Python program to convert
# JSON file to CSV


import json
import csv
from os import sep


# Opening JSON file and loading the data
# into the variable data
with open('output.json') as json_file:
	data = json.load(json_file)

catalog_book = data['catalog']['book']

# now we will open a file for writing
data_file = open('output.csv', 'w')

# create the csv writer object
csv_writer = csv.writer(data_file)

# Counter variable used for writing
# headers to the CSV file
count = 0

for catalog in catalog_book:
	if count == 0:

		# Writing headers of CSV file
		header = catalog.keys()
		csv_writer.writerow(header)
		count += 1

	# Writing data of CSV file
	csv_writer.writerow(catalog.values())

data_file.close()

```
## 4- You are working on a server, there is a need of a python script to check if a server is up or down.

``` python
import socket 

def is_running(site):
    """This function attempts to connect to the given server using a socket.
        Returns: Whether or not it was able to connect to the server."""
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((site, 80))
        return True
    except:
        return False

if __name__ == "__main__":
    while True:
        site = input('Website to check: ')
        if is_running(f'{site}'):
            print(f"{site} is running!")
        else:
            print(f'There is a problem with {site}!')

        if input("Would You like to check another website(Y/N)? ") in {'n', 'N'}:
            break
```
