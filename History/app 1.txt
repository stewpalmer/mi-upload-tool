from flask import Flask, request, render_template
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient
import csv
import io

app = Flask(__name__)
app.secret_key = 'mysecretkey'

# Enter your connection string and container name here
#connection_string = 'DefaultEndpointsProtocol=https;AccountName=myalliancemispend;AccountKey=aHI3NzcCY8Q4GZqrknEynUDypWOtMLNX4S9zhJH02Aya4EtAW1ffj9WjtPt3kXWoHcRs1jTmiest+AStMSp70Q==;EndpointSuffix=core.windows.net'
#container_name = 'mi-spend'

connection_string = 'DefaultEndpointsProtocol=https;AccountName=pipmispend;AccountKey=jQ3T5PZAr77MhoVc7W/yPws+mBBZEknk6E5ozrb00R+Cp7aaBx1nIAcB5yzDQsMIMTPdvSCJckW9+ASt2yf2wg==;EndpointSuffix=core.windows.net'
container_name = 'pip-mi-spend'

@app.route('/', methods=['GET', 'POST'])
def upload():
    if request.method == 'POST':
        # Get the file from the form data
        file = request.files['csv_file']
        if file:
            try:
                if '-' in file.filename:
                    filename_parts = file.filename.split('-')
                    if len(filename_parts[0]) == 6 and filename_parts[0].isdigit():
                        # hyphen is at the correct posion and the text to the left of the hyphen are numbers - carry on !!
                      
                        # Check if the file name starts with '2023'
                        if not file.filename.startswith('2023'):
                            # The file does not begin with '2023'
                            return render_template('upload.html', message='Invalid file name. Please make sure the file name starts with 2023 followed by the month, e.g. 01, 02, 03, etc.')

                        # Read the CSV file into a list of lists
                        valid_file_names = ['EOECPH.csv', 'NHSLPP.csv', 'NOECPC.csv', 'NHSCS.csv']
                        ##########################################################################################
                        ### use 2 apps; one for utf-8 the other for csv
                        # UTF-8 (CSV)
                        csv_data = list(csv.reader(io.StringIO(file.stream.read().decode("utf-8-sig"))))
                        # CSV (Standard)
                        # csv_data = list(csv.reader(io.StringIO(file.stream.read().decode('iso-8859-1'))))
                        ##########################################################################################
                        if not file.filename.endswith(tuple(valid_file_names)):
                            return render_template('upload.html', message='Invalid file name. Please make sure the csv file name is saved as CSV UTH-8 (Comma delimited file) and ends with either EOECPH, NHSLPP, NOECPC or NHSCS')

                        # Check whether the CSV file has the expected header
                        expected_header = ['Framework', 'Framework ID', 'Supplier Name', 'Supplier Company House No', 'Member trusts NHS Org Code (ODS)', 'Member trust name', 'Framework Organisation', 'MI Reporting Month', 'Invoice Date', 'ABI', 'MI Value (Ex VAT)']
                        if csv_data[0] != expected_header:
                            return render_template('upload.html', message='Invalid CSV file. Please make sure the header row is correct.')

                        # Check whether the CSV file has at least one row of data
                        if len(csv_data) < 2:
                            return render_template('upload.html', message='Invalid CSV file. Please make sure the file contains at least one row of data.')

                        # Check whether the CSV file has any blank rows and identify the row number of the first blank row
                        blank_row_num = None
                        for i, row in enumerate(csv_data):
                            if not all(row):
                                blank_row_num = i + 1
                                break
                        if blank_row_num:
                            return render_template('upload.html', message=f'Invalid CSV file. Please make sure the file does not contain any blank cells. Blank cell have been found in row number {blank_row_num}.')           
                
                        # Create a blob client using the connection string
                        blob_service_client = BlobServiceClient.from_connection_string(connection_string)
                        container_client = blob_service_client.get_container_client(container_name)
 
                        # Upload the file to the container
                        blob_client = container_client.upload_blob(name=file.filename, data=file.stream.read())
			
			# File uploaded successfully!
                        return render_template('upload.html', message='File uploaded successfully!')
                        
                    else:
                        # hyphen is not at the required position
                        return render_template('upload.html', message='Hyphen is not at the required position in the filename.')
                else:
                    # filename does not contain a hyphen
                    return render_template('upload.html', message='File name does not contain a hyphen.')

            except Exception as e:
                return render_template('upload.html', message=f'Error uploading file: {str(e)}')

    # Render the upload form template for GET requests
    return render_template('upload.html')

if __name__ == '__main__':
    app.run()
