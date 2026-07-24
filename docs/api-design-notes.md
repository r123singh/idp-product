Aim - Create APIs spec for document extraction for the Assistant clients

1. CRUD APIs for document type along with the fields to expect
2. Extract information from file by documentType with additional option to identify document

3. Document types - Passport, Resident ID.
4. Possible API features should support -
    - Main file to process
    - Image(PNG/JPEG) of the document
    - PDF document
    - Document type - Passport, Resident ID
    - Additional options identifying document
    - Fields to be included in the response
    - Response format - JSON
    - Handling any errors in processing the file uploaded
    - Handling LLM related errors in response generation
    - Errors with wrong document type provided or invalid file upload
    - Backside and Frontside separate files ?

Process -
1. API will enable extracting data from documents on transactional basis. 
2. It will trigger and start with consuming a document whenever user uploads it in the chat.
3. Finally, it will extract the data fields from the document as per expected fields and return the data as JSON object.


I went through the document - should we create 2 generic APIs one for document extraction and other for validation based on the document type or simply create API step wise for each step as mentioned in the process flow in the sheet. The generic API can be complex dealing with different repsonses for each document type but it will be single API easy to maintain. In case of step wise API, there can be document processing API and validation API per step and response creations will be easier. 

Generic API -
1. API will enable extracting data from documents on transactional basis. 
2. It will trigger and start with consuming a document whenever user uploads it in the chat.
3. Finally, it will extract the data fields from the document as per expected fields and return the data as JSON object.

Step wise API -
1. API will enable extracting data from documents on transactional basis. 
2. It will trigger and start with consuming a document whenever user uploads it in the chat.
3. Finally, it will extract the data fields from the document as per expected fields and return the data as JSON object.