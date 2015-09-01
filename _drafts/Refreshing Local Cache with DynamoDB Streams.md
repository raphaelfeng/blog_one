# Verifying the Ideas

1. iswrite a POC of replicate the one table changes to another
2. write a test app to verify it will not run out of sync

POC:
One Producer:
Keep inserting, changing and deleting items in one DynamoDB table

Multiple Consumers
scan the table to init local cache
read the the dynamodb streams to update their local cache

Test Case:
Basic Test
Start the Producer,
Start three consumers
after 10 min, stop producer
after 11 min, stop consumers and compare their local caches
save back to dynamodb and compare with original dynamodb table

B. Failure recovery
restart some consumer threads
C. Duplicate Stream Record
set the checkpoint to an earlier point for some of the consumers

DynamoDB Table
id: (String)
timestamp:(String)
value: (Number)

