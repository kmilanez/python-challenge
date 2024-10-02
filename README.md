# In a nutshell...
We need to process a file from another application that contains batch job information in JSON format.

The files are of two types:

* Full listings of jobs - for initial backfill and weekly reconciliation.
* Incremental - for daily changes (e.g. today's delta file will contain changes since yesterday's file).
JSON attributes:

* Extract information
  * extract_from_dt_utc
  * extract_type
* Job information
    * source
  * job_id
  * action - add or deleted
  * hostnames
  * days_of_week
  * timezone
  * depends_on - see L19, the job has a dependency on job at L60

# Requirements
1. Implement a parser to convert the following JSON objects.
2. Implement logic in the parser that can look up job dependencies.
3. Implement a main process to handle the two types of extract.

### Example content

```json

{
    "extract_from_dt_utc": "2024-08-27T13:43:11Z",
    "extract_type": "delta",
    "jobs": [
 
        {
            "source": "Batch Scheduler X",
            "job_id": "prd-1.ct.weekly.eow.job1",
            "action": "DEL"
        },
 
        {
            "source": "Batch Scheduler X",
            "job_id": "prd-1.ct.weekly.eow.job2",
            "action": "ADD",
            "days_of_week": [ "sun" ],
            "times_of_day": [ "04:00" ],
            "timezone": "US/Eastern",
            "depends_on": [ "prd-2.ct.daily.eod.group1.job2.subjob3" ]
        },
 
        {
            "source": "Batch Scheduler Y",
            "job_id": "prd-2.ct.daily.eod.group1",
            "action": "ADD",
            "days_of_week": [ "mon", "tue", "wed", "thu", "fri", "sat", "sun" ],
            "times_of_day": [ "03:00" ],
            "timezone": "US/Eastern",
            "child_jobs": [
                {
                    "job_id": "prd-2.ct.daily.eod.group1.job1",
                    "action": "ADD",
                    "hostnames": [ "dc1.machine.a-domain.net" ],
                    "days_of_week":[ "mon", "wed", "fri", "sun" ],
                    "times_of_day": [ "03:00" ],
                    "timezone": "US/Eastern"
                },
                {
                    "job_id": "prd-2.ct.daily.eod.group1.job2",
                    "action": "ADD",
                    "days_of_week": ["tue", "thu", "sat", "sun"],
                    "times_of_day": [ "03:00" ],
                    "timezone": "US/Eastern",
                    "child_jobs": [
                        {
                            "job_id": "prd-2.ct.daily.eod.group1.job2.subjob1",
                            "action": "ADD",
                            "hostnames": [ "dc2-1.machine.a-domain.net" ],
                            "days_of_week": [ "tue", "thu" ],
                            "times_of_day": [ "03:00" ]
                        },
                        {
                            "job_id": "prd-2.ct.daily.eod.group1.job2.subjob2",
                            "action": "ADD",
                            "hostnames": [ "dc2-2.machine.a-domain.net" ],
                            "days_of_week": [ "sat" ],
                            "times_of_day": [ "03:05" ]
                        },
                        {
                            "job_id": "prd-2.ct.daily.eod.group1.job2.subjob3",
                            "action": "ADD",
                            "hostnames": [ "dc2-2.machine.a-domain.net" ],
                            "days_of_week": [ "sun" ],
                            "times_of_day": [ "03:00" ]
                        }
                    ]
                }
            ]
        }
    ]
}
```

# Architecture design
Design a data pipeline to process the data with the following requirements.

* File lands at our S3 bucket.
* Process the file and persist into a database.
* Send notification for processing failure.

