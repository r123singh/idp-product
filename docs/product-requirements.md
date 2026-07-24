## Objective

Aim here is to create an API led system to automate the Share Transfer workflow in the registry (Free Zone Registry). Share Transfer is the process of transfer of a given number of shares from the current shareholder(s) (seller) to a new shareholder(s) (buyer). Currently this process involves doing multiple document checks including Passport, Emirates ID, Share Transfer certificate etc to validate and review the transfer. The main document here is Share Transfer certificate containing all info of the involved selling/buying shareholder(s).

**The API led system backed by Assistant capabilities will enable \-** 

1. User interaction in initiating the Share transfer validation request process along with uploading of the required documents at each workflow step

2. Processing of the uploaded documents at each step of the workflow for extraction of the required data. Ensuring all required data is either extracted or confirmed by asking for user inputs

3. Validation checks at each step of workflow to ensure data extracted is valid. Cross validation of all data points extracted at any with corresponding data points recorded from previous step's documents to ensure consistent flow.

4.  Final Report generation which will combine the data records extracted from all documents uploaded for the user review and download purposes in the end.

 

## API based workflow system design

The workflow system needs to be designed in such a way that APIs should be invoked for performing any document related actions such as extraction, validation. Main 2 key APIs identified to handle these actions \- 

1. **Extraction API** \- API to extracting fields based on the document types listed below. Based on it will return required fields listed for type in a structured response

2. **Validation API** \- API to validate as per the document type requirements. It should perform all validations as per checks, cross-checks listed for each document type. 

### 1\. Share Transfer certificate document \- 

**Following fields need to be extracted (classified under different sections):**

**Company Info** \-

1. **Company Name** \- Name of the company. It can be extracted from the header section (top) of the document. It needs to be checked if it's empty.  
     
2. **Entity Number** \- This unique id of the company in the registry system. It can be extracted from the header section of the document. It needs to be checked if it follows the pattern \- “**Registry \+ digits**” for example \- “ENT120900”.  
     
3. **Document Title** \- This will be the title of the document which is "Share Transfer Form". It can be extracted from the header section of the document. Check if it contains "Share Transfer" in the text.

**Current Share structure** \-

4. **Current Shareholder Name** \- This will be the name of the shareholder who is selling the shares. It can be extracted from the "current share structure" section of the document in a table format. It needs to be checked if its missing.  
     
5. **Share Type** \- This will be the type of the share for example "Ordinary Share" or "Preference Share". It can be extracted from the "current share structure" section of the document in a table format. It should be common across the document.  
     
6. **Number of Shares** \- This will be the number of shares currently held by the current shareholder. It can be extracted from the "current share structure" section of the document in a table format. It needs to be checked if its >=  number of transferred shares.  
     
7. **Value per Share (AED)** \- This will be value per share in AED. It can be extracted from the "current share structure" section of the document in a table format. The currency should be checked if it matches with currency in other places in the document.

**Transfer Details** \-

8. **Seller Name** \- This will be the name of the shareholder who is selling the shares referred as "Transferor". It should match with the name of the current shareholder and also with the seller's name in the signature section of the document. It can be extracted from the document in a table section. It needs to be checked if its missing.  
     
9. **Share Type (Transferred)** \- This will be the type of share being transferred present in the transfer details section of the document(table section). It needs to be checked if it matches with the share type extracted in point 5\.  
     
10. **Number of Shares Transferred** \- This will be the number of shares being transferred present in transfer details section of the document(table section). It needs to be checked if its \<= number of shares currently held by the current shareholder.  
      
11. **Value per Share** \- This will be value per share being transferred present in transfer details section of the document(table section). It needs to be checked if its matches with value per share extracted in point 7 and any other places.  
      
12. **Buyer Name (New Shareholder)** \- This will be name of the shareholder who is buying the shares referred as "Transferee". It can be extracted from the transfer details section of the document(table section). It needs to checked if it matches with the name of the new shareholder in the post-transfer section of the document as well as in signature section of the document.  
      
