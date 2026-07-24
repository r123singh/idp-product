1. Configure Document type API - API for document type object configuration (onboarding step one time, not needed for the POC)

  - This API can figure out matching document structure object from a given document type (for ex. passport, share transfer certificate)
  - API will be provided with document type input to resolve matching document type object to extract
  - It will return in response the schema object for that document in response
  - Extraction API will then use that response schema


2. Extraction API - API to retreive the content from uploaded document (for bulk upload or one by one handling)

  - Determine the document schema based 
  - Once the schema object is available, Extraction API will be invoked with schema as input
    - 2 options - 
      - This document type, understanding only once. Pre-fetch the scehma itself.
      - No document type specified, model to identify document type with Tool call. Document type determines next the schema. Next LLM model to process the extraction using the schema. 
      
  - The API can internally call an LLM model.
  - The API can handle validation of the document as well.
  - It can return the results as boolean value per validation rule. 
      - If its True for a rule, it means the rule is is satisfied.
      - If False, it means the rule is not satisfied. The API will the validation rule message as part of the response
  - API will be invoked after document upload finished by user, document type configured for that document with the Configure document type API and finally document content extracted using the Extraction API

API Python Program -

I want to create a Python program lambda function handler for the Extraction of document. The program is for Extraction of data from documents based on the schemas. It should support both for Local reads and S3/HTTP based source files. It will be deployed over lambda handle and HTTP gateway. The same code is to be referenced for local runner, lambda handler files. It will internally invoke an LLM model for the business logic of extraction. It should have optional input for content type of document to be extracted. In case its not provided, it should the model must be asked specifically to first identify document type and then extract the data based on the suitable schema file. If its already spcified about the document type, it should pre-fetch the schema from suitable schema file and then delegate the extraction to the LLM model invocation. For now the schemal files can be placed in the same folder as the Python program as path files.