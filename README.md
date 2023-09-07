# CV_expansion

## Collections:
### About the project
The call+ platform enables the collection of loans through digital means. Herein we have the functionality of calling & texting the Loan Account Holders.
The calls made via this can be of two types: Either Inbound or Outbound.

Inbound calls: A list of allocations is uploaded to the dialer service and it starts calling them. The number of threads for calling is determined by the number of agents tagged for Inbound Calling. 
For eg. If only 10 agents are tagged for inbound calling, we can have 15 - 20 active threads for calling.

Outbound calls: Each agent tagged for Outbound is explicitly allocated with loan account holders and a deadline to reach out to all the allocations adhering to the deadlines.

The Outbound calls are made using the same dialer that is used in Inbound calling, just these calls go through a layer. This layer is the Call+ platform.

Call+:
- Enables the management and tagging of agents to the Banks, cycle of allocations, and their hierarchy.
- Defining daily targets of agents by their Team Leaders.
- Performance analysis of the agents and Team Leaders by their reporting supervisors and department heads.
- Bifurcating the allocations, basis their dispositions.
### My contributions to Collections:
- Designed & created a standalone service from scratch for generating Call log reports, Agent performance reports, and contact list reports.

  Why?
  - These report generation requests were served by the same server hosting the whole of the application.
  - At times report generation slowed the process of calling which ultimately resulted in business loss.
  How?
  Decoupled the tightly coupled code.
  Created a service with APIs exposed for these report generation.
  The final report is pushed to the S3 bucket.
  A pre-signed URL is made available for the users to download it.
  Result:
  Report Generation is not hampering the actual business of calling.
  Reports no longer reside on servers instead, they are on S3 which is comparatively low-cost storage.
  These reports are comparatively less vulnerable to be exploited as they can be accessed only for a limited period of time.

- Standardising the dispositions across all platforms enabling it to be tracked universally.

  Why?
  There are multiple channels through which a loan account holder is approached eg. Calling, SMS, WhatsApp, Field, etc.
  Most of these channels are built at different timelines and under different client requirements.
  Very often there used to be the case when different channels referred to the same disposition under different names because of the differences in the channels, resulting in confusion & miscommunication.
  How?
  All the existing channels were brought together. A list of common standard dispositions was created along with their ranks.
  A new service Standardised dispositions is created which will cater to all disposition requirements of all channels.
  Updation of Existing dispositions in line with the standardized dispositions.
  Integration of Standardised Dispositions Service with Call+.
  Result:
  Clear communication of the status of a particular loan across all channels.
  This enabled the Data Science team to have cleaner data and efficient processing of the same.

- Designed and implemented the feature of "unsubscribing" a phone no. or loan account no.

  Why?
  A person who has already paid the loan should not be approached again through any of the channels.
  How?
  Provided Call+ agent with the option of marking a phone no. or loan account no. as unsubscribed.
  This API call will remove the details of that particular loan account from call+ DB and insert them into unsubscribe logs which is used as a check when taking in new allocations from a bank.
  Result:
  We stopped facing escalations from banks for approaching a closed loan account.

- Designed and Implemented a generic service for archiving the data from multiple data sources.

  Why?
  MySql storage is expensive, hence keeping the data that is not accessed frequently results in increased billing.
  How?
  This generic service takes inputs as table_name & query.
  The input query will be executed on the given table in chunks(in order to limit the mem & cpu usage), these chunks are dumped to a file having the name of the table and the date ranges of the records along with a current timestamp.
  Once we have the total output of the query in a file, that file is then pushed to a designated bucket.
  A delete query is run on the table in order to purge the records.
  Result:
  Saving the cost by only utilizing the expensive databases for the data which is used frequently.
  Faster and smoother execution of any or all types of query execution.
  
## Dhani:
### About the project
It is a subsidiary of Indiabulls Group. It is an e-commerce platform that helps small shop owners and vendors sell their goods all across the country.
Dhani has 6 warehouses as of Dec 2022 across different locations in the country which enabled the quick and cost-effective delivery of goods at the buyers' doorstep.
### My contributions to Dhani Store:
- Implemented "product pricing" product-wise promotional feature.

  Why?
  At times Dhani wanted to sell its products at different prices.
  How?
  Created a table "product_pricing" in which only those products will reside on which the promotion is going to be active.
  The quantity of the product and the updated price of the product will reside in this table and this data will overwrite any associated data of this product for the time it is in promotion.
  Result:
  Dhani no longer had to go and change the prices of a number of products manually for any timeline.
  As a result, Dhani could organize sales very quickly.
  
