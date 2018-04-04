## Performance

**Intended Audience**: this document provides guidance for diagnosing and improving performance of the [Reporting Data Warehouse](../README.md) (RDW). Operations, system administration, and developers may find it useful.

Currently this document is a placeholder for performance-specific advice. Related documents:
* [Tuning Aurora](PerformanceTuning.Aurora.md)
* [Tuning Redshift](PerformanceTuning.Redshift.md)

### Redshift Analyze and Vacuum
In general to get the best performance from your Redshift Database, you must ensure that database tables are regularly analyzed and vacuumed. For more information please refer to the Redshift documentation:
* [Vacuuming Tables](http://docs.aws.amazon.com/redshift/latest/dg/t_Reclaiming_storage_space202.html)
* [Analyzing Tables](http://docs.aws.amazon.com/redshift/latest/dg/t_Analyzing_tables.html)

Because data is regularly ingested and rarely deleted from RDW, it is important to analyze regularly but vacuuming should seldom be necessary. Based on empirical observations, analyzing every week or two is sufficient.
Operations should set up a scheduled task to analyze the reporting schema. There is a utility that may be used: https://github.com/awslabs/amazon-redshift-utils/tree/master/src/AnalyzeVacuumUtility.

TODO - test and otherwise vet this:
Using examples from [deployment](../deploy/Deployment.AWS.md] an invocation to analyze may look like this:
```
python analyze-vacuum-schema.py --db opus --db-user root --db-pwd password --db-host rdw-opus.[aws-randomization] --schema-name reporting  --output-file ./test.log --vacuum-flag false --analyze-flag true --stats-off-pct 10
```

TODO - create a deployment package for AWS Lambda and instructions for setting up the lambda function
TODO   https://docs.aws.amazon.com/lambda/latest/dg/python-programming-model-handler-types.html
