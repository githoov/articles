``` sql
/* source json @ s3://your-bucket/events/source.json */

{
	"agent": "foo", 
	"user_id": 1, 
	"event_attributes" : {
			"recipient" : "bar"
			}
}


/* create redshift table */

create table public.events (
	agent varhcar(256)
	, user_id integer
	, recipient varhcar(256))
distkey(user_id)
sortkeys(user_id);

/* store json schema in S3 bucket @ s3://your-bucket/jsonpaths/events.json */

{
	"jsonpaths" : 
	[
    "$.agent",
    "$.user_id",
    "$.event_attributes.recipient"
	]
}


/* issue copy to redshift, with destination table and path to json schema */

copy public.events 	 -- copy to above-created table
from 's3://your-bucket/events/source.json'	-- copy source json file
with credentials 'aws_access_key_id=$AWS_KEY;aws_secret_access_key=$AWS_SECRET'
json 's3://looker-db-loader/scott/jsonpaths/events.json';	-- references json schema above


/* all of this yields */

agent | user_id | recipient
---------------------------
"foo"    1         "bar"

-- if one runs

select *
from public.events
```
