# InferringMissingRecords
Inferring Missing Records for Historical Dimensions in ETL
When working with different partner’s databases with different admins, database tables can get very messy. It is no surprise to come up with a record that looks in contrast to all other records—unwritten protocols and lack of documentation and test plans can cause such errors. Take the following example,

·        While transferring data from OLTP database to OLAP database, an odd record rises: a customer seems to have made a transaction on a date when no active account of that customer could be associated to. Such inconsistency means that there is a missing account that has not been declared in the database.

Historical dimension tables holding customers’ accounts usually change through any of three types of policies for Slowly Changing Dimensions:

        i.           Type one: Update Changes

      ii.           Type two: Keep Historical

    iii.           Type three: Preserve Limited History

The most common type among the above is type two to store historical data in different rows with FromDate and ToDate columns indicating the most recent one with a true RecentFlag. Each record is assigned a surrogate key to each keep track of changes of a dimension table attribute. In our example, this could mean a simple change of branch or termination of the account: in the former case, after detecting the change, one needs to update the existing ToDate, and insert a new record with new FromDate; in the latter case, after detecting the change, one only needs to update the existing ToDate.

With this background in mind, assume that you receive a transaction whose date lies outside of your reported intervals. The only logical approach is to match the customer’s information through the intervals at hand. We then mark the data as inferred type two in a different table and assign an recognizable RecentFlag, so we can precisely update it later when we have full information about the account.

Let me elaborate through our example. Identification of the account associated with the transaction whose date is not during an active account, consists of:

1.      If the date of the transaction falls between two consecutive intervals, we can infer that the transaction belongs to an account whose activity spans from the ToDate of the first interval to the FromDate of the second one.

2.      If the date of the transaction is later than any activity interval of the customer’s accounts, we may infer that the account associated with the transaction starts from the latest ToDate of existing accounts of the customer and ends at a maximum date.

3.      By the same token, if the date of the transaction is sooner than any activity interval of the customer’s accounts, the transaction is assigned with an inferred account starting from the earliest FromDate of the actual customer’s accounts.

The inferred accounts above should then be reported as an inferred one in a separate table.

This method could be implemented by different ETL tools. I chose to implement the above scenario described in the example as a container in Datastage, IBM environment. However, due to historical nature of our accounts, we do not handle gaps between account activities and consider every new record within a series of interval or outside of the whole activity. In other words, we only deal with second and third rule of identification above, and we keep the changes in accounts of our customers through unique surrogate keys:




1.      At first, the input transactions get to join with the dimension table of users in data-warehouse (including those deemed as inferred ones) to lookup their historical attributes. We want all transactions conform to surrogate key of accounts we have already collected.

2.      As for null IDs, we resign to creating a new (fake) account, starting from minimum StartDate (FromDate) to maximum EndDate (ToDate). Any other IDs are separated to be checked on the transaction date.

3.      Then, the transaction’s date is determined whether it is sooner than StartDate or later than maximum EndDate. In respective cases, a minimum StartDate or a maximum EndDate and a distinguishable RecentFlag are all considered for such inferred records, which will take place in the proceeding transformers.

4.      Otherwise when the date is valid, there won’t be any inference involved and one can firmly decide which account ID corresponds to the record (Join_54 and RejectNullFieldsInUserID_ID) which we label as CleanLink.

5.      Transactions with null IDs or an inferred interval will receive distinct surrogate keys before blending with other data.

6.      A copy of above cases is inserted into our inferred database. Next, they are sent with the rest of the records to the output stage of the container.

I guess this is a pretty simple method that could be utterly useful in similar situations. Feel free to leave comments to let me know how you think of this method.