13. **Buyer Passport / Registration Numbe**r \- This will be the passport/registration number of the new shareholder. It can be extracted from the transfer details section of the document(table section). It needs to be checked if its available or not. Its required for cross-checking with passport documents later in the process.

**Buyer & Resolution** \-

14. **Buyer Name (New Shareholder) Name \-** This will be the name of the new shareholder referred to as "Transferee". It can be extracted from the post-transfer (new shareholder) details section of the document(table). It needs to be checked if it matches with buyer name extracted in point 12\.  
      
15. Buyer Passport Number \- This will be the passport number of the new shareholder. It can be extracted from the post-transfer (new shareholder) details section of the document(table). It should match with the passport number extracted in point 13\.  
      
16. Share Type \- It can be extracted from the post-transfer (new shareholder) details section of the document(table). It needs to be checked if it matches with all sections where its present.  
      
17. Number of Shares After Transfer \- This will be the number of shares after transfer present in the post-transfer (new shareholder) details section of the document(table). It needs to be checked if it equals to "Number of shares transferred" extracted in point 10\.  
      
18. Value per Share \- This will be value per share after transfer present in post-transfer (new shareholder) details section of the document(table). It needs to be checked if its matches with value extracted in other sections where present.

**Signatures** \-

19. Transfer Date \- This will be the date of share transfer. It can be found near the signatures section of the document. It will have a date pattern as DD/MM/YYYY. It should be checked if it's in the past or before the current date.  
      
20. Buyer Name \- This will be name of Buyer(s) as present near the signatures section of the document. It needs to be checked if its matches with buyer(s) name extracted earlier in the document.  
      
21. Buyer Signature Present \- This will be signature of Buyer(s) as present in the signatures section of the document. It needs to be checked if signature ink or image has been detected.  
      
22. Seller Name \- This will be the name of Seller(s) as present in the signatures section of the document. It needs to be checked if its matches with seller name extracted earlier in the document.  
      
23. Seller Signature Present \- This will be signature of Seller(s) as present in the signatures section of the document. It needs to be checked if signature ink or image has been detected.  
      
24. Approving Shareholder Name \- This will be name of Approving Shareholder(s) as present in the signatures section of the document. It needs to be checked the name(s) matches with the name(s) of current shareholder(s) extracted earlier in the document.  
      
25. Approving Signature Present \- This will be signature of Approving Shareholder(s) as present in the signatures section of the document. It needs to be checked if signature ink or image has been detected.

Note:

1. For any missing fields, ask the user to re-upload the document.  
2. In case of wrong document uploaded, ask the user to upload the correct document.  
3. There can be more than 1 buyer. In case of multiple buyers extract each record separately and check if all the fields are present for each buyer.  
4. There can be more than 1 seller or existing shareholder. In case of multiple sellers extract each record separately and check if all the fields are present for each seller.

### 2.Buyer(s)/Seller Passport(s) \-

Following fields need to be extracted from the passport document for each buyer/seller (as recorded and stored on the completion of the processing the Share Transfer certificate document earlier):

**MRZ(Machine-Readable Zone)**

- This will be the data extracted from the MRZ text field of the passport. MRZ can be found at the bottom of the passport. It will have 2 lines. Both lines should be extracted. It will contains Passport number, Name, DOB, Gender, Expiry Date. These values are required to v erify with the values extracted from other places in the passport document.

**Document Info** \-

1. **Document Type**: This will be the type of the document uploaded which should be "passport". Check if the keywords passport is present in the document title or any other place to identify if it's a standard passport or not.  
     
2. **Issuing Country Code**: This will be the country code of the country where the passport was issued. It will be a 3 letter code. Later it should be matched with the country code present in MRZ extracted from the passport.  
     
3. **Passport Number**: This will be the passport number of the passport. It should match with the passport number extracted in the Share Transfer certificate document earlier.

**Personal Info** \-

4. **Full Name**: This will be full name of the passport holder. It can be found under "Name" like field in the passport. It should match with name present in MRZ extracted from the passport.  
     
5. **First Name**: This will be first name of the passport holder. It needs to be extracted from "Full Name" as the first word.  
     
6. **Last Name**: This will be last name of the passport holder. It needs to be extracted from "Full Name" as the last word. It should match with last name present in MRZ extracted from the passport.  
     
7. **Date of Birth**: This will be date of birth of the passport holder. It can be found under label "Date of Birth" like field in the passport. It should match with date of birth present in MRZ extracted from the passport.  
     
8. **Gender**: This will be gender of the passport holder. It can be found under label "Sex" like field in the passport. It should match with gender present in MRZ extracted from the passport.  
     
9. **Place of Birth**: This will be place of birth of the passport holder. It can be found under label "Place of Birth" like field in the passport.  
     
10. **Mother Name**: This will be mother name of the passport holder. It can be found under label "Mother's Name" like field in the passport. It should be extracted if its available in the passport.

**Passport Details** \-

11. **Date of Issue**: This will be date of issue of the passport. It can be found under label "Date of Issue" like field in the passport.  
      
12. **Date of Expiry**: This will be expiry date of the passport. It can be found under label "Date of Expiry" like field in the passport. It should be checked if its not expired. Expiry date should be \> current date.  
      
13. **Issuing Authority**: This will be issuing authority of the passport. It can be found under label "Authority" like field in the passport.  
      
14. **Address**: This will be the address of the passport holder. It can be found under label "Address" like field in the passport. It should be checked if its present in the passport and if its not empty.

**Biometrics** \- TBD

#### **Cross-checking Passport with Share Transfer certificate document \-**

1. The Passport holder name should exactly match with the one of name(s) of the either seller or buyer as extracted from Share Transfer certificate document earlier. If there is some mismatch, ask the user to re-upload the passport  
     
2. Next check if the passport number matches with the passport number for the buyer/seller as extracted from Share Transfer certificate document earlier. If there is some mismatch, ask the user to re-upload the passport  
     
3. Identify the role of the passport holder if its a buyer or seller. If it can't be found, then ask the user to provide the role name (seller or buyer)

### 3\. Emirate ID document \- 

**Following fields need to be extracted from the Emirate ID document for each buyer/seller (as stored from the Share Transfer certificate document processing earlier). The card can have both front and back side images (JPEG/PNG) or both in a single document (PDF). Following are the fields classified under different sections which need to be extracted and stored \-**

**(Back Side) MRZ data** \-

- This will be the data extracted from the MRZ text field of the Emirate ID card (in the backside). MRZ can be found at the bottom of the document. It can have 3 lines. All the lines should be extracted. It will contain Emirates ID number, Name, DOB, Gender, Expiry date. These field values need to be extracted and verified against corresponding values extracted for the same fields in the others places of the document.

**(Front Side) Document Info** \-

1. **Document Type**: This will be the type of the document uploaded which should be like "emirate ID/ resident ID". Check for keywords "emirate ID" or "resident ID" in the top section or any other place to ensure its the standard emirate ID document or not.  
     
2. **Emirates ID Number**: This will be the ID number of the Emirate ID card holder. It can be found under the label "ID Number" like field in the document. It should match the pattern "**784-XXXX-XXXXXXX-X**".  
     
3. **Full Name**: This will be the full name of the Emirate ID card holder. It can be found with the label "Name" like field in the document. It should match the name present in MRZ extracted from the Emirate ID document.

**(Front Side) Personal Info** \-

4. **Date of Birth**: This will be the date of birth of the Emirate ID card holder. It can be found with the label "Date of Birth" like field in the document. It should match the date of birth present in MRZ extracted from the Emirate ID document.  
     
5. **Nationality**: This will be the nationality of the Emirate ID card holder. It can be found with the label "Nationality" like field in the document. It should match the nationality value present in MRZ extracted from the Emirate ID document.  
     
6. **Gender**: This will be the gender of the Emirate ID card holder. It can be found with label "Sex" like field in the document. It should match the gender value present in MRZ extracted from the Emirate ID document.

**(Front Side / Back Side) Card Details** \-

7. **Issuing Date:** This will be the date of issue of the Emirate ID card (in the front side). It can be found with the label "Issuing Date" like field in the document. It should be a valid date with pattern DD/MM/YYYY. It should be checked if it's in the past date. .  
     
8. **Expiry Date**: This will be expiry date of the Emirate ID card (in the front side). It can be found with a label "Expiry Date" like field in the document.It should be a valid date. It should be checked if its not expired \- expiry date should be \> current date by at least 3 months.  
     
9. **Card Number:** This will be a unique card ID number for the Emirate ID document (in the backside). It can be found with the label "Card Number" like field.  
     
10. **Issuing place**: This will be the place of issue of the Emirate ID card (in the backside). It can be found with the label "Issuing Place" like field in the document.

**(Back Side) Employment Info** \-

10. **Occupation**: This will be the occupation of the Emirate ID card holder. It can be found with the label "Occupation" like field in the document.  
      
11. **Employer Name**: This will be name of the employer for the Emirate ID card holder. It can be found with the label "Employer" like field in the document.

**(Front Side) Signature** \- This will be the signature of the Emirate ID card holder. It can be found below a label "Signature" in the document. It should be checked if it can be detected as either ink or image presence.

#### **Cross-checking Emirate ID with Passport details extracted earlier \-**

1. The Full name extracted from the Emirate ID document should match with one of the full name(s) extracted from Passport(s) earlier. In case of mismatch, ask the user to re-upload the Emirate ID document. Same should be done for Date of birth, Gender and Nationality.  
     
2. Separate Emirate ID records should be maintained in case of multiple passport(s) holders so each EID record is linked with only one passport record created earlier.

### 4. Proof of Address document

Following are the fields that need to be extracted from the document for each buyer/seller (as recorded and stored after processing the Share Transfer certificate document earlier):

**Document Info \-**

1. **Document Type** \- This will be the type of the document uploaded which should be "Proof of Address". The same should be identified by checking for keywords like "Invoice", "Bill", "Statement" in the document header section or any other place. It could be a utility bill, internet bill, tenant agreement, etc. (List of documents is yet to be provided). If valid document is not detected, ask the user to re-upload the document.  
     
2. Document Issuer Name \- This will be name of company or entity that issued the document. It can be found in the document header or title section usually. This is required to ensure the document source.

**Personal Info:**

3. **Full Name** \- This will be the full name of the document holder. It can be found somewhere in the top section of the document. It can be some user name label to whom the document is addressed to.

4. **Full Address** \- This will be the address of the document holder currently residing. It can be somewhere near the full name field. Complete addresses like blocks should be identified and extracted. If a partial address is extracted, ask the user to re-upload the document.

5. **City** \- This will be the city where the document holder is currently residing. It should be identified and extracted from the "Full address" value. It is an optional field if so it can be skipped for missing fields check.

6. **Country** \- Same should be repeated here as done for "City" field.

**Document details:**

7. Date \- This should be the date of issue of the document like invoice date, or statement date. It should be extracted from the issue date like section in the document. It should be checked if it's missing or not a valid date. It should not be more than 3 months old.

#### **Cross-checking Proof of Address with Share Transfer certificate document and Passport(s) details extracted earlier \-**

 

1. The Full name extracted from the Proof of Address document should be checked against full names(s) which were extracted from Passport(s). It should match with one of the full names. This check can ignore case, spaces etc. in the names. In case of complete mismatch, ask the user to re-upload the address document.

2. Full name should be checked against buyer(s)/ seller(s) name(s) extracted from Share Transfer certificate document. It should match with one of the names. In case of complete mismatch, ask the user to re-upload the address document.